---
title: "Upgrade Cozystack"
linkTitle: "Upgrade"
description: "Upgrade Cozystack system components."
weight: 31
---

## Upgrade Cozystack

### Check current cozystack status
```bash
kubectl get hr -A | grep -v "True"
```
Repair helmreleases if nessesary.  
Make sure that the cozystack config map contains all the necessary variables.

```bash
kubectl get configmap -n cozy-system cozystack -oyaml
```
Example output:
```yaml
apiVersion: v1
data:
  api-server-endpoint: https://33.44.55.66:6443
  #bundle-enable: telepresence if nessesary
  bundle-name: paas-full
  ipv4-join-cidr: 100.64.0.0/16
  ipv4-pod-cidr: 10.244.0.0/16
  ipv4-pod-gateway: 10.244.0.1
  ipv4-svc-cidr: 10.96.0.0/16
  # oidc-enabled: "true" if nessesary
  root-host: example.org
kind: ConfigMap
...
```
And add missing key: value data.

### Apply new version
```bash
version=vX.Y.Z
kubectl apply -f https://github.com/cozystack/cozystack/raw/$version/manifests/cozystack-installer.yaml
```

### Check cozystack status
```bash
kubectl get pods -n cozy-system
kubectl get hr -A | grep -v "True"
```

If pod status is failure, check logs.
```bash
kubectl logs deploy/cozystack -n cozy-system --previous
```
