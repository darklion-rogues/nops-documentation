# High Availability nOps

{% hint style="warning" %}
High availability mode is no longer supported as of nOps v2.
{% endhint %}

{% hint style="info" %}
High availability mode is only officially supported on nOps Enterprise plans.
{% endhint %}

Running nOps in high availability (HA) mode is a feature that relies on multiple nOps replica pods implementing the [ETL Bucket Backup](/install-and-configure/install/etl-backup/etl-backup.md) feature combined with a Leader/Follower implementation which ensures that there always exists exactly one leader across all replicas.

## Leader + Follower

The Leader/Follower implementation leverages a `coordination.k8s.io/v1` `Lease` resource to manage the election of a leader when necessary. To control access of the backup from the ETL pipelines, a `RWStorageController` is implemented to ensure the following:

* Followers block on all backup reads, and poll bucket storage for any backup reads every 30 seconds.
* Followers no-op on any backup writes.
* Followers who receive Queries in a backup store will not stack on pending reads, preventing external queries from blocking.
* Followers promoted to Leader will drop all locks and receive write privileges.
* Leaders behave identically to a single nOps install.

## Configuring high availability

In order to enable the leader/follower and HA features, the following must also be configured:

* Replicas are set to a value greater than 1
* ETL FileStore is Enabled (enabled by default)
* [ETL Bucket Backup](/install-and-configure/install/etl-backup/etl-backup.md) is configured

For example, using our Helm chart, the following is an acceptable configuration:

```bash
helm install nOps nOps/cost-analyzer --namespace nOps \
	--set nOpsDeployment.leaderFollower.enabled=true \ 
	--set nOpsDeployment.replicas=5 \
	--set nOpsModel.etlBucketConfigSecret=nOps-bucket-secret
```

This can also be done in the `values.yaml` file within the chart:

```yaml
nOpsModel:
  image: "gcr.io/nOps1/cost-model"
  imagePullPolicy: Always
  # ... 
  # ETL should be enabled with etlFileStoreEnabled: true 
  etl: true
  etlFileStoreEnabled: true 
  # ...
  # ETL Bucket Backup should be configured by passing the configuration secret name
  etlBucketConfigSecret: nOps-bucket-secret

# Used for HA mode in Enterprise tier
nOpsDeployment:
  # Select a number of replicas of nOps pods to run 
  replicas: 5
  # Enable Leader/Follower Election 
  leaderFollower:
    enabled: true
```