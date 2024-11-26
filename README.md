# cross-DB-docker-compose"

# ClickHouse Multi-Region Cluster Setup with Disaster Recovery

This repository contains the configuration and instructions for setting up a multi-region ClickHouse cluster with disaster recovery capabilities using ClickHouse Keeper.

## Architecture Overview

### Region One (172.25.0.0/16)
- 3 ClickHouse servers (172.25.0.12-14)
- 3 Keeper nodes (172.25.0.2-4)

### Region Two (172.26.0.0/16)
- 3 ClickHouse servers (172.26.0.12-14)
- 3 Keeper nodes (172.26.0.2-4)

## Prerequisites

- Docker and Docker Compose installed
- Basic understanding of ClickHouse and distributed systems
- Sufficient resources to run multiple Docker containers
- Network connectivity between regions

## Detailed Setup Steps

### 1. Start Keeper Services

First, launch the Keeper services in both regions:

```bash
# Region One
cd region_one
docker compose -f docker-compose-keeper.yaml up -d

# Region Two
cd region_two
docker compose -f docker-compose-keeper.yaml up -d
```

### 2. Start ClickHouse Servers

After Keeper services are up, start the ClickHouse servers:

```bash
# Region One
cd region_one
docker compose -f docker-compose-clickhouse.yaml up -d

# Region Two
cd region_two
docker compose -f docker-compose-clickhouse.yaml up -d
```

### 3. Configure Cross-Region Networking

Connect containers across regions:

```bash
#Create the external network:
docker network create --subnet=172.25.0.0/16 clickhouse_network_region_one
docker network create --subnet=172.26.0.0/16 clickhouse_network_region_two

# Connect region two containers to region one network
docker network connect clickhouse_network_region_one clickhouse04-keeper
docker network connect clickhouse_network_region_one clickhouse05-keeper
docker network connect clickhouse_network_region_one clickhouse06-keeper
docker network connect clickhouse_network_region_one clickhouse04-server
docker network connect clickhouse_network_region_one clickhouse05-server
docker network connect clickhouse_network_region_one clickhouse06-server

# Connect region one containers to region two network
docker network connect clickhouse_network_region_two clickhouse01-keeper
docker network connect clickhouse_network_region_two clickhouse02-keeper
docker network connect clickhouse_network_region_two clickhouse03-keeper
docker network connect clickhouse_network_region_two clickhouse01-server
docker network connect clickhouse_network_region_two clickhouse02-server
docker network connect clickhouse_network_region_two clickhouse03-server
```

### 4. Verify Keeper Setup

Check the Keeper configuration and status:

```bash
# Connect to keeper
docker exec -it clickhouse01-keeper bash
clickhouse-keeper-client
get "/keeper/config"

# Check all keeper states
for i in {01..06}; do     
    echo -e "\nServer State for clickhouse${i}-keeper:";     
    echo "mntr" | docker exec -i clickhouse${i}-keeper clickhouse-keeper-client -p 9181 | grep "zk_server_state"; 
done
```

### 5. Create Test Database and Table

Connect to ClickHouse and create test structures:

```bash
# Connect to ClickHouse
docker exec -it clickhouse02-server bash
clickhouse-client 

# Create database
CREATE DATABASE test ON CLUSTER 'dr_cluster';

# Create table
CREATE TABLE sales ON CLUSTER dr_cluster
(
    `id` UInt32,
    `date` Date,
    `customer_id` UInt32,
    `amount` Decimal(10, 2),
    `region` String
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{cluster}/{shard}/default/sales', '{replica}')
PARTITION BY toYYYYMM(date)
ORDER BY (id, date);
```

### 6. Test Data Insertion

Insert sample data to verify the setup:

```sql
-- Insert sample data
INSERT INTO sales (id, date, customer_id, amount, region) VALUES
    (1, '2024-01-01', 101, 525.50, 'North'),
    (2, '2024-01-01', 102, 125.75, 'South'),
    (3, '2024-01-01', 103, 850.00, 'East'),
    (4, '2024-01-02', 101, 225.25, 'North'),
    (5, '2024-01-02', 104, 975.00, 'West'),
    (6, '2024-01-03', 105, 340.80, 'Central'),
    (7, '2024-01-03', 102, 450.25, 'South'),
    (8, '2024-01-04', 106, 675.50, 'North'),
    (9, '2024-01-04', 107, 780.90, 'East'),
    (10, '2024-01-05', 108, 445.60, 'West');

-- Insert larger dataset
INSERT INTO sales
SELECT
    number as id,
    toDate('2024-01-01') + toIntervalDay(number % 90) as date,
    randCanonical() * 1000 + 1 as customer_id,
    toDecimal64(round(randCanonical() * 1000, 2), 2) as amount,
    arrayElement(['North', 'South', 'East', 'West', 'Central'], rand() % 5 + 1) as region
FROM numbers(100);
```

