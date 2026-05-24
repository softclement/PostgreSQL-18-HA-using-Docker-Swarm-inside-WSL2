# PostgreSQL 18 HA PoC using Docker Swarm inside WSL2

## Environment

```text
                Windows 11
                     |
                 WSL2 Ubuntu
                     |
                Docker Swarm
                     |
        -----------------------------------------
        |                 |                     |
    pg-primary       pg-standby1         pg-standby2
```

---

# STEP 1 — Complete Cleanup Before Starting PoC

This step removes:

* Existing Docker Swarm cluster
* Containers
* Images
* Volumes
* Networks
* Previous PoC folders

```bash
# Remove existing swarm stack
docker stack rm pgcluster

# Remove all containers
docker ps -aq | xargs -r docker rm -f

# Remove all docker images
docker images -q | xargs -r docker rmi -f

# Remove all docker volumes
docker volume ls -q | xargs -r docker volume rm -f

# Remove unused docker networks
docker network prune -f

# Leave docker swarm cluster
docker swarm leave --force

# Remove old PoC directory
sudo rm -rf ~/pg-swarm-ha
```

---

# STEP 2 — Verify Clean Environment

```bash
echo 'Docker Images'
docker images

echo '-------------------------------------------'

echo 'Docker Networks'
docker network ls

echo '-------------------------------------------'

echo 'Docker Containers'
docker container ls -a

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

Expected:

* No PostgreSQL containers
* No PostgreSQL images
* No volumes
* No services
* Swarm inactive OR empty

---

# STEP 3 — Verify Docker Installation

```bash
docker --version

docker-compose --version
```

Example:

```text
Docker version 29.x.x

docker-compose version 1.29.x
```

---

# STEP 4 — Pull PostgreSQL 18 Docker Image

```bash
docker pull postgres:18
```

Verify:

```bash
docker images
```

---

# STEP 5 — Create Working Directory

```bash
mkdir ~/pg-swarm-ha

cd ~/pg-swarm-ha
```

---

# STEP 6 — Initialize Docker Swarm

Sometimes in WSL2, Docker Swarm initialization fails because multiple IP addresses exist.

Example error:

```text
could not choose an IP address to advertise
```

Fix:

```bash
docker swarm init --advertise-addr <your-ip>
```

Example:

```bash
docker swarm init --advertise-addr 172.22.26.73
```

Verify:

```bash
docker node ls
```

Expected:

```text
Leader
Ready
Active
```

---

# STEP 7 — Create Docker Swarm Stack YAML

Create file:

```bash
vi docker-stack.yml
```

Add:

```yaml
version: '3.9'

volumes:
  pg-primary-data:
  pg-standby1-data:
  pg-standby2-data:

services:

  pg-primary:
    image: postgres:18
    user: "${UID}:${GID}"
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "5434:5432"
    volumes:
      - ./primary:/var/lib/postgresql/18/docker
    deploy:
      replicas: 1
    networks:
      - pg-swarm-net

  pg-standby1:
    image: postgres:18
    user: "${UID}:${GID}"
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "6001:5432"
    volumes:
      - ./standby1:/var/lib/postgresql/18/docker
    deploy:
      replicas: 1
    networks:
      - pg-swarm-net

  pg-standby2:
    image: postgres:18
    user: "${UID}:${GID}"
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "6002:5432"
    volumes:
      - ./standby2:/var/lib/postgresql/18/docker
    deploy:
      replicas: 1
    networks:
      - pg-swarm-net

networks:
  pg-swarm-net:
    external: true
