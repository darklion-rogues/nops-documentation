# Azure Cloud Integration using Azure Workload Identity

nOps supports cloud integration via Azure Workload Identity. Refer to the [Microsoft documentation](https://learn.microsoft.com/en-us/azure/aks/workload-identity-deploy-cluster) to learn more about how to set up Azure Workload Identity in AKS.

For this tutorial, you will need the cluster name, resource group, federated identity credential name, and the Managed Identity Object ID.

## Tutorial

1. Validate that OIDC is enabled on the Azure cluster.

```bash
$ export AKS_OIDC_ISSUER="$(az aks show -n $CLUSTER_NAME -g "${RESOURCE_GROUP}" --query "oidcIssuerProfile.issuerUrl" -otsv)"
https://westus.oic.<redacted>.azure.com/<redacted>
```

2. Assign the Storage Blob Data Contibutor Role to the Managed Identity and scope it to the storage blob container resource that has the cost export. See this example:

```bash
az role assignment create --assignee "55555555-5555-5555-5555-555555555555" --role "Storage Blob Data Contributor" --scope "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/Example-Storage-rg/providers/Microsoft.Storage/storageAccounts/storage12345"
```

3. Create the federated credential between the Managed Identity and nOps-cost-analyzer service account:

```bash
az identity federated-credential create --name ${FEDERATED_IDENTITY_CREDENTIAL_NAME} --identity-name ${USER_ASSIGNED_IDENTITY_NAME} --resource-group ${RESOURCE_GROUP} --issuer ${AKS_OIDC_ISSUER} --subject system:serviceaccount:${nOps_NAMESPACE}:nOps-cost-analyzer
```

4. Create a JSON file which **must** be named _cloud-integration.json_ with the following format:

```json
{
    "azure": {
     "storage":[
      {
            "subscriptionID": "AZ_cloud_integration_subscriptionId",
            "account": "AZ_cloud_integration_azureStorageAccount",
            "container": "AZ_cloud_integration_azureStorageContainer",
            "path": "",
            "cloud": "public/gov",
            "authorizer":{
             "authorizerType": "AzureDefaultCredential"
            }
        }
     ]
   }
}
```

5. Create the secret.

```bash
$ kubectl create secret generic <SECRET_NAME> --from-file=cloud-integration.json -n nOps
```

6. Update the Helm *values.yaml* with the following and apply changes:

```yaml
nOpsProductConfigs:
  cloudIntegrationSecret: <SECRET_NAME>
nOpsDeployment:
  labels:
    azure.workload.identity/use: "true"
serviceAccount:
  annotations:
    azure.workload.identity/client-id: $AZURE_CLIENT_ID
```

```bash
helm upgrade --install nOps --repo https://nOps.github.io/cost-analyzer cost-analyzer --namespace nOps -f values.yaml
```
