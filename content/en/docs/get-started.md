---
title: "Get Started with Cozystack"
linkTitle: "Get Started"
description: "Get Started with Cozystack."
weight: 30
---

This tutorial shows you how to bootstrap Cozystack on a few servers in your infrastructure

## Before you begin

![Cozystack deployment](/img/cozystack-deployment.png)

You need 3 physical servers or VMs with nested virtualisation:

```yaml
CPU: 4 cores
CPU model: host
RAM: 8-16 GB
HDD1: 32 GB
HDD2: 100GB (raw)
```

And one management VM or physical server connected to the same network.
Any Linux system installed on it (eg. Ubuntu should be enough)

{{% alert color="warning" %}}
:warning: This VM should support `x86-64-v2` architecture, the most probably you can achieve this by setting cpu model to `host`
{{% /alert %}}

## Objectives

- Bootstrap Cozystack on three servers using PXE
- Configure Storage
- Configure Networking interconnection
- Access Cozystack dashboard
- Deploy etcd, ingress and monitoring stack

## Install dependencies:

- `docker`
- `talosctl`
- `dialog`
- `nmap`
- `make`
- `yq`
- `kubectl`
- `helm`

## Netboot server

Start matchbox with prebuilt Talos image for Cozystack:

```bash
sudo docker run --name=matchbox -d --net=host ghcr.io/aenix-io/cozystack/matchbox:v0.6.0 \
  -address=:8080 \
  -log-level=debug
```

Start DHCP-Server:
```bash
sudo docker run --name=dnsmasq -d --cap-add=NET_ADMIN --net=host quay.io/poseidon/dnsmasq \
  -d -q -p0 \
  --dhcp-range=192.168.100.3,192.168.100.254 \
  --dhcp-option=option:router,192.168.100.1 \
  --enable-tftp \
  --tftp-root=/var/lib/tftpboot \
  --dhcp-match=set:bios,option:client-arch,0 \
  --dhcp-boot=tag:bios,undionly.kpxe \
  --dhcp-match=set:efi32,option:client-arch,6 \
  --dhcp-boot=tag:efi32,ipxe.efi \
  --dhcp-match=set:efibc,option:client-arch,7 \
  --dhcp-boot=tag:efibc,ipxe.efi \
  --dhcp-match=set:efi64,option:client-arch,9 \
  --dhcp-boot=tag:efi64,ipxe.efi \
  --dhcp-userclass=set:ipxe,iPXE \
  --dhcp-boot=tag:ipxe,http://192.168.100.250:8080/boot.ipxe \
  --log-queries \
  --log-dhcp
```

Where:
- `192.168.100.3,192.168.100.254` range to allocate IPs from
- `192.168.100.1` your gateway
- `192.168.100.250` is address of your management server

Check status of containers:

```
docker ps
```

example output:

```console
CONTAINER ID   IMAGE                                        COMMAND                  CREATED          STATUS          PORTS     NAMES
22044f26f74d   quay.io/poseidon/dnsmasq                     "/usr/sbin/dnsmasq -…"   6 seconds ago    Up 5 seconds              dnsmasq
231ad81ff9e0   ghcr.io/aenix-io/cozystack/matchbox:v0.6.0   "/matchbox -address=…"   58 seconds ago   Up 57 seconds             matchbox
```

## Bootstrap cluster

Write configuration for Cozystack:

```yaml
cat > patch.yaml <<\EOT
machine:
  kubelet:
    nodeIP:
      validSubnets:
      - 192.168.100.0/24
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

Run [talos-bootstrap](https://github.com/aenix-io/talos-bootstrap/) to deploy cluster:

```bash
talos-bootstrap install
```

Save admin kubeconfig to access your Kubernetes cluster:
```bash
cp -i kubeconfig ~/.kube/config
```

Check connection:
```bash
kubectl get ns
```

example output:
```console
NAME              STATUS   AGE
default           Active   7m56s
kube-node-lease   Active   7m56s
kube-public       Active   7m56s
kube-system       Active   7m56s
```


{{% alert color="warning" %}}
:warning: All nodes should currently show as `READY: False`, don't worry about that, this is because you disabled the default CNI plugin in the previous step. Cozystack will install it's own CNI-plugin on the next step.
{{% /alert %}}

## Install Cozystack

write config for cozystack, refer to [bundles](/docs/bundles/) documentation for configuration parameters

{{% alert color="warning" %}}
:warning: please make sure that you written the same setting specified in `patch.yaml` and `patch-controlplane.yaml` files.
{{% /alert %}}

```yaml
cat > cozystack-config.yaml <<\EOT
apiVersion: v1
kind: ConfigMap
metadata:
  name: cozystack
  namespace: cozy-system
