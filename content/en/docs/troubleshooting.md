---
title: "Troubleshooting cheatsheet"
linkTitle: "Troubleshooting"
description: "Showcase various ways to get more information out of Flux controllers to debug potential problems."
weight: 60
---

## Getting basic information

You can see the logs of main installer by executing:

```yaml
kubectl logs -n cozy-system deploy/cozystack -f
```

All the platform components are installed using fluxcd HelmRelases.

You can get all installed HelmRelases:

```bash
# kubectl get hr -A
NAMESPACE                        NAME                        AGE    READY   STATUS
cozy-cert-manager                cert-manager                4m1s   True    Release reconciliation succeeded
cozy-cert-manager                cert-manager-issuers        4m1s   True    Release reconciliation succeeded
cozy-cilium                      cilium                      4m1s   True    Release reconciliation succeeded
cozy-cluster-api                 capi-operator               4m1s   True    Release reconciliation succeeded
cozy-cluster-api                 capi-providers              4m1s   True    Release reconciliation succeeded
cozy-dashboard                   dashboard                   4m1s   True    Release reconciliation succeeded
cozy-fluxcd                      cozy-fluxcd                 4m1s   True    Release reconciliation succeeded
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

Normaly all of them should be `Ready` and `Release reconciliation succeeded`


### Diagnosing `install retries exhausted` error

Sometimes you can face with the error:

```bash
# kubectl get hr -A -n cozy-dashboard dashboard
NAMESPACE          NAME          AGE   READY   STATUS
cozy-dashboard     dashboard     15m   False   install retries exhausted
```

You can try to figure out by checking events:

```bash
kubectl describe hr -n cozy-dashboard dashboard
```

if `Events: <none>` then suspend and reume the release:

```bash
kubectl patch hr -n cozy-dashboard dashboard -p '{"spec": {"suspend": true}}' --type=merge
kubectl patch hr -n cozy-dashboard dashboard -p '{"spec": {"suspend": null}}' --type=merge
```

and check the events again:

```bash
kubectl describe hr -n cozy-dashboard dashboard
```
