---
title: On-Prem with scalability
menu:
  docs_{{ .version }}:
    identifier: index-on-prem-with-scalability
    name: On-Prem with scalability
    parent: guides
    weight: 1
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# On-Premises S3 Object Storage with Scalability

## Overview
NooBaa provides a scalable on-premises S3-compatible object storage solution. It allows you to set up local S3 storage and dynamically scale storage capacity and performance as needed.

## Key Features
- **Local S3 Setup**: Deploy NooBaa as an on-premises S3-compatible storage system.
- **Dynamic Scaling**: Scale storage capacity and performance based on workload demands.

## Installation Process

### Install NooBaa CRDs and Operator
Follow the steps in the `installation.md` file to install the NooBaa CRDs and Operator.


### Create Backing Store

Let's create a separate backing store for testing on premise S3 with storage scalability.

```bash
apiVersion: noobaa.io/v1alpha1
kind: BackingStore
metadata:
  name: pv-pool-backing-store
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

### Create Bucket Class

Let's create a bucket class, where we can specify tiering and placement policy. As this doc is for storage scalability, we will use the minimal setup for bucket class.

```yaml
apiVersion: noobaa.io/v1alpha1
kind: BucketClass
metadata:
  name: pv-pool-bucket-class
spec:
  placementPolicy:
    tiers:
      - backingStores:
          - pv-pool-backing-store
```

here,
- `tiers` is the placement policy.`
- `pv-pool-backing-store` is the backing store we created above.

### Create a Bucket using COSI

Now, let's create a bucket using the bucket class we created above.

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: kubestash-obc
spec:
  generateBucketName: kubestash
  storageClassName: noobaa.noobaa.io
  additionalConfig:
    bucketclass: pv-pool-bucket-class
```

Here, 
- `generateBucketName` is the name of the bucket.
- `additionalConfig` is the bucket class we created above.
- `storageClassName` comes default while noobaa is installed.

### Verify Bucket Creation and Access the Bucket

Now, let's connect with the NooBaa S3 endpoint and verify the bucket creation.

```bash
export EXTERNAL_IP=$(kubectl get svc s3 -n noobaa -o json | jq -r '.status.loadBalancer.ingress[0].ip')
export NOOBAA_ACCESS_KEY=$(kubectl get secret noobaa-admin -n noobaa -o json | jq -r '.data.AWS_ACCESS_KEY_ID|@base64d')
export NOOBAA_SECRET_KEY=$(kubectl get secret noobaa-admin -n noobaa -o json | jq -r '.data.AWS_SECRET_ACCESS_KEY|@base64d')
alias s3='AWS_ACCESS_KEY_ID=$NOOBAA_ACCESS_KEY AWS_SECRET_ACCESS_KEY=$NOOBAA_SECRET_KEY aws --endpoint https://$EXTERNAL_IP:443 --no-verify-ssl s3'
export AWS_REQUEST_CHECKSUM_CALCULATION=when_required
export AWS_RESPONSE_CHECKSUM_CALCULATION=when_required
```

Let's verify the bucket creation.
```bash
s3 ls 
2025-08-26 18:57:55 first.bucket
2025-08-27 12:16:05 kubestash-20a8a66c-8b4a-408d-a763-e9fe4d0f5309

```
Here, bucket has been created named `kubestash-20a8a66c-8b4a-408d-a763-e9fe4d0f5309` for genereatedBucketName `kubestash` of `kubestash-obc`.

### Scaling Storage Capacity

Now, let's check the storage capacity of the bucket. Noobaa CLI has build in commands to check the storage capacity.

```bash
noobaa bucket status kubestash-20a8a66c-8b4a-408d-a763-e9fe4d0f5309

Bucket status:
  Bucket                 : kubestash-20a8a66c-8b4a-408d-a763-e9fe4d0f5309
  OBC Namespace          : noobaa
  OBC BucketClass        : pv-pool-bucket-class
  Type                   : REGULAR
  Mode                   : OPTIMAL
  ResiliencyStatus       : OPTIMAL
  QuotaStatus            : QUOTA_NOT_SET
  Num Objects            : 0
  Data Size              : 0.000 B
  Data Size Reduced      : 0.000 B
  Data Space Avail       : 15.452 GB
  Num Objects Avail      : 9007199254740991
```

Here,
- `Data Size` is the total size of the bucket.
- `Data Size Reduced` is the size of total objects after reduced data.
- `Data Space Avail` is the storage capacity of the bucket. It refers the `resources` request of the backing store.
- `Num Objects Avail` is the number of objects that can be stored in the bucket.

**Scaling Storage**

Now, let's increase the `storage capacity` of the bucket. To do that we've to increase `numVolumes` of the backing store.

```yaml
  pvPool:
    numVolumes: 2 # Increasing num of volumes
```
Let's increase the `numVolumes` to `2`, that means the `Data Space Avail` field will be doubled. let's apply this and check the bucket status again.

```bash
noobaa bucket status kubestash-20a8a66c-8b4a-408d-a763-e9fe4d0f5309

Bucket status:
  Bucket                 : kubestash-20a8a66c-8b4a-408d-a763-e9fe4d0f5309
  OBC Namespace          : noobaa
  OBC BucketClass        : pv-pool-bucket-class
  Type                   : REGULAR
  Mode                   : OPTIMAL
  ResiliencyStatus       : OPTIMAL
  QuotaStatus            : QUOTA_NOT_SET
  Num Objects            : 0
  Data Size              : 0.000 B
  Data Size Reduced      : 0.000 B
  Data Space Avail       : 30.904 GB
  Num Objects Avail      : 9007199254740991
```
So, `Data Space Avail` has been increased to `30.904 GB`. We've successfully scaled the storage capacity of the bucket.

> Note: Keep in mind that we can't scale down `numVolumes` of backing store, it's a limitation of the underlying storage.




