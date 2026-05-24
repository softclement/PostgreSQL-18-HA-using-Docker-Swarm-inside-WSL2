# PostgreSQL 18 HA using Docker Swarm inside WSL2

## Architecture

```text
                Windows 11
                     |
                 WSL2 Ubuntu
                     |
                 Docker Swarm
                     |
        -----------------------------------------
        |                  |                    |
    pg-primary        pg-standby1         pg-standby2
         |                  |                    |
         -------- Streaming Replication ---------
```

---

# PostgreSQL 18 Docker Swarm HA PoC

This PoC demonstrates:

* PostgreSQL 18
* Docker Swarm
* WSL2 Ubuntu
* Streaming Replication
* Manual Failover Concepts
* Container-based PostgreSQL HA

Without using:

* Patroni
* Kubernetes
* Pgpool
* Repmgr

---

# STEP 1 — Clean Old Environment

## Remove Old Swarm Stack

```bash
docker stack rm pgcluster
```

---

## Remove Old Containers

```bash
docker ps -aq | xargs docker rm -f
```

---

## Remove Old Images

```bash
docker images -q | xargs docker rmi -f
```

---

## Remove Old Volumes

```bash
docker volume ls -q | xargs docker volume rm -f
```

---

## Remove Old Networks

```bash
docker network rm pg-swarm-net
```

---

## Remove Old Folders

```bash
sudo rm -rf primary standby1 standby2 standby_base
```

---

# STEP 2 — Verify Docker Environment

```bash
echo 'Docker Images'
docker images

echo '-------------------------------------------'

echo 'Docker Networks'
docker network ls

echo '-------------------------------------------'

echo 'Docker Containers'
docker container ls

echo '-------------------------------------------'

echo 'Docker Volumes'
docker volume ls

echo '-------------------------------------------'

echo 'Docker Services'
docker service ls

echo '-------------------------------------------'

echo 'Docker Nodes'
docker node ls
```

---

# STEP 3 — Prerequisites

## Verify Docker

```bash
docker --version
```

---

## Verify Docker Compose

```bash
docker-compose --version
```

---

## Pull PostgreSQL 18 Image

```bash
docker pull postgres:18
```

---

# STEP 4 — Initialize Docker Swarm

## Standard Command

```bash
docker swarm init
```

---

## Possible Error

```text
Error response from daemon:
could not choose an IP address to advertise
since this system has multiple addresses
```

---

## Fix

```bash
docker swarm init --advertise-addr <your-ip>
```

Example:

```bash
docker swarm init --advertise-addr 172.22.26.73
```

---

## Verify Swarm

```bash
docker node ls
```

---

# STEP 5 — Create Project Directory

```bash
mkdir ~/pg-swarm-ha
cd ~/pg-swarm-ha
```

---

# STEP 6 — Create Docker Swarm Stack YAML

Create file:

```bash
vi docker-stack.yml
```

---

## docker-stack.yml

```yaml
version: '3.9'

volumes:
  pg-primary-data:
  pg-standby1-data:
  pg-standby2-data:

services:

  pg-primary:
    image: postgres:18
    hostname: pg-primary
    user: postgres
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "5434:5432"
    volumes:
      - ./primary:/var/lib/postgresql/18/docker
    deploy:
      replicas: 1
    networks:
      - pgnet

  pg-standby1:
    image: postgres:18
    hostname: pg-standby1
    user: postgres
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "6001:5432"
    volumes:
      - ./standby1:/var/lib/postgresql/18/docker
    deploy:
      replicas: 1
    networks:
      - pgnet

  pg-standby2:
    image: postgres:18
    hostname: pg-standby2
    user: postgres
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "6002:5432"
    volumes:
      - ./standby2:/var/lib/postgresql/18/docker
    deploy:
      replicas: 1
    networks:
      - pgnet

networks:
  pgnet:
```

---

# STEP 7 — Create Required Directories

```bash
mkdir primary standby1 standby2 standby_base
```

---

# STEP 8 — Deploy Docker Swarm Stack

## Deploy

```bash
docker stack deploy -c docker-stack.yml pgcluster
```

By default Docker deploys services in detached/background mode.

---

## Deploy with Progress Display

```bash
docker stack deploy --detach=false -c docker-stack.yml pgcluster
```

Useful for learning and monitoring deployment progress.

---

## Deploy Without Progress

```bash
docker stack deploy --detach=true -c docker-stack.yml pgcluster
```

---

# STEP 9 — Verify Services

## List Services

```bash
docker service ls
```

---

## List Containers

```bash
docker ps
```

---

## List Service Tasks

