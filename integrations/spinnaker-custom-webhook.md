# Using Custom Webhook to Create a nOps Stage in Spinnaker

Adding the example webhook below to Spinnaker will enable a custom stage to query nOps for recommendations on a container. More info on [Spinnaker custom webhooks](https://spinnaker.io/guides/operator/custom-webhook-stages/#creating-a-custom-webhook-stage).

{% code overflow="wrap" %}
```
webhook:
  preconfigured:
  - label: "nOps: Get Sizing"
    type: getRequestSizing
    enabled: true
    description: Custom stage to get request sizing for a running container
    method: GET
    url: "${parameterValues['nOps_url']}//model/savings/requestSizing?algorithm=max-headroom&window=${parameterValues['time_window']}&targetCPUUtilization=${parameterValues['target_cpu_utilization']}&targetRAMUtilization=${parameterValues['target_ram_utilization']}&filterContainers=${parameterValues['container_name']}&filterControllers=${parameterValues['controller_name']}&filterNamespaces=${parameterValues['namespace']}"
    parameters:
      - label: "nOps API URL"
        name: nOps_url
        description: "Fully qualified Url to the requestSizing api"
        type: string
      - label: "Controller Name"
        name: controller_name
        description: "Name of the controller"
        type: string
      - label: "Container Name"
        name: container_name
        description: "Name of the container within the deployment"
        type: string
      - label: "Namespace"
        name: namespace
        description: "Namespace where controller and container are running"
        type: string
      - label: "Target CPU Utilization"
        name: target_cpu_utilization
        description: "Target CPU utilization for the recommendation"
        type: string
        defaultValue: 0.65
      - label: "Target RAM Utilization"
        name: target_ram_utilization
        description: "Target RAM utilization for the recommendation"
        type: string
        defaultValue: 0.65
      - label: "Time Window"
        name: time_window
        description: "Time window to look back to build recommendation [format: xd]"
        type: string
        defaultValue: "7d"
```
{% endcode %}
