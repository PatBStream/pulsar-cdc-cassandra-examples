# Pulsar Change Data Capture (CDC) with Cassandra running in Docker
This repo documents the setup and installation step to run Pulsar, Cassandra with CDC, in Docker on Windows Subsystem for Linux (WSL2).   

- [Pulsar Change Data Capture (CDC) with Cassandra running in Docker](#pulsar-change-data-capture-cdc-with-cassandra-running-in-docker)
  - [Overview](#overview)
- [Assumptions and Requirements](#assumptions-and-requirements)
- [Setup Cassandra Single node and CDC configuration](#setup-cassandra-single-node-and-cdc-configuration)
- [Setup Pulsar Source Connector for CDC](#setup-pulsar-source-connector-for-cdc)
- [Setup Docker Network for Pulsar and Cassandra Containers](#setup-docker-network-for-pulsar-and-cassandra-containers)
- [Start Pulsar in Docker](#start-pulsar-in-docker)
- [Start Cassandra and setup Keyspace and Table for CDC](#start-cassandra-and-setup-keyspace-and-table-for-cdc)
  - [Add Keyspace and Table for CDC](#add-keyspace-and-table-for-cdc)
- [Configure Pulsar CDC Source Connector](#configure-pulsar-cdc-source-connector)
- [View Pulsar CDC logs](#view-pulsar-cdc-logs)
  
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

# Setup Cassandra Single node and CDC configuration 
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
# Setup Pulsar Source Connector for CDC
Download the DataStax Cassandra Source Connector for Apache Pulsar (CSC) file:  
https://downloads.datastax.com/#cassandra-source-connector  
Untar the file as needed:
Untar the file as needed
```
mylaptop$ tar xvf cassandra-source-connectors-1.0.5.tar
.
.
.
mylaptop$ ls -la ~/pulsar-cdc
total nnnnnn
drwxr-xr-x  8 user1  user1       4096 Dec  6 08:17 ./
drwxr-xr-x 35 user1  user1       4096 Dec  6 08:55 ../
drwxr-xr-x 10 user1  user1       4096 Nov 10 04:14 cassandra-source-connectors-1.0.5/
-rw-r--r--  1 user1  user1  264417280 Dec  5 11:05 cassandra-source-connectors-1.0.5.tar

```
Copy the Agent Jar and Connector NAR to the **"config"** directory so Cassandra and Pulsar containers have access to these files.  Afterwards, **config** should contain:
```
mylaptop$ ls -la ~/pulsar-cdc/config
total nnnnn
drwxr-xr-x 2 user1 user1     4096 Dec  6 08:55 .
drwxr-xr-x 8 user1 user1     4096 Dec  6 08:17 ..
-rw-r--r-- 1 user1 user1 46188456 Apr 11  2022 agent-dse4-luna-1.0.5-all.jar
-rw-r--r-- 1 user1 user1 41894694 Apr 11  2022 agent-dse4-pulsar-1.0.5-all.jar
-rw-r--r-- 1 user1 user1    15029 Dec  6 08:55 cassandra-env.sh
-rw-r--r-- 1 user1 user1    72690 Dec  5 14:26 cassandra.yaml
-rw-r--r-- 1 user1 user1 84160763 Apr 11  2022 pulsar-cassandra-source-1.0.5.nar

```
# Setup Docker Network for Pulsar and Cassandra Containers
Setup a Docker network so the containers can communiate like normal host/servers.  We'll label our network **pulsarcdcnet**.
```
mylaptop$ docker network create pulsarcdcnet

```
# Start Pulsar in Docker
Start Pulsar Standalone in a Docker container:
```
mylaptop$ docker run -it -p 6650:6650  -p 8080:8080 -d -h pulsarhost --name pulsar  -v ~/pulsar-cdc/config:/config --net pulsarcdcnet datastax/lunastreaming:2.10_2.0 bin/pulsar standalone

```
**NOTE** Reference in the startup command-line for "--net pulsarcdcnet" parameter.  This is required for Docker container-to-container communication.
# Start Cassandra and setup Keyspace and Table for CDC
Start Cassandra in Docker:
```
mylaptop$ docker run -e DS_LICENSE=accept -e JVM_EXTRA_OPTS="-javaagent:/config/agent-dse4-luna-1.0.5-all.jar" --name dsehost -h dsehost -v ~/pulsar-cdc/config:/config -d --net pulsarcdcnet -e CASSANDRA_BROADCAST_ADDRESS=dsehost datastax/dse-server:6.8.29-1
```  
**NOTE** Reference in the startup command-line for "--net pulsarcdcnet" parameter.  This is required for Docker container-to-container communication.

## Add Keyspace and Table for CDC 
(TODO)
```
mylaptop$  docker exec -it dsehost /bin/bash
dse@dsehost~$ bin/cqlsh

Cqlsh commands:
create keyspace ks1 with replication = {'class':'SimpleStrategy', 'replication_factor' : 1};
CREATE TABLE ks1.table1 (a int, b text, PRIMARY KEY(a)) WITH cdc=true;
insert into ks1.table1 (a , b ) VALUES ( 0, 'hello');
update ks1.table1 set b = 'Updated Hello1' where a = 1;
delete from ks1.table1 where a = 2;
select * from ks1.table1;
```

# Configure Pulsar CDC Source Connector
(TODO)

```
pulsar-admin source create \ --name cassandra-source-1 \ --archive /config/pulsar-cassandra-source-1.0.5.nar \ --tenant public \ --namespace default \ --destination-topic-name persistent://public/default/data-ks1.table1 \ --parallelism 1 \ --source-config '{ "events.topic": "persistent://public/default/events-ks1.table1", "keyspace": "ks1", "table": "table1", "contactPoints": "dsehost", "port": 9042, "loadBalancing.localDc": "dc1", "auth.provider": "PLAIN", "auth.username": "<username>", "auth.password": "<password>" }'
```

# View Pulsar CDC logs
To view the Pulsar CDC Source Connector logfiles, attach to the running Docker container "pulsar" and then tail or cat the logfile.
```
mylaptop$ docker exec -it pulsar /bin/bash

I have no name!@pulsarhost:/pulsar$ tail -200f logs/functions/public/default/cassandra-source-1/cassandra-source-1-0.log

2022-12-07T22:44:09,832+0000 [main] INFO  org.apache.pulsar.functions.runtime.JavaInstanceStarter - JavaInstance S
erver started, listening on 37463
2022-12-07T22:44:09,834+0000 [main] INFO  org.apache.pulsar.functions.runtime.JavaInstanceStarter - Starting runti
meSpawner
2022-12-07T22:44:09,835+0000 [main] INFO  org.apache.pulsar.functions.runtime.RuntimeSpawner - public/default/cass
andra-source-1-0 RuntimeSpawner starting function
2022-12-07T22:44:09,837+0000 [main] INFO  org.apache.pulsar.common.nar.FileUtils - Jar file /pulsar/download/pulsa
r_functions/public/default/cassandra-source-1/0/pulsar-cassandra-source-1.0.5.nar contains META-INF/bundled-depend
encies, it may be a NAR file
2022-12-07T22:44:09,838+0000 [main] INFO  org.apache.pulsar.functions.runtime.thread.ThreadRuntime - Trying Loadin
g file as NAR file: /pulsar/download/pulsar_functions/public/default/cassandra-source-1/0/pulsar-cassandra-source-
1.0.5.nar
```

