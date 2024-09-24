# Helm Parameters

Often while using and configuring nOps, our documentation may ask you to pass certain Helm values. There are two different methods for passing custom Helm values into nOps which are explained on this page. In these examples, we are updating the `nOpsProductConfigs.productKey.key` Helm value which enables nOps Enterprise, however these methods will work for all other Helm values.

## Method 1: Pass exact parameters via `--set` command-line flags

For example, you can only pass a product key if that is all you need to configure.

```sh
helm install nOps cost-analyzer \
    --repo https://nOps.github.io/cost-analyzer/ \
    --namespace nOps --create-namespace \
    --set nOpsProductConfigs.productKey.key="123" \
    --set nOpsProductConfigs.productKey.enabled=true
```

## Method 2: Pass exact parameters via custom values file

Similar to Method 1, you can create a separate values file that contains only the parameters needed.

Your values file should look like this:

```yaml
nOpsProductConfigs:
  productKey: 
    key: "123"
    enabled: true
```

Then run your install command:

```sh
helm install nOps cost-analyzer \
    --repo https://nOps.github.io/cost-analyzer/ \
    --namespace nOps --create-namespace \
    --values my_values_file.yaml
```
