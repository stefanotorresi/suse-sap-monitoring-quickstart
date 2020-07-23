# SUSE SAP Monitoring Solution Quick-Start preview

The SHAP Monitoring Solution aims at bringing a modern open-source stack to monitor SAP systems.

It is based on a few notable free open-source projects: [Prometheus](http://prometheus.io), [Grafana](http://grafana.com/oss/grafana) and [Loki](http://grafana.com/oss/loki).

This document will guide you in installing and configuring all the required packages and start monitoring an existing SAP cluster.


## 0. Table of contents

1. [Prerequisites](#1-prerequisites)
2. [Architecture](#2-architecture)
3. [Provisioning the monitoring server](#3-provisioning-the-monitoring-server)
    1. [Adding the upstream repositories](#31-adding-the-upstream-repository)
    2. [Installing packages](#32-installing-packages)
    3. [Configuring Prometheus](#33-configuring-prometheus)
    4. [Configuring Grafana](#34-configuring-grafana)
    5. [Configuring Loki](#35-configuring-loki)
    6. [Enabling and starting the services](#36-enabling-and-starting-the-services)
4. [Provisioning the target nodes](#4-provisioning-the-target-nodes)
    1. [Adding the upstream repositories](#41-adding-the-upstream-repository)
    2. [Installing packages](#42-installing-packages)


## 1. Prerequisites

Throughout the rest of this document, it is assumed the following: 

- You have some SAP system in place already. Installing and configuring SAP HANA, SAP NetWeaver or SAP S4/HANA will not be covered by this guide;
- You can create a new node in the network topology of the existing cluster.
- All the nodes are running _SUSE Linux Enterprise for SAP Applications_, version 12SP3 or later.
- You have full root access to all the nodes;
- You have access to the network configuration of the cluster.

**Disclaimer**: Please, do not attempt to execute this guide on a live production system.


## 2. Architecture

Introducing monitoring in an existing system requires provisioning an additional node to host the monitoring services; we'll refer to this node as the _monitoring server_ from now on.

The existing nodes you wish to monitor will be referred as _target nodes_ instead.

The _monitoring server_ will both listen for connections and perform connections towards the _target nodes_, so inbound and outbound network traffic will be needed between the _monitoring server_ and all the _target nodes_.


## 3. Provisioning the _monitoring server_

Once you have a new _SUSE Linux Enterprise for SAP Applications_ node in place, note down it's publicly accessible IP address; we'll refer to this as `$MONITORING_SERVER_IP` throughout the rest of the document.


### 3.1. Adding the upstream repository

```
export SLE_VERSION=SLE_15_SP2 # set according to your environment
zypper add-repo https://download.opensuse.org/repositories/network:/ha-clustering:/sap-deployments:/devel/$SLE_VERSION/network:ha-clustering:sap-deployments:devel.repo
```


### 3.2. Installing packages

```
zypper install golang-github-prometheus-prometheus \
               golang-github-prometheus-alertmanager \
               grafana \
               loki
```


### 3.3. Configuring Prometheus

TBD


### 3.4. Configuring Grafana

TBD


### 3.5. Configuring Loki

TBD


### 3.6. Enabling and starting the services

```
systemctl enable --now prometheus grafana-server loki
```


## 4. Provisioning the _target nodes_

### 4.1. Adding the upstream repository

See step [3.1.](#31-adding-the-upstream-repository)


### 4.2. Installing packages

The packages providing `node_exporter` and `promtail` are needed in all the _target nodes_:

```
zypper install golang-github-prometheus-node_exporter \
               loki
```

Furthermore, you should install the relevant exporters, depending on the workload each node is dedicated to.

For Pacemaker HA cluster member nodes:
```
zypper install prometheus-ha_cluster_exporter
```

For HANA nodes:
```
zypper install prometheus-hanadb_exporter
```

For NetWeaver or S4/HANA nodes:
```
zypper install prometheus-sap_host_exporter
```

Note: for some workloads, a combination of multiple of the above options might be appropriate, e.g. Highly Available HANA or Highly Available Enqueue Replication Server.


### 4.3 Configuring 