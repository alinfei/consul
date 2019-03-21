--- 
layout: "docs"
page_title: "Kubernetes Consul Reference Architecture"
sidebar_current: "docs-guides-k8s-reference-architecture"
description: |-
  This document provides recommended practices and a reference architecture. 
---

# Consul and Kubernetes Reference Architecture

Preparing your Kubernetes cluster to successfully deploy and run Consul is an
important first step in your production deployment process. In this guide you
will prepare your Kubernetes cluster, independent of its underlying platform
(AKS, EKS, GCP, etc). However, we will call out cloud specific differences when
applicable. Before starting this guide you should have experience with
Kubernetes, and have `kubectl` and helm configured locally. 

By the end of this guide, you will be able to select the right resource limits
for Consul pods, select the Consul datacenter design that meets your use case,
and understand the minimum networking requirements. 

## Infrastructure Requirements

Consul server agents are responsible for the cluster state, responding to RPC
queries, and processing all write operations. Since the Consul servers are
highly active and are responsible for maintaining the cluster state, server
sizing is critical for the overall performance efficiency and health of the
Consul cluster. Review the [Consul Reference
Architecture](/advanced/day-1-operations/reference-architecture#consul-servers)
guide for sizing recommendations for small and large Consul datacenters. 

The CPU and memory recommendations can be used when you select the resources
limits for the Consul pods. The disk recommendations can also be used when
selecting the resources limits and configuring persistent volumes. You will
need to set both `limits` and `required` in the Helm chart. Below is an example
Helm chart for a Consul server in a large environment.

```yaml 
server 
  resources: | 
    requests: 
      memory: "32Gi" 
      cpu: "4" 
      disk: "50Gi"
   limits: 
     memory: "32Gi"
     cpu: "4" 
     disk: "50Gi" 
```

You should also set [resource limits for Consul
clients](https://www.consul.io/docs/platform/k8s/helm.html#v-client-resources),
so that the client pods do not unexpectedly consume more resources than
expected. 

[Persistent
volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) (PV)
allow you to have a fixed disk location for the Consul data. This ensures that
if a Consul server is lost, the data will not be lost. This is an important
feature of Kubernetes, but may take some additional configuration. If you are
running Kubernetes on one of the major cloud platforms, persistent volumes
should already be configured; be sure to read their documentation for more
details. In addition to setting up the PV resource in Kubernetes, you will need
to map the Consul server to that volume with the [storage class
parameter](https://www.consul.io/docs/platform/k8s/helm.html#v-server-storageclass).

Finally, you will need to enable RBAC on your Kubernetes cluster. Review
[Kubernetes
RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/),
[AWS](https://docs.aws.amazon.com/eks/latest/userguide/managing-auth.html),
[GCP](https://cloud.google.com/kubernetes-engine/docs/how-to/role-based-access-control),
and
[Azure](https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest#az-aks-create).
In Azure, RBAC is enabled by default . 

## Datacenter Design 

There are many configurations for running Consul with Kubernetes. In this guide
we will cover three of the most common.

1. Consul agents can be solely deployed within kubernetes.  
1. Consul servers
can be deployed outside of kubernetes and clients inside of Kubernetes.  
1. Multiple Consul datacenters with agents inside and outside of Kubernetes.  

Review the Kubernetes
[documentation](https://www.consul.io/docs/platform/k8s/index.html#use-cases)
for additional use case information. 

Since all three use cases will also need catalog sync, review the
implementation details for catalog sync.  

### Consul Datacenter Deployed in Kubernetes 

Deploying a Consul cluster, servers and clients, in Kubernetes can be done with
the official [Helm
chart](https://www.consul.io/docs/platform/k8s/helm.html#using-the-helm-chart).
This configuration is useful for managing services within Kubernetes and is
common for users who do not already have a production Consul datacenter.

![Reference Diagram](/assets/images/k8s-consul-simple.png "Consul in Kubernetes Reference Diagram")

The Consul datacenter in Kubernetes will function the same as a platform
independent Consul datacenter. Agents will communicate over LAN Gossip, servers
will participate in the Raft consensus and quorum, and client requests will be
forwarded to the leading server with the RPC.

### Consul Datacenter with a Kubernetes Cluster

To use an existing Consul cluster to manage services in Kubernetes, Consul
clients can be deployed within the Kubernetes cluster. This will also allow the
Kubernetes services to be synced to Consul. This design allows Consul tools
such as envconsul, consul-template, and more to work on Kubernetes. It will
also register each Kubernetes node with the Consul catalog for full visibility
into your infrastructure.

![Reference Diagram](/assets/images/k8s-cluster-consul-datacenter.png "Consul and Kubernetes Reference Diagram")

This type of deployment in Kubernetes can also be set up with the official Helm
chart.


### Multiple Consul Clusters with a Kubernetes Cluster

Consul clusters in different datacenters running the same service can be joined
by WAN links. The clusters can operate independently and only communicate over
the WAN. This type datacenter design is detailed in the [Reference Architecture
guide](/advanced/day-1-operations/reference-architecture#multiple-datacenters).
In this setup, you can have a Consul cluster running outside of Kubernetes and
a Consul cluster running inside of Kubernetes. 

### Catalog Sync

To use catalog sync, you must enable it in the [Helm
chart](https://www.consul.io/docs/platform/k8s/helm.html#v-synccatalog).
Catalog sync allows you to sync services between Consul and Kubernetes. The
sync can be unidirectional, in either direction, or bidirectional. Read the
[documentation](https://www.consul.io/docs/platform/k8s/service-sync.html) to
learn more about the configuration. 

Services synced from Kubernetes to Consul will be discoverable, like any other
service within the Consul datacenter. Read more in the [network
connectivity](#networking-connectivity) section to learn more about related
Kubernetes configuration. Services synced from Consul to Kubernetes will be
discoverable with the built-in Kubernetes DNS once a [Consul stub
domain](https://www.consul.io/docs/platform/k8s/dns.html) is deployed. When
bidirectional catalog sync is enabled, it will behave like the two
unidirectional setups. 

## Networking Connectivity 

When running Consul inside Kubernetes as a pod, the Consul servers will be
automatically configured with the appropriate addresses. However, when running
Consul servers outside of the Kubernetes cluster and clients inside Kubernetes
as pods, there are additional [networking
considerations](/consul/advanced/day-1-operations/reference-architecture#network-connectivity).

### Network Connectivity for Services

When using Consul catalog sync, to sync Kubernetes services to Consul, you will
need to ensure the Kubernetes services are supported [service
types](https://www.consul.io/docs/platform/k8s/service-sync.html#kubernetes-service-types)
and configure correctly in Kubernetes. If the service is configured correctly,
it will be discoverable by Consul like any other service in the datacenter. 

### Network Security

Finally, you should consider securing your Consul datacenter with
[ACLs](bootrapping). ACLs should be used with [Consul
Connect](https://www.consul.io/docs/platform/k8s/connect.html) to secure
service to service communication. The Kubernetes cluster should also be
secured. 

## Summary 

You are now prepared to deploy Consul with Kubernetes. In this
guide, you were introduced to several a datacenter design for a variety of use
cases. This guide also outlined the Kubernetes prerequisites, resource
requirements for Consul, and networking considerations. Continue onto the
[Deploying Consul with Kubernetes
guide](https://learn.hashicorp.com/consul/getting-started-k8s/helm-deploy) for
information on deploying Consul with the official Helm chart or continue
reading about Consul Operations in the [Day 1 Path](https://learn.hashicorp.com/consul/?track=advanced#advanced). 
 