data:
  bundle-name: "paas-full"
  ipv4-pod-cidr: "10.244.0.0/16"
  ipv4-pod-gateway: "10.244.0.1"
  ipv4-svc-cidr: "10.96.0.0/16"
  ipv4-join-cidr: "100.64.0.0/16"
EOT
```

Create namesapce and install Cozystack system components:

```bash
kubectl create ns cozy-system
kubectl apply -f cozystack-config.yaml
kubectl apply -f https://github.com/aenix-io/cozystack/raw/v0.6.0/manifests/cozystack-installer.yaml
```

(optional) You can track the logs of installer:
```bash
kubectl logs -n cozy-system deploy/cozystack -f
```

Wait for a while, then check the status of installation:
```bash
kubectl get hr -A
```

Wait until all releases become to `Ready` state:
```console
NAMESPACE                        NAME                        AGE    READY   STATUS
cozy-cert-manager                cert-manager                4m1s   True    Release reconciliation succeeded
cozy-cert-manager                cert-manager-issuers        4m1s   True    Release reconciliation succeeded
cozy-cilium                      cilium                      4m1s   True    Release reconciliation succeeded
cozy-cluster-api                 capi-operator               4m1s   True    Release reconciliation succeeded
cozy-cluster-api                 capi-providers              4m1s   True    Release reconciliation succeeded
cozy-dashboard                   dashboard                   4m1s   True    Release reconciliation succeeded
cozy-grafana-operator            grafana-operator            4m1s   True    Release reconciliation succeeded
cozy-kamaji                      kamaji                      4m1s   True    Release reconciliation succeeded
cozy-kubeovn                     kubeovn                     4m1s   True    Release reconciliation succeeded
cozy-kubevirt-cdi                kubevirt-cdi                4m1s   True    Release reconciliation succeeded
cozy-kubevirt-cdi                kubevirt-cdi-operator       4m1s   True    Release reconciliation succeeded
cozy-kubevirt                    kubevirt                    4m1s   True    Release reconciliation succeeded
cozy-kubevirt                    kubevirt-operator           4m1s   True    Release reconciliation succeeded
cozy-linstor                     linstor                     4m1s   True    Release reconciliation succeeded
cozy-linstor                     piraeus-operator            4m1s   True    Release reconciliation succeeded
cozy-mariadb-operator            mariadb-operator            4m1s   True    Release reconciliation succeeded
cozy-metallb                     metallb                     4m1s   True    Release reconciliation succeeded
cozy-monitoring                  monitoring                  4m1s   True    Release reconciliation succeeded
cozy-postgres-operator           postgres-operator           4m1s   True    Release reconciliation succeeded
cozy-rabbitmq-operator           rabbitmq-operator           4m1s   True    Release reconciliation succeeded
cozy-redis-operator              redis-operator              4m1s   True    Release reconciliation succeeded
cozy-telepresence                telepresence                4m1s   True    Release reconciliation succeeded
cozy-victoria-metrics-operator   victoria-metrics-operator   4m1s   True    Release reconciliation succeeded
tenant-root                      tenant-root                 4m1s   True    Release reconciliation succeeded
```

## Configure Storage

Setup alias to access LINSTOR:
```bash
alias linstor='kubectl exec -n cozy-linstor deploy/linstor-controller -- linstor'
```

list your nodes
```bash
linstor node list
```

example output:

```console
+-------------------------------------------------------+
| Node | NodeType  | Addresses                 | State  |
|=======================================================|
| srv1 | SATELLITE | 192.168.100.11:3367 (SSL) | Online |
| srv2 | SATELLITE | 192.168.100.12:3367 (SSL) | Online |
| srv3 | SATELLITE | 192.168.100.13:3367 (SSL) | Online |
+-------------------------------------------------------+
```

list empty devices:

```bash
linstor physical-storage list
```

example output:
```console
+--------------------------------------------+
| Size         | Rotational | Nodes          |
|============================================|
| 107374182400 | True       | srv3[/dev/sdb] |
|              |            | srv1[/dev/sdb] |
|              |            | srv2[/dev/sdb] |
+--------------------------------------------+
```


create storage pools:




{{< tabs name="create_storage_pools" >}}
{{% tab name="ZFS" %}}
```bash
linstor ps cdp zfs srv1 /dev/sdb --pool-name data --storage-pool data
linstor ps cdp zfs srv2 /dev/sdb --pool-name data --storage-pool data
linstor ps cdp zfs srv3 /dev/sdb --pool-name data --storage-pool data
```
{{% /tab %}}

{{% tab name="LVM" %}}
```bash
linstor ps cdp lvm srv1 /dev/sdb --pool-name data --storage-pool data
linstor ps cdp lvm srv2 /dev/sdb --pool-name data --storage-pool data
linstor ps cdp lvm srv3 /dev/sdb --pool-name data --storage-pool data
```
{{% /tab %}}
{{< /tabs >}}

list storage pools:

```bash
linstor sp l
```

example output:

```console
+-------------------------------------------------------------------------------------------------------------------------------------+
| StoragePool          | Node | Driver   | PoolName | FreeCapacity | TotalCapacity | CanSnapshots | State | SharedName                |
|=====================================================================================================================================|
| DfltDisklessStorPool | srv1 | DISKLESS |          |              |               | False        | Ok    | srv1;DfltDisklessStorPool |
| DfltDisklessStorPool | srv2 | DISKLESS |          |              |               | False        | Ok    | srv2;DfltDisklessStorPool |
| DfltDisklessStorPool | srv3 | DISKLESS |          |              |               | False        | Ok    | srv3;DfltDisklessStorPool |
| data                 | srv1 | ZFS      | data     |    96.41 GiB |     99.50 GiB | True         | Ok    | srv1;data                 |
| data                 | srv2 | ZFS      | data     |    96.41 GiB |     99.50 GiB | True         | Ok    | srv2;data                 |
| data                 | srv3 | ZFS      | data     |    96.41 GiB |     99.50 GiB | True         | Ok    | srv3;data                 |
+-------------------------------------------------------------------------------------------------------------------------------------+
```


Create default storage classes:
```yaml
kubectl create -f- <<EOT
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: linstor.csi.linbit.com
parameters:
  linstor.csi.linbit.com/storagePool: "data"
  linstor.csi.linbit.com/layerList: "storage"
  linstor.csi.linbit.com/allowRemoteVolumeAccess: "false"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: replicated
