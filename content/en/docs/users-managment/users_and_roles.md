---
title: Creating users and add roles for them
linkTitle: Users and roles
description: "How to create users and add roles for them"
weight: 30
---

Creating users and add roles for them

### Overview

When a tenant is created in Cozy (starting with version 1.6.0), roles, RoleBindings and keycloak groups will automatically be created in the Kubernetes cluster.

## Assigning a Role to a User for a Tenant

1. **Access Keycloak**:
   To retrieve login credentials, check the secret by running the following command:
   ```bash
   kubectl get secret keycloak-credentials -n cozy-keycloak -o yaml
   ```
   **Keycloak Address**:
   The Keycloak address will match the value specified in the cozystack ConfigMap. For example, if your ConfigMap looks like this:
   ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
    name: cozystack
    namespace: cozy-system
    data:
    root-host: infra.example.org
    api-server-adress: 55.21.33.22
    bundle-name: "paas-full"
    ipv4-pod-cidr: "10.244.0.0/16"
    ipv4-pod-gateway: "10.244.0.1"
    ipv4-svc-cidr: "10.96.0.0/16"
    ipv4-join-cidr: "100.64.0.0/16"
   ```
   Then Keycloak will be available at: `keycloak.infra.example.org`

## Configure Roles for Each Tenant in Cozy:

- **`tenant-abc-view`**
  - Read-only access to resources from our API.
  - Ability to view logs.

- **`tenant-abc-use`**
  - All previous permissions
  - VNC access for virtual machines.

- **`tenant-abc-admin`**
  - All previous permissions
  - Ability to delete pods, along with all permissions from `tenant-abc-use`.
  - Ability to create, update, and delete resources from our API (excluding `tenant`, `monitoring`, `etcd`, `ingress`).

- **`tenant-abc-super-admin`**
  - All previous permissions
  - Ability to create, update, and delete `tenant`, `monitoring`, `etcd`, and `ingress`.
