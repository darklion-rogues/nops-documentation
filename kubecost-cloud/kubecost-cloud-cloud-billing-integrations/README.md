# nOps Cloud: Cloud Billing Integrations

{% hint style="info" %}
This documentation should only be consulted when using nOps Cloud. For information about the configuring cloud integrations with self-hosted nOps, see [here](/install-and-configure/install/cloud-integration/README.md).
{% endhint %}

Integration with cloud service providers (CSPs) via their respective billing APIs allows nOps to display out-of-cluster (OOC) costs (e.g. AWS S3, Google Cloud Storage, Azure Storage Account). Additionally, it allows nOps to reconcile nOps's in-cluster estimates with actual billing data to improve accuracy.

To learn more about how nOps provides accurate cost data or how to manage existing cloud integrations, read below. Otherwise, see any of the following articles to get started on integrating a specific CSP:

* [nOps Cloud AWS Integration](/nOps-cloud/nOps-cloud-cloud-billing-integrations/nOps-cloud-aws-integration.md)
* [nOps Cloud GCP Integration](/nOps-cloud/nOps-cloud-cloud-billing-integrations/nOps-cloud-gcp-integration.md)
* [nOps Cloud Azure Integration](/nOps-cloud/nOps-cloud-cloud-billing-integrations/nOps-cloud-azure-integration.md)

## Reconciliation

Reconciliation matches in-cluster assets with items found in the billing data pulled from the CSP. This allows nOps to display the most accurate depiction of your in-cluster spending. Additionally, the reconciliation process creates `Network` assets for in-cluster nodes based on the information in the billing data. The main drawback of this process is that the CSPs have between a 6 to 24-hour delay in releasing billing data, and reconciliation requires a complete day of cost data to reconcile with the in-cluster assets. This requires a 48-hour window between resource usage and reconciliation. If reconciliation is performed within this window, asset cost is deflated to the partially complete cost shown in the billing data.

Cost-based metrics are based on on-demand pricing unless there is definitive data from a CSP that the node is not on-demand. This way estimates are as accurate as possible. If a new reserved instance is provisioned or a node joins a savings plan:

1. nOps continues to emit on-demand pricing until the node is added to the cloud bill.
2. Once the node is added to the cloud bill, nOps starts emitting something closer to the actual price.
3. For the time period where nOps assumed the node was on-demand but it was actually reserved, reconciliation adjusts the price to reflect the actual cost.

{% hint style="info" %}
The reconciled assets will inherit the labels from the corresponding items in the billing data. If there exist identical label keys between the original assets and those of the billing data items, the label value of the original asset will take precedence.
{% endhint %}

## Managing cloud integrations

You can view your existing cloud integrations and their success status in the nOps UI by visiting _Settings_, then scrolling to Cloud Integrations. To create a new integration or learn more about existing integrations, select _View additional details_ to go to the Cloud Integrations page.

Here, you can view your existing integrations. For non-successful integrations, nOps will display a diagnostic error message in the Status column to contextualize steps toward successful integration.

![Example successful and failed cloud integrations](../../.gitbook/assets/Snag\_d920b39.png)

Select an individual integration to view a side panel that contains details about that integration. You will be able to select _Edit_ to adjust any details about your integration that were configured during the original setup.

Cloud integrations that have _not_ gathered any data can be deleted from the side panel by selecting the trash can icon next to the integration name. At the moment, integrations that have gathered data can only be deleted by the nOps Cloud team. For help with this, contact [cloud@nOps.com](mailto:cloud@nOps.com).
