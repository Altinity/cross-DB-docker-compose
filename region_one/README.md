# clickhouse-replication-docker
Data Replication in ClickHouse (Docker Based Setup) , 3 Node Replication Cluster

## Requirement -- Download these files from Github repo

```
docker-compose.yaml
config.xml
clusters.xml
listen_host.xml
zookeeper.xml
macros_1.xml
macros_2.xml
macros_3.xml
```


If you want to run specific version of clickhouse then just update image for specific version in docker-compose.yaml file.
Example : image: clickhouse/clickhouse-server:23.5.2.7-alpine, Update for all ClickHouse Nodes in Cluster. 
Visit docker hub to get correct image tag : https://hub.docker.com/r/clickhouse/clickhouse-server/tags


Execute below commands on docker Host 

```
root@ip-10-0-6-24:~/ashwini_workdir/clickhouse# docker-compose up -d
Starting zookeeper ... done
Starting clickhouse03 ... done
Starting clickhouse02 ... done
Starting clickhouse01 ... done

root@ip-10-0-6-24:~# docker ps
CONTAINER ID   IMAGE                                 COMMAND                  CREATED       STATUS          PORTS                                                                                                                                       NAMES
c3ab220e510b   clickhouse/clickhouse-server:latest   "/entrypoint.sh"         3 hours ago   Up 36 seconds   9009/tcp, 0.0.0.0:8125->8123/tcp, :::8125->8123/tcp, 0.0.0.0:9002->9000/tcp, :::9002->9000/tcp                                              clickhouse02
a98ee0b4784b   clickhouse/clickhouse-server:latest   "/entrypoint.sh"         3 hours ago   Up 35 seconds   9009/tcp, 0.0.0.0:8126->8123/tcp, :::8126->8123/tcp, 0.0.0.0:9003->9000/tcp, :::9003->9000/tcp                                              clickhouse03
f67bd83c305a   clickhouse/clickhouse-server:latest   "/entrypoint.sh"         3 hours ago   Up 35 seconds   9009/tcp, 0.0.0.0:8124->8123/tcp, :::8124->8123/tcp, 0.0.0.0:9001->9000/tcp, :::9001->9000/tcp                                              clickhouse01
1faf23b1ebbf   bitnami/zookeeper:latest              "/opt/bitnami/script…"   3 hours ago   Up 37 seconds   0.0.0.0:2888->2888/tcp, :::2888->2888/tcp, 0.0.0.0:3888->3888/tcp, :::3888->3888/tcp, 8080/tcp, 0.0.0.0:2182->2181/tcp, :::2182->2181/tcp   zookeeper
```

When containers are running, connect to clickhouse-client and test

