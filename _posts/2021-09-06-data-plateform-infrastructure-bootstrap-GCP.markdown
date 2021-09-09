---
title:  "Data plateform infrastructure bootstrap in GCP"
date:   2021-09-05 16:36:37 +0100
categories: Kubernetes GCP Terraform Devops
tags:
  - Devops
  - GCP
  - Terraform
  - Kubernetes
---

## Introduction
I have promised to write an article about spinning up Kubernetes cluster on GCP in my [last article](https://dataops-sre.github.io/data-plateform-infrastructure-bootstrap/) a year ago, I had time to test with an free GCP account, it is a quite interesting learning.

## Setup Kubernetes cluster on GCP

Main process is very similar as on AWS, In GCP you need to first create a project and configure your service account under the project, then you will have access to gcloud service with gloud cli tool.
```sh
#create project
gcloud projects create gke-terragrunt-demo
gcloud config set project gke-terragrunt-demo
#configure service account
gcloud iam service-accounts create terragrunt-sa --display-name="Terragrunt service account"
gcloud projects add-iam-policy-binding gke-terragrunt-demo --member='serviceAccount:terragrunt-sa@gke-terragrunt-demo.iam.gserviceaccount.com' --role='roles/admin'
gcloud iam service-accounts keys create ~/.gcloud/key.json --iam-account=terragrunt-sa@gke-terragrunt-demo.iam.gserviceaccount.com
gcloud auth activate-service-account --key-file ~/.gcloud/key.json
```

For the terraform part, we can just use existing GKE terraform module on github, they also offer various usage examples, you can checkout [here](https://github.com/gruntwork-io/terraform-google-gke). The rest of setups are documented in my [git repository](https://github.com/dataops-sre/terragrunt-dataplaform-bootstrap-gcp)