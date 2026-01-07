---
title: NooBaa
menu:
  docs_{{ .version }}:
    identifier: index-overview
    name: NooBaa
    parent: what-is-noobaa-concepts
    weight: 1
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

# NooBaa

[NooBaa]() is an open-source, cloud-native data management and object storage platform. It enables unified data access, multi-cloud federation, and seamless scalability by abstracting underlying storage resources. NooBaa helps organizations manage, migrate, and protect data across on-premises and cloud environments efficiently.

## Features

| Features                           | Scope                                                                                      |
|------------------------------------|--------------------------------------------------------------------------------------------|
| Unified Data Access                | Access data from multiple sources (cloud, on-prem, S3, etc.) via a single API              |
| Multi-Cloud Federation             | Federate and manage data across multiple cloud providers and on-premises storage            |
| Scalability                        | Scale storage resources dynamically to meet growing data demands                            |
| Data Replication Policy            | Replicate data across different locations for high availability and disaster recovery       |
| High Availability                  | Ensure continuous data access and service uptime through redundancy and failover            |
| Data Migration                     | Seamlessly migrate data between different storage backends                                  |
| Policy-Based Data Management       | Apply policies for data placement, lifecycle, and access control                            |
| Monitoring and Analytics           | Gain insights into data usage, performance, and health with built-in monitoring tools       |
| Kubernetes Native                  | Integrates natively with Kubernetes for deployment, management, and automation              |

### Custom Resources

NooBaa uses [Custom Resource Definitions (CRDs)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) to manage its components and configuration in a Kubernetes-native way. This section provides an overview of the key custom resources used by NooBaa.

- **NooBaa**: This is the central CRD for a NooBaa deployment. It represents the entire NooBaa system and orchestrates all its components. Administrators interact with this resource to install, configure, and manage the NooBaa platform within a Kubernetes cluster.
- **BackingStore**: A `BackingStore` represents underlying storage resources that NooBaa can use to store data. These can be cloud-based storage services (like AWS S3, Azure Blob Storage, or Google Cloud Storage) or on-premises storage systems (like a PV pool). `BackingStore`s are used to create buckets and provide the physical storage layer.
- **NamespaceStore**: A `NamespaceStore` is a special type of `BackingStore` that represents an entire external bucket or a prefix within a bucket. Instead of using the external storage as a backing for NooBaa's own data services (like deduplication and compression), a `NamespaceStore` provides a direct, transparent connection to the external resource. This is useful for data federation and accessing existing data in place.
- **BucketClass**: A `BucketClass` defines policies for how data is placed and managed across one or more `BackingStore`s. It can specify data placement policies like mirroring or spreading data across different stores. When a user requests a new bucket, they can specify a `BucketClass` to control the behavior and characteristics of their bucket.
- **ObjectBucketClaim (OBC)**: An `ObjectBucketClaim` (OBC) is a Kubernetes-native way for a user or application to request a new object storage bucket, similar to how a `PersistentVolumeClaim` (PVC) requests block or file storage. When an OBC is created, the NooBaa operator provisions a corresponding `ObjectBucket` and makes the connection details available to the application via a `Secret` and a `ConfigMap`.


## Next Steps