```
root@ip-10-0-6-24:~/ashwini_workdir/clickhouse# docker exec -it clickhouse01  bash
root@clickhouse01:/#
root@clickhouse01:/#
root@clickhouse01:/#
root@clickhouse01:/# clickhouse-client
ClickHouse client version 23.3.1.2823 (official build).
Connecting to localhost:9000 as user default.
Connected to ClickHouse server version 23.3.1 revision 54462.

Warnings:
 * Linux is not using a fast clock source. Performance can be degraded. Check /sys/devices/system/clocksource/clocksource0/current_clocksource
 * Linux threads max count is too low. Check /proc/sys/kernel/threads-max
 * Available memory at server startup is too low (2GiB).


clickhouse01 :) CREATE DATABASE IF NOT EXISTS Example_DB ON CLUSTER '{cluster}';

CREATE DATABASE IF NOT EXISTS Example_DB ON CLUSTER `{cluster}`

Query id: d572b0db-23b0-432d-b144-3535445ed120

┌─host─────────┬─port─┬─status─┬─error─┬─num_hosts_remaining─┬─num_hosts_active─┐
│ clickhouse01 │ 9000 │      0 │       │                   2 │                0 │
│ clickhouse02 │ 9000 │      0 │       │                   1 │                0 │
│ clickhouse03 │ 9000 │      0 │       │                   0 │                0 │
└──────────────┴──────┴────────┴───────┴─────────────────────┴──────────────────┘

3 rows in set. Elapsed: 0.143 sec.

clickhouse01 :) CREATE TABLE Example_DB.product ON CLUSTER '{cluster}'
                               (
                                    created_at DateTime,
                                    product_id UInt32,
                                    category UInt32
                               ) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{cluster}/{shard}/Example_DB/product', '{replica}')
                               PARTITION BY toYYYYMM(created_at)
                               ORDER BY (product_id, toDate(created_at), category)
                               SAMPLE BY category;

CREATE TABLE Example_DB.product ON CLUSTER `{cluster}`
(
    `created_at` DateTime,
    `product_id` UInt32,
    `category` UInt32
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{cluster}/{shard}/Example_DB/product', '{replica}')
PARTITION BY toYYYYMM(created_at)
ORDER BY (product_id, toDate(created_at), category)
SAMPLE BY category

Query id: 26d805bd-14d7-4d88-9111-4aa3360ea863

┌─host─────────┬─port─┬─status─┬─error─┬─num_hosts_remaining─┬─num_hosts_active─┐
│ clickhouse01 │ 9000 │      0 │       │                   2 │                0 │
│ clickhouse02 │ 9000 │      0 │       │                   1 │                0 │
│ clickhouse03 │ 9000 │      0 │       │                   0 │                0 │
└──────────────┴──────┴────────┴───────┴─────────────────────┴──────────────────┘

3 rows in set. Elapsed: 0.309 sec.

clickhouse01 :) select * from Example_DB.product;

SELECT *
FROM Example_DB.product

Query id: dfd344dd-8bf1-4a55-a839-9de9e0e4ef7c

┌──────────created_at─┬─product_id─┬─category─┐
│ 2023-04-20 12:14:45 │        765 │       23 │
└─────────────────────┴────────────┴──────────┘

1 row in set. Elapsed: 0.002 sec.

clickhouse01 :) insert into Example_DB.product values (now(),9832,11);

INSERT INTO Example_DB.product FORMAT Values

Query id: ba051e61-3465-4d76-ad12-a9ccbc97f028

Ok.

1 row in set. Elapsed: 0.015 sec.

clickhouse01 :) select * from Example_DB.product;

SELECT *
FROM Example_DB.product

Query id: 78c6a9a3-bc66-4594-830c-87da8f9a6382

┌──────────created_at─┬─product_id─┬─category─┐
│ 2023-04-20 12:14:45 │        765 │       23 │
└─────────────────────┴────────────┴──────────┘
┌──────────created_at─┬─product_id─┬─category─┐
│ 2023-04-20 12:15:19 │       5465 │    23432 │
└─────────────────────┴────────────┴──────────┘

2 rows in set. Elapsed: 0.007 sec.
```

```
clickhouse01 :) SELECT hostName(), database, name FROM clusterAllReplicas(cluster_demo_ash, system.tables) WHERE database='Example_DB' and name='product'

SELECT
    hostName(),
    database,
    name
FROM clusterAllReplicas(cluster_demo_ash, system.tables)
WHERE (database = 'Example_DB') AND (name = 'product')

Query id: a25221f0-d818-4897-b27f-dce27acbdc8a

┌─hostName()───┬─database───┬─name────┐
│ clickhouse01 │ Example_DB │ product │
└──────────────┴────────────┴─────────┘
┌─hostName()───┬─database───┬─name────┐
│ clickhouse02 │ Example_DB │ product │
└──────────────┴────────────┴─────────┘
┌─hostName()───┬─database───┬─name────┐
│ clickhouse03 │ Example_DB │ product │
└──────────────┴────────────┴─────────┘

3 rows in set. Elapsed: 0.193 sec.

clickhouse01 :)
```

```
clickhouse01 :) SELECT * FROM system.clusters WHERE cluster='cluster_demo_ash' FORMAT Vertical;
clickhouse01 :) SELECT hostName(), database, name FROM clusterAllReplicas(cluster_demo_ash, system.tables) WHERE database='Example_DB' and name='product'

```


