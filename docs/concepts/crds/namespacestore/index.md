---
title: NamespaceStore
menu:
  docs_{{ .version }}:
    identifier: index-namespacestore
    name: NamespaceStore
    parent: crds
    weight: 1
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

# NamespaceStore

## What is NamespaceStore

A `NamespaceStore` in NooBaa is a custom resource that represents an external namespace-based storage backend. It allows NooBaa to connect to and manage data in external object stores, such as AWS S3, Azure Blob, Google Cloud Storage, or other S3-compatible services. 
By configuring NamespaceStores, administrators can extend NooBaa's data management capabilities to external cloud or on-premises object stores, supporting hybrid and multi-cloud scenarios.

## NamespaceStore CRD Concept

The `NamespaceStore` Custom Resource Definition (CRD) in NooBaa defines the configuration and connection details for external namespace-based storage backends. It is used in conjunction with BucketClass to manage data placement and access policies.

### Key Features
- Connects to external object stores (S3, Azure Blob, Google Cloud Storage, etc.)
- Supports hybrid and multi-cloud data management
- Enables data migration and federation
- Integrates with BucketClass for policy-based data placement
- Provides connection and credential management

## NamespaceStore `Spec`
A `NamespaceStore` CRD has the following spec:
- **type** - The type of the namespace store (e.g., awsS3, azureBlob, googleCloudStorage, s3Compatible).
- **secret** - Reference to a Kubernetes Secret containing credentials for accessing the namespace store.
- **endpoint** - The endpoint URL for the external object store.
- **targetBucket** - The bucket name in the external storage system.

## NamespaceStore `Status`
The status section provides information about the health and readiness of the namespace store, including:
- **phase** - Current phase (e.g., Ready, Failed)
- **conditions** - List of status conditions (e.g., Available, Progressing)

## NamespaceStore CRD Specifications

```yaml
apiVersion: noobaa.io/v1alpha1
kind: NamespaceStore
metadata:
  name: example-namespace-store
  namespace: noobaa
spec:
  type: s3-compatible
  endpoint: https://s3.example.com
  targetBucket: example-bucket
  secret:
    name: s3-credentials
    namespace: noobaa
status:
  phase: Ready
  conditions:
    - type: Available
      status: "True"
      reason: NamespaceStorePhaseReady
      message: NamespaceStore is ready
```