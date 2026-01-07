---
title: Multi Cloud Federation
menu:
  docs_{{ .version }}:
    identifier: index-multi-cloud-federation
    name: Multi Cloud Federation
    parent: guides
    weight: 1
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Multi Hybrid Cloud Data Federation

## Overview

NooBaa's multi-cloud data federation allows you to manage data across various storage locations—including public clouds like AWS, Azure, and Google Cloud, as well as on-premises S3-compatible storage—through a single, unified S3 endpoint. This enables you to create a flexible and resilient data fabric without being locked into a single provider.

This guide will walk you through connecting multiple cloud storage backends and defining data placement policies to mirror or spread data across them.


## Create Multiple Backing Stores
We've to create Multiple Backing Stores for each in cloud provider or on-premises location what you want to include in your data federation.
We're going to create one `s3` and another `Local PV` backing store and connect them to the data federation.

### Create `S3` Backing Store:

For AWS S3 Create a secret with your AWS credentials:
```bash
kubectl create secret generic aws-s3-creds \
  --from-literal=AWS_ACCESS_KEY_ID=<YOUR_AWS_ACCESS_KEY_ID> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<YOUR_AWS_SECRET_ACCESS_KEY>
```

Then, create the BackingStore manifest:
```yaml
apiVersion: noobaa.io/v1alpha1
kind: BackingStore
metadata:
  name: aws-s3-backingstore
  namespace: noobaa
spec:
  type: aws-s3
  awsS3:
    targetBucket: noobaa
    secret:
      name: aws-s3-creds
      namespace: noobaa
```
Here,
- `targetBucket` is the name of the bucket that will be created in the cloud provider.
- `secret` is the name of the secret that contains the AWS credentials.


### Create `Local PV` Backing Store:

Create the BackingStore manifest:

```yaml
apiVersion: noobaa.io/v1alpha1
kind: BackingStore
metadata:
  name: local-pv-backing-store
  namespace: noobaa
spec:
  pvPool:
    numVolumes: 1
    resources:
      requests:
        storage: 16Gi
        memory: 4Gi
        cpu: 2000m
      limits:
        memory: 4Gi
        cpu: 2000m
  type: pv-pool
```

## Define Data Placement Policies with a BucketClass

A BucketClass defines how data is placed across one or more BackingStores. You can choose between `Mirror` and `Spread` policies.

Here, in this guide, we're going to create a BucketClass with a Mirror policy that includes both the `aws-s3-backingstore` and `local-pv-backing-store`.

**Mirror Policy**

A Mirror policy replicates data to two or more BackingStores, providing data redundancy.

**Key Points About Mirror Policy:**
- `Mirror` Policy ensures that data is replicated to two or more BackingStores.
- If one backing store fails, the data is still available in the other.
- If both backing stores fail, the data is lost.
- As, we are using a cloud provider, the data is replicated to that cloud provider.So, data will never lost.

This is how we can achive multi-cloud data federation

Now, lets create a `BucketClass` manifest that `mirrors` data between `AWS` and `Local PV` stores:

```yaml
apiVersion: noobaa.io/v1alpha1
kind: BucketClass
metadata:
  name: mirror-bucket-class
spec:
  placementPolicy:
    tiers:
      - backingStores:
          - aws-s3-backingstore
          - local-pv-backingstore
        placement: Mirror
```

Here,
- `backingStores` is the list of backing stores that will be used to store the data.
- `placement` is the placement policy that will be used to place the data.

## Create a Bucket

Finally, create a bucket using an `ObjectBucketClaim` (OBC) that references above `BucketClass`.

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: federated-bucket
spec:
  generateBucketName: federated-bucket
  storageClassName: noobaa.noobaa.io
  additionalConfig:
    bucketclass: mirror-bucket-class
```

Any data written to my-federated-bucket will now be automatically mirrored between `AWS S3` and `Local PV` , all managed seamlessly through the `NooBaa S3 endpoint`.

## Verify

Let's verify that the mirror policy works by uploading an object, simulating a failure of the `AWS S3` BackingStore, and then confirming the object is still accessible from the `Local PV` store.

**Connect with the NooBaa S3 endpoint**

```bash
export EXTERNAL_IP=$(kubectl get svc s3 -n noobaa -o json | jq -r '.status.loadBalancer.ingress[0].ip')
export NOOBAA_ACCESS_KEY=$(kubectl get secret noobaa-admin -n noobaa -o json | jq -r '.data.AWS_ACCESS_KEY_ID|@base64d')
export NOOBAA_SECRET_KEY=$(kubectl get secret noobaa-admin -n noobaa -o json | jq -r '.data.AWS_SECRET_ACCESS_KEY|@base64d')
alias s3='AWS_ACCESS_KEY_ID=$NOOBAA_ACCESS_KEY AWS_SECRET_ACCESS_KEY=$NOOBAA_SECRET_KEY aws --endpoint https://$EXTERNAL_IP:443 --no-verify-ssl s3'
export AWS_REQUEST_CHECKSUM_CALCULATION=when_required
export AWS_RESPONSE_CHECKSUM_CALCULATION=when_required
```

**Verify the Bucket**

```bash
s3 ls
2025-08-27 16:08:14 federated-bucket-efcdec31-5658-4422-967a-b311eb14ce7d
2025-08-26 18:57:55 first.bucket
```
Here,
- `federated-bucket-efcdec31-5658-4422-967a-b311eb14ce7d` is the name of the bucket that was created.

**Upload an Object to the Bucket**

Let's upload a movie of `3.2 GB` to the bucket:
```bash
s3 cp  ~/random/dune.mkv s3://federated-bucket-efcdec31-5658-4422-967a-b311eb14ce7d/movies/dune.mkv

upload: ../../../../../../../../../random/dune.mkv to s3://federated-bucket-efcdec31-5658-4422-967a-b311eb14ce7d/movies/dune.mkv
```
So, this object is now available in both the `AWS S3` and `Local PV` stores.

**Simulate a Failure of the AWS S3 BackingStore**

To test failover, remove the `backingStores : aws-s3-backingstore` from the BucketClass and `placement: Mirror` field.

```bash
apiVersion: noobaa.io/v1alpha1
kind: BucketClass
metadata:
  name: mirror-bucket-class
spec:
  placementPolicy:
    tiers:
      - backingStores:
          - local-pv-backingstore
```

**Now check the object**

Now, Due to the Mirror policy, the object is still available in the `Local PV` store.

```bash
s3 cp  s3://federated-bucket-08070214-e6ee-4617-80e7-c0a317a4219f/movies/dune.mkv ~/Downloads/dune.mkv

download: s3://federated-bucket-08070214-e6ee-4617-80e7-c0a317a4219f/movies/dune.mkv to ../../../../../../../../../Documents/dune.mkv
```

So, the object is still available in the `Local PV` store.

This is how NooBaa's multi-cloud data federation works, providing seamless data management across multiple storage backends with redundancy and failover capabilities.

---