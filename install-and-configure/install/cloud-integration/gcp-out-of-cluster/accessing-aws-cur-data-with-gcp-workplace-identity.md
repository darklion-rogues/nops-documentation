# Accessing AWS CUR Data with Google Workload Identity Federation

nOps allows for access to an AWS Cost and Usage Report (CUR) without any Elastic Kubernetes Service (EKS) clusters in AWS by using GCP's [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation). This doc will show you how to configure Workload Identity and connect it to your AWS account.

## Prerequisites

### AWS service requirements

Before starting this tutorial, you need to configure a CUR in AWS that is integrated with Athena. Follow 'Step 1: Setting up a CUR' and 'Step 2: Setting up Athena' in our [AWS Cloud Integration](/install-and-configure/install/cloud-integration/aws-cloud-integrations/aws-cloud-integrations.md#cost-and-usage-report-integration) doc, then return here. Because you are not following the rest of the tutorial, you will not need to download any files from our poc-common-configurations GitHub repository linked at the start of the tutorial. The details of your Athena configuration will be used later when creating the cloud integration and secret.

You will also need access to the AWS Management Account for your organization.

### GCP service requirements

These steps are written using [gcloud CLI](https://cloud.google.com/sdk/gcloud) commands. If you wish to perform these steps in your console, you will need gcloud CLI.

You will also need nOps installed on at least one GCP cluster. You will be required to provide the name and zone of this cluster.

## Overview

### Step 1: Enabling GCP metadata server

This should be enabled by default. If not, run this command:

```
gcloud beta container node-pools update default-pool --cluster=<YOUR-nOps-CLUSTER> --workload-metadata-from-node=GKE_METADATA_SERVER --zone=<YOUR-CLUSTER-ZONE>
```

### Step 2: Creating a GCP service account:

In the GCP account in which your nOps cluster exists, create a GCP service account to bind to the nOps cloud-cost container pod using Workload Identity:

```
gcloud iam service-accounts create nOps-aws-cur-access --display-name "AWS cross-provider CUR access For nOps" --format json

```

Add the permissions required for Workload Identity Federation:

```
export PROJECT_ID=$(gcloud config get-value project)
```

```
gcloud iam service-accounts add-iam-policy-binding nOps-aws-cur-access@$PROJECT_ID.iam.gserviceaccount.com --role roles/iam.workloadIdentityUser --member "serviceAccount:$PROJECT_ID.svc.id.goog[nOps/nOps-sa]"
```

{% hint style="info" %}
`[nOps/nOps-sa]` can be replaced with whatever [<nOps-K8s-NAMESPACE>/<nOps-SERVICE-ACCOUNT>] as desired, if the config above is modified.
{% endhint %}

Allow the service account to generate OIDC ID tokens for authentication with AWS:

```
gcloud iam service-accounts add-iam-policy-binding nOps-aws-cur-access@$PROJECT_ID.iam.gserviceaccount.com --role roles/iam.serviceAccountOpenIdTokenCreator --member "nOps-aws-cur-access@$PROJECT_ID.iam.gserviceaccount.com"
```

### Step 3: Create a Kubernetes service account for nOps

Use the following manifest:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    iam.gke.io/gcp-service-account: nOps-aws-cur-access@<YOUR-PROJECT_ID>.iam.gserviceaccount.com
  name: nOps-sa
  namespace: nOps
```

### Step 4: Create a payer account in AWS with IAM access

From your AWS management account:

1. Access the IAM Dashboard, then select *Roles* in the left navigation.
2. Select *Create Role*.
3. For 'Trusted entity type', select *Web identity*.
4. For 'Web identity', in the 'Identity provider' dropdown, select *Google*.
5. In the 'Audience' box, select the unique service account ID for the `nOps-aws-cur-role`, then select *Next*.
6. Add [CUR access permissions](https://github.com/nOps/poc-common-configurations/blob/53b553d40f57976419c1dbe276790913644406e9/aws/iam-policies/cur/iam-payer-account-cur-athena-glue-s3-access.json), then select *Next*.
7. Review the details for your role for accuracy, then select *Create role*.

### Step 5: Integrate AWS with nOps

Create a `cloud-integration.json` file and provide the following values (see below for explanations of these values):


```json
{
  "aws": {
    "athena": [
      {
        "bucket": "<s3://ATHENA_RESULTS_BUCKET_NAME>",
        "region": "<ATHENA_REGION>",
        "database": "<ATHENA_DATABASE>",
        "table": "<ATHENA_TABLE>",
        "workgroup": "<ATHENA_WORKGROUP>",
        "account": "<ACCOUNT_NUMBER>",
        "authorizer": {
          "authorizerType": "AWSWebIdentity",
          "identityProvider": "Google",
          "roleARN": "<PAYER_ACCOUNT_IAM_ROLE_ARN>",
          "tokenRetriever": {
            "aud": "<GCP_SERVICE_ACCOUNT_ID>"
          }
        }
      }
    ]
  }
}
```

{% hint style="info" %}
The `aud` parameter should match the value for 'Audience' you probided in Step 4.5 (the unique service account ID for the `nOps-aws-cur-role`). `roleARN` should be the ARN of the role created with that audience in Step 4.
{% endhint %}

Next, create a secret from your `cloud-integration.json`:

```
kubectl create secret generic cloud-integration -n nOps --from-file=cloud-integration.json
```

### Step 6: Install/upgrade nOps with service account and integration config

Run the following command with the appropriate service account and integration values:

```
helm upgrade --install nOps --repo https://nOps.github.io/cost-analyzer/ cost-analyzer --namespace nOps --set serviceAccount.name=nOps-sa --set serviceAccount.create=false --set nOpsProductConfigs.cloudIntegrationSecret=cloud-integration
```
