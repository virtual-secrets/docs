---
title: Welcome | KubeObj
description: Welcome to KubeObj
menu:
  docs_{{ .version }}:
    identifier: readme-kubeobj
    name: Readme
    parent: welcome
    weight: -1
menu_name: docs_{{ .version }}
section_menu_id: welcome
url: /docs/{{ .version }}/welcome/
aliases:
  - /docs/{{ .version }}/
  - /docs/{{ .version }}/README/
---

# NooBaa

[NooBaa]() is an open-source, cloud-native data management and object storage platform. It enables unified data access, multi-cloud federation, and seamless scalability by abstracting underlying storage resources. NooBaa helps organizations manage, migrate, and protect data across on-premises and cloud environments efficiently.

## [Concepts](/docs/concepts/)
Concept explains some significant aspect of NooBaa. This is where you can learn about what NooBaa does and how it does it.

- [NooBaa Overview](/docs/concepts/what-is-noobaa/overview/index.md) Provides an introduction to NooBaa and gives an overview of the features it provides.
- [NooBaa API](/docs/concepts/crds/backingstore/index.md) Introduces NooBaa CRDs.


## [Setup](/docs/setup/)

Setup contains instructions for installing, uninstalling, and upgrading KubeStash.

- **Install NooBaa:** Provides instructions for installing NooBaa.
  - [Install](/docs/setup/install/index.md) How to install KubeStash.


## [Guides](/docs/guides/)

Guides section shows how to perform different operations with NooBaa.

- [Cloud Native Operations](/docs/guides/cloud-native-operation/index.md): Describes how to manage cloud-native experience for object storage management.
- [On Prem With Scalability](/docs/guides/on-prem-with-scalability/index.md): Explains how to deploy and scale NooBaa on-premises. 
- [Multi Cloud Federation](/docs/guides/multi-cloud-federation/index.md): Describes how to federate data across multiple clouds.
- [Unified Data Access](/docs/guides/unified-data-access/index.md): Guides on accessing data from various storage systems through a single endpoint.
- [Data Replication](/docs/guides/data-replication-policy/index.md): Provides instructions for setting up a highly available NooBaa deployment.
- [High Availability](/docs/guides/high-availability/index.md): Describes how we can achive highly available NooBaa deployment. 

We're always looking for help improving our documentation, so please don't hesitate to [file an issue](https://github.com/kubeobj/) if you see some problem.
