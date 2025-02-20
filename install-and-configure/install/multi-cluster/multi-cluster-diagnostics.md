# Multi-Cluster Diagnostics

{% hint style="info" %}
This feature is currently in beta. It is enabled by default.
{% endhint %}

Multi-Cluster Diagnostics offers a single view into the health of all the clusters you currently monitor with nOps.

Health checks include, but are not limited to:

1. Whether nOps is correctly emitting metrics
2. Whether nOps is being scraped by Prometheus
3. Whether Prometheus has scraped the required metrics
4. Whether nOps's ETL files are healthy

## Configuration

```yaml
# This is an abridged example. Full example in link below.
diagnostics:
  enabled: true
  primary:
    enabled: true  # Only enable this on your primary nOps cluster

# Ensure you have configured a unique CLUSTER_ID.
prometheus:
  server:
    global:
      external_labels:
        cluster_id: YOUR_CLUSTER_ID
nOpsProductConfigs:
  clusterName: YOUR_CLUSTER_ID

# Ensure you have configured a storage config secret.
nOpsModel:
  federatedStorageConfigSecret: federated-store
```

Additional configuration options can found in the [*values.yaml*](https://github.com/nOps/cost-analyzer-helm-chart/blob/develop/cost-analyzer/values.yaml) under `diagnostics:`.

## Architecture

The Multi-Cluster Diagnostics feature is a process run within the `nOps-cost-analyzer` deployment. It has the option to be run as an independent deployment for higher availability via `.Values.diagnostics.deployment.enabled`.

When run in each nOps deployment, it monitors the health of nOps and sends that health data to the central object store at the `/diagnostics` filepath. The below diagram depicts these interactions. This diagram is specific to the requests required for diagnostics only. For additional diagrams, see our [multi-cluster guide](multi-cluster.md).

![nOps-Agent-Diagnostics](/images/diagrams/Agent-Diagnostics-Architecture.png)

## API usage

The diagnostics API can be accessed on the `primary` via `/model/diagnostics/multicluster?window=1d`.

The `window` query parameter is required, which will return all diagnostics within the specified time window.

{% swagger method="get" path="/multi-cluster-diagnostics" baseUrl="http://<your-nOps-address>/model" summary="Multi-cluster Diagnostics API" %}
{% swagger-description %}
The Multi-cluster Diagnostics API provides a single view into the health of all the clusters you currently monitor with nOps.
{% endswagger-description %}

{% swagger-parameter in="path" name="window" type="string" required="true" %}
Duration of time over which to query. Accepts words like `today`, `week`, `month`, `yesterday`, `lastweek`, `lastmonth`; durations like `30m`, `12h`, `7d`; comma-separated RFC3339 date pairs like `2021-01-02T15:04:05Z,2021-02-02T15:04:05Z`; comma-separated Unix timestamp (seconds) pairs like `1578002645,1580681045`.
{% endswagger-parameter %}

{% swagger-response status="200: OK" description="" %}
```json
{
    "code": 200,
    "data": {
        "overview": {
            "nOpsEmittingMetricDiagnosticPassed": true,
            "prometheusHasnOpsMetricDiagnosticPassed": true,
            "prometheusHasCadvisorMetricDiagnosticPassed": true,
            "prometheusHasKSMMetricDiagnosticPassed": true,
            "dailyAllocationEtlHealthyDiagnosticPassed": true,
            "dailyAssetEtlHealthyDiagnosticPassed": true,
            "nOpsPodsNotOOMKilledDiagnosticPassed": true,
            "nOpsPodsNotPendingDiagnosticPassed": false
        },
        "clusters": [
            {
                "clusterId": "cluster_one",
                "latestRun": "2024-03-01T22:42:32Z",
                "nOpsVersion": "v2.1",
                "nOpsEmittingMetric": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "prometheusHasnOpsMetric": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "prometheusHasCadvisorMetric": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "prometheusHasKSMMetric": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "dailyAllocationEtlHealthy": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "dailyAssetEtlHealthy": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "nOpsPodsNotOOMKilled": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "nOpsPodsNotPending": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                }
            },
            {
                "clusterId": "cluster_two",
                "latestRun": "2024-03-01T22:40:17Z",
                "nOpsVersion": "v2.1",
                "nOpsEmittingMetric": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "prometheusHasnOpsMetric": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "prometheusHasCadvisorMetric": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "prometheusHasKSMMetric": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "dailyAllocationEtlHealthy": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "dailyAssetEtlHealthy": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "nOpsPodsNotOOMKilled": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "nOpsPodsNotPending": {
                    "diagnosticPassed": false,
                    "numFailures": 52,
                    "firstFailureDate": "2024-03-01T18:25:09Z",
                    "diagnosticOutput": "RunDiagnostic: checknOpsPodsNotPending: queryPrometheusCheckResultEmpty: the following query returned a non-empty result sum(kube_pod_status_phase{namespace='nOps-etl-fed', phase='Pending'}) by (pod,namespace) > 0"
                }
            },
            {
                "clusterId": "cluster_three",
                "latestRun": "2024-03-01T22:42:32Z",
                "nOpsVersion": "v2.1",
                "nOpsEmittingMetric": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "prometheusHasnOpsMetric": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "prometheusHasCadvisorMetric": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "prometheusHasKSMMetric": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "dailyAllocationEtlHealthy": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "dailyAssetEtlHealthy": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "nOpsPodsNotOOMKilled": {
                    "diagnosticPassed": true,
                    "numFailures": 0,
                    "firstFailureDate": "",
                    "diagnosticOutput": ""
                },
                "nOpsPodsNotPending": {
                    "diagnosticPassed": false,
                    "numFailures": 52,
                    "firstFailureDate": "2024-03-01T18:24:42Z",
                    "diagnosticOutput": "RunDiagnostic: checknOpsPodsNotPending: queryPrometheusCheckResultEmpty: the following query returned a non-empty result sum(kube_pod_status_phase{namespace='nOps-etl-fed', phase='Pending'}) by (pod,namespace) > 0"
                }
            }
        ]
    }
}
```
{% endswagger-response %}
{% endswagger %}
