---
title: ObjectBucket
menu:
  docs_{{ .version }}:
    identifier: index-objectbucket
    name: ObjectBucket
    parent: crds
    weight: 1
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

# ObjectBucket

## What is ObjectBucket

An `ObjectBucket` in NooBaa is a custom resource that represents a provisioned object bucket within a Kubernetes cluster. It is automatically created and managed by the NooBaa operator when an ObjectBucketClaim is processed. The ObjectBucket resource contains metadata, connection information, and links the bucket to its backing storage, enabling applications to access object storage through Kubernetes-native APIs.

By using ObjectBucket, administrators and users can integrate object storage into their Kubernetes workflows, leveraging NooBaa's abstraction and automation capabilities.

## ObjectBucket CRD Concept

The `ObjectBucket` Custom Resource Definition (CRD) in NooBaa manages the lifecycle and metadata of object buckets. It stores connection details, links to claims, and references to backing stores or bucket classes.

### Key Features
- Represents a provisioned object bucket in Kubernetes
- Stores connection details and metadata for the bucket
- Used internally for bucket lifecycle management

## ObjectBucket `Spec`
An `ObjectBucket` CRD has the following spec:
- **additionalState** - Contains metadata such as account, bucket class, and generation
- **claimRef** - Reference to the ObjectBucketClaim that requested this bucket
- **endpoint** - Connection details for accessing the bucket (host, port, bucket name, region)
- **reclaimPolicy** - Policy for resource cleanup (e.g., Delete, Retain)
- **storageClassName** - The storage class used for the bucket

## ObjectBucket `Status`
The status section provides information about the phase and readiness of the object bucket, including:
- **phase** - Current phase (e.g., Bound)

## ObjectBucket CRD Specifications

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucket
metadata:
  name: obc-noobaa-my-app-bucket
  namespace: noobaa
spec:
  additionalState:
    account: obc-account.my-app-bucket-e84d8c0b-5e5a-45fc-aa32-d3d6ee85fa91.68ccfaf6@noobaa.io
    bucketclass: noobaa-default-bucket-class
    bucketclassgeneration: "1"
  claimRef:
    apiVersion: objectbucket.io/v1alpha1
    kind: ObjectBucketClaim
    name: my-app-bucket
    namespace: noobaa
    uid: 5e923a6d-0e39-4a6f-bf10-d09daea2c0da
  endpoint:
    additionalConfig: {}
    bucketHost: s3.noobaa.svc
    bucketName: my-app-bucket-e84d8c0b-5e5a-45fc-aa32-d3d6ee85fa91
    bucketPort: 443
    region: ""
    subRegion: ""
  reclaimPolicy: Delete
  storageClassName: noobaa.noobaa.io
status:
  phase: Bound
```