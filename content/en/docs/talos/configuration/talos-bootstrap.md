---
title: Bootstrap a Talos Linux cluster for Cozystack using talos-bootstrap
linkTitle: talos-bootstrap
description: "Bootstrap a Talos Linux cluster for Cozystack using talos-bootstrap"
weight: 10
---

[talos-bootstrap](https://github.com/aenix-io/talos-bootstrap/) is an interactive script for bootstrapping Kubernetes clusters on Talos OS.

It was created by Ænix to simplify the installation of Talos Linux on bare metal nodes in a user-friendly manner.

## Install dependencies:

- `talosctl`
- `dialog`
- `nmap`

## Preparation

Create a new directory for holding your cluster configuration

```
mkdir cluster1
cd cluster1
```

Write configuration for Cozystack:

```yaml
cat > patch.yaml <<\EOT
machine:
  kubelet:
    nodeIP:
      validSubnets:
      - 192.168.100.0/24
    extraConfig:
      maxPods: 512
  kernel:
    modules:
    - name: openvswitch
    - name: drbd
      parameters:
        - usermode_helper=disabled
    - name: zfs
    - name: spl
  install:
    image: ghcr.io/aenix-io/cozystack/talos:v1.6.4
  files:
  - content: |
      [plugins]
        [plugins."io.containerd.grpc.v1.cri"]
          device_ownership_from_security_context = true
    path: /etc/cri/conf.d/20-customization.part
    op: create

cluster:
  network:
    cni:
      name: none
    dnsDomain: cozy.local
    podSubnets:
    - 10.244.0.0/16
    serviceSubnets:
    - 10.96.0.0/16
EOT

cat > patch-controlplane.yaml <<\EOT
cluster:
  allowSchedulingOnControlPlanes: true
  controllerManager:
    extraArgs:
      bind-address: 0.0.0.0
  scheduler:
    extraArgs:
      bind-address: 0.0.0.0
  apiServer:
    certSANs:
    - 127.0.0.1
  proxy:
    disabled: true
  discovery:
    enabled: false
  etcd:
    advertisedSubnets:
    - 192.168.100.0/24
EOT
```

Run [talos-bootstrap](https://github.com/aenix-io/talos-bootstrap/) to deploy the first node in a cluster:

```bash
talos-bootstrap install
```

{{% alert color="warning" %}}
:warning: If your nodes are running on an external network, you must specify each node explicitly in the argument:
```
talos-bootstrap install -n 1.2.3.4
```

Where `1.2.3.4` is the IP-address of your remote node.
{{% /alert %}}



{{% alert color="info" %}}
Talos-bootstrap will enable bootstrap on the first configured node in a cluster. If you want to rebootstrap the etcd cluster, simply remove the line `BOOTSTRAP_ETCD=false` from your `cluster.conf` file.
{{% /alert %}}

Repeat the step for the other nodes in a cluster.

Now follow **Get Started** guide starting from the [**Install Cozystack**](/docs/get-started/#install-cozystack) section, to continue the installation.



