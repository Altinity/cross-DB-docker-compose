# cross-DB-docker-compose

# ClickHouse Multi-Region Cluster Setup with Disaster Recovery

This repository contains the configuration and instructions for setting up a multi-region ClickHouse cluster with disaster recovery capabilities using ClickHouse Keeper.

## Architecture Overview

### Region One (172.25.0.0/16)
- 3 ClickHouse servers (172.25.0.12-14)
- 3 ClickHouse keepers (172.25.0.2-4)

### Region Two (172.26.0.0/16)
- 3 ClickHouse servers (172.26.0.12-14)
- 3 ClickHouse keepers (172.26.0.2-4)

## Prerequisites

- Docker and Docker Compose installed
- Basic understanding of ClickHouse and distributed systems
- Sufficient resources to run multiple Docker containers
- Network connectivity between regions

## Detailed Setup Steps

### 0. Clear everything

#### Delete containers and volumes

```bash
docker compose -f region_one/docker-compose-clickhouse.yaml down
docker compose -f region_two/docker-compose-clickhouse.yaml down
docker compose -f region_one/docker-compose-keeper.yaml down
docker compose -f region_two/docker-compose-keeper.yaml down
docker compose -f region_two/docker-compose-keeper-fail.yaml down
docker compose -f region_two/docker-compose-keeper-fail-step2.yaml down
docker volume rm $(docker volume ls -q | grep -E '^region_one_clickhouse[0-9]+-keeper-(data|logs)$')
docker volume rm $(docker volume ls -q | grep -E '^region_two_clickhouse[0-9]+-keeper-(data|logs)$')
```

#### Revert configuration changes

```bash
sed -E -i 's/server_id="1">(true|false)</server_id="1">true</g' ./region_*/keeper_config/keeper_config_common.xml
sed -E -i 's/server_id="2">(true|false)</server_id="2">true</g' ./region_*/keeper_config/keeper_config_common.xml
sed -E -i 's/server_id="3">(true|false)</server_id="3">true</g' ./region_*/keeper_config/keeper_config_common.xml
sed -E -i 's/server_id="4">(true|false)</server_id="4">false</g' ./region_*/keeper_config/keeper_config_common.xml
sed -E -i 's/server_id="5">(true|false)</server_id="5">false</g' ./region_*/keeper_config/keeper_config_common.xml
sed -E -i 's/server_id="6">(true|false)</server_id="6">false</g' ./region_*/keeper_config/keeper_config_common.xml
```

### 1. Create networks

```bash
# Create the external network:
docker network create --subnet=172.25.0.0/16 clickhouse_network_region_one
docker network create --subnet=172.26.0.0/16 clickhouse_network_region_two
```

### 2. Start Keeper Services

First, launch the Keeper services in both regions and attach them to pre-created networks:

```bash
# Region One
docker compose -f region_one/docker-compose-keeper.yaml up -d

# Region Two
docker compose -f region_two/docker-compose-keeper.yaml up -d
```

### 3. Start ClickHouse Servers

After Keeper services are up, start the ClickHouse servers:

```bash
# Region One
docker compose -f region_one/docker-compose-clickhouse.yaml up -d

# Region Two
docker compose -f region_two/docker-compose-clickhouse.yaml up -d
```

### 4. Configure Cross-Region Networking

Connect containers across regions:

