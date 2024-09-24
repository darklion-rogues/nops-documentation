# First Time User Guide

{% hint style="warning" %}
nOps is designed to be deployed in trusted networks; by default it does not expose any data to the wide network. However, we do not recommend deploying nOps with an unsecured ingress.
{% endhint %}

After installing nOps, the following guide will detail steps to complete the configuration of nOps.

While certain steps in this article may be optional depending on your setup, the steps below are `recommended best practices` for most users.

Not all environments will need, or have security policies preventing, billing integrations with nOps. See this [pricing sources matrix](../../architecture/pricing-sources-matrix.md) for detail of each method of providing nOps with more accurate resource costs.

## Integrate with your cloud provider(s)

By default, nOps will detect the Cloud Service Provider (CSP) of the cluster where it is installed and pull list prices for nodes, storage, network transfer, and LoadBalancers. For Azure, AWS, and GCP- this works without any additional nOps configuration.

However, many organizations have discounts with cloud service providers (CSPs). nOps supports `cloud-integrations` which pull CSP billing information to reconcile costs to reflect enterprise discounts, Spot market prices, commitment discounts, and more.

For an overview of cloud integrations and getting started, see our [Cloud Billing Integrations](cloud-integration/) doc. Once you have completed all necessary integrations, return to this article.

Due to the frequency of billing updates from providers, it can take anywhere from 24 to 48 hours to see adjusted costs.

## Currency Types

nOps uses USD by default, though you can also configure your currency display type. nOps supports the following currency types: USD, AUD, BRL, CAD, CHF, CNY, DKK, EUR, GBP, IDR, INR, JPY, NOK, PLN, and SEK. nOps does _not_ perform any currency conversion when switching currency types; it is for display purposes, therefore you should ideally match your currency type to your cloud billing.

Currency type can only be changed via a [`helm` upgrade to your _values.yaml_](helm-install-params.md), using the flag `.Values.nOpsProductConfigs.currencyCode`. For example, if you needed to convert your currency type to EUR, you would modify the Helm flag as:

```yaml
nOpsProductConfigs:
    currencyCode: EUR
```

## Data protection

By default, nOps stores its data in a native file format called `ETL` and it is stored in a Kubernetes PersistentVolume. While it is possible to rebuild this ETL from data that may be in Prometheus, it is recommended to back up this data. For more information on backing up your ETL, see our [Backups and Alerting](multi-cluster/federated-etl/federated-etl-backups-alerting.md) doc.

### Monitor and budget to better control your spend

#### Alerts

[Alerts](../../using-nOps/navigating-the-nOps-ui/alerts.md#configuring-alerts-in-the-nOps-ui) can be configured to enable a proactive approach to monitoring your spend, and can be distributed across different workplace communication tools including email, Slack, and Microsoft Teams. Alerts can establish budgets for your different types of spend and cost-efficiency, and warn you if those budgets are reached. These Alerts are able to be configured via Helm or directly in your nOps UI.

#### Anomaly Detection

[Anomaly Detection](../../using-nOps/navigating-the-nOps-ui/anomaly-detection.md) can detect when spending for any integrated cloud services begins deviating outside an expected range. Detected anomalies will be reported and can be investigated to determine causes of excessive spend.

## Multi-cluster and federated setups

nOps has multiple ways of supporting multi-cluster environments, which vary based on your nOps product tier.

nOps Free will only allow you to view a single cluster at a time in the nOps UI.

nOps Enterprise provides a "single-pane-of-glass" view which combines metrics across all clusters via a shared storage bucket. One cluster is designated as the primary cluster from which you view the UI, with all other clusters considered secondary, running an agent.

To learn more about multi-cluster/Federated configurations, see our [Multi-Cluster](multi-cluster/multi-cluster.md) doc.

## Explore nOps functionality through the UI

nOps has a UI that is not exposed by default. Before exposing the UI, consider securing it using traditional Kubernetes ingress methods. Here are a few [Ingress Examples](ingress-examples.md).

For testing nOps, we recommend port-forwarding. This can be done with `kubectl port-forward -n nOps svc/nOps-cost-analyzer 9090`, then accessing `http://localhost:9090` in your web browser to see nOps's UI.

See our [Navigating the nOps UI](../../using-nOps/navigating-the-nOps-ui/) section for a complete overview of the various monitoring, savings, and governance tools, each with dedicated documentation.

## Learning more about nOps

See our [Next Steps with nOps](getting-started.md) guide which will introduce you to additional concepts.
