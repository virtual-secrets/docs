---
title: BucketClass
menu:
  docs_{{ .version }}:
    identifier: index-bucketclass
    name: BucketClass
    parent: crds
    weight: 2
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

# BucketClass

## What is BucketClass

A `BucketClass` in NooBaa is a custom resource that defines policies and classes for provisioning object buckets. It allows administrators to specify data placement strategies, replication policies, and the selection of backing stores for buckets. BucketClass enables advanced features like tiering, mirroring, and custom data management rules, providing flexibility and control over how data is stored and accessed.

By configuring BucketClasses, organizations can optimize storage usage, ensure data durability, and meet specific application requirements within a Kubernetes environment.

## BucketClass CRD Concept

The `BucketClass` Custom Resource Definition (CRD) in NooBaa is used to manage the configuration and policies for object buckets. It determines how data is distributed across backing stores, the replication strategy, and other bucket-level behaviors.

### Key Features
- Policy-based data placement and management
- Supports tiering, mirroring, and custom replication strategies
- Integrates with BackingStore and NamespaceStore resources
- Facilitates multi-cloud and hybrid storage scenarios

## BucketClass `Spec`
A `BucketClass` CRD has the following spec:
- **placementPolicy** - Defines how data is placed across backing stores (e.g., tiering, mirroring).
  - **tiers** - List of tiers for tiered placement, each containing backing stores.
  - **namespaceStores** - List of NamespaceStore resources used by the bucket class.
- **namespacePolicy** - Configuration for namespace-based data placement.
- **replicationPolicy** - Specifies replication rules for data durability and availability.
- **backingStores** - List of BackingStore resources used by the bucket class.

## BucketClass `Status`
The status section provides information about the health and readiness of the bucket class, including:
- **phase** - Current phase (e.g., Ready, Failed)
- **conditions** - List of status conditions (e.g., Available, Progressing)

## BucketClass CRD Specifications

```yaml
apiVersion: noobaa.io/v1alpha1
kind: BucketClass
metadata:
  creationTimestamp: "2025-09-19T06:11:21Z"
  finalizers:
    - noobaa.io/finalizer
  generation: 1
  labels:
    app: noobaa
  name: noobaa-default-bucket-class
  namespace: noobaa
  ownerReferences:
    - apiVersion: noobaa.io/v1alpha1
      blockOwnerDeletion: true
      controller: true
      kind: NooBaa
      name: noobaa
      uid: ad72436d-ed05-43db-927e-81b19b5a6507
  resourceVersion: "889837"
  uid: eb2156d2-d3fc-42c6-9602-d40f09106d7b
spec:
  placementPolicy:
    tiers:
      - backingStores:
          - noobaa-default-backing-store
status:
  conditions:
    - lastHeartbeatTime: "2025-09-22T09:34:46Z"
      lastTransitionTime: "2025-09-22T09:34:46Z"
      message: noobaa operator completed reconcile - bucket class is ready
      reason: BucketClassPhaseReady
      status: "True"
      type: Available
    - lastHeartbeatTime: "2025-09-22T09:34:46Z"
      lastTransitionTime: "2025-09-22T09:34:46Z"
      message: noobaa operator completed reconcile - bucket class is ready
      reason: BucketClassPhaseReady
      status: "False"
      type: Progressing
    - lastHeartbeatTime: "2025-09-22T09:34:46Z"
      lastTransitionTime: "2025-09-19T06:40:11Z"
      message: noobaa operator completed reconcile - bucket class is ready
      reason: BucketClassPhaseReady
      status: "False"
      type: Degraded
    - lastHeartbeatTime: "2025-09-22T09:34:46Z"
      lastTransitionTime: "2025-09-22T09:34:46Z"
      message: noobaa operator completed reconcile - bucket class is ready
      reason: BucketClassPhaseReady
      status: "True"
      type: Upgradeable
  mode: OPTIMAL
  phase: Ready
```