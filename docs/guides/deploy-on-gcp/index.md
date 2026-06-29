---
title: Deploy Virtual Secrets on GCP
menu:
  docs_{{ .version }}:
    identifier: index-deploy-on-gcp
    name: Deploy on GCP
    parent: guides
    weight: 12
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Deploy Virtual Secrets on GCP

Kubernetes secrets are not really "secret". They are typically stored in plain-text inside etcd. Check out the [Kubernetes Docs](https://kubernetes.io/docs/concepts/configuration/secret/#:~:text=Kubernetes%20Secrets%20are,Kubernetes%20Secrets.) for reference.

This **Virtual Secrets** project offers a new `Secret` which is a Kubernetes Extended API resource that addresses these limitations of the core `Secret`. It supports the full api `Read` / `Write` / `Mount` operation for external/cloud secret management systems without using the k8s secrets rail.

## Virtual Secrets Design

**Virtual Secrets** introduces a new `Secret` resource under the `virtual-secrets.dev` API group, functioning similarly to the core Kubernetes `Secret` resource from a user point of view. However, instead of storing secret data in `etcd`, it securely stores it in an external secret manager.

The Secret resource is divided into two parts:

- **Secret Data**: Stored in an external secret manager to enhance security.

- **Secret Metadata**: Stored within the cluster, as it is less sensitive and improves performance.

## How to use Virtual Secrets

Let's go ahead and see how to use the `Secret` offered by **Virtual Secrets**.

To keep things isolated, let's create a namespace called `demo` where all the resources will be deployed.

```bash
$ kubectl create namespace demo
namespace/demo created
```

### Install Virtual Secrets Server

First, install the virtual-secret-server which is a custom api server for the `secrets.virtual-secrets.dev` resource.

```bash
$ helm repo add appscode https://charts.appscode.com/stable/
$ helm repo update
$ helm search repo appscode/virtual-secrets-server --version=v2025.3.14
$ helm upgrade -i virtual-secrets-server appscode/virtual-secrets-server \
    --version=v2025.3.14 -n kubevault --create-namespace 
```

### Deploy and Configure Vault Server

Before we create a custom `Secret`, we need to deploy a vault server where the secret data will be stored. Also, it needs to be configured to grant necessary permissions like create, update, read, list, delete and delete in a kv secret engine named `virtual-secrets.dev` to the virtual-secrets-server.

If you do not have a vault server running. You can use [KubeVault](https://kubevault.com) to deploy a vault server. Here is a [guide](https://kubevault.com/articles/how-to-use-hashicorp-vault-in-kubernetes-using-kubevault/#:~:text=cloud%2Dnative%20applications.-,Deploy%20Vault%20on%20Kubernetes,-Pre%2Drequisites) to do that.

### Create SecretStore

We need to create another resource called `SecretStore` which will contain the connection information to the external secret manager where the secrets will be stored.

Credentials Mode:

We need to create a gcp credentials secret named `gcp-cred` in the `demo` namespace with the following data:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gcp-cred
  namespace: demo
data:
  GCP_CREDENTIALS: <base64-encoded-gcp-cred-json>
```

```yaml
apiVersion: config.virtual-secrets.dev/v1alpha1
kind: SecretStore
metadata:
  name: gcp-cred-ss
spec:
  gcp:
    region: eastus
    secretRef:
      name: gcp-cred
      namespace: demo
```

Credentials-less Mode:
```yaml
apiVersion: config.virtual-secrets.dev/v1alpha1
kind: SecretStore
metadata:
  name: gcp-cred-ss2
spec:
  gcp:
    region: eastus
```

Here,
- `spec.gcp` - section describes the connection information for GCP Secret Manager.
- `spec.gcp.region` - contains the GCP region.
- `spec.gcp.secretRef` - contains the reference to the secret with GCP credentials (for credentials mode).

> **Note**: `spec.gcp` can be used to specify the connection information of the corresponding secret manager.

Granting necessary permissions to virtual-secrets-server for Credentials-less Mode:

Need to add an annotation to the service account of virtual-secrets-server and bind it with a GSA which has the needed permissions to access the secret manager. Here is how you can do it,

```bash
kubectl annotate sa virtual-secrets-server \
  iam.gke.io/gcp-service-account="rudrovs@appscode-testing.iam.gserviceaccount.com" \
  -n kubevault --overwrite

gcloud iam service-accounts add-iam-policy-binding rudrovs@appscode-testing.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:appscode-testing.svc.id.goog[kubevault/virtual-secrets-server]"

kubectl rollout restart deployment virtual-secrets-server -n kubevault
```

### Create Virtual Secret

Now, we can create custom `Secret` resource using the YAML below:

```yaml
apiVersion: virtual-secrets.dev/v1alpha1
kind: Secret
metadata:
  name: virtual-secret
  namespace: demo
stringData:
  username: appscode
  password: virtual-secret
secretStoreName: gcp
```

Here,
- `secretStoreName` - specifies the `SecretStore` we just created.
- Other than that, everything else is similar to a core Kubernetes `Secret`.

Let's go ahead and apply the `Secret`,

```bash
$ kubectl apply -f virtual-secrets/virtual-secret.yaml
secret.virtual-secrets.dev/virtual-secret created
```

Let's list the `Secrets` to see if it is created or not,

```bash
$ kubectl get secrets.virtual-secrets.dev -n demo
NAME             TYPE     DATA   AGE
virtual-secret   Opaque   2      1m6s
```

We can also get the whole definition of the `Secret`,

```bash
$ kubectl get secrets.virtual-secrets.dev -n demo virtual-secret -oyaml
apiVersion: virtual-secrets.dev/v1alpha1
data:
  password: dmlydHVhbC1zZWNyZXQ=
  username: YXBwc2NvZGU=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"virtual-secrets.dev/v1alpha1","kind":"Secret","metadata":{"annotations":{},"name":"virtual-secret","namespace":"demo"},"secretStoreName":"gcp","stringData":{"password":"virtual-secret","username":"appscode"}}
  creationTimestamp: "2025-03-25T10:37:45Z"
  generation: 1
  name: virtual-secret
  namespace: demo
  resourceVersion: "332000"
  uid: e2fd622a-020a-4ca4-acbe-501a9147a089
secretStoreName: gcp
type: Opaque
```

We can see that this `Secret` actually behaves identical of the core `Secret`. But the data is not stored in the `etcd` and it is way more secure than using the native k8s `Secret`.

### Mount Virtual Secret

`Secrets` are not that useful if we can not mount them to pods. We can mount the `virtual secrets` using [Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/).

Virtual Secrets comes with a custom provider of `Secrets Store CSI Driver`, named `secrets-store-csi-driver-provider-virtual-secrets` which leverages `virtual-secrets-server` to read secret data from `virtual secrets` and uses the `Secrets Store CSI Driver` to mount those into to the pods.

Let's go ahead and install `Secrets Store CSI Driver` and `secrets-store-csi-driver-provider-virtual-secrets` into our cluster,

```bash
$ helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
$ helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system

$ helm search repo appscode/secrets-store-csi-driver-provider-virtual-secrets --version=v2025.3.14
$ helm upgrade -i secrets-store-csi-driver-provider-virtual-secrets appscode/secrets-store-csi-driver-provider-virtual-secrets -n kube-system --create-namespace --version=v2025.3.14
```

If both of them are deployed we should see two new pods in the `kube-system` namespace.

```bash
$ kubectl get pods -n kube-system
NAME                                                      READY   STATUS    RESTARTS      AGE
csi-secrets-store-secrets-store-csi-driver-rvpvm          3/3     Running   0             61s
secrets-store-csi-driver-provider-virtual-secrets-m78gv   1/1     Running   0             34s
```

The `Secrets Store CSI Driver` uses a custom resource named `SecretProviderClass` to mount the secret. Let's go ahead and create that,

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: virtual-secret
  namespace: demo
spec:
  provider: virtual-secrets
  parameters:
    secretName: virtual-secret
```

Here,
- `spec.provider` - specifies the provider for `Secrets Store CSI Driver` to communicate and use.
- `parameters.secretName` - specifies the name of the `virtual secret` we want to mount.

> **Note**:
>   - We can also call the `mount` subresource of the `virtual secret` to create the `SecretProviderClass` for us.
>   - The namespace and the name of `SecretProviderClass` should be same as the `Virtual Secret` it is being used for.

Let's create the `SecretProviderClass`,

```bash
$ kubectl apply -f virtual-secrets/secret-provider-class.yaml 
secretproviderclass.secrets-store.csi.x-k8s.io/virtual-secret created
```

With the custom secret created, the authentication configured and role created, the `provider-virtual-secrets` extension installed and the `SecretProviderClass` defined it is finally time to create a pod that mounts the desired secret.

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: webapp
  namespace: demo
spec:
  containers:
    - image: jweissig/app:0.0.1
      name: webapp
      volumeMounts:
        - name: virtual-secrets-store
          mountPath: "/mnt/virtual-secrets"
          readOnly: true
  volumes:
    - name: virtual-secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "virtual-secret"
```

Here,
- In `spec.volumes[0]`, a volume with name `virtual-secrets-store` with necessary configs is specified.
- In `spec.containers[0].volumeMounts`, the volume is referred to be mounted in the `/mnt/virtual-secrets` path.

Let's create the pod,

```bash
$ kubectl apply -f virtual-secrets/app.yaml
pod/webapp created
```

If we get the pod we will see that it will get to the `Running` state after some period,

```bash
$ kubectl get pods -n demo
NAME                READY   STATUS    RESTARTS   AGE
webapp              1/1     Running   0          6m45s
```

Now, check the secret data written to the file system at /mnt/virtual-secrets on the webapp pod.

```bash
$ kubectl exec -n demo webapp -- cat /mnt/virtual-secrets/username
appscode⏎

$ kubectl exec -n demo webapp -- cat /mnt/virtual-secrets/password
virtual-secret⏎ 
```

The value displayed matches the username and password value for the custom secret named `virtual-secret` we created earlier.

## Support

To speak with us, please leave a message on [our website](https://appscode.com/contact/).

To receive product announcements, follow us on [Twitter/X](https://x.com/AppsCodeHQ).

To watch tutorials of various Production-Grade Kubernetes Tools Subscribe our [YouTube](https://youtube.com/@appscode) channel.

If you have found a bug with Virtual Secrets or want to request for new features, please [file an issue](https://github.com/virtual-secrets/project/issues/new).