provisioner: linstor.csi.linbit.com
parameters:
  linstor.csi.linbit.com/storagePool: "data"
  linstor.csi.linbit.com/autoPlace: "3"
  linstor.csi.linbit.com/layerList: "drbd storage"
  linstor.csi.linbit.com/allowRemoteVolumeAccess: "true"
  property.linstor.csi.linbit.com/DrbdOptions/auto-quorum: suspend-io
  property.linstor.csi.linbit.com/DrbdOptions/Resource/on-no-data-accessible: suspend-io
  property.linstor.csi.linbit.com/DrbdOptions/Resource/on-suspended-primary-outdated: force-secondary
  property.linstor.csi.linbit.com/DrbdOptions/Net/rr-conflict: retry-connect
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOT
```

list storageclasses:

```bash
kubectl get storageclasses
```

example output:
```console
NAME              PROVISIONER              RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local (default)   linstor.csi.linbit.com   Delete          WaitForFirstConsumer   true                   11m
replicated        linstor.csi.linbit.com   Delete          WaitForFirstConsumer   true                   11m
```

## Configure Networking interconnection

To access your services select the range of unused IPs, eg. `192.168.100.200-192.168.100.250`

{{% alert color="warning" %}}
:warning: these IPs should be from the same network as nodes or they should have all necessary routes for them.
{{% /alert %}}

Configure MetalLB to use and announce this range:
```yaml
kubectl create -f- <<EOT
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: cozystack
  namespace: cozy-metallb
spec:
  ipAddressPools:
  - cozystack
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: cozystack
  namespace: cozy-metallb
spec:
  addresses:
  - 192.168.100.200-192.168.100.250
  autoAssign: true
  avoidBuggyIPs: false
