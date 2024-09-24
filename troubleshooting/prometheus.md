Running a Query in nOps-Bundled Prometheus
==============================================

## Step 1: Connect to Prometheus

Here is an example command to connect if you've installed nOps in the nOps namespace:

```
kubectl port-forward -n nOps service/nOps-prometheus-server 9003:80
```

## Step 2: Visit Prometheus UI

View `http://localhost:9003/` in your web browser. You should be presented with a UI that looks like the following:

![Prometheus query UI](/images/prom-ui.png)

If you're unable to connect, confirm that the Prometheus server pod is in a `Running` state.

## Step 3: Input your desired query and execute
