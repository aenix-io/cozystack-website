---
title: "OIDC Server"
linkTitle: "OIDC Server"
description: "OIDC Server"
weight: 35
---

## Prerequisites

1. **OIDC Configuration**  
   Your API server must be configured to use OIDC. If you are using Talos Linux, your machine configuration should include the following parameters:

   ```yaml
   cluster:
     apiServer:
       extraArgs:
         oidc-issuer-url: "https://keycloak.example.org/realms/cozy"
         oidc-client-id: "kubernetes"
         oidc-username-claim: "preferred_username"
         oidc-groups-claim: "groups"
   ```

2. **Domain Reachability**  
   Ensure that the domain `keycloak.example.org` is accessible from the cluster and resolves to your root ingress controller.

3. **Storage Configuration**  
   Storage must be properly configured.

## Configuration

If all prerequisites are met, you can proceed with the configuration steps.

### Step 1: Enable OIDC in Cozystack

Edit your Cozystack ConfigMap to enable OIDC:

```bash
kubectl patch -n cozy-system configmap cozystack --type=merge -p '{"data":{"oidc-enabled": "true"}}'
```

Within one minute, CozyStack will reconcile the ConfigMap and create three new `HelmRelease` resources:

```bash
# kubectl get hr -n cozy-keycloak
cozy-keycloak                    keycloak                    26s    Unknown   Running 'install' action with a timeout of 5m0s
cozy-keycloak                    keycloak-configure          26s    False     dependency 'cozy-keycloak/keycloak-operator' is not ready
cozy-keycloak                    keycloak-operator           26s    False     dependency 'cozy-keycloak/keycloak' is not ready
```

### Step 2: Wait for Installation Completion  

Wait until all resources are successfully installed and reach the `Ready` state:

```bash
NAME                 AGE     READY   STATUS
keycloak             2m19s   True    Helm install succeeded for release cozy-keycloak/keycloak.v1 with chart cozy-keycloak@0.19.0
keycloak-configure   2m19s   True    Helm install succeeded for release cozy-keycloak/keycloak-configure.v1 with chart cozy-keycloak-configure@0.19.0
keycloak-operator    2m19s   True    Helm install succeeded for release cozy-keycloak/keycloak-operator.v1 with chart cozy-keycloak-operator@0.19.0
```

<!-- TODO: automate this -->
Reconcile tenants:

```
kubectl annotate -n tenant-root hr/tenant-root reconcile.fluxcd.io/forceAt=$(date +"%Y-%m-%dT%H:%M:%SZ") --overwrite
```

### Step 3: Access Keycloak  

You can now access Keycloak at `https://keycloak.example.org` (replace `example.org` with your infrastructure domain).

To get the Keycloak credentials, run the following command:

```bash
kubectl get secret -o yaml -n cozy-keycloak keycloak-credentials -o go-template='{{ printf "%s\n" (index .data "password" | base64decode) }}'
```

1. **Create a User in the Cozy Realm**  
   Follow the [Keycloak documentation](https://www.keycloak.org/docs/latest/server_admin/index.html#proc-creating-user_server_administration_guide) to create a user in the Cozy realm.

2. **Add User to the `kubeapps-admin` Group**  
   Assign the user to the `kubeapps-admin` group.

### Step 4: Get Kubeconfig

Now access the cluster via Dashboard 