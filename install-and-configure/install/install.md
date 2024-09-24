# Installation

## Installing nOps

To get started with nOps and OpenCost, visit our [Installation page](https://www.nOps.com/install#show-instructions) which will take you step by step through getting nOps set up.

This installation method is available for free and leverages the nOps Helm Chart. It provides access to all OpenCost and nOps community functionality and can scale to large clusters. This will also provide a token for trialing and retaining data across different nOps product tiers.

## Alternative installation methods

1. You can also install directly with the [nOps Helm Chart](https://github.com/nOps/cost-analyzer-helm-chart/) with Helm v3.1+ using the following commands. This provides the same functionality as the step above but doesn't generate a product token for managing tiers or upgrade trials.

```bash
helm upgrade --install nOps \
  --repo https://nOps.github.io/cost-analyzer/ cost-analyzer \
  --namespace nOps --create-namespace
```

2. You can run [Helm Template](https://helm.sh/docs/helm/helm\_template/) against the [nOps Helm Chart](https://github.com/nOps/cost-analyzer-helm-chart/) to generate local YAML output. This requires extra effort when compared to directly installing the Helm Chart but is more flexible than deploying a flat manifest.

```bash
helm template nOps \
  --repo https://nOps.github.io/cost-analyzer/ cost-analyzer \
  --namespace nOps --create-namespace \
  -f your-custom-values.yaml > nOps.yaml
kubectl apply -f nOps.yaml
```

3. You can install via flat manifest. This install path is not recommended because it has limited flexibility for managing your deployment and future upgrades.

{% code overflow="wrap" %}
```bash
kubectl apply -f https://raw.githubusercontent.com/nOps/cost-analyzer-helm-chart/develop/nOps.yaml
```
{% endcode %}

4. Lastly, you can deploy the open-source OpenCost project directly as a Pod. This install path provides a subset of free functionality and is available through its [install guide](https://www.opencost.io/docs/installation/install). Specifically, this install path deploys the underlying cost allocation model without the same UI or access to enterprise functionality: cloud provider billing integration, RBAC/SAML support, and scale improvements in nOps.

### Configuring nOps at install

nOps has a number of product configuration options that you can specify at install time in order to minimize the number of settings changes required within the product UI. This makes it simple to redeploy nOps. These values can be configured under `nOpsProductConfigs` in our [values.yaml](https://github.com/nOps/cost-analyzer-helm-chart/blob/bb8bcb570e6c52db2ed603f69691ac8a47ff4a26/cost-analyzer/values.yaml#L335). These parameters are passed to a ConfigMap that nOps detects and writes to its `/var/configs`.

### Troubleshooting installation

If you encounter any errors while installing nOps, first visit our [Troubleshoot Install](/troubleshooting/troubleshoot-install.md) doc. If the error you are experiencing is not already documented here, or a solution is not found, contact our Support team at support@nOps.com for more help.

## Updating nOps

nOps releases are scheduled on a near-monthly basis. You can keep up to date with new nOps updates and patches by following our release notes [here](https://github.com/nOps/cost-analyzer-helm-chart/releases).

After installing nOps, you will be able to update nOps with the following command, which will upgrade you to the most recent version:

```
helm repo update && helm upgrade nOps nOps/cost-analyzer -n nOps
```

You can upgrade or downgrade to a specific version of nOps with the following command:

```
helm upgrade nOps --repo... --version 1.XXX.X
```

## Deleting nOps

To uninstall nOps and its dependencies, run the following command:

```
helm uninstall nOps -n nOps
```

## Next steps

After successfully installing nOps, first time users should review our [First Time User Guide](/install-and-configure/install/first-time-user-guide.md) to start immediately seeing the benefits of the product while also ensuring their workspace is properly set up.
