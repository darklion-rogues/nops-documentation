# Multi-Cluster nOps with AWS Distro for OpenTelemetry (ADOT)

## See also

* [Amazon Managed Service for Prometheus (AMP) Overview](/install-and-configure/advanced-configuration/custom-prom/aws-amp-integration.md)
* [AWS Agentless AMP](/install-and-configure/advanced-configuration/custom-prom/nOps-agentless-amp.md)
* [AMP with nOps Prometheus (`remote_write`)](/install-and-configure/advanced-configuration/custom-prom/amp-with-remote-write.md)

## Overview

This guide will walk you through the steps to deploy nOps with [AWS Distro for Open Telemetry (ADOT)](https://aws-otel.github.io/) to collect metrics from your Kubernetes clusters utilizing the `EKS-Optimized` license.

{% hint style="info" %}
nOps `EKS-Optimized` allows for 15 days of query history. Unlock unlimited history with [nOps Enterprise](https://www.nOps.com/pricing).
{% endhint %}

![Architecture Diagram](/images/adot-architecture.png)

## Prerequisites

1. An AWS Managed Prometheus Workspace is required to use ADOT
2. AWS IAM permissions to add permissions for nOps to read from the workspace

Before following this guide, make sure you've reviewed AWS' [Set up metrics ingestion using AWS Distro for Open Telemetry on an Amazon Elastic Kubernetes Service cluster](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-onboard-ingest-metrics-OpenTelemetry.html) to enable the ADOT collector daemonSet.

This guide assumes that the nOps Helm release name and the nOps namespace have the same value (usually this will be `nOps`), which allows a global find and replace on `YOUR_NAMESPACE`.

## Configuration

Clone [this repository](https://github.com/nOps/poc-common-configurations/tree/main/aws/amp-with-adot) that contains all of the configuration files you will need to deploy nOps with ADOT.

```bash
git clone https://github.com/nOps/poc-common-configurations.git
cd poc-common-configurations/aws-amp/adot
```

Update all configuration files with your cluster name (replace all instances of `YOUR_CLUSTER_NAME_HERE`). The examples use `cluster_id` for the key of the key:value pair for the cluster name. You can use any key you want, including what is likely already being used.

### ADOT configuration

There are many options for deploying the ADOT daemonSet. At a minimum, nOps needs the provided scrape config to be added to the ADOT Prometheus ConfigMap. This sample ConfigMap also contains cAdvisor metrics, which is required by nOps.

```bash
kubectl apply -f example-configs/prometheus-daemonset.yaml -n adot-col
```

Alternatively, you can add these items to your [existing ConfigMap](https://github.com/nOps/poc-common-configurations/blob/main/aws/amp-with-adot/example-configs/nOps-adot-scrape-config.yaml).


{% hint style="info" %}
For the nOps `scrape_configs` job, `honor_labels: true` must be set. Without this, you will likely only see the `kube-system` or `nOps` namespace in the UI.
{% endhint %}

### nOps AWS IAM setup

1. Create the nOps namespace:

    ```bash
    kubectl create ns YOUR_NAMESPACE
    ```

1. Create the AWS IAM policy to allow nOps to query metrics from AMP:

    ```bash
    aws iam create-policy --policy-name nOps-read-amp-metrics --policy-document file://iam-read-amp-metrics.json
    ```

1. (Optional) Create the AWS IAM policy to allow nOps to find savings in the AWS Account:

    ```bash
    aws iam create-policy --policy-name DescribeResources --policy-document file://iam-describeCloudResources.json
    ```

1. (Optional) Create the AWS IAM policy to allow nOps to write to find account-level tags:

    ```bash
    aws iam create-policy --policy-name OrganizationListAccountTags --policy-document file://iam-listAccounts-tags.json
    ```

### nOps primary installation

1. Configure the nOps Service Account:

    ```bash
    eksctl create iamserviceaccount \
        --name nOps-sa \
        --namespace YOUR_NAMESPACE \
        --cluster YOUR_CLUSTER_NAME_HERE --region YOUR_REGION \
        --attach-policy-arn arn:aws:iam::AWS_ACCOUNT_NUMBER:policy/nOps-read-amp-metrics \
        --attach-policy-arn arn:aws:iam::AWS_ACCOUNT_NUMBER:policy/OrganizationListAccountTags \
        --attach-policy-arn arn:aws:iam::AWS_ACCOUNT_NUMBER:policy/DescribeResources \
        --override-existing-serviceaccounts --approve
    ```

1. Install nOps:

    ```bash
    aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws
    helm install YOUR_NAMESPACE \
        oci://public.ecr.aws/nOps/cost-analyzer \
        -f values-nOps-primary.yaml
        -f https://raw.githubusercontent.com/nOps/cost-analyzer-helm-chart/develop/cost-analyzer/values-eks-cost-monitoring.yaml
    ```

### nOps agent installation

This assumes you have created the IAM policies above. If using multiple AWS accounts, you will need to create the policies in each account (section titled nOps AWS IAM setup).

1. Create the nOps namespace:

    ```bash
    kubectl create ns YOUR_NAMESPACE
    ```

1. Configure the nOps Service Account:

    ```bash
    eksctl create iamserviceaccount \
        --name nOps-sa \
        --namespace YOUR_NAMESPACE \
        --cluster YOUR_CLUSTER_NAME_HERE --region YOUR_REGION \
        --attach-policy-arn arn:aws:iam::AWS_ACCOUNT_NUMBER:policy/nOps-read-amp-metrics \
        --attach-policy-arn arn:aws:iam::AWS_ACCOUNT_NUMBER:policy/OrganizationListAccountTags \
        --attach-policy-arn arn:aws:iam::AWS_ACCOUNT_NUMBER:policy/DescribeResources \
        --override-existing-serviceaccounts --approve
    ```

1. Deploy the nOps agent:

    ```bash
    aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws
    helm install YOUR_NAMESPACE \
        oci://public.ecr.aws/nOps/cost-analyzer \
        -f values-nOps-agent.yaml
        -f https://raw.githubusercontent.com/nOps/cost-analyzer-helm-chart/develop/cost-analyzer/values-eks-cost-monitoring.yaml
    ```

## ADOT daemonSet quick install

See this [example .yaml file](https://github.com/nOps/poc-common-configurations/blob/main/aws/amp-with-adot/example-configs/prometheus-daemonset.yaml) for an all-in-one ADOT DS config.

## Troubleshooting

For more help troubleshooting, see our [Amazon Managed Service for Prometheus (AMP) Overview](aws-amp-integration.md#troubleshooting) doc.
