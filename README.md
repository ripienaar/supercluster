## What?

Creates NATS super clusters - clusters of clusters - using Docker Compose.

I use this during development to create different sizes of clusters for testing some scenarios.

## Using?

The command below will:

 * Create a `docker-compose.yaml` file for 2 clusters with 3 nodes each
 * With users `one`, `two` and `system` all with the password `password`
 * Start a shell with `NATS_URL`, `NATS_USER` and `NATS_PASSWORD` set to connect to cluster 1 node 1


```nohighligh
$ PASSWORD=password rake supercluster
Cluster: 1
  Node: nc1.c1.example.net
    Port: 10000

  Node: nc2.c1.example.net
    Port: 10001

  Node: nc3.c1.example.net
    Port: 10002


Cluster: 2
  Node: nc1.c2.example.net
    Port: 10003

  Node: nc2.c2.example.net
    Port: 10004

  Node: nc3.c2.example.net
    Port: 10005


...wrote docker-compose.yaml
...wrote configs/cluster.conf

Use rake shell to access the network
```

Now start the cluster in another window using `docker-compose up` and you can see the cluster members:

You can connect to the cluster as follows:

```nohighlight
$ SHELL="2 1" rake shell
```

This opens a new shell configured to connect to cluster 2 node 1.

```nohighlight
[rip@dev1]% nats server list --user system
+----------------------------------------------------------------------------------------------------------------------+
|                                                   Server Overview                                                    |
+--------+------------+-----------+---------+-------+------+--------+-----+---------+-----+------+--------+------------+
| Name   | Cluster    | IP        | Version | Conns | Subs | Routes | GWs | Mem     | CPU | Slow | Uptime | RTT        |
+--------+------------+-----------+---------+-------+------+--------+-----+---------+-----+------+--------+------------+
| nc1-c1 | c1         | 0.0.0.0   | 2.1.7   | 1     | 71   | 2      | 1   | 6.6 MiB | 0.0 | 0    | 3m34s  | 2.904972ms |
| nc2-c1 | c1         | 0.0.0.0   | 2.1.7   | 0     | 71   | 2      | 1   | 6.4 MiB | 0.0 | 0    | 3m35s  | 3.031357ms |
| nc3-c1 | c1         | 0.0.0.0   | 2.1.7   | 0     | 71   | 2      | 1   | 6.3 MiB | 0.0 | 0    | 3m34s  | 3.151222ms |
| nc3-c2 | c2         | 0.0.0.0   | 2.1.7   | 0     | 70   | 2      | 1   | 6.1 MiB | 0.0 | 0    | 3m34s  | 3.253285ms |
| nc1-c2 | c2         | 0.0.0.0   | 2.1.7   | 0     | 70   | 2      | 1   | 6.1 MiB | 0.0 | 0    | 3m34s  | 3.35053ms  |
| nc2-c2 | c2         | 0.0.0.0   | 2.1.7   | 0     | 70   | 2      | 1   | 6.9 MiB | 0.0 | 0    | 3m35s  | 3.441887ms |
+--------+------------+-----------+---------+-------+------+--------+-----+---------+-----+------+--------+------------+
|        | 2 Clusters | 6 Servers |         | 1     | 423  |        |     | 38 MiB  |     | 0    |        |            |
+--------+------------+-----------+---------+-------+------+--------+-----+---------+-----+------+--------+------------+

+--------------------------------------------------------------+
|                       Cluster Overview                       |
+---------+------------+-------------------+-------------------+
| Cluster | Node Count | Outgoing Gateways | Incoming Gateways |
+---------+------------+-------------------+-------------------+
| c2      | 3          | 3                 | 3                 |
| c1      | 3          | 3                 | 3                 |
+---------+------------+-------------------+-------------------+
|         | 6          | 6                 | 6                 |
+---------+------------+-------------------+-------------------+
```

Possible settings are:

|Setting        |Description                       |Default    |
|-----------------|----------------------------------|-----------|
|`JETSTREAM`      |enables JetStream clustered mode||
|`DOMAIN`         |the domain to create containers in|example.net|
|`IMAGE`          |the docker image to use|nats|
|`CLUSTERS`       |how many clusters to create|2|
|`CLUSTER_MEMBERS`|how many members in each cluster|3|
|`PORT`           |the base port to start adding each node to|10000|
|`IMAGE_CONFIG`   |the path in the image to use for the configuration|/nats-server.conf|
|`PASSWORD`       |set this to enable multiple accounts using this password||
|`NATS`           |opens a shell after creating the configurations for a specific cluster and host. Set to '1 1' for cluster 1 node 1||
|`TC`             |sets up Linux traffic control within the cluster|

## Traffic Control

The https://github.com/lukaszlach/docker-tc system is setup to facilitate `tc` based traffic control allowing for custom
packet loss, delays, duplication etc to be configured.

To use this set the `TC` variable to `1`

NOTE: `tc` only works on egress traffic.

## TODO

 * Set up NATS Surveyor
 * TLS
