---
title: NooBaa
menu:
  docs_{{ .version }}:
    identifier: index-noobaa
    name: NooBaa
    parent: crds
    weight: 1
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

# NooBaa

## What is NooBaa

A `NooBaa` resource in NooBaa is the main custom resource that represents the deployment and management of the NooBaa system within a Kubernetes cluster. It acts as the central controller, orchestrating all NooBaa components, including object buckets, backing stores, namespace stores, and bucket classes. The NooBaa resource is responsible for the lifecycle, configuration, and overall health of the NooBaa deployment.

By managing the NooBaa resource, administrators can deploy, upgrade, and monitor the entire NooBaa data management platform in Kubernetes.

## NooBaa CRD Concept

The `NooBaa` Custom Resource Definition (CRD) in NooBaa is used to manage the deployment, configuration, and status of the NooBaa system. It includes settings for storage, networking, resource allocation, and integration with other NooBaa CRDs.

### Key Features
- Central management of NooBaa components and resources
- Handles deployment, upgrades, and scaling
- Integrates with BackingStore, NamespaceStore, BucketClass, and ObjectBucket resources
- Provides health and status monitoring for the NooBaa system
- Supports multi-cloud and hybrid deployments

## NooBaa `Spec`
A `NooBaa` CRD has the following spec:

The `.spec` field of a NooBaa resource defines the desired state of the NooBaa system. It includes configuration options for various components and behaviors. Below are some of the key fields within the `spec`:

- **affinity**: Affinity (optional) passed through to noobaa's pods
- **annotations**: The annotations-related configuration to add/set on each Pod related object.
- **bucketNotifications**: BucketNotifications (optional) controls bucket notification options
- **cleanupPolicy**: CleanupPolicy (optional) Indicates user's policy for deletion
- **coreResources**: CoreResources (optional) overrides the default resource requirements for the
  server container
- **dbConf**: DBConf (optional) overrides the default postgresql db config
- **dbImage**: DBImage (optional) overrides the default image for the db container
- **dbResources**: DBResources (optional) overrides the default resource requirements for the
  db container
- **dbSpec**: DBSpec (optional) DB spec for a managed postgres cluster
- **dbStorageClass**: DBStorageClass (optional) overrides the default cluster StorageClass for the
  database volume.
- **dbType**: DBType (optional) overrides the default type image for the db container.
  The only possible value is postgres
- **dbVolumeResources**: DBVolumeResources (optional) overrides the default PVC resource requirements
  for the database volume. For the time being this field is immutable and can only be set on system
  creation. This is because volume size updates are only supported for increasing the
  size, and only if the storage class specifies `allowVolumeExpansion: true`,
- **debugLevel**: <integer> enum: all, nsfs, warn, default_level. DebugLevel (optional) sets the debug level.
- **disableLoadBalancerService**:DisableLoadBalancerService (optional) sets the service type to ClusterIP
  instead of LoadBalancer.
- **disableRoutes**: DisableRoutes (optional) disables the reconciliation of openshift route
  resources in the cluster.
- **endpoints**: Endpoints (optional) sets configuration info for the noobaa endpoint
  deployment.
- **externalPgSSLRequired**: ExternalPgSSLRequired (optional) holds an optional boolean to force ssl
  connections to the external Postgres DB
- **externalPgSSLSecret**: ExternalPgSSLSecret (optional) holds an optional secret with client key and
  cert used for connecting to external Postgres DB
- **externalPgSSLUnauthorized**: ExternalPgSSLUnauthorized (optional) holds an optional boolean to allow
  unauthorized connections to external Postgres DB
- **externalPgSecret**: ExternalPgSecret (optional) holds an optional secret with a url to an
  extrenal Postgres DB to be used
- **image**: Image (optional) overrides the default image for the server container
- **imagePullSecret**: ImagePullSecret (optional) sets a pull secret for the system image
- **joinSecret**: JoinSecret (optional) instructs the operator to join another cluster
  and point to a secret that holds the join information
- **labels**:	<map[string]map[string]string> The labels-related configuration to add/set on each Pod related object.
- **loadBalancerSourceSubnets**:LoadBalancerSourceSubnets (optional) if given will allow access to the
  NooBaa services only from the listed subnets. This field will have no effect if
  `DisableLoadBalancerService` is set to true
- **logResources**:	LogResources (optional) overrides the default resource requirements for the
  noobaa-log-processor container
- **manualDefaultBackingStore**: ManualDefaultBackingStore (optional - default value is false) if true the
  default backingstore/namespacestore will not be reconciled by the operator and it should be manually handled by
  the user. It will allow the user to delete DefaultBackingStore/DefaultNamespaceStore, user needs to
  delete associated buckets and update the admin account with new BackingStore/NamespaceStore in order to
  delete the DefaultBackingStore/DefaultNamespaceStore. 
- **pvPoolDefaultStorageClass**: PVPoolDefaultStorageClass (optional) overrides the default cluster
  StorageClass for the pv-pool volumes. This affects where the system stores data chunks (encrypted).
  Updates to this field will only affect new pv-pools, but updates to existing pools are not supported by the operator.
- **region**: Region (optional) provide a region for the location info
  of the endpoints in the endpoint deployment