```

---

# STEP 8 — Create Required Directories

```bash
mkdir primary standby1 standby2 standby_base
```

Verify:

```bash
ls -ltr
```

Expected:

```text
primary
standby1
standby2
standby_base
docker-stack.yml
```

---

# STEP 8.1 — Docker Network creation


```bash
docker network create \
--driver overlay \
--attachable \
pg-swarm-net
```
Verify:

```bash
docker network ls
```
---
# STEP 9 — Deploy Docker Swarm Stack

## Deploy with Progress Display

```bash
docker stack deploy --detach=false -c docker-stack.yml pgcluster
```

## Deploy in Background

```bash
docker stack deploy --detach=true -c docker-stack.yml pgcluster
```

Explanation:

By default, Docker Swarm deploys services in background mode.

Using:

```bash
--detach=false
```

shows deployment progress synchronously.

Verify:

```bash
docker service ls
```

Expected:

```text
1/1 replicas running
```

---

# STEP 10 — Verify Running Containers

```bash
docker ps
```

Expected:

```text
pgcluster_pg-primary
pgcluster_pg-standby1
pgcluster_pg-standby2
```

---

# STEP 11 — Connect to Primary PostgreSQL

```bash
docker exec -it $(docker ps -q -f name=pg-primary) psql -U postgres
```

---

# STEP 12 — Create Replication User

```sql
CREATE ROLE replicator
WITH REPLICATION
LOGIN
PASSWORD 'replpass';
```

Verify PostgreSQL replication settings:

```sql
show wal_level;

show max_wal_senders;
```

Expected:

```text
wal_level = replica

max_wal_senders = 10
```

Exit:

```sql
\q
```

---

# STEP 13 — Configure pg_hba.conf

Instead of installing vim inside container, edit PostgreSQL files directly from WSL2 host machine.

Simple analogy:

```text
Container = Running Engine

Mounted Volume = Actual Hard Disk
```

Edit from host:

```bash
cd ~/pg-swarm-ha
```

Open:

```bash
sudo vi primary/pg_hba.conf
```

Add:

```text
host replication replicator 0.0.0.0/0 md5
```

Restart PostgreSQL service:

```bash
docker service update --force pgcluster_pg-primary
```

Test replication login:

```bash
docker exec -it $(docker ps -q -f name=pg-primary) psql -U replicator -d postgres
```

---

# STEP 14 — Take Base Backup from Primary

Connect to primary container:

```bash
docker exec -it $(docker ps -q -f name=pg-primary) bash
```

Run:

```bash
pg_basebackup \
-h pg-primary \
-D /tmp/standby \
-U replicator \
-P \
-R
```

Exit container:

```bash
exit
```

---

# STEP 15 — Copy Base Backup to Host

```bash
docker cp $(docker ps -q -f name=pg-primary):/tmp/standby ./standby_base
```

Verify:

```bash
ls -ltr standby_base
```

Expected:

```text
PG_VERSION
backup_label
backup_manifest
pg_wal
postgresql.conf
standby.signal
```

---

# STEP 16 — Copy Standby Data

```bash
sudo cp -r standby_base/* standby1/

sudo cp -r standby_base/* standby2/
```

---

# STEP 17 — Restart Standby Services

```bash
docker service update --force pgcluster_pg-standby1

docker service update --force pgcluster_pg-standby2
```

---

# STEP 18 — Verify Streaming Replication

Connect to primary:

```bash
docker exec -it $(docker ps -q -f name=pg-primary) psql -U postgres
```

Run:

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

# STEP 19 — Test Replication

On Primary:

```sql
CREATE TABLE test1(id int, name text);

INSERT INTO test1 VALUES (1,'Clement');
```

---

# STEP 20 — Verify Replication on Standby

Connect standby:

```bash
docker exec -it $(docker ps -q -f name=pg-standby1) psql -U postgres
```

Verify:

```sql
SELECT * FROM test1;
```

Expected:

```text
1 | Clement
```

---

# STEP 21 — Useful Docker Swarm Verification Commands

## List Services

```bash
docker service ls
```

## Service Task Status

```bash
docker service ps pgcluster_pg-primary
```

## View Container Logs

```bash
docker logs <container-id>
```

## List Swarm Nodes

```bash
docker node ls
```

## List Docker Volumes

```bash
docker volume ls
```

## List Docker Networks

```bash
docker network ls
```

---

# STEP 22 — Complete Cleanup After PoC

```bash
# Remove swarm stack
docker stack rm pgcluster

# Remove all containers
docker ps -aq | xargs -r docker rm -f

# Remove all images
docker images -q | xargs -r docker rmi -f

# Remove all docker volumes
docker volume ls -q | xargs -r docker volume rm -f

# Remove unused docker networks
docker network prune -f

# Leave docker swarm cluster
docker swarm leave --force

# Remove PoC directory
sudo rm -rf ~/pg-swarm-ha
```