```bash
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

### 5. Keeper setup monitoring 

Constantly check the Keeper configuration and status in separate shell (shell 2):

```bash
watch -x bash -c 'for i in {01..06}; do     echo -e "\nServer State for clickhouse${i}-keeper:";     docker exec -i clickhouse${i}-keeper clickhouse-keeper-client -q mntr | grep "zk_server_state" ; done'
```


### 6. Enable ingestion of data 

Start inserting sample data to verify the setup in separate shell (shell 3):

```bash
watch -x bash -c 'for i in {01..03}; do     echo -e "\nInsert statement for clickhouse${i}-server:";     docker exec -i clickhouse${i}-server clickhouse-client -q "INSERT INTO track_status SELECT rand()" ; done'
```

## Disaster Recovery Procedures

### A. Emergency Failover

1. Initial Configuration Check:
   - One Region Two keepers should start with `can_be_leader: true`
   - Two Region Two keepers should start with `can_be_leader: false`

2. Start quorum in Region Two with only one node:
   - Restart keeper node with `--force-restore` flag
   - Restart keeper node as normal 

3. Clear state for two keeper nodes in Region Two
   - Remove content of log and data dirs for keeper node

4. Connect two nodes as learners in Region Two
   - Restart two nodes with `can_be_leader: false` configuration

5. Promote two nodes in Region Two to participants
   - For two nodes in Region Two update keeper configuration to set `can_be_leader: true` and restart node


Current Keeper Configuration:

```plaintext
server.1=172.25.0.2:9234;participant;1
server.2=172.25.0.3:9234;participant;1
server.3=172.25.0.4:9234;participant;1
server.4=172.26.0.2:9234;learner;1
server.5=172.26.0.3:9234;learner;1
server.6=172.26.0.4:9234;learner;1
```

Target Keeper Configuration:

```plaintext
server.1=172.25.0.2:9234;learner;1
server.2=172.25.0.3:9234;learner;1
server.3=172.25.0.4:9234;learner;1
server.4=172.26.0.2:9234;participant;1
server.5=172.26.0.3:9234;participant;1
server.6=172.26.0.4:9234;participant;1
```


#### 1. Stop keeper ensemble in Region One:

```bash
docker stop clickhouse01-keeper
docker stop clickhouse02-keeper
docker stop clickhouse03-keeper
```

#### 2. Change configuration for keeper servers

We will make single node quorum cluster using keeper server 4 (or whatever you like from Region Two):

```bash
sed -E -i 's/server_id="1">(true|false)</server_id="1">false</g' ./region_*/keeper_config/keeper_config_common.xml
sed -E -i 's/server_id="2">(true|false)</server_id="2">false</g' ./region_*/keeper_config/keeper_config_common.xml
sed -E -i 's/server_id="3">(true|false)</server_id="3">false</g' ./region_*/keeper_config/keeper_config_common.xml
sed -E -i 's/server_id="4">(true|false)</server_id="4">true</g' ./region_*/keeper_config/keeper_config_common.xml
sed -E -i 's/server_id="5">(true|false)</server_id="5">false</g' ./region_*/keeper_config/keeper_config_common.xml
sed -E -i 's/server_id="6">(true|false)</server_id="6">false</g' ./region_*/keeper_config/keeper_config_common.xml
```

#### 3. Start one keeper with force restore and two nodes with empty state 

Restart keeper server 4 with --force-restore flag.
Restart keeper server 5, 6 with new volumes.

```bash
docker compose -f region_two/docker-compose-keeper-fail.yaml up -d
```

#### 4. Restart keeper as normal

```bash
docker compose -f region_two/docker-compose-keeper-fail-step2.yaml up -d
```

#### 5. Switch ingestion to Region Two

Switch ingestion of sample data to use servers in Region Two, in separate shell (shell 3):

```bash
watch -x bash -c 'for i in {04..06}; do     echo -e "\nServer State for clickhouse${i}-server:";     docker exec -i clickhouse${i}-server clickhouse-client -q "INSERT INTO track_status SELECT rand()" ; done'
```

#### 6. Promote keeper 5 node to participant

```bash
### Stop CH05
docker stop clickhouse05-keeper

### Change config for CH05
sed -E -i 's/server_id="5">(true|false)</server_id="5">true</g' ./region_*/keeper_config/keeper_config_common.xml

### Remove CH05 from config
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='reconfig remove "5"'
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='get "/keeper/config"'

