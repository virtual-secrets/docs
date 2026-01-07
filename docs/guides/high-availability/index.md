---
title: High Availability
menu:
  docs_{{ .version }}:
    identifier: index-high-availability
    name: High Availability
    parent: guides
    weight: 1
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# High Availability in NooBaa

High Availability (HA) is critical for any production storage system. A highly available NooBaa deployment ensures that your data services remain online and accessible even if individual components or nodes fail. This is achieved through redundancy at multiple levels: core services, metadata database, and data placement.

### 1. Core Component High Availability

NooBaa's core services, such as the main server and the S3 endpoint, can be configured for high availability. If one instance fails, Kubernetes automatically directs traffic to the healthy ones, ensuring service continuity.

- The NooBaa server (core) can be scaled by setting a fixed number of replicas.
- The S3 endpoint is managed by a Horizontal Pod Autoscaler (HPA) and scales automatically between a minimum and maximum count based on CPU utilization.

You can configure these settings directly in the NooBaa custom resource.

```yaml
apiVersion: noobaa.io/v1alpha1
kind: NooBaa
metadata:
  name: noobaa
  namespace: noobaa
spec:
  coreResources:
    requests:
      cpu: "1"
      memory: "4Gi"
    limits:
      cpu: "2"
      memory: "4Gi"
      
  # Configure autoscaling for the S3 endpoint
  endpoints:
    minCount: 2
    maxCount: 4
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "1"
        memory: "1Gi"
```

Here,
- `endpoints.minCount:` Sets the minimum number of S3 endpoint pods. This ensures at least two pods are running to handle traffic even under low load.
- `endpoints.maxCount:` Sets the maximum number of S3 endpoint pods the HPA can scale up to during high traffic periods.


### 2. Database High Availability

NooBaa stores all its metadata—about buckets, objects, accounts, and system configuration—in a PostgreSQL database.
The availability of this database is paramount for the entire NooBaa system. As we're using KubeDB managed PostgreSQL, the database is automatically backed up and restored in case of a failure.
