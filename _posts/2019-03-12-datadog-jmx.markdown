---
title:  "Collect JMX metrics from a Kubernetes deployment with Datadog"
date:   2019-03-12 16:36:37 +0100
categories: Datadog Monitoring Jmx Kubernetes
tags:
  - Devops
  - Datadog
---
Since 2018, my team has been using Datadog to collect application-related metrics.

We use Kafka as our message queuing service, and we have Kafka Connect/Kafka Streams applications (JVM-based) to perform real-time data enrichment, transformation, and unload tasks. Our applications are deployed on AWS-managed Kubernetes clusters. In Kafka Connect/Streams frameworks, JMX metrics are available by default. So we need to collect those metrics consistently, even when Kubernetes pods restart or applications are redeployed.

Datadog provides Kubernetes integration that allows you to collect application JMX metrics through service autodiscovery. You need to run a [Kubernetes daemonset in your cluster](https://docs.datadoghq.com/agent/kubernetes) to act as a metric collection agent for pods. You also need to follow their [JMX guide](https://docs.datadoghq.com/integrations/java) to configure your pod annotations. Of course, it was not as easy as advertised. There were many bugs in their `jmxfetch` back then. After quite a bit of back-and-forth with Datadog engineers, we managed to fix many of those bugs and eventually got a relatively satisfactory integration.

## Java app-side configuration
For a Kafka Connector application, we use the following Datadog JMX integration configuration:
```yaml
{% raw  %}
instances:
  - jmx_url: service:jmx:rmi:///jndi/rmi://%%host%%:9999/jmxrmi
    name: {{ include "kafka-connect-helm.fullname" . }}
    tags:
    - kafka:connect
    - env:{{ .Values.ENV }}
    - app:{{ include "kafka-connect-helm.fullname" . }}
logs:
  - type: file
    path: /var/log/kafka/server.log
...
{% endraw %}
```

The whole config file is [here](https://github.com/mrmuggymuggy/kafka-connect-s3offload/blob/master/kafka-connect-helm/monitoring/custom_metrics.yaml).

Datadog autodiscovery relies on the Datadog daemonset to read the JMX configuration from Kubernetes deployment annotations, so you need to transform the YAML file into JSON format inside the annotations.

```yaml
{% raw  %}
   {{- if and (.Values.datadog.enabled) }}
      annotations:
        iam.amazonaws.com/role: '{{ .Values.kafka_connect.CONNECT_IAM_ROLE }}'
        #Logs
        ad.datadoghq.com/{{ include "kafka-connect-helm.fullname" . }}.logs: '{{ toJson (fromYaml (tpl (.Files.Get "monitoring/custom_metrics.yaml") .)).logs }}'
        # label for existing template on file
        ad.datadoghq.com/{{ include "kafka-connect-helm.fullname" . }}.check_names: '["{{ include "kafka-connect-helm.fullname" . }}-{{- uuidv4 | trunc 5 -}}"]'  # becomes instance tag in Datadog
        ad.datadoghq.com/{{ include "kafka-connect-helm.fullname" . }}.init_configs: '[{{ toJson (fromYaml (tpl (.Files.Get "monitoring/custom_metrics.yaml") .)).init_config }}]'
        ad.datadoghq.com/{{ include "kafka-connect-helm.fullname" . }}.instances: '{{ toJson (fromYaml (tpl (.Files.Get "monitoring/custom_metrics.yaml") .)).instances }}'
    {{- end }}
    spec:
      volumes:
{% endraw %}
```

The whole config file is [here](https://github.com//mrmuggymuggy/kafka-connect-s3offload/blob/master/kafka-connect-helm/templates/deployment.yaml).

The code snippet uses Helm (Go template) syntax:
```jinja
{% raw  %}
'[{{ toJson (fromYaml (tpl (.Files.Get "monitoring/custom_metrics.yaml") .)).init_config }}]'
{% endraw %}
```
to transform the YAML configuration into JSON and inject it as part of the Kubernetes deployment annotations at runtime.

{% capture notice-2 %}
Important points:
1. The `name` in `ad.datadoghq.com/$name` in the annotation must be the same as your container name.
2. `ad.datadoghq.com/$name.check_names` should ideally contain a UUID. The Datadog daemonset stores check name and host IP pairs. During a service redeployment, the host IP changes. If the check name stays the same, the Datadog daemonset may continue checking an outdated host IP, and you will lose metrics.
{% endcapture %}
<div class="notice">{{ notice-2 | markdownify }}</div>

## Datadog daemonset configuration

Everything worked fine with the initial setup, but after a few weeks we suddenly started seeing missing application metrics. After investigating, we found that the `jmxfetch` process inside the Datadog daemonset was being constantly killed by the host OOM killer.

The Datadog daemonset spawns the `jmxfetch` (Java) process as a subprocess with the JVM options `-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap`. As the main Datadog agent process consumed more memory over time, eventually the Java process was no longer able to allocate memory and got killed by the host OOM killer.

Datadog had implemented auto-restart for `jmxfetch`, so once the other processes consumed too much memory, it kept trying to restart `jmxfetch` again and again. At that point, we started losing JMX metrics.

Based on this finding, I implemented a solution that forces the Datadog daemonset liveness probe to fail by patching the existing pod liveness probe checks:

Memory check based on the container cgroup:
[container_memory_check.py](https://github.com/mrmuggymuggy/data-team-bootstrap/blob/master/data-team-bootstrap-helm/monitoring/container_memory_check.py)
```python
#!/usr/bin/env python
import psutil
import sys

CGROUP_MEM_LIMIT_FILE = '/sys/fs/cgroup/memory/memory.limit_in_bytes'

def main():
  mem_total = 0
  f = open(CGROUP_MEM_LIMIT_FILE, "r")
  container_cgroup_limit = float(f.read())
  f.close()

  for p in psutil.process_iter(): 
    p_mem = p.memory_full_info()
    mem_total = mem_total + (p_mem.rss - p_mem.shared)

  mem_used_percent = (mem_total/container_cgroup_limit) * 100
  print("container cgroup limit : {0}, total container process memory : {1}".format(container_cgroup_limit, mem_total))
  print("total memory use percentage {0}".format(mem_used_percent))
  if mem_used_percent > 90:
    sys.exit(1)
  else:
    sys.exit(0)

if __name__== "__main__":
  main()
```
When the total memory used by all processes exceeds 90% of the allocated resource, the memory check fails.

Add these checks to the end of the Datadog liveness probe:
[probe.sh](https://github.com/mrmuggymuggy/data-team-bootstrap/blob/master/data-team-bootstrap-helm/monitoring/probe.sh)
```bash
#!/bin/sh
set -e

/opt/datadog-agent/bin/agent/agent health
python container_memory_check.py
```

You can also take a look at my [data-team-bootstrap](https://github.com/mrmuggymuggy/data-team-bootstrap) project, which contains the Datadog daemonset deployment setup.
