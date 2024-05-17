---
title: Bootstrap a Talos Linux cluster for Cozystack using Talm
linkTitle: Talm
description: "Bootstrap a Talos Linux cluster for Cozystack using Talm"
weight: 20
---

[Talm](https://github.com/aenix-io/talm) is a Helm-like utility for declarative configuration management of Talos Linux.

It was created by Ænix to allow more declarative and custom configurations for cluster management.

Talm comes with pre-built presets for Cozystack.

Initialize the new project:

```
mkdir -p cluster1
cd cluster1
talm init --preset cozystack
```

The structure of the project mostly mirrors an ordinary Helm chart:

- `charts` - a directory that includes a common library chart with functions used for querying information from Talos Linux.
- `Chart.yaml` - a file containing the common information about your project; the name of the chart is used as the name for the newly created cluster.
- `templates` - a directory used to describe templates for the configuration generation.
- `secrets.yaml` - a file containing secrets for your cluster.
- `values.yaml` - a common values file used to provide parameters for the templating.
- `nodes` - an optional directory used to describe and store generated configuration for nodes.

You're free to edit the files: `Chart.yaml`, `values.yaml`, and `templates/*` to meet your environment requirements.

Now, create a nodes directory and collect the information from your node into a specific file:

```bash
talm template -e 1.2.3.4 -n 1.2.3.4 -t templates/controlplane.yaml -i > nodes/srv1.yaml
```

Where `1.2.3.4` is the IP address of your remote node.

Check the generated file, and if everything is okay, apply it:

```bash
talm apply -f nodes/srv1.yaml -i
```
In future operations, you can also use the following options:

- `--dry-run` - dry run mode will show a diff with the existing configuration.
- `-m try` - try mode will rollback the configuration in 1 minute.

```bash
talm bootstrap -f nodes/srv1.yaml
```

To access the cluster, generate an admin kubeconfig:

```bash
talm kubeconfig kubeconfig -f nodes/srv1.yaml
export KUBECONFIG=$PWD/kubeconfig
```

Now follow **Get Started** guide starting from the [**Install Cozystack**](/docs/get-started/#install-cozystack) section, to continue the installation.
