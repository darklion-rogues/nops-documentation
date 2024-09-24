# Installation nOps with Istio (Rancher)

The following requirements are given:

* Rancher with default monitoring
* Use of an existing Prometheus and Grafana (nOps will be installed without Prometheus and Grafana)
* Istio with gateway and sidecar for deployments

## Activation of Istio

1. Istio is activated by editing the namespace. To do this, execute the command `kubectl edit namespace nOps` and insert the label `istio-injection: enabled`
2. After Istio has been activated, some adjustments must be made to the deployment with `kubectl -n nOps edit deployment nOps-cost-analyzer` to allow communication within the namespace. For example, the healtch-check is completed successfully. When editing the deployment, the two annotations must be added:

```
annotations:
	traffic.sidecar.istio.io/excludeOutboundIPRanges: "10.43.0.1/32"
	sidecar.istio.io/rewriteAppHTTPProbers: "true"
```

## Authorization polices

An authorization policy governs access restrictions in namespaces and specifies how resources within a namespace are allowed to access it.

### ap-ingress: communication with Istio

```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: ap-ingress
  namespace: nOps
spec:
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account
```

### ap-intern: communication with nOps

```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: ap-intern
  namespace: nOps
spec:
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/nOps/sa/nOps-cost-analyzer
```

### ap-extern: as a port share (9003) for communication from Prometheus (namespace "cattle-monitoring-system") to nOps (namespace "nOps")

```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: ap-extern
  namespace: nOps
spec:
  rules:
  - to:
    - operation:
        ports:
          - "9003"
```

## Peer Authentication

Peer authentication is used to set how traffic is tunneled to the Istio sidecar. In the example, enforcing TLS is disabled so that Prometheus can grab the metrics from nOps (if this action is not performed, it returns at HTTP 503 error).

### pa-default.yaml

```
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: pa-default
  namespace: nOps
spec:
  mtls:
    mode: PERMISSIVE
```

## Destination Rule

A destination rule is used to specify how traffic should be handled after routing to a service. In my case, TLS is disabled for connections from nOps to Prometheus and Grafana (namespace "cattle-monitoring-system").

### dr-prometheus.yaml

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-prometheus
  namespace: nOps
spec:
  host: rancher-monitoring-prometheus.cattle-monitoring-system.svc.cluster.local
  trafficPolicy:
    tls:
      mode: DISABLE
```

### dr-grafana.yaml

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-grafana
  namespace: nOps
spec:
  host: rancher-monitoring-grafana.cattle-monitoring-system.svc.cluster.local
  trafficPolicy:
    tls:
      mode: DISABLE
```

## Virtual Service

A virtual service is used to direct data traffic specifically to individual services within the service mesh. The virtual service defines how the routing should run. A gateway is required for a virtual service.

### vs-nOps.yaml

```
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  labels:
    cattle.io/creator: norman
  name: vs-nOps
  namespace: nOps
spec:
  gateways:
  - ${gateway}
  hosts:
  - ${host}
  http:
  - match:
    - uri:
        prefix: /nOps
    rewrite:
      uri: /
    route:
    - destination:
        host: nOps-cost-analyzer.nOps.svc.cluster.local
        port:
          number: 9090
```

After creating the virtual service, nOps should be accessible at the URL `http(s)://${gateway}/nOps/`.

## Troubleshooting

### nOps is displaying excessively high cost metrics in the billions

Setting the Helm flag `customPricesEnabled` to `true`, or enabling [Custom Pricing](/architecture/pricing-sources-matrix.md#custom-pricing) through the UI has been shown to fix an error where Rancher clusters displayed absurdly high cost metrics.