### Start CH05
docker start clickhouse05-keeper

### Add CH05 to config back as participant
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='reconfig add "server.5=172.26.0.3:9234;participant;1"'
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='get "/keeper/config"'
```

#### 7. Promote keeper 6 node to participant

```bash
### Stop CH06
docker stop clickhouse06-keeper

### Change config for CH06
sed -E -i 's/server_id="6">(true|false)</server_id="6">true</g' ./region_*/keeper_config/keeper_config_common.xml

### Remove CH06 from config
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='reconfig remove "6"'
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='get "/keeper/config"'

### Start CH06
docker start clickhouse06-keeper

### Add CH06 to config
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='reconfig add "server.6=172.26.0.4:9234;participant;1"'
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='get "/keeper/config"'
```

### B. Resync Region One:

#### 1. Stop running keeper in Region One

```bash
docker compose -f region_one/docker-compose-keeper.yaml down
```

#### 2. Connect network back

```bash
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

#### 3. Clear volumes for keeper in Region One

```bash
docker volume rm $(docker volume ls -q | grep -E '^region_one_clickhouse[0-9]+-keeper-(data|logs)$')
```

#### 4. Demote keepers in Region One to learners

```bash
sed -E -i 's/server_id="1">(true|false)</server_id="1">false</g' ./region_*/keeper_config/keeper_config_common.xml
sed -E -i 's/server_id="2">(true|false)</server_id="2">false</g' ./region_*/keeper_config/keeper_config_common.xml
sed -E -i 's/server_id="3">(true|false)</server_id="3">false</g' ./region_*/keeper_config/keeper_config_common.xml
```

#### 5. Start keeper in Region One

```bash
docker compose -f region_one/docker-compose-keeper.yaml up -d
```

### C. Planned Switchover

1. Initial Configuration Check:
   - All Region One keepers should start with `can_be_leader: false`

2. Enable Leadership in Region One:
   - For each node in Region One update keeper configuration to set `can_be_leader: true` and restart node

3. Move leadership to Region One
   - Execute `rqld` on one of keeper node from Region One, check that leader was changed

Current Keeper Configuration:

```plaintext
server.1=172.25.0.2:9234;learner;1
server.2=172.25.0.3:9234;learner;1
server.3=172.25.0.4:9234;learner;1
server.4=172.26.0.2:9234;participant;1
server.5=172.26.0.3:9234;participant;1
server.6=172.26.0.4:9234;participant;1
```

Target Keeper Configuration:

```plaintext
server.1=172.25.0.2:9234;learner;1
server.2=172.25.0.3:9234;participant;1
server.3=172.25.0.4:9234;participant;1
server.4=172.26.0.2:9234;learner;1
server.5=172.26.0.3:9234;learner;1
server.6=172.26.0.4:9234;learner;1
```

#### 1. Promote keeper 1 node to participant

```bash
### Stop CH01
docker stop clickhouse01-keeper

### Change config for CH01
sed -E -i 's/server_id="1">(true|false)</server_id="1">true</g' ./region_*/keeper_config/keeper_config_common.xml

### Remove CH01 from config
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='reconfig remove "1"'
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='get "/keeper/config"'

### Start CH01
docker start clickhouse01-keeper

### Add CH01 to config
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='reconfig add "server.1=172.25.0.2:9234;participant;1"'
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='get "/keeper/config"'
```

#### 2. Promote keeper 2 node to participant

```bash
### Stop CH02
docker stop clickhouse02-keeper

### Change config for CH02
sed -E -i 's/server_id="2">(true|false)</server_id="2">true</g' ./region_*/keeper_config/keeper_config_common.xml

### Remove CH02 from config
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='reconfig remove "2"'
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='get "/keeper/config"'

### Start CH02
docker start clickhouse02-keeper

### Add CH02 to config
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='reconfig add "server.2=172.25.0.3:9234;participant;1"'
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='get "/keeper/config"'
```

