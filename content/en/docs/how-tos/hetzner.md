---
title: How to install Cozystack in Hetnzer
linkTitle: Hetzner
description: "How to install Cozystack in Hetzner"
weight: 10
---

{{% alert color="warning" %}}
:warning: Secure Boot is currently not supported.

If your server configured to use Secure Boot, you need to disable this feature in your BIOS. Otherwise, it will block the server from booting after Talos Linux installation.
{{% /alert %}}

Switch your server into rescue mode

Login to the server using SSH

Check Disks:

```console
# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
nvme0n1     259:4    0 476.9G  0 disk
nvme1n1     259:0    0 476.9G  0 disk
```

Download Talos Linux asset from the Cozystack's [releases page](https://github.com/aenix-io/cozystack/releases), and write it into disk:

```bash
cd /tmp
wget https://github.com/aenix-io/cozystack/releases/download/v0.2.0/metal-amd64.raw.xz
xz -d -c /tmp/metal-amd64.raw.xz | dd of=/dev/nvme0n1 bs=4M oflag=sync
```

Resize the partition table and prepare additional partition for the cloud-init data:

```bash
# resize gpt partition
sgdisk -e /dev/nvme0n1

# Create 20MB partition in the end of disk
end=$(sgdisk -E /dev/nvme0n1)
sgdisk -n7:$(( $end - 40960 )):$end -t7:ef00 /dev/nvme0n1

# Create fat partion for cloud-init
mkfs.vfat -n CIDATA /dev/nvme0n1p7

# Mount cloud-init partition
mount  /dev/nvme0n1p7 /mnt
```

Write cloud-init configuration

```bash
echo 'local-hostname: us-talos-1' > /mnt/meta-data
echo '#cloud-config' > /mnt/user-data
echo 'version: 2' > /mnt/network-config
```

Edit network-config and specify your network settings using [network-config-format-v2](https://cloudinit.readthedocs.io/en/latest/reference/network-config-format-v2.html).

```yaml
version: 2
ethernets:
  eth0:
    match:
      macaddress: 'a6:34:9e:c0:25:6b'
    dhcp4: false
    addresses:
      - "1.2.3.4/26"
      - "2a01:4f3:341:524::2/64"
    gateway4: "1.2.3.1"
    gateway6: "fe80::1"
    nameservers:
      search: [cluster.local]
      addresses: [8.8.8.8]
```

You can find more comprehensive example under the [link](https://github.com/siderolabs/talos/blob/10f958cf41ec072209f8cb8724e6f89db24ca1b6/internal/app/machined/pkg/runtime/v1alpha1/platform/nocloud/testdata/metadata-v2.yaml)

Umount cloud-init partition, sync changes, and reboot the server

```bash
umount /mnt
sync
reboot
```

Now, when it is booted to Talos Linux maintenance mode, you can use [talos-bootstrap](https://github.com/aenix-io/talos-bootstrap) to bootstrap the cluster:

```
talos-bootstrap install -n 1.2.3.4
```

Where `1.2.3.4` is the IP-address of your remote node.


Now follow **Get Started** guide starting from the [**Bootstrap cluster**](/docs/get-started/#bootstrap-cluster) section, to continue the installation.
