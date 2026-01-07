---
title: Cloud Native Operation
menu:
  docs_{{ .version }}:
    identifier: index-cloud-native-operation
    name: Cloud Native Operation
    parent: guides
    weight: 1
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Cloud-Native S3 Operations with NooBaa

NooBaa is designed to provide a cloud-native experience for object storage management within a Kubernetes environment. It leverages Kubernetes Custom Resource Definitions (CRDs) to allow developers and administrators to manage `S3-compatible buckets` declaratively, just like any other Kubernetes resource.

### Prerequisites

Follow the steps in `01_installation.md` to install the NooBaa operator and its custom resource definitions (CRDs).

---

### 1. Declarative Bucket Provisioning with ObjectBucketClaim (OBC)

The `ObjectBucketClaim` (OBC) is a Kubernetes custom resource that allows users to request an S3 bucket by simply defining a YAML manifest and applying it to the cluster. The NooBaa operator watches for new OBCs and automatically provisions the corresponding resources.

When you create an `ObjectBucketClaim`, the NooBaa operator creates:

*   An `ObjectBucket` (OB) representing the provisioned bucket.
*   A `Secret` containing the S3 access keys (`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`).
*   A `ConfigMap` containing the bucket's endpoint and name.

This enables a powerful self-service workflow for developers.

#### Create an ObjectBucketClaim

Let's create a simple OBC that requests a bucket using the default `BucketClass`.

Create the following `obc.yaml` file:

```bash
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: my-app-bucket
  namespace: noobaa
spec:
  generateBucketName: my-app-bucket
  storageClassName: noobaa.noobaa.io
```

Here,
- `generateBucketName:` A prefix for the bucket name. NooBaa will append a unique ID to ensure the name is globally unique.
- `storageClassName:` Must be `noobaa.noobaa.io` to be handled by the NooBaa operator.

#### Verify the Provisioned Resources

After applying the OBC, you can verify that NooBaa has created the associated resources.

Now, check for the Secret and ConfigMap that were created:

```bash
kubectl get secret my-app-bucket -n noobaa
NAME            TYPE     DATA   AGE
my-app-bucket   Opaque   2      50s


kubectl get configmap my-app-bucket -n noobaa
NAME            DATA   AGE
my-app-bucket   5      77s
```
These resources contain the access credentials and connection information for your application to use the bucket.


### 2. Consuming the Bucket in an Application

The true power of the `OBC` model is how easily applications can consume the provisioned S3 bucket. The Secret and ConfigMap created by the OBC can be mounted directly into your application's Pod as environment variables.

This decouples your application from the storage infrastructure. Your application code doesn't need to know the S3 endpoint, bucket name, or credentials at build time.


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: s3-cli-tester
  namespace: noobaa
spec:
  containers:
    - name: aws-cli
      image: amazon/aws-cli:latest
      envFrom:
        - secretRef:
            name: my-app-bucket # Name of the Secret created by the OBC
        - configMapRef:
            name: my-app-bucket # Name of the ConfigMap created by the OBC
      command:
        - "/bin/sh"
        - "-c"
        - |
          echo "Creating a test file..."
          echo "Hello from NooBaa!" > /tmp/hello.txt
          echo "Hello from KubeStash!" > /tmp/kubestash.txt

          echo "Uploading file to bucket: $BUCKET_NAME"
          aws --endpoint-url http://$BUCKET_HOST s3 cp /tmp/hello.txt s3://$BUCKET_NAME/
          aws --endpoint-url http://$BUCKET_HOST s3 cp /tmp/kubestash.txt s3://$BUCKET_NAME/

          echo "Listing objects in bucket: $BUCKET_NAME"
          aws --endpoint-url http://$BUCKET_HOST s3 ls s3://$BUCKET_NAME/
  restartPolicy: Never
```

This Pod does the following:


1. Mounts the Secret and ConfigMap from the my-app-bucket OBC as environment variables.
2. Creates two simple text file.
3. Uses the aws s3 cp command to upload the file. Note the --endpoint-url flag, which is constructed from the BUCKET_HOST variable provided by the ConfigMap.
4. Uses aws s3 ls to list the contents of the bucket to verify the upload.


Apply the Pod manifest and check its logs:


```bash
# Create the Pod
kubectl apply -f s3-cli-pod.yaml -n noobaa

# Wait for the pod to complete
kubectl wait --for=condition=Ready pod/s3-cli-tester --timeout=120s -n noobaa

# Check the logs
kubectl logs s3-cli-tester -n noobaa

Creating a test file...
Uploading file to bucket: my-app-bucket-3c188abd-b4cb-4f95-a7b2-01ec44868d11
upload: ../tmp/hello.txt to s3://my-app-bucket-3c188abd-b4cb-4f95-a7b2-01ec44868d11/hello.txt
upload: ../tmp/kubestash.txt to s3://my-app-bucket-3c188abd-b4cb-4f95-a7b2-01ec44868d11/kubestash.txt
Listing objects in bucket: my-app-bucket-3c188abd-b4cb-4f95-a7b2-01ec44868d11
2025-09-04 08:33:45         19 hello.txt
2025-09-04 08:33:51         22 kubestash.txt

```

