# nOps Data Status Metrics

## Overview

The metrics listed below are emitted by nOps and scraped by Prometheus to help monitor the status of nOps data pipelines:

1. `nOps_allocation_data_status`, which presents the active allocation data's time series status
2. `nOps_asset_data_status`, which presents the time series status of the active asset data

These metrics provide data status through to proactively alert and analyze the allocation and asset data at a point in time.

## Metric details

### Allocation metrics

The metrics below depict the status of active allocation data at a point in time. The resolution is either daily or hourly, which aligns one-to-one with the data status of allocation daily and hourly store. Each hourly and daily stores have four types of status

1. **Empty**: Depicts the total number of empty allocationSet in each store hourly or daily at a point in time.
2. **Error**: Depicts the total number of errors in the allocationSet in each store hourly or daily at a point in time.
3. **Success**: Depicts the total number of successful allocationSet in each store hourly or daily at a point in time.
4. **Warning**: Depicts the total number of warnings in all allocationSet in each store hourly or daily at a point in time.

![Prometheus metrics nOps_allocation_data_status after port-forwarding](/images/metric-nOps-allocation-data-status.png)

### Asset metrics

The metrics below depict the status of active asset data at a point in time. The resolution is either daily or hourly, which aligns one-to-one with the data status of asset daily and hourly store. Each hourly and daily stores have four types of status

1. **Empty**: Depicts the total number of empty assetSet in each store hourly or daily at a point in time.
2. **Error**: Depicts the total number of errors in the assetSet in each store hourly or daily at a point in time.
3. **Success**: Depicts the total number of successful assetSet in each store hourly or daily at a point in time.
4. **Warning**: Depicts the total number of warnings in all assetSet in each store hourly or daily at a point in time.

![Prometheus metrics nOps_asset_data_status after port-forwarding](/images/metric-nOps-asset-data-status.png)

## Usage details

* `nOps_asset_data_status` is written to Prometheus during the assetSet and assetLoad events.
* `nOps_allocation_data_status` is written to Prometheus during the allocationSet and allocationLoad events.
* During the cleanup operation, the corresponding entries of each allocation and asset are deleted to avoid the metrics having those particular entries having parity with respective allocation and asset stores.