## Disaster Recovery Procedures

### A. Planned Switchover

1. Initial Configuration Check:
   - All Region Two keepers should start with `can_be_leader: false`

2. Enable Leadership in Region Two:
   - Update Keeper configuration to set `can_be_leader: true` for all Region Two nodes
   - Restart Keeper services in both regions

3. Current Keeper Configuration:
```plaintext
server.1=172.25.0.2:9234;participant;1
server.2=172.25.0.3:9234;participant;1
server.3=172.25.0.4:9234;participant;1
server.4=172.26.0.2:9234;learner;1
server.5=172.26.0.3:9234;learner;1
server.6=172.26.0.4:9234;learner;1
```

### B. Emergency Failover

1. Request Leadership Transfer:
```bash
docker exec -it clickhouse04-keeper bash
clickhouse-keeper-client
rqld  # Send leadership request to leader
```

2. Stop Region One Keepers:
```bash
cd region_one
docker compose -f docker-compose-keeper.yaml down
```

3. Start Region Two Failover Configuration:
```bash
cd region_two
docker compose -f docker-compose-keeper-fail.yaml up -d
```

4. Clean Coordination Directories (if needed) on all keeper nodes:
```bash
# Execute for each keeper (04-06)
docker exec -it clickhouse04-keeper bash
rm -rf /var/lib/clickhouse/coordination/log/*
rm -rf /var/lib/clickhouse/coordination/snapshots/*
```

5. Restart Keeper Services:
```bash
for i in {4..6}; do  
    echo "Checking clickhouse0$i-keeper";
    docker restart clickhouse0$i-keeper;  
done
```

6. Verify Keeper Status:
```bash
docker exec -it clickhouse04-keeper bash
clickhouse-keeper-client
get "/keeper/config"
```

### C. Post-Failover Recovery

1. Check Data Accessibility:
```sql
USE test;
SELECT * FROM sales;
```

2. If Inserts Fail, Restore Replica:
```sql
SYSTEM RESTART REPLICA test.sales;
SYSTEM RESTORE REPLICA test.sales;
```

3. Verify Data Insert Functionality:
```sql
INSERT INTO sales (id, date, customer_id, amount, region) VALUES
    (1, '2024-01-01', 101, 525.50, 'North'),
    (2, '2024-01-01', 102, 125.75, 'South');
```

## Configuration Files

### Region Two Keeper Configuration Example (Post-Failover)
```xml
<clickhouse>
    <listen_host>0.0.0.0</listen_host>
    <keeper_server>
        <tcp_port>9181</tcp_port>
        <server_id>1</server_id>
        <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
        <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>
        <coordination_settings>
            <operation_timeout_ms>10000</operation_timeout_ms>
            <session_timeout_ms>30000</session_timeout_ms>
            <raft_logs_level>trace</raft_logs_level>
            <startup_timeout>60</startup_timeout>
            <force_sync>false</force_sync>
            <quorum_reads>false</quorum_reads>
            <max_requests_batch_size>100</max_requests_batch_size>
            <reserved_log_items>1000000</reserved_log_items>
            <snapshots_to_keep>3</snapshots_to_keep>
            <stale_log_gap>10000</stale_log_gap>
            <fresh_log_gap>200</fresh_log_gap>
        </coordination_settings>
        <raft_configuration>
            <server>
                <id>1</id>
                <hostname>172.26.0.2</hostname>
                <port>9234</port>
                <can_become_leader>true</can_become_leader>
            </server>
            <server>
                <id>2</id>
                <hostname>172.26.0.3</hostname>
                <port>9234</port>
                <can_become_leader>true</can_become_leader>
            </server>
            <server>
                <id>3</id>
                <hostname>172.26.0.4</hostname>
                <port>9234</port>
                <can_become_leader>true</can_become_leader>
            </server>
        </raft_configuration>
    </keeper_server>
</clickhouse>
```


