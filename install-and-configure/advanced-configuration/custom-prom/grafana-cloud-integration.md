# Grafana Cloud Integration for nOps

[Grafana Cloud](https://grafana.com/products/cloud/) is a composable observability platform, integrating metrics, traces and logs with Grafana. Customers can leverage the best open source observability software without the overhead of installing, maintaining, and scaling your observability stack.

This document will show you how to integrate the Grafana Cloud Prometheus metrics service with nOps.

## Prerequisites

* You have access to a running Kubernetes cluster
* You have created a Grafana Cloud account
* You have permissions to create Grafana Cloud API keys

## Step 1: Install the Grafana Agent on your cluster

Install the Grafana Agent for Kubernetes on your cluster. On the existing K8s cluster that you intend to install nOps, run the following commands to install the Grafana Agent to scrape the metrics from nOps `/metrics` endpoint. The script below installs the Grafana Agent with the necessary scraping configuration for nOps, you may want to add additional scrape configuration for your setup. Please remember to replace the following values with your actual Grafana cloud's values:

* `REPLACE-WITH-GRAFANA-PROM-REMOTE-WRITE-ENDPOINT`
* `REPLACE-WITH-GRAFANA-PROM-REMOTE-WRITE-USERNAME`
* `REPLACE-WITH-GRAFANA-PROM-REMOTE-WRITE-API-KEY`
* `REPLACE-WITH-YOUR-CLUSTER-NAME`

{% code overflow="wrap" %}
```
cat <<'EOF' |

kind: ConfigMap
metadata:
  name: grafana-agent
apiVersion: v1
data:
  agent.yaml: |
    metrics:
      wal_directory: /var/lib/agent/wal
      global:
        scrape_interval: 60s
        external_labels:
          cluster: <REPLACE-WITH-YOUR-CLUSTER-NAME>
      configs:
      - name: integrations
        remote_write:
        - url: https://<REPLACE-WITH-GRAFANA-PROM-REMOTE-WRITE-ENDPOINT>
          basic_auth:
            username: <REPLACE-WITH-GRAFANA-PROM-REMOTE-WRITE-USERNAME>
            password: <REPLACE-WITH-GRAFANA-PROM-REMOTE-WRITE-API-KEY>
        scrape_configs: #Need further scrape config update
        - job_name: nOps
          honor_labels: true
          scrape_interval: 1m
          scrape_timeout: 10s
          metrics_path: /metrics
          scheme: http
          dns_sd_configs:
          - names:
            - nOps-cost-analyzer.nOps
            type: 'A'
            port: 9003
        - job_name: nOps-networking
          kubernetes_sd_configs:
            - role: pod
          relabel_configs:
          # Scrape only the the targets matching the following metadata
            - source_labels: [__meta_kubernetes_pod_label_app]
              action: keep
              regex:  'nOps-network-costs'
  
EOF
(export NAMESPACE=nOps && kubectl apply -n $NAMESPACE -f -)

MANIFEST_URL=https://raw.githubusercontent.com/grafana/agent/v0.24.2/production/kubernetes/agent-bare.yaml NAMESPACE=nOps /bin/sh -c "$(curl -fsSL https://raw.githubusercontent.com/grafana/agent/v0.24.2/production/kubernetes/install-bare.sh)" | kubectl apply -f -
```
{% endcode %}

You can also verify if `grafana-agent` is scraping data with the following command  (optional):

```
kubectl -n nOps logs grafana-agent-0
```

To learn more about how to install and config Grafana agent as well as additional scrape configuration, please refer to [Grafana Agent](https://grafana.com/docs/tempo/latest/configuration/grafana-agent/) documentation or you can check nOps Prometheus scrape config at this [GitHub repository](https://github.com/nOps/cost-analyzer-helm-chart/blob/ebe7e088debecd23f90e6dd75b425828901a246c/cost-analyzer/charts/prometheus/values.yaml#L1152).

## Step 2: Create `dbsecret` to allow nOps to query the metrics from Grafana Cloud Prometheus:

Create two files in your working directory, called `USERNAME` and `PASSWORD` respectively

```
export PASSWORD=<REPLACE-WITH-GRAFANA-PROM-REMOTE-WRITE-API-KEY>
export USERNAME=<REPLACE-WITH-GRAFANA-PROM-REMOTE-WRITE-USERNAME>
printf "${PASSWORD}" > PASSWORD
printf "${USERNAME}" > USERNAME
```

Verify that you can run queries against your Grafana Cloud Prometheus query endpoint (optional):

{% code overflow="wrap" %}
```
cred="$( echo $NAME:$PASSWORD | base64 )"; curl -H "Authorization: Basic $cred" https://<REPLACE-WITH-GRAFANA-PROM-QUERY-ENDPOINT>/api/v1/query?query=up
```
{% endcode %}

Create K8s secret name `dbsecret`:

```
kubectl create secret generic dbsecret \
    --namespace nOps \
    --from-file=USERNAME \
    --from-file=PASSWORD
```

Verify if the credentials appear correctly (optional):

```
kubectl -n nOps get secret dbsecret -o json | jq '.data | map_values(@base64d)'
```

## Step 3 (optional): Configure nOps recording rules for Grafana Cloud using Cortextool

To set up recording rules in Grafana Cloud, download the [Cortextool CLI utility](https://github.com/grafana/cortex-tools). While they are optional, they offer improved performance.

After installing the tool, create a file called _nOps\_rules.yaml_ with the following command:

{% code overflow="wrap" %}
```yaml
cat << EOF > nOps-rules.yaml
namespace: "nOps"
groups:
  - name: CPU
    rules:
      - expr: sum(rate(container_cpu_usage_seconds_total{container_name!=""}[5m]))
        record: cluster:cpu_usage:rate5m
      - expr: rate(container_cpu_usage_seconds_total{container_name!=""}[5m])
        record: cluster:cpu_usage_nosum:rate5m
      - expr: avg(irate(container_cpu_usage_seconds_total{container_name!="POD", container_name!=""}[5m])) by (container_name,pod_name,namespace)
        record: nOps_container_cpu_usage_irate
      - expr: sum(container_memory_working_set_bytes{container_name!="POD",container_name!=""}) by (container_name,pod_name,namespace)
        record: nOps_container_memory_working_set_bytes
      - expr: sum(container_memory_working_set_bytes{container_name!="POD",container_name!=""})
        record: nOps_cluster_memory_working_set_bytes
  - name: Savings
    rules:
      - expr: sum(avg(kube_pod_owner{owner_kind!="DaemonSet"}) by (pod) * sum(container_cpu_allocation) by (pod))
        record: nOps_savings_cpu_allocation
        labels:
          daemonset: "false"
      - expr: sum(avg(kube_pod_owner{owner_kind="DaemonSet"}) by (pod) * sum(container_cpu_allocation) by (pod)) / sum(kube_node_info)
        record: nOps_savings_cpu_allocation
        labels:
          daemonset: "true"
      - expr: sum(avg(kube_pod_owner{owner_kind!="DaemonSet"}) by (pod) * sum(container_memory_allocation_bytes) by (pod))
        record: nOps_savings_memory_allocation_bytes
        labels:
          daemonset: "false"
      - expr: sum(avg(kube_pod_owner{owner_kind="DaemonSet"}) by (pod) * sum(container_memory_allocation_bytes) by (pod)) / sum(kube_node_info)
        record: nOps_savings_memory_allocation_bytes
        labels:
          daemonset: "true"
EOF
```
{% endcode %}

Then, make sure you are in the same directory as your `_nOps\_rules.yaml_`, and load the rules using Cortextool. Replace the address with your Grafana Cloud’s Prometheus endpoint (Remember to omit the /api/prom path from the endpoint URL).

```
cortextool rules load \
--address=<REPLACE-WITH-GRAFANA-PROM-ENDPOINT> \
--id=<REPLACE-WITH-GRAFANA-PROM-REMOTE-WRITE-USERNAME> \
--key=<REPLACE-WITH-GRAFANA-PROM-REMOTE-WRITE-API-KEY> \
nOps_rules.yaml
```

Print out the rules to verify that they’ve been loaded correctly:

```
cortextool rules print \
--address=<REPLACE-WITH-GRAFANA-PROM-ENDPOINT> \
--id=<REPLACE-WITH-GRAFANA-PROM-REMOTE-WRITE-USERNAME> \
--key=<REPLACE-WITH-GRAFANA-PROM-REMOTE-WRITE-API-KEY>
```

## Step 4: Install nOps on the cluster

Install nOps on your K8s cluster with Grafana Cloud Prometheus query endpoint and `dbsecret` you created in Step 2.

```
helm upgrade -i -n nOps nOps nOps/cost-analyzer \
    --namespace nOps \
    --set global.prometheus.fqdn=https://<REPLACE-WITH-GRAFANA-PROM-QUERY-ENDPOINT> \
    --set global.prometheus.enabled=false \
    --set global.prometheus.queryServiceBasicAuthSecretName=dbsecret
```

The process is complete. By now, you should have successfully completed the nOps integration with Grafana Cloud.

Optionally, you can also add our [nOps Dashboard for Grafana Cloud](https://grafana.com/grafana/dashboards/15714) to your organization to visualize your cloud costs in Grafana.
