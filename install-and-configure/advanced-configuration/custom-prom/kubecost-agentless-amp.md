# nOps with AWS Managed Prometheus Agentless Monitoring

## See also

* [AMP Overview](/install-and-configure/advanced-configuration/custom-prom/aws-amp-integration.md)
* [AWS Distro for Open Telemetry](/install-and-configure/advanced-configuration/custom-prom/nOps-aws-distro-open-telemetry.md)
* [AMP with nOps Prometheus (`remote_write`)](/install-and-configure/advanced-configuration/custom-prom/amp-with-remote-write.md)

## Overview

{% hint style="info" %}
Using AMP allows multi-cluster nOps with EKS-Optimized licenses.
{% endhint %}

This guide will walk you through the steps to deploy nOps with AWS Agentless AMP to collect metrics from your Kubernetes cluster.

{% hint style="info" %}
Keep in mind that "agentless" refers to the Prometheus scraper, not the nOps agent. The nOps agent is still required to collect metrics from the cluster.
{% endhint %}

The guide below assumes a multi-cluster setup will be used, which is supported with the EKS-Optimized license that is enabled by following the below guide.

## Prerequisites

Follow this [Using an AWS managed collector](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-collector-how-to.html) guide to enable the managed collector.

This guide assumes that the nOps Helm release name and the nOps namespace are equal, which allows a global find-and-replace on `$nOps_NAMESPACE`.

## Architecture diagram

![Agentless AMP Architecture](../../../images/diagrams/AMP-agentless-multi-cluster-Prometheus-nOps-architecture.png)

## Agentless AMP Configuration

### AMP setup

