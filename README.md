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

# STEP 1 — Clean Existing Docker/Swarm Environment

This step completely removes:

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

# Remove PoC working directory
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

* No containers
* No postgres images
* No volumes
* No services
* Swarm inactive OR no active services

---

# STEP 3 — Verify Docker Installation

```bash
docker --version

docker-compose --version
```

Example:

```bash
Docker version 29.x.x
docker-compose version 1.29.x
```

---

# STEP 4 — Pull PostgreSQL 18 Docker Image

```bash
docker pull postgres:18
```

Verify image:

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

First attempt may fail in WSL2 because multiple IP addresses exist.

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

services:

  pg-primary:
    image: postgres:18
    hostname: pg-primary
    ports:
      - "5434:5432"
    environment:
      POSTGRES_PASSWORD: postgres
    volumes:
      - pg-primary-data:/var/lib/postgresql
    networks:
      - pgnet

  pg-standby1:
    image: postgres:18
    hostname: pg-standby1
    ports:
      - "6001:5432"
    environment:
      POSTGRES_PASSWORD: postgres
    volumes:
      - pg-standby1-data:/var/lib/postgresql
    networks:
      - pgnet

  pg-standby2:
    image: postgres:18
    hostname: pg-standby2
    ports:
      - "6002:5432"
    environment:
      POSTGRES_PASSWORD: postgres
    volumes:
      - pg-standby2-data:/var/lib/postgresql
    networks:
      - pgnet

networks:
  pgnet:

volumes:
  pg-primary-data:
  pg-standby1-data:
  pg-standby2-data:
```

---

# STEP 8 — Deploy Docker Swarm Stack

## Deploy with Progress Display

```bash
docker stack deploy --detach=false -c docker-stack.yml pgcluster
```

## Deploy in Background

```bash
docker stack deploy --detach=true -c docker-stack.yml pgcluster
```

Verify:

```bash
docker service ls
```

Expected:

```text
1/1 replicas running
```

---

# STEP 9 — Verify Containers

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

# STEP 10 — Connect to Primary PostgreSQL

```bash
docker exec -it $(docker ps -q -f name=pg-primary) psql -U postgres
```

---

# STEP 11 — Create Replication User

```sql
CREATE ROLE replicator
WITH REPLICATION
LOGIN
PASSWORD 'replpass';
```

Verify:

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

# STEP 12 — Configure pg_hba.conf

Instead of installing vim inside container, edit files directly from WSL2 host machine.

Simple analogy:

```text
Container = Running Engine
Mounted Volume = Actual Hard Disk
```

Edit directly from host:

```bash
cd ~/pg-swarm-ha
```

Open:

```bash
vi primary/pg_hba.conf
```

Add:

```text
host replication replicator 0.0.0.0/0 md5
```

Restart services:

```bash
docker service update --force pgcluster_pg-primary
```

Test replication user:

```bash
docker exec -it $(docker ps -q -f name=pg-primary) psql -U replicator -d postgres
```

---

# STEP 13 — Take Base Backup from Primary

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

# STEP 14 — Copy Standby Base Backup

Create standby folders:

```bash
mkdir standby1 standby2
```

Copy backup:

```bash
docker cp $(docker ps -q -f name=pg-primary):/tmp/standby ./standby_base
```

Copy standby data:

```bash
cp -r standby_base/* standby1/

cp -r standby_base/* standby2/
```

---

# STEP 15 — Restart Standby Services

```bash
docker service update --force pgcluster_pg-standby1

docker service update --force pgcluster_pg-standby2
```

---

# STEP 16 — Verify Streaming Replication

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

# STEP 17 — Test Replication

On Primary:

```sql
CREATE TABLE test1(id int, name text);

INSERT INTO test1 VALUES (1,'Clement');
```

---

# STEP 18 — Verify on Standby

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

# STEP 19 — Useful Docker Swarm Verification Commands

## List Services

```bash
docker service ls
```

## Service Task Status

```bash
docker service ps pgcluster_pg-primary
```

## Container Logs

```bash
docker logs <container-id>
```

## List Nodes

```bash
docker node ls
```

## List Volumes

```bash
docker volume ls
```

---

# STEP 20 — Complete Cleanup

```bash
# Remove swarm stack
docker stack rm pgcluster

# Remove all containers
docker ps -aq | xargs -r docker rm -f

# Remove all images
docker images -q | xargs -r docker rmi -f

# Remove all volumes
docker volume ls -q | xargs -r docker volume rm -f

# Remove unused networks
docker network prune -f

# Leave swarm cluster
docker swarm leave --force

# Remove working directory
sudo rm -rf ~/pg-swarm-ha
```

