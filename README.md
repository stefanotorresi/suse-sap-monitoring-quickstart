# SUSE SAP Monitoring Solution Quick-Start preview

The SHAP Monitoring Solution aims at bringing a modern open-source stack to monitor SAP systems.

It is based on a few notable free open-source projects: 

- [Prometheus](http://prometheus.io), a monitoring system and time series database.
- [Grafana](http://grafana.com/oss/grafana), a GUI platform for monitoring and observability.
- [Loki](http://grafana.com/oss/loki), a log aggregation system.

This document will guide you in installing and configuring all the required packages and start monitoring an existing SAP cluster.



## Table of contents

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
    3. [Configuring the exporters](#43-configuring-the-exporters)


## 1. Prerequisites

This document assumes the following conditions: 

- You have some SAP system in place already. Installing and configuring SAP HANA, SAP NetWeaver or SAP S4/HANA will not be covered by this guide; for this exact purpose, you can refer to our [SAP Automation](https://documentation.suse.com/sles-sap/15-SP2/single-html/SLES-SAP-sol-automation/).
- You can create a new node in the network topology of the existing cluster;
- All the nodes are running _SUSE Linux Enterprise for SAP Applications_, version 12SP3 or later;
- You have full root access to all the nodes;
- You have access to the network configuration of the cluster.

**Disclaimer**: Do not attempt to execute this guide directly on a live production system!


## 2. Architecture and Terminology

Introducing monitoring in an existing system requires provisioning an additional node to host the monitoring services; we'll refer to this node as the _monitoring server_ from now on.

The existing nodes you wish to monitor will be referred as _target nodes_ instead.

The _monitoring server_ both listens for and performs connections towards the _target nodes_. Inbound and outbound network traffic is needed between the _monitoring server_ and all the _target nodes_.


## 3. Provisioning the _Monitoring Server_

Once you have a new _SUSE Linux Enterprise for SAP Applications_ node in place, write down its publicly accessible IP address. We'll refer to this IP as `$MONITORING_SERVER_IP` throughout the rest of this document.


### 3.1. Adding the upstream repository

Some of the packages required to configure the solution are not yet delivered in official SLES channels, so - for demonstration purposes - we will use the latest development codestream.

Browse the [`network:ha-clustering:sap-deployments:devel`](https://download.opensuse.org/repositories/network:/ha-clustering:/sap-deployments:/devel/) and select the distribution version that is most relevant to you.  
Copy the link to the `.repo` file, and use it as shown below (the example uses `SLE_15_SP2`):

```
zypper add-repo -f https://download.opensuse.org/repositories/network:/ha-clustering:/sap-deployments:/devel/SLE_15_SP2/network:ha-clustering:sap-deployments:devel.repo
```

Don't forget to refresh the repos! 😉
```
zypper --gpg-auto-import-keys refresh
```


### 3.2. Installing Packages

Install the following packages on the _monitoring server_:

```
zypper install golang-github-prometheus-prometheus \
               golang-github-prometheus-alertmanager \
               golang-github-prometheus-node_exporter \
               grafana \
               loki
```

In a production environment, you probably want to distribute the Prometheus, Loki and Grafana services across multiple hosts, depending on the volume of your workloads. (Future versions of this guide may explain how to do so by deploying the monitoring stack in a Kubernetes cluster - ed.).


### 3.3. Configuring Prometheus

We'll want to change the Prometheus "jobs" configuration and deviate a little from the defaults. 

We group each set of exporters by destination of use.

In the YAML example below, we have a `monitoring` job which targets the exporters running in the monitoring host itself. Additionally, we have a `hana` job, which targets two SAP HANA nodes in a active/passive HA deployment:

<details>
<summary>/etc/prometheus/prometheus.yaml</summary>

```yaml
scrape_configs:
- job_name: "monitoring"
  static_configs:
  - targets:
    - "localhost:9090" # Prometheus self monitoring
    - "localhost:9100" # node_exporter

- job_name: 'hana':
  static_configs:
  - targets: 
    - "$HANA_NODE_1_IP:9100" # node_exporter
    - "$HANA_NODE_1_IP:9664" # ha_cluster_exporter
    - "$HANA_NODE_2_IP:9100" # node_exporter
    - "$HANA_NODE_2_IP:9664" # ha_cluster_exporter
    - "$HANA_FLOATING_IP:9668" # hanadb_exporter
```
</details>

Of course, you must change the `$HANA_NODE_1_IP`, `$HANA_NODE_2_IP` and `$HANA_FLOATING_IP` example variables according to your existing environment.

Note that in HA deployments, you only have one `hanadb_exporter` instance (port 9668 by default), targeted at the current active node. If you do not have a HA deployment, you can target the only present node, for example:

```yaml
  - targets: 
    - "$HANA_NODE_IP:9100" # node_exporter
    - "$HANA_NODE_IP:9664" # ha_cluster_exporter
    - "$HANA_NODE_IP:9668" # hanadb_exporter
```

### 3.4. Configuring Grafana

A sample configuration file is provided at `/etc/grafana/grafana.ini`. Default values works without changes.

Data sources must be added to let Grafana query the underlying Prometheus and Loki servers:

The following example uses the public monitoring server IP to access Prometheus. This is done to let Grafana correctly expose hyperlinks to the Prometheus query debug panel:

<details><summary>/etc/grafana/provisioning/datasources/datasources.yml</summary>

```yaml
apiVersion: 1

datasources:
- name: Prometheus
  type: prometheus
  access: proxy
  url: http://$MONITORING_SERVER_IP:9090
  isDefault: True

- name: Loki
  type: loki
  access: proxy
  url: http://localhost:3100
```
</details>

Alternatively, you can add data sources manually from the web GUI.

### 3.5. Configuring Loki

A sample configuration file is provided at `/etc/loki/loki.yaml`.

The default configuration uses the local filesystem storage, which is only suitable for small workloads.


### 3.6. Enabling and starting the services

```
systemctl enable --now prometheus grafana-server loki
```

The first time browse Grafana's web UI at `http://$MONITORING_SERVER_IP:3000`, you are asked to create an adminstrator user.


## 4. Provisioning the _Target Nodes_

### 4.1. Adding the Upstream Repository

See step [3.1.](#31-adding-the-upstream-repository)


### 4.2. Installing Packages

The packages providing `node_exporter` and `promtail` are needed in all the _target nodes_.
Install them with:

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

Note: for some workloads, a combination of multiple of the above options might be appropriate, for example, Highly Available HANA or Highly Available Enqueue Replication Server.


### 4.3 Configuring the Exporters

TBD
