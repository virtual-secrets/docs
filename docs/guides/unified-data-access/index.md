---
title: Unified Data Access
menu:
  docs_{{ .version }}:
    identifier: index-unified-data-access
    name: Unified Data Access
    parent: guides
    weight: 1
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Unified Data Access with NooBaa

NooBaa provides unified data access by allowing you to connect to various storage resources, whether they are in the cloud or on-premises, and manage them through a single S3-compatible endpoint. This is primarily achieved using **Namespace Stores**.

## What is a Namespace Store?

A Namespace Store is a NooBaa resource that represents an underlying storage location, such as an existing AWS S3 bucket, Azure Blob container, or any S3-compatible object store. Unlike a `BackingStore`, which is used for storing NooBaa's own data chunks, a `NamespaceStore` provides read and/or write access to data that already exists in the target store.

This enables you to create "Namespace Buckets" in NooBaa that act as a virtual view into one or more of these external storage locations.

### Key Use Cases

-   **Hybrid and Multi-Cloud Access**: Access data from multiple public clouds (AWS, Azure, GCP) and on-prem S3 stores through a single S3 endpoint.
-   **Transparent Data Migration**: Migrate data between storage systems in the background while applications continue to have uninterrupted access.
-   **Unified Data Access**: Access data from multiple storage systems through a single S3 endpoint.

## How to Configure a Namespace Store

Setting up a Namespace Store involves three main steps:
1.  Create a `Secret` containing the credentials for the external storage provider.
2.  Create a `NamespaceStore` custom resource that points to the target bucket/container.
3.  Create a `BucketClass` that uses the `NamespaceStore`.
4.  Create an `ObjectBucketClaim` (OBC) to provision the Namespace Bucket.

This guide demonstrates how to configure a `Namespace` Store using an existing `Azure` Blob Storage container. Once configured, you can access the data in your Azure container through NooBaa's S3-compatible endpoint.

### Create a Provider Secret for Azure

First, create a Kubernetes secret containing your Azure Storage Account credentials. NooBaa will use this secret to authenticate with Azure.

```bash
kubectl create secret generic azure-secret -n noobaa \
  --from-literal=Azure_Storage_Account_Name=<your-azure-storage-account-name> \
  --from-literal=Azure_Storage_Account_Key=<your-azure-storage-account-key>
```

### Create the Azure NamespaceStore

Next, define a `NamespaceStore` resource that points to your `Azure` Blob container. This resource tells NooBaa how to connect to the external store.

```bash
apiVersion: noobaa.io/v1alpha1
kind: NamespaceStore
metadata:
  name: azure-namespacestore
  namespace: noobaa
spec:
  type: azure-blob
  azureBlob:
    targetBlobContainer: noobaa
    secret:
      name: azure-secret
      namespace: noobaa
```

Here,

- type: azure-blob: Specifies the provider type.
- targetBucket: The name of your existing container in Azure Blob Storage.
- secret: A reference to the secret created in the previous step.

### Create a BucketClass

```yaml
apiVersion: noobaa.io/v1alpha1
kind: BucketClass
metadata:
  name: azure-namespace-bucket-class
spec:
  namespacePolicy:
    type: Single
    single:
      resource: azure-namespacestore
```

### Create an ObjectBucketClaim (OBC)

Finally, create an ObjectBucketClaim (OBC) to provision the S3-compatible bucket that your applications will use. This OBC references the BucketClass you just created.

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: azure-bucket
spec:
  generateBucketName: azure-bucket
  storageClassName: noobaa.noobaa.io
  additionalConfig:
    bucketclass: azure-namespace-bucketclass
```

### Accessing Your Data via the S3 Endpoint

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
2025-08-27 16:08:14 azure-bucket-33ef992e-68aa-4489-bc72-94aefa23cb11
2025-08-26 18:57:55 first.bucket
```
Here,
- `azure-bucket-33ef992e-68aa-4489-bc72-94aefa23cb11` is the name of the bucket for the obectbucketclaim we created.


**List objects in the bucket**
```bash
s3 ls s3://azure-bucket-33ef992e-68aa-4489-bc72-94aefa23cb11/ --recursive
```

Now, whatever we upload here, it'll redirect to the Azure Blob Storage container.

**Upload a file**

```bash
echo "hello azure" > test.txt
s3 cp test.txt s3://azure-bucket-33ef992e-68aa-4489-bc72-94aefa23cb11 
upload: ./test.txt to s3://azure-bucket-33ef992e-68aa-4489-bc72-94aefa23cb11/test.txt
```

Now, verify that the file is in the Azure Blob Storage container.

```bash
az storage blob list \
        --container-name noobaa \
        --account-name kubestash \
        --output table
Name      Blob Type    Blob Tier    Length    Content Type    Last Modified              Snapshot
--------  -----------  -----------  --------  --------------  -------------------------  ----------
test.txt  BlockBlob    Hot          12        text/plain      2025-09-01T08:33:49+00:00
```

You should see `test.txt` listed in the output, confirming that the data was successfully written to Azure through NooBaa.