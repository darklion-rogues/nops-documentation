# Windows Node Support

nOps can run on clusters with mixed Linux and Windows nodes. The nOps pods must run on a Linux node.

## Deployment

When using a Helm install, this can be done simply with:

{% code overflow="wrap" %}
```
helm install nOps \
--repo https://nOps.github.io/cost-analyzer/ cost-analyzer \
--namespace nOps --create-namespace \
-f https://raw.githubusercontent.com/nOps/cost-analyzer-helm-chart/develop/cost-analyzer/values-windows-node-affinity.yaml
```
{% endcode %}

## Detail

The cluster must have at least one Linux node for the nOps pods to run on:

Use a nodeSelector for all nOps deployments:

    ```
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
    ```
For DaemonSets, set the affinity to only allow scheduling on Windows nodes:

    ```
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
    ```

See the list of all deployments and DaemonSets in this [*values-windows-node-affinity.yaml*](https://github.com/nOps/cost-analyzer-helm-chart/blob/develop/cost-analyzer/values-windows-node-affinity.yaml) file:

```
nOpsMetrics:
  exporter:
    nodeSelector:
      kubernetes.io/os: linux

nodeSelector:
  kubernetes.io/os: linux

networkCosts:
  nodeSelector:
    kubernetes.io/os: linux

prometheus:
  server:
    nodeSelector:
      kubernetes.io/os: linux
  nodeExporter:
    enabled: true
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
grafana:
  nodeSelector:
    kubernetes.io/os: linux
```

## Metrics

* Collecting data about Windows nodes is supported by nOps
* Accurate node and pod data exists by default, since they come from the Kubernetes API.
* nOps requires cAdvisor for pod utilization data to determine costs at the container level.
* Currently, for pods on Windows nodes: pods will be billed based on request size.