```bash
docker service ps pgcluster_pg-primary

docker service ps pgcluster_pg-standby1

docker service ps pgcluster_pg-standby2
```

---

# STEP 10 — Configure Primary PostgreSQL

## Login to Primary

```bash
docker exec -it $(docker ps -q -f name=pg-primary) psql -U postgres
```

---

## Create Replication User

```sql
CREATE ROLE replicator
WITH REPLICATION
LOGIN
PASSWORD 'replpass';
```

---

## Verify WAL Settings

```sql
SHOW wal_level;

SHOW max_wal_senders;
```

Expected:

```text
wal_level = replica
max_wal_senders = 10
```

---

# STEP 11 — Edit PostgreSQL Configuration

## Important Observation

Installing vim inside container may fail because container runs with limited permissions.

---

## Best Practice

Edit mounted PostgreSQL files directly from WSL2 host machine.

---

## Simple Analogy

```text
Container = Running engine
Mounted volume = Actual hard disk

You edit the disk from outside
instead of entering engine room.
```

---

## Edit Files from Host

```bash
cd ~/pg-swarm-ha

vi primary/postgresql.conf

vi primary/pg_hba.conf
```

---

## Add in pg_hba.conf

```text
host replication replicator 0.0.0.0/0 md5
host all all 0.0.0.0/0 md5
```

---

# STEP 12 — Restart Stack

```bash
docker stack rm pgcluster
```

Wait few seconds.

---

## Deploy Again

```bash
docker stack deploy -c docker-stack.yml pgcluster
```

---

# STEP 13 — Test Replication Connection

```bash
docker exec -it $(docker ps -q -f name=pg-primary) \
psql -U replicator -d postgres
```

If login works, replication configuration is correct.

---

# STEP 14 — Take Base Backup

```bash
docker exec -it $(docker ps -q -f name=pg-primary) bash
```

---

## Create Base Backup

```bash
pg_basebackup \
-h pg-primary \
-D /tmp/standby \
-U replicator \
-P \
-R
```

Password:

```text
replpass
```

Exit container.

---

# STEP 15 — Copy Standby Backup to Host

```bash
docker cp $(docker ps -q -f name=pg-primary):/tmp/standby ./standby_base
```

---

# STEP 16 — Stop Stack

```bash
docker stack rm pgcluster
```

Wait few seconds.

---

# STEP 17 — Copy Standby Data

```bash
cp -r standby_base/* standby1/

cp -r standby_base/* standby2/
```

No permission fix required.

---

# STEP 18 — Start Stack Again

```bash
docker stack deploy -c docker-stack.yml pgcluster
```

---

# STEP 19 — Verify Replication

## Check on Primary

```bash
docker exec -it $(docker ps -q -f name=pg-primary) \
psql -U postgres
```

---

## Verify Streaming Replication

```sql
SELECT application_name,
       client_addr,
       state
FROM pg_stat_replication;
```

Expected:

```text
streaming
```

---

# STEP 20 — Replication Testing

## Create Table on Primary

```sql
CREATE TABLE test1(id int, name text);

INSERT INTO test1 VALUES (1,'Clement');
```

---

## Verify on Standby

```bash
docker exec -it $(docker ps -q -f name=pg-standby1) \
psql -U postgres
```

---

## Query

```sql
SELECT * FROM test1;
```

Expected:

```text
 id |  name
----+---------
  1 | Clement
```

---

# STEP 21 — Verify Docker Internal Details

## Docker Images

```bash
docker images
```

---

## Docker Volumes

```bash
docker volume ls
```

---

## Docker Networks

```bash
docker network ls
```

---

## Docker Services

```bash
docker service ls
```

---

## Docker Nodes

```bash
docker node ls
```

---

## Docker Container Details

```bash
docker inspect <container_id>
```

---

## Actual Docker Image Location

```bash
docker info | grep "Docker Root Dir"
```

Usually:

```text
/var/lib/docker
```

---

# STEP 22 — Cleanup Complete PoC

## Remove Stack

```bash
docker stack rm pgcluster
```

---

## Remove Containers

```bash
docker ps -aq | xargs docker rm -f
```

---

## Remove Images

```bash
docker images -q | xargs docker rmi -f
```

---

## Remove Volumes

```bash
docker volume ls -q | xargs docker volume rm -f
```

---

## Remove Networks

```bash
docker network rm pg-swarm-net
```

---

## Remove Local Directories

```bash
sudo rm -rf primary standby1 standby2 standby_base
```

---

# Final Result

Successfully built:

* PostgreSQL 18 HA
* Docker Swarm
* Streaming Replication
* Primary + Multiple Standbys
* WSL2-based PostgreSQL Lab Environment
* Containerized PostgreSQL Learning Platform
