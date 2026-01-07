---
title: Bucket Data Replication
menu:
  docs_{{ .version }}:
    identifier: index-data-replication-policy
    name: Bucket Data Replication
    parent: guides
    weight: 1
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Bucket Data Replication

NooBaa supports data replication policies to asynchronously replicate data from a source bucket to one or more destination buckets. This is essential for disaster recovery and high availability, ensuring your data is protected across multiple locations.

![s3-replication.png](images/s3-replication.png)

In this guide, we will configure a primary bucket that uses a local PV pool for storage. We will then apply a replication policy to this bucket so that any data written to it is automatically replicated to a secondary bucket hosted on AWS S3.


### Notes
- **Asynchronous Replication**: Data replication from the source to the destination is not instantaneous.

### Prerequisites
Follow the steps in `01_installation.md` to install the NooBaa operator and its custom resource definitions (CRDs).


### Bucket Class Replication

Bucket replication policies can be applied to a BucketClass. In those cases, the policy will automatically be 'inherited' by all bucket claims that utilize the bucketclass.

**Replication Policy**

We can set replication policy under the `spec.placementPolicy.` field of the BucketClass.

PlacementPolicy is an array of rules:

- `rule_id` defines the unique identifier of the rule.
- `destination_bucket` definers the replication target bucket.
- `filter` field defines the filter to apply to the source bucket during replication.

---

### High-Level Steps

1.  **Create the Replication Target Bucket on AWS S3**:
    *   Create a `NamespaceStore` to connect to an external S3 bucket.
    *   Create a `BucketClass` that uses this `NamespaceStore`.
    *   Create an `ObjectBucketClaim` (OBC) to provision the target bucket.
2.  **Create the Primary Bucket with a Replication Policy**:
    *   Create a `BackingStore` using a local PV Pool.
    *   Create a `BucketClass` that uses the PV Pool `BackingStore` and includes a `replicationPolicy` pointing to the target bucket.
    *   Create an `ObjectBucketClaim` (OBC) to provision the primary bucket.
3.  **Verify Replication**:
    *   Upload files to the primary bucket.
    *   Confirm that the files are replicated to the target bucket on AWS S3.

---

### 1. Create the Replication Target Bucket on AWS S3

First, we'll set up the destination bucket on AWS S3 where the data will be replicated.

#### Create AWS Credentials Secret
Create a Kubernetes secret containing your AWS credentials.

```bash
kubectl create secret generic aws-s3-creds \
  --from-literal=AWS_ACCESS_KEY_ID=<YOUR_AWS_ACCESS_KEY_ID> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<YOUR_AWS_SECRET_ACCESS_KEY> \
  --namespace noobaa
```

#### Create a NamespaceStore for AWS S3

This `NamespaceStore` will connect NooBaa to your target S3 bucket. Create the following `aws-namespace-store.yaml` file:

```yaml
apiVersion: noobaa.io/v1alpha1
kind: NamespaceStore
metadata:
  name: aws-namespace-store
spec:
  awsS3:
    region: us-east-1
    secret:
      name: aws-s3-creds
      namespace: noobaa
    targetBucket: pv-pool-bucket-replication
  type: aws-s3
```

Here,
- `targetBucket` The name of your existing S3 bucket.
- `secret` is the name of the secret that contains the AWS credentials.

#### Create a BucketClass for the NamespaceStore

This `BucketClass` defines how to use the NamespaceStore. Create the following `aws-namespace-bucket-class.yaml`:
```yaml
apiVersion: noobaa.io/v1alpha1
kind: BucketClass
metadata:
  name: aws-namespace-bucket-class
spec:
  namespacePolicy:
    type: Single
    single:
      resource: aws-namespace-store
```

Here,
- `type` is the type of the namespace store.
- `single.resource` is the namespace store we created above.

#### Create an ObjectBucketClaim for the Target Bucket

This OBC provisions a `NooBaa` bucket that maps to your external S3 bucket. This will be our replication destination. Create the following `aws-object-bucket-claim.yaml`:

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: pv-pool-bucket-replication
spec:
  generateBucketName: pv-pool-bucket-replication
  storageClassName: noobaa.noobaa.io
  additionalConfig:
    bucketclass: aws-namespace-bucket-class
