---
title: Core Concepts
description: Core Concepts of Cozystack.
weight: 10
---

These are some core concepts in Cozystack.

## Platform Architecture

The core principle of the platform is that the end-user never has direct access to the API management cluster. Instead, the platform facilitates easy integration with the provider’s system, which in turn creates all the necessary services for the user. Then, it provides the credentials for user access to these services.

Besides managed services, the platform allows to bootstrap tenant Kubernetes clusters. The users have full access to their tenant Kubernetes clusters but have no access to the management cluster.

All controllers serving the user’s cluster are run externally from it, which allows to fully isolate the management cluster API from tenants and minimizes the potential for attacks on the management cluster.

Once access is granted to the tenant Kubernetes, the user can order physical volumes, load balancers, and eventually other services using tenant Kubernetes API.

![Cozystack for public cloud](/img/case-public-cloud.png)

All use cases are presented here:

* [**Using Cozystack to build public cloud**](/docs/use-cases/public-cloud/)  
  How to use Cozystack to build public cloud

* [**Using Cozystack to build private cloud**](/docs/use-cases/private-cloud/)  
  How to use Cozystack to build private cloud

* [**Using Cozystack as Kubernetes distribution**](/docs/use-cases/kubernetes-distribution/)
  How to use Cozystack as Kubernetes distribution


## Tenant system

A tenant is the main unit of security on the platform. The closest analogy would be Linux kernel namespaces.

Tenants can be created recursively and are subject to the following rules:

#### Higher-level tenants can access lower-level ones.

Higher-level tenants can view and manage the applications of all their children.

#### Each tenant has its own domain

By default (unless otherwise specified), it inherits the domain of its parent with a prefix of its name, for example, if the parent had the domain `example.org`, then `tenant-foo` would get the domain `foo.example.org` by default.

Kubernetes clusters created in this tenant namespace would get domains like: `kubernetes-cluster.foo.example.org`

Example:
```
tenant-root (example.org)
└── tenant-foo (foo.example.org)
    └── kubernetes-cluster1 (kubernetes-cluster1.foo.example.org)
```

#### Lower-level tenants can access the cluster services of their parent (provided they do not run their own)

Thus, you can create `tenant-u1` with a set of services like `etcd`, `ingress`, `monitoring`. And create another tenant namespace `tenant-u2` inside of `tenant-u1`.

Let's see what will happen when you run Kubernetes and Postgres under `tenant-u2` namesapce.

Since `tenant-u2` does not have its own cluster services like `etcd`, `ingress`, and `monitoring`, the applications will use the cluster services of the parent tenant.  
This in turn means:

- The Kubernetes cluster data will be stored in etcd for `tenant-u1`.
- Access to the cluster will be through the common ingress of `tenant-u1`.
- Essentially, all metrics will be collected in the monitoring from `tenant-u1`, and only it will have access to them.


Example:
```
tenant-u1
├── etcd
├── ingress
├── monitoring
└── tenant-u2
    ├── kubernetes-cluster1
    └── postgres-db1
```
