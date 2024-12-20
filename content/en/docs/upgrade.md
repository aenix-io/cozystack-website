---
title: "Upgrade Cozystack"
linkTitle: "Upgrade"
description: "Upgrade Cozystack system components."
weight: 60
---

## Upgrade Cozystack

### Check current cozystack status
```bash
kubectl get hr -A | grep -v "True"
```
Repair helmreleases if nessesary.

### Apply new version
```bash
version=vX.Y.Z
kubectl apply -f https://github.com/aenix-io/cozystack/raw/$version/manifests/cozystack-installer.yaml
```

### Check cozystack status
```bash
kubectl get pods -n cozy-system
kubectl get hr -A | grep -v "True"
```
