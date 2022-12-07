- [Pulsar Change Data Capture (CDC) with Cassandra running in Docker](#pulsar-change-data-capture-cdc-with-cassandra-running-in-docker)
  - [Overview](#overview)
- [Assumptions and Requirements](#assumptions-and-requirements)
- [Setup Cassandra Single node in Docker](#setup-cassandra-single-node-in-docker)

# Pulsar Change Data Capture (CDC) with Cassandra running in Docker
This repo documents the setup and installation step to run Pulsar, Cassandra with CDC, in Docker on Windows Subsystem for Linux (WSL2).   

## Overview
We'll detail the steps of install and implementation of 3 components for Cassandra CDC:  Pulsar, Pulsar Cassandra Source Connector, and Cassandra.  All components run in Docker Desktop on Windows 11 with WSL2.

Additionally, examples are provided for C* tables and entries to demo CDC in action with Pulsar.

For this demo and examples, we'll use the following components, OSS and community editions:
* Pulsar 2.10_2 from DataStax Luna Streaming (https://hub.docker.com/r/datastax/lunastreaming)
* Cassandra from DataStax DSE (https://hub.docker.com/r/datastax/dse-server)
* CDC for Cassandra (https://docs.datastax.com/en/cdc-for-cassandra/cdc-apache-cassandra/2.2.1/install.html#_deploy_cdc_for_cassandra)
* Pulsar Connector for Cassandra CDC (https://downloads.datastax.com/#cassandra-source-connector)

# Assumptions and Requirements
* Access to the internet from laptop
* Docker Desktop installed and running normally
* WSL2 installed and running Ubuntu 

# Setup Cassandra Single node in Docker
Download the following files into a local directory/folder.  For example, local directory ~/pulsar-cdc/
[DataStax Change Agent for Apache Cassandra (CAC)](https://downloads.datastax.com/#cassandra-change-agent)
Untar the file as needed
```
mylaptop$ tar xvf cassandra-source-agents-<version>.tar
.
.
.
mylaptop$ ls -la
total nnnnnn
drwxr-xr-x  8 user1  user1       4096 Dec  6 08:17 ./
drwxr-xr-x 35 user1  user1       4096 Dec  6 08:55 ../
drwxr-xr-x 10 user1  user1       4096 Nov 10 04:14 cassandra-source-agents-1.0.5/
-rw-r--r--  1 user1  user1  264417280 Dec  5 11:05 cassandra-source-agents-1.0.5.tar

```
Create a new directory/folder called "config" to store Cassandra configuration files for CDC.
```
mylaptop$ mkdir ~/pulsar-cdc/config

```
Download the following Cassandra config files into "config".  
https://github.com/datastax/docker-images/blob/master/config-templates/DSE/6.8.1/cassandra.yaml  
https://github.com/datastax/docker-images/blob/master/config-templates/DSE/6.8.1/cassandra-env.sh  

```
mylaptop$ cd ~/pulsar-cdc/config
mylaptop$ ls -la
total 168312
drwxr-xr-x 2 user1 user1     4096 Dec  6 08:55 .
drwxr-xr-x 8 user1 user1     4096 Dec  6 08:17 ..
-rw-r--r-- 1 user1 user1    15029 Dec  6 08:55 cassandra-env.sh
-rw-r--r-- 1 user1 user1    72690 Dec  5 14:26 cassandra.yaml

```
In file: **cassandra-env.sh**, add this param at the end: **export CDC_PULSAR_SERVICE_URL="pulsar://pulsarhost:6650"**  
In file: **cassandra.yaml**, add these lines to the end-of-file:
```
cdc_enabled: true
commitlog_sync_period_in_ms: 2000
cdc_total_space_in_mb: 50000
```