- **tolerations**	<[]Object> Tolerations (optional) passed through to noobaa's pods


## NooBaa `Status`
The status section provides information about the health, phase, and readiness of the NooBaa system, including:
- **phase**: Current phase of the NooBaa system (e.g., `Ready`, `Failed`). When the system is fully operational, the phase will be `Ready`.
- **conditions**: A list of status conditions that provide more detailed information about the state of the system. Each condition includes:
    - `type`: The type of condition (e.g., `Available`, `Progressing`, `Degraded`).
    - `status`: The status of the condition (`True`, `False`, or `Unknown`).
    - `message`: A human-readable message describing the condition.
    - `reason`: A brief reason for the condition's last transition.
- **readme**: A welcome message and initial instructions for interacting with the NooBaa S3 service.
- **services**: Information about the services created by NooBaa, including their internal and external endpoints. This includes services for S3, management, and STS.
- **accounts**: Information about the created accounts, including the admin account and its corresponding secret.
- **endpoints**: Details about the S3 endpoint, including the number of ready endpoints and the virtual hosts.

## NooBaa CRD Specifications
Here is an example of a `NooBaa` custom resource:

```yaml
apiVersion: noobaa.io/v1alpha1
kind: NooBaa
metadata:
  name: noobaa
  namespace: noobaa
spec:
  dbType: postgres
  externalPgSSLUnauthorized: true
  externalPgSecret:
    name: noobaa-external-pg-db
    namespace: noobaa
  image: ghcr.io/kubeobj/noobaa-core:5.20-ac
  manualDefaultBackingStore: true
status:
  accounts:
    admin:
      secretRef:
        name: noobaa-admin
        namespace: noobaa
  actualImage: ghcr.io/kubeobj/noobaa-core:5.20-ac
  conditions:
  - lastHeartbeatTime: "2025-09-24T11:42:27Z"
    lastTransitionTime: "2025-09-24T11:42:27Z"
    message: noobaa operator completed reconcile - system is ready
    reason: SystemPhaseReady
    status: "True"
    type: Available
  - lastHeartbeatTime: "2025-09-24T11:42:27Z"
    lastTransitionTime: "2025-09-24T11:42:27Z"
    message: noobaa operator completed reconcile - system is ready
    reason: SystemPhaseReady
    status: "False"
    type: Progressing
  - lastHeartbeatTime: "2025-09-24T11:42:27Z"
    lastTransitionTime: "2025-09-19T06:10:09Z"
    message: noobaa operator completed reconcile - system is ready
    reason: SystemPhaseReady
    status: "False"
    type: Degraded
  - lastHeartbeatTime: "2025-09-24T11:42:27Z"
    lastTransitionTime: "2025-09-24T11:42:27Z"
    message: noobaa operator completed reconcile - system is ready
    reason: SystemPhaseReady
    status: "True"
    type: Upgradeable
  - lastHeartbeatTime: "2025-09-24T11:42:26Z"
    lastTransitionTime: "2025-09-19T06:10:09Z"
    status: k8s
    type: KMS-Type
  - lastHeartbeatTime: "2025-09-24T11:42:26Z"
    lastTransitionTime: "2025-09-19T06:10:09Z"
    status: Sync
    type: KMS-Status
  endpoints:
    readyCount: 1
    virtualHosts:
      - s3.noobaa.svc
  observedGeneration: 2
  phase: Ready
  readme: |
    Welcome to NooBaa!
    -----------------
    NooBaa Core Version:     5.20.0-11e425a
    NooBaa Operator Version: 5.20.0

    Lets get started:

    Test S3 client:

        kubectl port-forward -n noobaa service/s3 10443:443 &
        NOOBAA_ACCESS_KEY=$(kubectl get secret noobaa-admin -n noobaa -o json | jq -r '.data.AWS_ACCESS_KEY_ID|@base64d')
        NOOBAA_SECRET_KEY=$(kubectl get secret noobaa-admin -n noobaa -o json | jq -r '.data.AWS_SECRET_ACCESS_KEY|@base64d')
        alias s3='AWS_ACCESS_KEY_ID=$NOOBAA_ACCESS_KEY AWS_SECRET_ACCESS_KEY=$NOOBAA_SECRET_KEY aws --endpoint https://localhost:10443 --no-verify-ssl s3'
        s3 ls
  services:
    serviceMgmt:
      internalDNS:
        - https://noobaa-mgmt.noobaa.svc:443
      internalIP:
        - https://10.43.246.52:443
      nodePorts:
        - https://10.2.0.162:0
      podPorts:
        - https://10.42.0.106:8443
    serviceS3:
      externalIP:
        - https://10.2.0.162:443
      internalDNS:
        - https://s3.noobaa.svc:443
      internalIP:
        - https://10.43.125.228:443
      nodePorts:
        - https://10.2.0.162:32250
      podPorts:
        - https://10.42.0.107:6443
    serviceSts:
      internalDNS:
        - https://sts.noobaa.svc:443
      internalIP:
        - https://10.43.235.107:443
      nodePorts:
        - https://10.2.0.162:31884
      podPorts:
        - https://10.42.0.107:7443
    serviceSyslog: {}
```