EOT
```

## Setup basic applications

Get token from `tenant-root`:
```bash
kubectl get secret -n tenant-root tenant-root -o go-template='{{ printf "%s\n" (index .data "token" | base64decode) }}'
```

Enable port forward to cozy-dashboard:
```bash
kubectl port-forward -n cozy-dashboard svc/dashboard 8000:80
```

Open: http://localhost:8000/

- Select `tenant-root`
- Click `Upgrade` button
- Into `host` section write a domain which you're going to use as parent domain for all deployed applications

  {{% alert color="warning" %}}
  :warning: if you have no domain yet, you can use `192.168.100.200.nip.io` where `192.168.100.200` is a first IP address in your network addresses range.

   alternatively you can leave the default value, however you'll be need to modify your `/etc/hosts` every time you want to access specific application.
  {{% /alert %}}

- Set `etcd`, `monitoring` and `ingress` to enabled position
- Click Deploy


Check persistent volumes provisioned:

```bash
kubectl get pvc -n tenant-root
```

example output:
```console
NAME                                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
data-etcd-0                              Bound    pvc-4cbd29cc-a29f-453d-b412-451647cd04bf   10Gi       RWO            local          <unset>                 2m10s
data-etcd-1                              Bound    pvc-1579f95a-a69d-4a26-bcc2-b15ccdbede0d   10Gi       RWO            local          <unset>                 115s
data-etcd-2                              Bound    pvc-907009e5-88bf-4d18-91e7-b56b0dbfb97e   10Gi       RWO            local          <unset>                 91s
grafana-db-1                             Bound    pvc-7b3f4e23-228a-46fd-b820-d033ef4679af   10Gi       RWO            local          <unset>                 2m41s
grafana-db-2                             Bound    pvc-ac9b72a4-f40e-47e8-ad24-f50d843b55e4   10Gi       RWO            local          <unset>                 113s
vmselect-cachedir-vmselect-longterm-0    Bound    pvc-622fa398-2104-459f-8744-565eee0a13f1   2Gi        RWO            local          <unset>                 2m21s
vmselect-cachedir-vmselect-longterm-1    Bound    pvc-fc9349f5-02b2-4e25-8bef-6cbc5cc6d690   2Gi        RWO            local          <unset>                 2m21s
vmselect-cachedir-vmselect-shortterm-0   Bound    pvc-7acc7ff6-6b9b-4676-bd1f-6867ea7165e2   2Gi        RWO            local          <unset>                 2m41s
vmselect-cachedir-vmselect-shortterm-1   Bound    pvc-e514f12b-f1f6-40ff-9838-a6bda3580eb7   2Gi        RWO            local          <unset>                 2m40s
vmstorage-db-vmstorage-longterm-0        Bound    pvc-e8ac7fc3-df0d-4692-aebf-9f66f72f9fef   10Gi       RWO            local          <unset>                 2m21s
vmstorage-db-vmstorage-longterm-1        Bound    pvc-68b5ceaf-3ed1-4e5a-9568-6b95911c7c3a   10Gi       RWO            local          <unset>                 2m21s
vmstorage-db-vmstorage-shortterm-0       Bound    pvc-cee3a2a4-5680-4880-bc2a-85c14dba9380   10Gi       RWO            local          <unset>                 2m41s
vmstorage-db-vmstorage-shortterm-1       Bound    pvc-d55c235d-cada-4c4a-8299-e5fc3f161789   10Gi       RWO            local          <unset>                 2m41s
```

Check all pods are running:


```bash
kubectl get pod -n tenant-root
```

example output:
```console
NAME                                           READY   STATUS    RESTARTS       AGE
etcd-0                                         1/1     Running   0              2m1s
etcd-1                                         1/1     Running   0              106s
etcd-2                                         1/1     Running   0              82s
grafana-db-1                                   1/1     Running   0              119s
grafana-db-2                                   1/1     Running   0              13s
grafana-deployment-74b5656d6-5dcvn             1/1     Running   0              90s
grafana-deployment-74b5656d6-q5589             1/1     Running   1 (105s ago)   111s
root-ingress-controller-6ccf55bc6d-pg79l       2/2     Running   0              2m27s
root-ingress-controller-6ccf55bc6d-xbs6x       2/2     Running   0              2m29s
root-ingress-defaultbackend-686bcbbd6c-5zbvp   1/1     Running   0              2m29s
vmalert-vmalert-644986d5c-7hvwk                2/2     Running   0              2m30s
vmalertmanager-alertmanager-0                  2/2     Running   0              2m32s
vmalertmanager-alertmanager-1                  2/2     Running   0              2m31s
vminsert-longterm-75789465f-hc6cz              1/1     Running   0              2m10s
vminsert-longterm-75789465f-m2v4t              1/1     Running   0              2m12s
vminsert-shortterm-78456f8fd9-wlwww            1/1     Running   0              2m29s
vminsert-shortterm-78456f8fd9-xg7cw            1/1     Running   0              2m28s
vmselect-longterm-0                            1/1     Running   0              2m12s
vmselect-longterm-1                            1/1     Running   0              2m12s
vmselect-shortterm-0                           1/1     Running   0              2m31s
vmselect-shortterm-1                           1/1     Running   0              2m30s
vmstorage-longterm-0                           1/1     Running   0              2m12s
vmstorage-longterm-1                           1/1     Running   0              2m12s
vmstorage-shortterm-0                          1/1     Running   0              2m32s
vmstorage-shortterm-1                          1/1     Running   0              2m31s
```

Now you can get public IP of ingress controller:
```
kubectl get svc -n tenant-root root-ingress-controller
```

example output:
```console
NAME                      TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)                      AGE
root-ingress-controller   LoadBalancer   10.96.16.141   192.168.100.200   80:31632/TCP,443:30113/TCP   3m33s
```

Use `grafana.example.org` (under 192.168.100.200) to access system monitoring, where `example.org` is your domain specified for `tenant-root`

- login: `admin`
- to get password:
  ```bash
  kubectl get secret -n tenant-root grafana-admin-password -o go-template='{{ printf "%s\n" (index .data "password" | base64decode) }}'
  ```
