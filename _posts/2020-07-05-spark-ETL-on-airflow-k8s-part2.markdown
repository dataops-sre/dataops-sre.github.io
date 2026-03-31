---
title: "Spark ETL with Airflow on Kubernetes — Part 2"
date: 2020-07-05 22:36:37 +0100
categories: Data-Engineering Kubernetes Airflow
tags:
  - SRE
  - Data Engineering
  - Data Processing
  - Airflow
  - Apache Spark
  - Kubernetes
  - AWS
  - ETL
---

Let’s talk about the pitfalls and inconveniences of Airflow.

This article is the follow-up to [Part 1]({{ site.baseurl }}/spark/kubernetes/airflow/spark-ETL-on-airflow-k8s-part1/), where I explained how we run Spark ETL workloads with Airflow on Kubernetes.

## Pitfalls and inconveniences

Keep your Airflow DAGs simple.

The points below are based on my own experience and apply to our setup:

* Airflow on Kubernetes with the Local executor
* Tasks executed through `KubernetesPodOperator`

These are the problems that stood out the most for me:

1. A DAG’s [`dagrun_timeout`](https://airflow.apache.org/docs/stable/_modules/airflow/models/dag.html#DAG) and an operator’s `execution_timeout` parameter do not apply properly, because `KubernetesPodOperator` does not implement [`on_kill`](https://stackoverflow.com/questions/50054777/airflow-task-is-not-stopped-when-execution-timeout-gets-triggered).
2. Backfill ignores the DAG’s `max_active_runs` parameter when retriggered. At the time this article was written, this was still not fixed. See [here](https://issues.apache.org/jira/browse/AIRFLOW-137).
3. A DAG can be triggered with parameters like this: `airflow trigger_dag 'example_dag' -r 'run_id' --conf '{"key":"value"}'`. However, this does not work correctly when you retrigger a backfill. For example, if you backfill day `x` with `--conf '{"key":"value1"}'` and then rerun the backfill for the same day with `--conf '{"key":"value2"}'`, the `key` still keeps `value1`.
4. You cannot configure Kubernetes ephemeral storage resources with `KubernetesPodOperator`.
5. DAG run history is not always cleaned up properly. During DAG development, I often adjust DAG or task parameters, and this can mess up the scheduler database. Some DAG runs cannot be marked as failed or cleared. Even after deleting and redeploying the DAG, problems can persist. In those cases, I had to run SQL cleanup queries directly against the Airflow metadata database.
6. Kubernetes attached volume names are not templated fields. Spark jobs can use a large amount of disk space when memory is not enough, so attaching volumes to Kubernetes pods becomes important. This becomes problematic for parallel backfills. For example, if we want to reprocess data in parallel from day `01-01-2020` to `04-01-2020`, I would like to create volume names like `{% raw  %}"spark-pv-claim-{{ ds_nodash }}"{% endraw  %}`. In practice, since `persistentVolumeClaim` names are not templated fields in `KubernetesPodOperator`, volumes cannot be attached dynamically the way we need.

## Solutions and workarounds

Here are the solutions and workarounds we used for the problems above:

1. Use the shell `timeout` command. For example, configure `KubernetesPodOperator` with `"cmds": ["timeout", "45m", "/bin/sh", "-c"]` to enforce a 45-minute timeout for the Spark task.
2. Make DAG executions independent and avoid dependencies on past runs. Since the `max_active_runs` problem mainly happens during backfills, clear the DAG history before retriggering.
3. Use Airflow variables instead. For example, we have a `force_reprocess` variable in the DAG. Before a backfill, we set it with `airflow variables --set force_reprocess false;`, then run `airflow backfill "historical_reprocess" --start_date "y" --end_date "y" --reset_dagruns -y;`. Inside the DAG, we read it with `force_reprocess = Variable.get("force_reprocess", default_var=True)`. You just need to remember to delete the variable after the backfill is done.
4. Airflow version [1.10.11](https://github.com/apache/airflow/blob/1.10.11rc1/airflow/kubernetes/pod.py) adds ephemeral storage support.
5. Run SQL cleanup queries against the Airflow database. You can find useful examples in this [post](https://www.astronomer.io/guides/airflow-queries/).
6. This feature was crucial for us, so I extended `KubernetesPodOperator` to manage volumes dynamically by adding a new templated field called `pvc_name`.

```python
from airflow.contrib.operators.kubernetes_pod_operator import KubernetesPodOperator
from airflow.contrib.kubernetes.volume import Volume
from airflow.contrib.kubernetes.volume_mount import VolumeMount
from airflow.utils.decorators import apply_defaults


class KubernetesPodWithVolumeOperator(KubernetesPodOperator):
    template_fields = (*KubernetesPodOperator.template_fields, "pvc_name")

    @apply_defaults
    def __init__(self, pvc_name=None, *args, **kwargs):
        super(KubernetesPodWithVolumeOperator, self).__init__(*args, **kwargs)
        self.pvc_name = pvc_name

    def execute(self, context):
        if self.pvc_name:
            volume_mount = VolumeMount(
                "spark-volume", mount_path="/tmp", sub_path=None, read_only=False
            )

            volume_config = {"persistentVolumeClaim": {"claimName": f"{self.pvc_name}"}}
            volume = Volume(name="spark-volume", configs=volume_config)

            self.volumes.append(volume)
            self.volume_mounts.append(volume_mount)
        super(KubernetesPodWithVolumeOperator, self).execute(context)
```

Usage of the custom operator:

```python
{% raw  %}
historical_process = KubernetesPodWithVolumeOperator(
    namespace=os.environ['AIRFLOW__KUBERNETES__NAMESPACE'],
    name="historical-process",
    image=historical_process_image,
    image_pull_policy="IfNotPresent",
    cmds=["/bin/sh", "-c"],
    arguments=[spark_submit_sh],
    env_vars=envs,
    task_id="historical-process-1",
    is_delete_operator_pod=True,
    in_cluster=True,
    hostnetwork=False,
    # important env vars to run spark submit
    pod_runtime_info_envs=pod_runtime_info_envs,
    pvc_name="spark-pv-claim-{{ ds_nodash }}",
)
{% endraw  %}
```

## Conclusion

Despite the many inconveniences and pitfalls, our pipelines have been running fine so far with the Airflow scheduler. We achieved a significant cost reduction by moving away from AWS Glue. Historical data reprocessing can now be done with a single backfill command. Real-time and historical data processing now live in the same code base, and the pipeline steps are transparent thanks to DAG definitions in Python.

In the future, I would also like to study alternatives such as [Prefect Core](https://docs.prefect.io/), [Kedro](https://kedro.readthedocs.io/en/stable/index.html), or [Dagster](https://github.com/dagster-io/dagster/). There may be better tools out there.
