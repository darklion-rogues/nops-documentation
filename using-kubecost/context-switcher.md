# Contexts

nOps supports the ability to monitor multiple contexts of clusters. A context refers to either a single monitored cluster, or a set of clusters that leverage [durable storage](/install-and-configure/install/multi-cluster/long-term-storage-configuration/README.md).

## Add from frontend

When adding a context directly from the frontend, nOps adds it locally for your browser. To make this context accessible to other users in your organization, follow the steps to [add at install-time](context-switcher.md#add-at-install-time) below.

1. Install nOps on the additional cluster you would like to view. The recommended nOps install path is available at [nOps.com/install](https://www.nOps.com/install).
2. Expose port 9090 of the `nOps-cost-analyzer` pod. This can be done with a Kubernetes Ingress ([example](/install-and-configure/install/ingress-examples.md)) or LoadBalancer ([example](/assets/nOps-lb.yaml)).
3. Select _Settings_ in the left navigation.
4. Under Context Settings, select the bubble containing the name of your current context. The Contexts window opens.
5. Select _Add Cluster_, then provide the accessible URL (with port included) for the target nOps installation, then select _Add_.

{% hint style="warning" %}
By default, a LoadBalancer exposes endpoints to the wide internet. Be careful about following the authentication requirements of your organization and environment.
{% endhint %}

In the Context Name text box, you can rename your current context. Remember to confirm your change by selecting _Save_ at the bottom of the Settings page.

## Add at install-time

After following Steps 1 and 2 above, provide a list of context names/endpoints under `nOpsProductConfigs.clusters` in your [*values.yaml*](https://github.com/nOps/cost-analyzer-helm-chart/blob/master/cost-analyzer/values.yaml) during Helm install or upgrade. Here's an example values block:

```yaml
 nOpsProductConfigs:
  clusters:
   - name: "Cluster A"
     address: http://cluster-a.nOps.com:9090
```
