---
title: BackingStore
menu:
  docs_{{ .version }}:
    identifier: index-backingstore
    name: BackingStore
    parent: crds
    weight: 1
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

# BackingStore

## What is BackingStore

A `BackingStore` in `NooBaa` is a custom resource that defines a storage backend for object buckets. It acts as an abstraction layer, allowing NooBaa to connect to various storage types such as cloud providers (AWS S3, Azure Blob, Google Cloud Storage), local persistent volumes, or namespace-based stores. BackingStores provide the actual storage capacity for buckets managed by NooBaa and enable features like dynamic scaling, high availability, and policy-based data placement.

By configuring `BackingStores`, administrators can flexibly manage where and how data is stored, replicated, and accessed within a Kubernetes environment.

## BackingStore CRD Concept

The `BackingStore` Custom Resource Definition (CRD) in NooBaa defines a storage backend for object buckets. It allows NooBaa to connect and manage different types of storage resources, such as cloud storage, local PV pools, or namespace stores. BackingStores are essential for providing the actual storage capacity behind buckets managed by NooBaa.

### Key Features
- Supports multiple storage types: AWS S3, Azure Blob, Google Cloud Storage, PVC pools, and more
- Enables dynamic scaling and management of storage resources
- Provides health and status monitoring for each backing store
- Allows for high availability and redundancy by combining multiple stores
- Integrates with bucket classes for policy-based data placement

## BackingStore `Spec`
A `BackingStore` CRD has the following spec:
- **type** - The type of the backing store (e.g., S3, Azure, Google, pv-pool).
- **secret** - Reference to a Kubernetes Secret containing credentials for accessing the backing store. Required for cloud providers and some namespace stores.
- **endpoint** - The endpoint URL for cloud or remote storage backends (e.g., S3 endpoint, Azure Blob URL).
- **targetBucket** - The bucket name in the external storage system (for cloud providers like S3, Azure, Google).
- **pvPool** - Configuration for local persistent volume pools. Includes number of volumes, resource requests/limits, and secret for access.

This flexible spec allows NooBaa to support a wide range of storage backends and deployment scenarios.

## BackingStore `Status`
The status section provides information about the health, phase, and mode of the backing store, including:
- **Available** - Indicates if the backing store is ready for use.
- **Degraded** - Indicates issues or reduced capacity.
- **mode** - Current operational mode (e.g., LOW_CAPACITY).
- **phase** - Current phase (e.g., Ready).

## BackingStore CRD Specifications
Like any other CRD, the `BackingStore` CRD has a spec section.

A sample `BackingStore` object for a `pvPool` is shown below:

```yaml
apiVersion: noobaa.io/v1alpha1
kind: BackingStore
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"noobaa.io/v1alpha1","kind":"BackingStore","metadata":{"annotations":{},"name":"noobaa-default-backing-store","namespace":"noobaa"},"spec":{"pvPool":{"numVolumes":1,"resources":{"limits":{"cpu":"1000m","memory":"2Gi"},"requests":{"cpu":"1000m","memory":"2Gi","storage":"16Gi"}},"secret":{}},"type":"pv-pool"}}
  creationTimestamp: "2025-09-19T06:38:01Z"
  finalizers:
  - noobaa.io/finalizer
  generation: 2
  labels:
    app: noobaa
  name: noobaa-default-backing-store
  namespace: noobaa
  resourceVersion: "888587"
  uid: a4d22c56-c9f0-459f-8c22-9f546714ed2b
spec:
  pvPool:
    numVolumes: 1
    resources:
      limits:
        cpu: "1"
        memory: 2Gi
      requests:
        cpu: "1"
        memory: 2Gi
        storage: 16Gi
    secret: {}
  type: pv-pool
status:
  conditions:
  - lastHeartbeatTime: "2025-09-22T09:22:27Z"
    lastTransitionTime: "2025-09-22T09:22:27Z"
    message: BackingStorePhaseReady
    reason: 'Backing store mode: LOW_CAPACITY'
    status: "True"
    type: Available
  - lastHeartbeatTime: "2025-09-22T09:22:27Z"
    lastTransitionTime: "2025-09-22T09:22:27Z"
    message: BackingStorePhaseReady
    reason: 'Backing store mode: LOW_CAPACITY'
    status: "False"
    type: Progressing
  - lastHeartbeatTime: "2025-09-22T09:22:27Z"
    lastTransitionTime: "2025-09-19T06:40:11Z"
    message: BackingStorePhaseReady
    reason: 'Backing store mode: LOW_CAPACITY'
    status: "False"
    type: Degraded
  - lastHeartbeatTime: "2025-09-22T09:22:27Z"
    lastTransitionTime: "2025-09-22T09:22:27Z"
    message: BackingStorePhaseReady
    reason: 'Backing store mode: LOW_CAPACITY'
    status: "True"
    type: Upgradeable
  mode:
    modeCode: LOW_CAPACITY
    timeStamp: 2025-09-19 06:40:11.207848024 +0000 UTC m=+2288.042493713
  phase: Ready
```