---
title: "Cozystack Roadmap"
linkTitle: "Roadmap"
description: "Cozystack Roadmap"
weight: 50
---

### Platform

- [x] Installing the platform on a group of servers, ordering VM, MySQL, Postgres, Redis, LoadBalancer and VPN.
- [x] Running databases in replicated master/slave mode with the ability to manage users and rights
- [x] Creation of a basic graphical interface with authorization by tokens.
- [x] Collecting metrics from all services and dashboards for displaying detailed status.
- [x] Monitoring as a general system service.
- [ ] Enhanced etcd management
- [ ] Community repository - a repository similar to Archlinux AUR, where each user can publish their custom package

### Monitoring

- [x] Ability to launch a separate Monitoring Stack for each tenant.
- [x] Each tenant has a separate database for collecting logs and metrics
- [ ] Collects metrics from user workloads into their own Monitoring Stack.
- [ ] Alerts from all components get collected into a common IRM
- [ ] Routing alerts into Slack and Telegram
- [ ] Collecting logs from all services into separate databases for each tenant

### Managed Kubernetes

- [x] Ability to run Managed Kubernetes clusters with isolated control-plane
- [x] Ability to order PVCs and LoadBalancer type services from a tenant's Kubernetes cluster
- [ ] Automatic logs and metrics collection from tenant's Kubernetes clusters into the tenant’s Monitoring Stack

### Applications

- [X] Managed VPN service
- [X] PostgreSQL users and database management
- [X] PostgreSQL backups
- [X] MariaDB users and database management
- [X] MariaDB backups
- [ ] MariaDB recovery enhancements
- [ ] MariaDB to support maxscale
- [ ] Enhanced Virtual Machines management

### Security

- [x] Basic management of tenants
- [ ] Network isolation between tenants
- [ ] Improvement of the authorization and access control system
- [ ] User creation and granular permission assignment
- [ ] Integration with external authentication systems (LDAP, OIDC, etc.)
- [ ] Common authentication gateway and security logging portal
- [ ] VPCs and network isolation
