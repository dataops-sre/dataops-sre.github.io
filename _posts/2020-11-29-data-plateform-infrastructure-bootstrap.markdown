---
title: "Bootstrapping a data platform infrastructure on Kubernetes"
date: 2020-11-29 16:36:37 +0100
categories: DevOps Cloud-Infrastructure AWS Kubernetes
tags:
  - DevOps
  - SRE
  - Data Engineering
  - Data Processing
  - Cloud Infrastructure
  - AWS
  - Terraform
  - Terragrunt
  - Kubernetes
---

The goal of this article is to show how easily you can bootstrap a resilient, scalable, and budget-friendly data processing platform in the cloud with Terraform and Terragrunt. In this article, I focus only on AWS. I may cover the same setup on GCP in another article.

In my previous articles about Spark on Kubernetes, I took a resilient and scalable infrastructure for granted. In many companies, that part is usually managed by a platform team. But if your organization does not use Kubernetes company-wide, getting such an environment provisioned for your team can easily take months.

The good news is that a data processing platform is actually much simpler to set up than a general-purpose application platform. It usually does not need to handle incoming public traffic, service mesh concerns, or DNS/SSL certificate management. That makes it a very good candidate for a self-service infrastructure setup.

## Setting up a Kubernetes cluster on AWS

If you are not familiar with [Terraform](https://blog.gruntwork.io/an-introduction-to-terraform-f17df9c6d180) or [Terragrunt](https://blog.gruntwork.io/terragrunt-how-to-keep-your-terraform-code-dry-and-maintainable-f61ae06959d8), have a quick look at their articles first. They are very powerful infrastructure-as-code tools.

TL;DR: if you just want the code, check out the full implementation in my [Git repository](https://github.com/dataops-sre/terragrunt-dataplaform-bootstrap). It also contains a deployment guide.

```hcl
{% raw  %}
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  version         = "13.2.1"
  cluster_name    = var.cluster_name
  cluster_version = "1.17"
  subnets         = module.vpc.private_subnets
  vpc_id          = module.vpc.vpc_id
  enable_irsa     = true

  worker_groups_launch_template = [
    {
      name                    = "on-demand-1"
      override_instance_types = var.ondemand_instance_types
      asg_max_size            = var.ondemand_asg_max_size
      kubelet_extra_args      = "--node-labels=node.kubernetes.io/lifecycle=ondemand"
      suspended_processes     = ["AZRebalance"]
      tags = [
        {
          "key"                 = "k8s.io/cluster-autoscaler/enabled"
          "propagate_at_launch" = "false"
          "value"               = "true"
        },
        {
          "key"                 = "k8s.io/cluster-autoscaler/${var.cluster_name}"
          "propagate_at_launch" = "false"
          "value"               = "true"
        }
      ]
    },
    {
      name                    = "spot-1"
      override_instance_types = var.spot_instance_types
      spot_instance_pools     = 4
      asg_max_size            = var.spot_asg_max_size
      asg_desired_capacity    = 1
      kubelet_extra_args      = "--node-labels=node.kubernetes.io/lifecycle=spot --register-with-taints=node-role.kubernetes.io/spot=true:PreferNoSchedule"
      tags = [
        {
          "key"                 = "k8s.io/cluster-autoscaler/enabled"
          "propagate_at_launch" = "false"
          "value"               = "true"
        },
        {
          "key"                 = "k8s.io/cluster-autoscaler/${var.cluster_name}"
          "propagate_at_launch" = "false"
          "value"               = "true"
        }
      ]
    },
  ]

  worker_additional_security_group_ids = [aws_security_group.all_worker_mgmt.id]
}
{% endraw  %}
```

This setup uses the existing [Terraform EKS module](https://github.com/terraform-aws-modules/terraform-aws-eks/). It defines two worker node pools:

- one pool with on-demand instances to host critical components
- one pool with spot instances to run compute workloads more cheaply

Later, in your Kubernetes deployments, you can decide which node pool to target by using Kubernetes [taints and tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/).

This split works very well in practice: keep the essential services on stable nodes, and push bursty or fault-tolerant compute workloads onto spot capacity.

## Deploying the cluster autoscaler and the spot instance handler

Once the cluster is in place, the next step is to deploy the cluster autoscaler and the spot instance termination handler. I use the Terraform Helm provider for that.

```hcl
{% raw  %}
resource "helm_release" "cluster-autoscaler" {
  name       = "cluster-autoscaler"
  version    = "8.0.0"
  repository = "https://charts.helm.sh/stable"
  chart      = "cluster-autoscaler"
  namespace  = "kube-system"
}

resource "helm_release" "spot-handler" {
  name       = "spot-handler"
  version    = "1.4.9"
  repository = "https://charts.helm.sh/stable"
  chart      = "k8s-spot-termination-handler"
  namespace  = "kube-system"
}
{% endraw  %}
```

At this point, you already have a managed, scalable, and resilient Kubernetes cluster on AWS.

The cluster autoscaler adjusts node capacity automatically based on workload demand, while the spot termination handler helps the cluster react more gracefully to spot interruption events. Together, they make spot-based compute much more practical for data workloads.

## Deploying Airflow

The next step is to deploy Airflow into the cluster so it can orchestrate Spark streaming jobs and Spark batch ETL pipelines, as discussed in my previous articles.

```hcl
{% raw  %}
resource "helm_release" "aiflow" {
  name       = "airflow"
  repository = "https://dataops-sre.github.io/helm-charts/"
  chart      = "airflow"
  namespace  = "default"
}
{% endraw  %}
```

I packaged my Airflow Helm chart in my own [Helm repository](https://dataops-sre.github.io/helm-charts/).

## Conclusion

And that is basically it: a resilient, scalable, and budget-friendly data platform infrastructure running on AWS.

What I like about this setup is that it gives a small data team a lot of autonomy. You do not need a huge platform organization to get started. With a relatively small amount of Terraform and Helm configuration, you can build a solid Kubernetes foundation for Spark, Airflow, and other data workloads.

Of course, this is only the bootstrap phase. In a real platform, you will probably also want to add observability, IAM fine-tuning, backups, cost monitoring, and security hardening. But as a starting point, this setup already gives you a strong and very usable base.
