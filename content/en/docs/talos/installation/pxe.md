---
title: How to install Cozystack using PXE
linkTitle: PXE
description: "How to install Cozystack using PXE"
weight: 10
---

![Cozystack deployment](/img/cozystack-deployment.png)

## Install dependencies:

- `docker`
- `kubectl`

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


Now, when it is booted to Talos Linux maintenance mode, you can use [talos-bootstrap](https://github.com/aenix-io/talos-bootstrap) or [Talm](https://github.com/aenix-io/talm) to bootstrap the cluster


Just follow **Get Started** guide starting from the [**Bootstrap cluster**](/docs/get-started/#bootstrap-cluster) section, to continue the installation.