1. Clone this [poc-common-configurations](https://github.com/nOps/poc-common-configurations) repository that contains all of the configuration files you will need to deploy nOps with AWS Agentless AMP.

    ```sh
    git clone https://github.com/nOps/poc-common-configurations.git
    cd poc-common-configurations/aws/amp-agentless
    ```

2. Update all configuration files with your cluster name (replace all `YOUR_CLUSTER_NAME_HERE`).

3. Build the configuration variables:

    ```sh
    CLUSTER_NAME=YOUR_CLUSTER_NAME_HERE
    CLUSTER_REGION=us-east-2
    nOps_NAMESPACE=nOps
    WORKSPACE_ID=ws-YOUR_WORKSPACE_ID
    AWS_ACCOUNT_ID=11111111111
    WORKSPACE_ARN=$(aws amp describe-workspace --workspace-id $WORKSPACE_ID --output json | jq -r .workspace.arn)
    CLUSTER_JSON=$(aws eks describe-cluster --name $CLUSTER_NAME --region $CLUSTER_REGION --output json)
    CLUSTER_ARN=$(echo $CLUSTER_JSON | jq -r .cluster.arn)
    SECURITY_GROUP_IDS=$(echo $CLUSTER_JSON | jq -r .cluster.resourcesVpcConfig.clusterSecurityGroupId)
    SUBNET_IDS=$(echo $CLUSTER_JSON | jq -r '.cluster.resourcesVpcConfig.subnetIds | @csv')
    ```

4. Create the nOps scraper:

    ```sh
    nOps_SCRAPER_OUTPUT=$(aws amp create-scraper --output json \
        --alias nOps-scraper \
        --source eksConfiguration="{clusterArn=$CLUSTER_ARN, securityGroupIds=[$SECURITY_GROUP_IDS],subnetIds=[$SUBNET_IDS]}" \
        --scrape-configuration configurationBlob="$(base64 scraper-nOps-with-networking.yaml|tr -d '\n')" \
        --destination ampConfiguration="{workspaceArn=$WORKSPACE_ARN}")
    echo $nOps_SCRAPER_OUTPUT
    nOps_SCRAPER_ID=$(echo $nOps_SCRAPER_OUTPUT|jq -r .scraperId)
    echo $nOps_SCRAPER_ID
    ```

5. Get the ARN of the scraper:

    ```sh
    ARN_PART=$(aws amp describe-scraper --output json --region $CLUSTER_REGION --scraper-id $nOps_SCRAPER_ID | jq -r .scraper.roleArn | cut -d'_' -f2)
    ROLE_ARN_nOps_SCRAPER="arn:aws:iam::$AWS_ACCOUNT_ID:role/AWSServiceRoleForAmazonPrometheusScraper_$ARN_PART"
    echo $ROLE_ARN_nOps_SCRAPER
    ```

6. Add the ARN of the scraper to the `kube-system/aws-auth` configMap:

    ```sh
    eksctl create iamidentitymapping \
        --cluster $CLUSTER_NAME --region $CLUSTER_REGION \
        --arn $ROLE_ARN_nOps_SCRAPER \
        --username aps-collector-user
    ```

7. Create a scraper for cAdvisor and node exporter. Node exporter is optional. cAdvisor is required, but may already be available.

    ```sh
    CADVSIOR_SCRAPER_OUTPUT=$(aws amp create-scraper --output json \
        --alias cadvisor-scraper \
        --source eksConfiguration="{clusterArn=$CLUSTER_ARN, securityGroupIds=[$SECURITY_GROUP_IDS],subnetIds=[$SUBNET_IDS]}" \
        --scrape-configuration configurationBlob="$(base64 scraper-cadvisor-node-exporter.yaml|tr -d '\n')" \
        --destination ampConfiguration="{workspaceArn=$WORKSPACE_ARN}")
    echo $CADVSIOR_SCRAPER_OUTPUT
    CADVSIOR_SCRAPER_ID=$(echo $CADVSIOR_SCRAPER_OUTPUT|jq -r .scraperId)
    echo $CADVSIOR_SCRAPER_ID
    ```

8. Get the ARN of the scraper:

    ```sh
    ARN_PART=$(aws amp describe-scraper --output json --region $CLUSTER_REGION --scraper-id $CADVSIOR_SCRAPER_ID | jq -r .scraper.roleArn | cut -d'_' -f2)
    ROLE_ARN_CADVSIOR_SCRAPER="arn:aws:iam::$AWS_ACCOUNT_ID:role/AWSServiceRoleForAmazonPrometheusScraper_$ARN_PART"
    echo $ROLE_ARN_CADVSIOR_SCRAPER
     ```

9. Add the ARN of the scraper to the kube-system/aws-auth configmap:

    ```sh
    eksctl create iamidentitymapping \
        --cluster $CLUSTER_NAME --region $CLUSTER_REGION \
        --arn $ROLE_ARN_CADVSIOR_SCRAPER \
        --username aps-collector-user
    ```

10. Apply the agentless RBAC permissions:

    ```sh
    kubectl apply -f rbac.yaml
    ```

### nOps primary cluster installation

1. Create the nOps namespace:

    ```bash
    kubectl create ns $nOps_NAMESPACE
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

1. Configure the nOps Service Account:

    * If the following fails, be sure that IRSA is enabled on your EKS cluster. <https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html>

    ```bash
    eksctl create iamserviceaccount \
    --name nOps-sa \
    --namespace $nOps_NAMESPACE \
    --cluster $CLUSTER_NAME --region $CLUSTER_REGION \
    --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/nOps-read-amp-metrics \
    --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/OrganizationListAccountTags \
    --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/DescribeResources \
    --override-existing-serviceaccounts --approve
    ```

1. Update the placeholder values such as `YOUR_CLUSTER_NAME_HERE` in *values-nOps-primary.yaml*.

1. Install nOps on your primary:

    ```bash
    aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws
    helm install $nOps_NAMESPACE -n $nOps_NAMESPACE \
        oci://public.ecr.aws/nOps/cost-analyzer \
        -f https://raw.githubusercontent.com/nOps/cost-analyzer-helm-chart/develop/cost-analyzer/values-eks-cost-monitoring.yaml \
        -f values-nOps-primary.yaml
    ```

### nOps agents installation

Follow the above AMP setup section to configure the scraper(s) on each cluster.

This assumes you have created the AWS IAM policies above. If using multiple AWS accounts, you will need to create the policies in each account.

1. Update the placeholder values such as `YOUR_CLUSTER_NAME_HERE` in *values-nOps-agent.yaml*.

1. Create the nOps namespace:

    ```bash
    kubectl create ns $nOps_NAMESPACE
    ```

1. Configure the nOps Service Account:

    ```bash
    eksctl create iamserviceaccount \
        --name nOps-sa \
        --namespace $nOps_NAMESPACE \
        --cluster $CLUSTER_NAME --region $CLUSTER_REGION \
        --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/nOps-read-amp-metrics \
        --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/OrganizationListAccountTags \
        --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/DescribeResources \
        --override-existing-serviceaccounts --approve
    ```

1. Deploy the nOps agent:

    ```bash
    aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws
    helm install $nOps_NAMESPACE -n $nOps_NAMESPACE \
        oci://public.ecr.aws/nOps/cost-analyzer \
        -f https://raw.githubusercontent.com/nOps/cost-analyzer-helm-chart/develop/cost-analyzer/values-eks-cost-monitoring.yaml \
        -f values-nOps-agent.yaml
    ```

## Troubleshooting

It will take a few minutes for the scrapers start.

For more help troubleshooting, see our [Amazon Managed Service for Prometheus (AMP) Overview](aws-amp-integration.md#troubleshooting) doc.