#### 3. Promote keeper 3 node to participant

```bash
### Stop CH03
docker stop clickhouse03-keeper

### Change config for CH03
sed -E -i 's/server_id="3">(true|false)</server_id="3">true</g' ./region_*/keeper_config/keeper_config_common.xml

### Remove CH03 from config
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='reconfig remove "3"'
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='get "/keeper/config"'

### Start CH03
docker start clickhouse03-keeper

### Add CH03 to config
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='reconfig add "server.3=172.25.0.4:9234;participant;1"'
docker exec -it clickhouse04-keeper clickhouse-keeper-client -q='get "/keeper/config"'
```

#### 4. Move leader to Region One

```bash
docker exec -it clickhouse01-keeper clickhouse-keeper-client -q='rqld'
# Sent leadership request to leader.
docker exec -it clickhouse01-keeper clickhouse-keeper-client -q='mntr'
# zk_server_state leader repeat rqld command if needed
```

#### 5. Switch ingestion to Region One

Switch ingestion of sample data to use servers in Region One, in separate shell (shell 3):

```bash
watch -x bash -c 'for i in {01..03}; do     echo -e "\nServer State for clickhouse${i}-server:";     docker exec -i clickhouse${i}-server clickhouse-client -q "INSERT INTO track_status SELECT rand()" ; done'
```

#### 5. Demote keeper 4 node to learner

```bash
### Stop CH04
docker stop clickhouse04-keeper

### Change config for CH04
sed -E -i 's/server_id="4">(true|false)</server_id="4">false</g' ./region_*/keeper_config/keeper_config_common.xml

### Remove CH04 from config
docker exec -it clickhouse01-keeper clickhouse-keeper-client -q='reconfig remove "4"'
docker exec -it clickhouse01-keeper clickhouse-keeper-client -q='get "/keeper/config"'

### Start CH04
docker start clickhouse04-keeper

### Add CH04 to config
docker exec -it clickhouse01-keeper clickhouse-keeper-client -q='reconfig add "server.4=172.26.0.2:9234;learner;1"'
docker exec -it clickhouse01-keeper clickhouse-keeper-client -q='get "/keeper/config"'
```

#### 6. Demote keeper 5 node to learner

```bash
### Stop CH05
docker stop clickhouse05-keeper

### Change config for CH05
sed -E -i 's/server_id="5">(true|false)</server_id="5">false</g' ./region_*/keeper_config/keeper_config_common.xml

### Remove CH05 from config
docker exec -it clickhouse01-keeper clickhouse-keeper-client -q='reconfig remove "5"'
docker exec -it clickhouse01-keeper clickhouse-keeper-client -q='get "/keeper/config"'

### Start CH05
docker start clickhouse05-keeper

### Add CH05 to config
docker exec -it clickhouse01-keeper clickhouse-keeper-client -q='reconfig add "server.5=172.26.0.3:9234;learner;1"'
docker exec -it clickhouse01-keeper clickhouse-keeper-client -q='get "/keeper/config"'
```

#### 7. Demote keeper 6 node to learner

```bash
### Stop CH06
docker stop clickhouse06-keeper

### Change config for CH06
sed -E -i 's/server_id="6">(true|false)</server_id="6">false</g' ./region_*/keeper_config/keeper_config_common.xml

### Remove CH06 from config
docker exec -it clickhouse01-keeper clickhouse-keeper-client -q='reconfig remove "6"'
docker exec -it clickhouse01-keeper clickhouse-keeper-client -q='get "/keeper/config"'

### Start CH06
docker start clickhouse06-keeper

### Add CH06 to config
docker exec -it clickhouse01-keeper clickhouse-keeper-client -q='reconfig add "server.6=172.26.0.4:9234;learner;1"'
docker exec -it clickhouse01-keeper clickhouse-keeper-client -q='get "/keeper/config"'
```
