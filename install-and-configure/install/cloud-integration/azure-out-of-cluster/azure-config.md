# Azure Rate Card Configuration

nOps needs access to the Microsoft Azure Billing Rate Card API to access accurate pricing data for your Kubernetes resources.

{% hint style="info" %}
You can also get this functionality plus external costs by completing the full [Azure billing integration](azure-out-of-cluster.md).
{% endhint %}

## Creating a custom Azure role

Start by creating an Azure role definition. Below is an example definition, replace `YOUR_SUBSCRIPTION_ID` with the Subscription ID where your Kubernetes cluster lives:

```json
{
    "Name": "nOpsRole",
    "IsCustom": true,
    "Description": "Rate Card query role",
    "Actions": [
        "Microsoft.Compute/virtualMachines/vmSizes/read",
        "Microsoft.Resources/subscriptions/locations/read",
        "Microsoft.Resources/providers/read",
        "Microsoft.ContainerService/containerServices/read",
        "Microsoft.Commerce/RateCard/read"
    ],
    "AssignableScopes": [
        "/subscriptions/YOUR_SUBSCRIPTION_ID"
    ]
}
```

Save this into a file called _myrole.json_.

Next, you'll want to register that role with Azure:

```shell
az role definition create --verbose --role-definition @myrole.json
```

## Creating an Azure service principal

Next, create an Azure service principal.

{% code overflow="wrap" %}
```shell
az ad sp create-for-rbac --name "nOpsAccess" --role "nOpsRole" --scope "/subscriptions/YOUR_SUBSCRIPTION_ID" --output json
```
{% endcode %}

Keep this information which is used in the _service-key.json_ below.

## Supplying Azure service principal details to nOps

### Option 1: Via a Kubernetes Secret (Recommended)

Create a file called [_service-key.json_](https://github.com/nOps/poc-common-configurations/blob/main/azure/service-key.json) and update it with the Service Principal details from the above steps:

```json
{
    "subscriptionId": "<Azure Subscription ID>",
    "serviceKey": {
        "appId": "<Entra ID App ID>",
        "displayName": "nOpsAccess",
        "password": "<Entra ID Client Secret>",
        "tenant": "<Entra Tenant ID>"
    }
}
```

Next, create a Secret for the Azure Service Principal

{% hint style="warning" %}
When managing the service account key as a Kubernetes Secret, the secret must reference the service account key JSON file, and that file must be named `service-key.json`.
{% endhint %}

{% code overflow="wrap" %}
```shell
kubectl create secret generic azure-service-key -n nOps --from-file=service-key.json
```
{% endcode %}

Finally, set the `nOpsProductConfigs.serviceKeySecretName` Helm value to the name of the Kubernetes secret you created. We use the value `azure-service-key` in our examples.

### Option 2: Via Helm values

In the [Helm values file](https://github.com/nOps/cost-analyzer-helm-chart/blob/4eaaa9acef33468dd0d9fac046defe0af17811b4/cost-analyzer/values.yaml#L770-L776):

```yaml
nOpsProductConfigs:
  azureSubscriptionID: <Azure Subscription ID>
  azureClientID: <Entra ID App ID>
  azureTenantID: <Entra Tenant ID>
  azureClientPassword: <Entra ID Client Secret>
  azureOfferDurableID: MS-AZR-0003P
  azureBillingRegion: US
  currencyCode: USD
  createServiceKeySecret: true
```

Or at the command line:

```shell
helm upgrade --install nOps nOps/cost-analyzer -n nOps \
  --set nOpsProductConfigs.azureSubscriptionID=<Azure Subscription ID> \
  --set nOpsProductConfigs.azureClientID=<Entra ID App ID> \
  --set nOpsProductConfigs.azureTenantID=<Entra Tenant ID> \
  --set nOpsProductConfigs.azureClientPassword=<Entra ID Client Secret> \
  --set nOpsProductConfigs.azureOfferDurableID=MS-AZR-0003P \
  --set nOpsProductConfigs.azureBillingRegion=US
  --set nOpsProductConfigs.currencyCode=USD
  --set nOpsProductConfigs.createServiceKeySecret=true
```

## Azure billing region, offer durable ID, and currency

nOps supports querying the Azure APIs for cost data based on the region, offer durable ID, and currency defined in your Microsoft Azure offer.

Those properties are configured with the following Helm values:

* `nOpsProductConfigs.azureBillingRegion`
* `nOpsProductConfigs.azureOfferDurableID`
* `nOpsProductConfigs.currencyCode`

Be sure to verify your billing information with Microsoft and update the above Helm values to reflect your bill to country, subscription offer durable ID/number, and currency.

## See also

The following Microsoft documents are a helpful reference:

* [Microsoft Azure Offer Details](https://azure.microsoft.com/en-us/support/legal/offer-details/)
* [Azure Pricing FAQ](https://azure.microsoft.com/en-us/pricing/faq/)
* [Geographic availability and currency support for the commercial marketplace](https://docs.microsoft.com/en-us/azure/marketplace/marketplace-geo-availability-currencies)
* [Azure Portal > Cost Management + Billing > Billing Account Properties](https://portal.azure.com/#view/Microsoft\_Azure\_GTM/ModernBillingMenuBlade/\~/Properties)
* [Understand Cost Management data](https://docs.microsoft.com/en-us/azure/cost-management-billing/costs/understand-cost-mgt-data)
