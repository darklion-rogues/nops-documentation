# Installing nOps with Rafay

[Rafay](https://rafay.co) is a SaaS-first Kubernetes Operations Platform (KOP) with enterprise-class scalability, zero-trust security and interoperability for managing applications across public clouds, data centers & edge.

See [Rafay documentation](https://docs.rafay.co/) to learn more about the platform and how to use it.

This document will walk you through installing nOps on a cluster that has been provisioned or imported using the Rafay controller. The steps below describe how to create and use a custom cluster blueprint via the [Rafay Web Console](https://console.rafay.dev/). The entire workflow can also be fully automated and embedded into an automation pipeline using the [RCTL CLI utility](https://docs.rafay.co/cli/overview/) or [Rafay REST APIs](https://docs.rafay.co/automation/api/apis/).

## Prerequisites

You have already [provisioned or imported](https://docs.rafay.co/learn/overview/) one or more Kubernetes clusters using the [Rafay controller](https://console.rafay.dev/).

## Step 1: Create a repository

Under _Integrations_:

* Select _Repositories_ and create a new repository named `nOps` of type _Helm._
* Select _Create._

![Create Repository](/images/rafay-nOps-repository-1.png)

* Enter the endpoint value of `https://nOps.github.io/cost-analyzer/`.
* Select _Save._

## Step 2: Customize values

You'll need to override the default _values.yaml_ file. Create a new file called _nOps-custom-values.yaml_ with the following content:

```yaml
# Custom values for nOps
reporting:
  valuesReporting: false
# Replace token with the value you get from nOps.com/install
# after entering your email address
nOpsToken: 'token_string'
```

## Step 3: Create a namespace

* Login to the [Rafay Web Console](https://console.rafay.dev/) and navigate to your Project as an _Org Admin_ or _Infrastructure Admin._
* Under _Infrastructure_, select _Namespaces_ and create a new namespace called `nOps`, and select type _Wizard._

![Create Namespace](/images/rafay-nOps-namespace-1.png)

* Select _Save & Go to Placement._
* Select the cluster(s) that the namespace will be added to. Select _Save & Go To Publish._
* Select _Publish_ to publish the namespace to the selected cluster(s).
* Once the namespace has been published, select _Exit._
* Under _Infrastructure_, select _Clusters._
* Select the _kubectl_ button on the cluster to open a virtual terminal.
* Verify that the `nOps` namespace has been created by running the following command:

```
$ kubectl get ns nOps

NAME            STATUS   AGE
nOps        Active   44m
```

## Step 4: Create an add-on

From the [Web Console](https://console.rafay.dev/):

* Select _Add-ons_ and _Create_ a new add-on called `nOps.`
* Select _Bring your own._
* Select _Helm 3_ for type.
* Select _Pull files from repository._
* Select _Helm_ for the repository type.
* Select `nOps` for the namespace.
* Select _Select._

![Create Addon](/images/rafay-nOps-addon-1.png)

* Create a new version of the add-on.
* Select _New Version._
* Provide a version name such as `v1`.
* Select `nOps` for the repository.
* Enter `cost-analyzer` for the chart name.
* Upload the `nOps-custom-values.yaml` file that was previously created.
* Select _Save Changes._

## Step 5: Create a blueprint

Once you've created the nOps add-on, use it in assembling a custom cluster blueprint. You can add other add-ons to the same custom blueprint.

* Under _Infrastructure_, select _Blueprints._
* Create a new blueprint and give it a name such as `nOps`.
* Select _Save._

![Create Blueprint](/images/rafay-nOps-blueprint-1.png)

* Create a new version of the blueprint.
* Select _New Version._
* Provide a version name such as `v1`.
* Under Add-Ons, select the `nOps` Add-on and the version that was previously created.
* Select _Save Changes._

![Create Blueprint](/images/rafay-nOps-blueprint-2.png)

## Step 6: Apply blueprint

You may now apply this custom blueprint to a cluster.

* Select _Options_ for the target cluster in the Web Console.
* Select _Update Blueprint_ and select the `nOps` blueprint and version you created previously.
* Select _Save and Publish._

![Update Blueprint](/images/rafay-nOps-blueprint-3.png)

This will start the deployment of the add-ons configured in the `nOps` blueprint to the targeted cluster. The blueprint sync process can take a few minutes. Once complete, the cluster will display the current cluster blueprint details and whether the sync was successful or not.

## Step 7: Verify deployment

You can optionally verify whether the correct resources have been created on the cluster. Select the `kubectl` button on the cluster to open a virtual terminal.

Then, verify the pods in the `nOps` namespace. Run `kubectl get pod -n nOps`, and check that the output is similar to the example below.

```
$ kubectl get pod -n nOps

NAME                                          READY   STATUS    RESTARTS   AGE
nOps-cost-analyzer-8544c4bbd4-gx4nl       3/3     Running   0          6m23s
nOps-grafana-768655466d-vlsmq             3/3     Running   0          6m23s
nOps-kube-state-metrics-f99c657b5-mh5mt   1/1     Running   0          6m23s
nOps-prometheus-node-exporter-26fwv       1/1     Running   0          6m23s
nOps-prometheus-node-exporter-zfkvw       1/1     Running   0          6m23s
nOps-prometheus-server-5cc6745978-z98f8   2/2     Running   0          6m23s
```

## Step 8: Enable port forwarding

In order to access the nOps UI, you'll need to enable access to the frontend application using [port-forward](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/). To do this, download and use the `Kubeconfig` with the KubeCTL CLI (`../../accessproxy/kubectl_cli/`).

```
kubectl port-forward --namespace nOps deployment/nOps-cost-analyzer 9090

Forwarding from 127.0.0.1:9090 -> 9090
Forwarding from [::1]:9090 -> 9090
Handling connection for 9090
```

## Step 9: View data

You can now access the nOps UI by visiting `http://localhost:9090` in your browser.

![nOps Dashboards](/images/rafay-nOps-view-1.png)

You have now successfully created a custom cluster blueprint with the `nOps` add-on and applied to a cluster. Use this blueprint on as many clusters as you require.

You can find [Rafay's documentation](https://docs.rafay.co/recipes/cost/nOps/) on nOps as well as guides for how to create or import a cluster using the Rafay controller on the [Rafay P](https://docs.rafay.co/clusters/overview/)[roduct Documentation](https://docs.rafay.co/clusters/overview/) site.