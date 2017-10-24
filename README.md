## Overview
This is a self project hosting a Cassandra cluster on top of rpi boards and each rpi node services a Cassandra container.

The base image is based on ARM architecture; therefore, this container **must be run on an ARM based SoC board**, and should not run on a x86_64 machine  - unless you have a ARM based system hosted on top of a virtual machine. There are some nice [tutorial on Raspberry PI emulation on PC](http://www.makeuseof.com/tag/emulate-raspberry-pi-pc/).

You may refer to this [git repo](https://github.com/mcfongtw/docker-rpi-cassandra/tree/rpi-cassandra), cloned from [official git repo](https://github.com/docker-library/cassandra) with several modifications including base image being changed to Raspbian.

This note that this repo is **NOT** fully tested by any CI. Travis (and others), AFAIK, only supports x86 based containers. I have tested on my own board manually. You may try this for fun.

Most functions belows are identical to [Docker official Cassandra image](https://hub.docker.com/_/cassandra/)

### Differences from official repository
1. JMX server is *enabled* by default and JMX authentication is *disabled* for simplicity.
2. Some tunable configurations are adjusted to fit in Raspberry Pi 3 hardware. See details below.
3. Most of the example and commands are heavy docker-compose focused, as it is personal preference over plain old docker cli commands.

## Latest Supported Versions
Supported tags plus associated dockerfile and docker compose files respective:
* 3.11.1 ([Dockerfile](https://github.com/mcfongtw/docker-rpi-cassandra/blob/rpi-cassandra/3.11/Dockerfile.armhf) / [Docker Compose](https://github.com/mcfongtw/docker-rpi-cassandra/blob/rpi-cassandra/3.11/docker-compose.yml) )
* 2.2 / 2.2.11 ([Dockerfile](https://github.com/mcfongtw/docker-rpi-cassandra/blob/rpi-cassandra/2.2/Dockerfile.armhf) / [Docker Compose](https://github.com/mcfongtw/docker-rpi-cassandra/blob/rpi-cassandra/2.2/docker-compose.yml))

## Configurations

### Exposed Ports
| Port | Usage                                       | Note                 |
| ---- |:-------------:                              | -----:               |
| 7000 | Unencrypted Intra-node gossip communication |                      |
| 7001 | TLS Intra-node gossip communication         |                      |
| 9042 | CQL native service                          |                      |
| 7199 | JMX server                                  |                      |
| 9160 | Thrift service                              | Disabled by default. |

Raspberry pi is a very resource limited hardware; thus it is highly recommended to run Cassandra as a dedicated container service on each node and expose ports to public network. User could run client applications / test with cqlsh against the rpi network address.

### Data Storage
Cassandra data file is mapped to host machine by default at `/var/lib/cassandra`. You may change this value by editing the volumn section in docker-compose.yml

### Configuration Properties
Some configurations are adjustable and provided as environment variables. Default value to some variables are defined in docker-compose.yaml. Those variables are:

| Property Name               | Usage                                                                                                 | Default        | Note  |
| ----------------------------|:-------------:                                                                                        | -----:         | -----:|
| CASSANDRA_CLUSTER_NAME      | The universal name across all nodes in the cluster                                                    | 'Cassandra4PI' |       |
| CASSANDRA_BROADCAST_ADDRESS |   The "public" IP address this node uses to broadcast to other nodes                                  |                |       |
| CASSANDRA_SEEDS             | A comma-separated list of IP addresses used by gossip for bootstrapping new nodes joining a cluster   |                |       |
| CASSANDRA_NUM_TOKENS        | Number of tokens for this node.                                                                       | 8              |       |
| MAX_HEAP_SIZE               | Heap size settings (Xms and Xmx)                                                                      | 384m           |       |
| HEAP_NEW_SIZE               | Size for young generation (Xmn).                                                                      | 90m            |       |

## Common Administrative Operations

### How to Deploy a Single Node Cluster on Raspbery Pi
Assume the 1st node has public address: 192.168.0.1
1. Update the following properties in docker-compose.yml
   * JVM_EXTRA_OPTS=-Djava.rmi.server.hostname=192.168.0.1
   * CASSANDRA_BROADCAST_ADDRESS=192.168.0.1
2. Execute the command: `docker-compose run --service-ports -d cassandra`

### How to Join a Second node (onward) to an existing Cluster on Raspbery Pi
Assume the 2nd node has public address: 192.168.0.2
1. Update the following properties in docker-compose.yml
   * JVM_EXTRA_OPTS=-Djava.rmi.server.hostname=192.168.0.2
   * CASSANDRA_BROADCAST_ADDRESS=192.168.0.2
   * CASSANDRA_SEEDS=192.168.0.1
2. Execute the command: `docker-compose run --service-ports -d cassandra`

### Perform cqlsh / nodetool Operation Remotely
CQL shell:
`./cqlsh 192.168.0.1`

NodeTool:
`./nodetool -h 192.168.0.1 -p 7199 <action>`

### How to Decommission a Running Node
Let us say we wish to decommission the 2nd node, and we can execute
`./nodetool -h 192.168.0.2 -p 7199 decommission`

### Troubleshooting Tips
1. To check container runtime status `docker-compose ps`.
Check the 'State' column, which normally should be 'Up'

2. To retrieve Cassandra runtime log incrementally (like tail -f)
`docker logs -f (container-name or container-id)`

3. To /bin/bash into Cassandra container
`docker exec -it rpi_cassandra_run_1 /bin/bash`