```
After a moment, a bucket will be created. You can find its full name by listing the buckets.

#### Accessing Your Data via the NooBaa S3 Endpoint**

Connect with the NooBaa S3 endpoint and verify the bucket creation.

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
2025-08-27 16:08:14 pv-pool-bucket-replication-180c1b36-bf4e-451e-977f-1cd996f22627
2025-08-26 18:57:55 first.bucket
```
Here,
- `pv-pool-bucket-replication-180c1b36-bf4e-451e-977f-1cd996f22627` is the name of the bucket for the obectbucketclaim we created.




### 2. Create the Primary Bucket with a Replication Policy

Now, we'll create the primary bucket that stores data locally and replicates it to the AWS bucket we just configured.


#### Create a PV Pool BackingStore

This `BackingStore` will use a PersistentVolume (PV) to store data locally. Create the following `pv-pool-backing-store.yaml`:

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

#### Create a BucketClass with a Replication Policy

This `BucketClass` uses the local `PV pool` for storage and includes a `replicationPolicy`. Create the following `replication-bucket-class.yaml`:

```yaml
apiVersion: noobaa.io/v1alpha1
kind: BucketClass
metadata:
  name: replication-bc
  namespace: noobaa
spec:
  placementPolicy:
    tiers:
      - backingStores:
          - pv-pool-backing-store
  replicationPolicy: |
    {
      "rules": [
        {
          "rule_id": "rule-1",
          "destination_bucket": "pv-pool-bucket-replication-180c1b36-bf4e-451e-977f-1cd996f22627"
        }
      ]
    }
```

Here,
- `tiers` is the list of backing stores that will be used for the bucket.
- `rule_id` is the unique identifier of the rule.
- `destination_bucket` is the name of the bucket that was created earlier.

#### Create an ObjectBucketClaim for the Primary Bucket

This OBC will create our primary source bucket. Create the following `replication-object-bucket-claim.yaml`:

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: pv-pool-bucket
spec:
  generateBucketName: pv-pool-bucket
  storageClassName: noobaa.noobaa.io
  additionalConfig:
    bucketclass: replication-bc
```

Now, for the disaster scenario of this `pv-pool-bucket` bucket we want to replicate this bucket to another cloud bucket what we created and added to the replication policy of the bucket class.


### 3.Verify Bucket Replication

Now we'll upload a file to the source bucket and verify that it's replicated to the target bucket.

#### List S3 Buckets
List the buckets to find the full names generated by NooBaa.

```bash
s3 ls 

# Expected output (names will vary):
2025-09-03 13:27:09 replicated-source-bucket-58d15153-54a6-47af-bd7b-e88e7a72e7db
2025-09-03 10:58:57 aws-replication-target-bucket-180c1b36-bf4e-451e-977f-1cd996f22627
```

#### Upload some Objects to the Source Bucket
```bash
## Generate some demo txt file
echo "Hello World" > demo.txt
echo "Hello Kubestash" > kubestash.txt
echo "Hello NooBaa" > noobaa.txt

## Upload the file to the bucket
s3 cp  demo.txt   s3://pv-pool-bucket-58d15153-54a6-47af-bd7b-e88e7a72e7db/demo.txt
s3 cp  kubestash.txt  s3://pv-pool-bucket-58d15153-54a6-47af-bd7b-e88e7a72e7db/kubestash.txt
s3 cp  noobaa.txt s3://pv-pool-bucket-58d15153-54a6-47af-bd7b-e88e7a72e7db/noobaa.txt

## List the objects in the bucket
s3 ls s3://pv-pool-bucket-58d15153-54a6-47af-bd7b-e88e7a72e7db/
2025-09-03 14:35:11         12 demo.txt
2025-09-03 14:35:30         16 kubestash.txt
2025-09-03 14:35:35         13 noobaa.txt

```

#### Verify Replication to the Target Bucket

```bash
s3 ls s3://pv-pool-bucket-replication-180c1b36-bf4e-451e-977f-1cd996f22627/

2025-09-03 14:40:49         12 demo.txt
2025-09-03 14:40:48         16 kubestash.txt
2025-09-03 14:40:48         13 noobaa.txt
```

You can also verify the presence of these objects from your AWS S3 console.

![replicated-data.png](images/replicated-data.png)

This confirms that the replication policy is working correctly.