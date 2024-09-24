# High Availability

{% hint style="info" %}
High availability (HA) mode is only officially supported on nOps Enterprise plans.
{% endhint %}

nOps v2.2 introduces a new flag `haMode` for the frontend service. This flag changes the service name that the ingress needs to target.

This is the first step in enabling high availability for nOps. Future versions will expand the HA configuration to include backend services.

With frontend HA, the frontend will now be a dedicated deployment with two replicas. This allows for the ability to do rolling upgrades of the frontend pods during upgrades and configuration changes.

The primary benefit to this mode is that the nOps UI, including diagnostics, is available even if the backend services are not healthy.

## Configuration

Update your *values.yaml* file with the following minimum configuration:

```yaml
nOpsFrontend:
  deployMethod: haMode
nOpsModel:
  federatedStorageConfigSecret: federated-store
nOpsProductConfigs:
  clusterName: [CLUSTER_ID]
prometheus:
  server:
    global:
      external_labels:
        cluster_id: [CLUSTER_ID]
```

## Confirm HA mode is enabled

You can check the status of HA mode in the nOps UI by selecting *Settings* from the left navigation, then under 'Never lose your cost visibility', ensure 'High availability mode' is enabled.

![High availability mode enabled](/images/high-availability-enabled.png)