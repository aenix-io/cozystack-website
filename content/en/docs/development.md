---
title: Development
description: Cozystack Internals and Development
weight: 15
---

Cozystack Internals and Development information

## How it works

In short, cozystack is a seed container that bootstraps the entire platform. It consists of:

- **installer.sh** script: Contains minimal business logic, performs migrations, installs
  FluxCD, and prepares Kubernetes to run the platform chart.

- **platform chart**: A Helm chart that bootstraps the remaining configuration. It reads
  the state from the Kubernetes cluster using Helm lookup functions and templates
  all necessary manifests. The `installer.sh` script continuously runs the platform chart
  to ensure changes in the cluster are properly reconciled.

- **HTTP server**: Serves assets (Helm charts and Grafana dashboards) built from Cozystack repository.

- **FluxCD**: The main engine that applies all necessary Helm charts from the HTTP server
  to Kubernetes according to the configuration applied by the platform chart.

## Repository Structure

The main structure of the [cozystack](https://github.com/aenix-io/cozystack) repository is:

```shell
.
├── packages        # Main directory for cozystack packages
│   ├── core            # Core platform logic charts
│   ├── system          # System charts used to configure the system
│   ├── apps            # User-facing charts shown in the dashboard catalog
│   └── extra           # Tenant-specific applications that can be deployed as tenant options
├── dashboards      # Grafana dashboards
├── hack            # Helper scripts for local development
└── scripts         # Scripts mainly used by the cozystack container
    └── migrations  # Migrations between versions
```

Development can be done locally by modifying and updating files in this repository.

## Packages

### [core](https://github.com/aenix-io/cozystack/tree/main/packages/core)

Core packages are used to bootstrap Cozystack and its configuration itself.

It consists of two packages:

#### installer

Provides everything needed for the initial bootstrap of cozystack: the installer.sh script and an HTTP server with assets built from this repository.

#### platform

Reads the configuration from the cluster, templates the manifests, and applies them
back to the cluster.

It primarily reads the `cozystack` ConfigMap from `cozy-system` namespace, and templates
manifests according to the specified options. This ConfigMap
specifies the [bundle](/docs/bundles/) name and options, detailing which system components should be
installed in the cluster.

{{% alert color="info" %}}
Core packages do not use Helm to apply manifests; they are intended to be used only as `helm template . | kubectl apply -f -`.
{{% /alert %}}

### [system](https://github.com/aenix-io/cozystack/tree/main/packages/system)

System packages configure the system to manage and deploy user applications. The
necessary system components are specified in the bundle configuration. This can also
include components without an operator, whose installation can be triggered as part
of user applications.

{{% alert color="info" %}}
System packages use Helm to install and are managed by FluxCD.
{{% /alert %}}

### [apps](https://github.com/aenix-io/cozystack/tree/main/packages/apps)

These user-facing applications appear in the dashboard and include manifests to be applied to the cluster.

They should not contain business logic, because they are managed by operators installed from system.

### [extra](https://github.com/aenix-io/cozystack/tree/main/packages/extra)

Similar to `apps` but not shown in the application catalog. They can only be installed as part of a tenant.
They are allowed to use by bottom tenants installed in current tenant namespace.

Read more about [Tenant System](/docs/concepts/#tenant-system) on Core Concepts page.

It is possible to use only one application type within a single tenant namespace.

{{% alert color="info" %}}
Apps and extra packages use Helm for application and are installed from the dashboard and managed by FluxCD.
{{% /alert %}}

## Package Structure

Every package is a typical Helm chart containing all necessary images and manifests
for the platform. We follow an umbrella chart logic to keep upstream charts in the
`./charts` directory and override values.yaml in the application's root.
This structure simplifies upstream chart updates.

```
.
├── Chart.yaml  # Helm chart definition and parameter description
├── Makefile    # Common targets for simplifying local development
├── charts      # Directory for upstream charts
├── images      # Directory for Docker images
├── patches     # Optional directory for upstream chart patches
├── templates   # Additional manifests for the upstream Helm chart
└── values.yaml # Override values for the upstream Helm chart
```

## Development

Each application includes a Makefile to simplify the development process. We follow this logic for every package:

```shell
make update  # Update Helm chart and versions from the upstream source
make image   # Build Docker images used in the package
make show    # Show output of rendered templates
make diff    # Diff Helm release against objects in a Kubernetes cluster
make apply   # Apply Helm release to a Kubernetes cluster
```

For example, to update cilium:

```shell
cd packages/system/cilium         # Go to application directory
make update                       # Download new version from upstream
make image                        # Build cilium image
git diff .                        # Show diff with changed manifests
make diff                         # Show diff with applied cluster manifests
make apply                        # Apply changed manifests to the cluster 
kubectl get pod -n cozy-cilium    # Check if everything works as expected
git commit -m "Update cilium"     # Commit changes to the branch
```

To build the cozystack container with an updated chart:

```shell
cd packages/core/installer        # Go to the cozystack package
make image-cozystack              # Build cozystack image
make apply                        # Apply to the cluster
kubectl get pod -n cozy-system    # Check if everything works as expected
kubectl get hr -A                 # Check HelmRelease objects
```

After development, run E2E tests by calling `./hack/e2e.sh`.

{{% alert color="info" %}}
When rebuilding images, specify the `REGISTRY` environment variable to point to your Docker registry.

Feel free to look inside each Makefile to better understand the logic.
{{% /alert %}}
