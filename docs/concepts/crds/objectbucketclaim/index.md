---
title: ObjectBucketClaim
menu:
  docs_{{ .version }}:
    identifier: index-objectbucketclaim
    name: ObjectBucketClaim
    parent: crds
    weight: 1
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

# ObjectBucketClaim

## What is ObjectBucketClaim

An `ObjectBucketClaim` in NooBaa is a custom resource that allows users or applications to request the creation of an object bucket within a Kubernetes cluster. When an ObjectBucketClaim is created, the NooBaa operator provisions an ObjectBucket and provides connection details for accessing the bucket. This resource enables Kubernetes-native workflows for object storage provisioning and access.

By using ObjectBucketClaim, users can dynamically request and manage object buckets as part of their application deployments.

## ObjectBucketClaim CRD Concept

The `ObjectBucketClaim` Custom Resource Definition (CRD) in NooBaa manages the request and lifecycle of object buckets. It specifies the desired bucket name, storage class, and access policies, and links to the provisioned ObjectBucket resource.

### Key Features
- Allows users to request object buckets via Kubernetes APIs
- Automates bucket provisioning and connection management
- Integrates with ObjectBucket, BackingStore, and BucketClass resources
- Supports dynamic and declarative object storage workflows

## ObjectBucketClaim `Spec`
An `ObjectBucketClaim` CRD has the following spec:
- **bucketName** - The desired name of the object bucket
- **storageClassName** - The storage class to use for provisioning the bucket
- **additionalConfig** - Optional configuration for bucket access
- **generateBucketName** - Option to auto-generate a bucket name

## ObjectBucketClaim `Status`
The status section provides information about the phase and readiness of the claim, including:
- **phase** - Current phase (e.g., Bound, Pending)
- **objectBucketName** - Name of the provisioned ObjectBucket

## ObjectBucketClaim CRD Specifications

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: my-app-bucket
  namespace: noobaa
spec:
  bucketName: my-app-bucket
  storageClassName: noobaa.noobaa.io
  additionalConfig: {}
status:
  phase: Bound
  objectBucketName: obc-noobaa-my-app-bucket
```