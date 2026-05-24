# PostgreSQL 18 HA PoC using Docker Swarm inside WSL2

## Key Notes for PostgreSQL 18

- Mount at `/var/lib/postgresql` not `/var/lib/postgresql/data`. Data lives at `/var/lib/postgresql/18/docker/`.
- Named volumes required — bind mounts cause Swarm task scheduling to hang in WSL2.
- Use `psql -tAc 'SHOW data_directory'` dynamically instead of hardcoding paths.
- `primary_conninfo` is written to `postgresql.auto.conf` by `pg_basebackup -R`.
- Use full Swarm service DNS name `pgcluster_pg-primary` as hostname, not `pg-primary`.

---

# STEP 1 — Complete Cleanup

```bash
docker stack rm pgcluster
sleep 5
docker ps -aq | xargs -r docker rm -f
docker images -q | xargs -r docker rmi -f
docker volume ls -q | xargs -r docker volume rm -f
docker network prune -f
docker swarm leave --force
sudo rm -rf ~/pg-swarm-ha
```

# STEP 2 — Verify Clean Environment

```bash
docker images && docker network ls && docker container ls -a && docker volume ls
docker service ls && docker node ls 2>/dev/null || echo "Swarm inactive"
```

# STEP 3 — Pull PostgreSQL 18

```bash
docker pull postgres:18
docker images
```

# STEP 4 — Create Working Directory

```bash
mkdir ~/pg-swarm-ha
cd ~/pg-swarm-ha
mkdir primary standby1 standby2 standby_base
```

# STEP 5 — Initialize Docker Swarm

```bash
MY_IP=$(hostname -I | awk '{print $1}')
echo "Using IP: $MY_IP"
docker swarm init --advertise-addr $MY_IP
docker node ls
```

Expected: `Leader / Ready / Active`

# STEP 6 — Create Overlay Network (MUST be before stack deploy)

```bash
docker network create --driver overlay --attachable pg-swarm-net
docker network ls
```

# STEP 7 — Create docker-stack.yml

```bash
cat > ~/pg-swarm-ha/docker-stack.yml << 'EOF'
version: '3.9'

services:

  pg-primary:
    image: postgres:18
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "5434:5432"
    volumes:
      - pg-primary-data:/var/lib/postgresql
    deploy:
      replicas: 1
    networks:
      - pg-swarm-net

  pg-standby1:
    image: postgres:18
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "6001:5432"
    volumes:
      - pg-standby1-data:/var/lib/postgresql
    deploy:
      replicas: 1
    networks:
      - pg-swarm-net

  pg-standby2:
    image: postgres:18
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "6002:5432"
    volumes:
      - pg-standby2-data:/var/lib/postgresql
    deploy:
      replicas: 1
    networks:
      - pg-swarm-net

volumes:
  pg-primary-data:
  pg-standby1-data:
  pg-standby2-data:

networks:
  pg-swarm-net:
    external: true
EOF
```

Key differences from older PostgreSQL:
- Mount is `/var/lib/postgresql` not `/var/lib/postgresql/data`
- Named volumes not bind mounts
- Volumes declared in `volumes:` section

# STEP 8 — Deploy Stack

```bash
cd ~/pg-swarm-ha
docker stack deploy --detach=false -c docker-stack.yml pgcluster
docker service ls
docker ps
```

Expected: three containers running.

# STEP 9 — Create Replication User

```bash
docker exec -it $(docker ps -q -f name=pg-primary) psql -U postgres
```

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replpass';
SHOW wal_level;        -- expected: replica
SHOW max_wal_senders;  -- expected: 10
\q
```

# STEP 10 — Configure pg_hba.conf

Use SHOW data_directory dynamically to avoid hardcoding path:

```bash
docker exec -it $(docker ps -q -f name=pg-primary) bash -c \
  "echo 'host replication replicator 0.0.0.0/0 md5' >> \$(psql -U postgres -tAc 'SHOW data_directory')/pg_hba.conf"

docker exec -it $(docker ps -q -f name=pg-primary) bash -c \
  "tail -3 \$(psql -U postgres -tAc 'SHOW data_directory')/pg_hba.conf"
```

Restart to reload config:

```bash
docker service update --force pgcluster_pg-primary
sleep 15
docker ps -f name=pg-primary
```

Test replication login:

```bash
docker exec -it $(docker ps -q -f name=pg-primary) \
  psql -U replicator -d postgres -c "SELECT 1;"
```

# STEP 11 — Take Base Backup

Use full Swarm service DNS name as host:

```bash
docker exec -it $(docker ps -q -f name=pg-primary) bash -c "
  pg_basebackup \
    -h pgcluster_pg-primary \
    -D /tmp/standby \
    -U replicator \
    -P \
    -R \
    --port=5432
"
```

Password: `replpass`

Expected: `23722/23722 kB (100%), 1/1 tablespace`

# STEP 12 — Copy Backup to Host

```bash
docker cp $(docker ps -q -f name=pg-primary):/tmp/standby/. ~/pg-swarm-ha/standby_base/
ls -ltr ~/pg-swarm-ha/standby_base/
```

# STEP 13 — Verify primary_conninfo

The -R flag writes to postgresql.auto.conf not postgresql.conf:

```bash
cat ~/pg-swarm-ha/standby_base/postgresql.auto.conf
```

Confirm `host=pgcluster_pg-primary`. If wrong, fix:

```bash
sed -i "s/host=pg-primary/host=pgcluster_pg-primary/" \
  ~/pg-swarm-ha/standby_base/postgresql.auto.conf
```

# STEP 14 — Copy Standby Data into Containers

In PostgreSQL 18, data directory inside container is /var/lib/postgresql/18/docker/:

```bash
docker cp ~/pg-swarm-ha/standby_base/. \
  $(docker ps -q -f name=pg-standby1):/var/lib/postgresql/18/docker/

docker cp ~/pg-swarm-ha/standby_base/. \
  $(docker ps -q -f name=pg-standby2):/var/lib/postgresql/18/docker/

docker exec $(docker ps -q -f name=pg-standby1) \
  ls /var/lib/postgresql/18/docker/standby.signal

docker exec $(docker ps -q -f name=pg-standby2) \
  ls /var/lib/postgresql/18/docker/standby.signal
```

# STEP 15 — Restart Standby Services

```bash
docker service update --force pgcluster_pg-standby1
docker service update --force pgcluster_pg-standby2
docker ps
```

# STEP 16 — Verify Streaming Replication

```bash
docker exec -it $(docker ps -q -f name=pg-primary) psql -U postgres -c \
  "SELECT application_name, client_addr, state, sync_state FROM pg_stat_replication;"
```

Expected: two rows, both `state = streaming`

# STEP 17 — Test Replication End-to-End

Write on primary:

```bash
docker exec -it $(docker ps -q -f name=pg-primary) psql -U postgres -c "
  CREATE TABLE test1 (id int, name text);
  INSERT INTO test1 VALUES (1, 'Clement');
"
```

Read from standbys:

```bash
docker exec -it $(docker ps -q -f name=pg-standby1) psql -U postgres -c "SELECT * FROM test1;"
docker exec -it $(docker ps -q -f name=pg-standby2) psql -U postgres -c "SELECT * FROM test1;"
```

Expected: `1 | Clement` on both.

Confirm read-only:

```bash
docker exec -it $(docker ps -q -f name=pg-standby1) psql -U postgres -c \
  "INSERT INTO test1 VALUES (2, 'test');"
```

Expected: `ERROR: cannot execute INSERT in a read-only transaction`

# STEP 18 — Useful Verification Commands

```bash
docker service ls
docker service ps pgcluster_pg-primary
docker logs $(docker ps -q -f name=pg-primary)
docker node ls
docker volume ls
docker network ls
docker exec $(docker ps -q -f name=pg-primary) psql -U postgres -tAc "SHOW data_directory"
```

# STEP 19 — Complete Cleanup

```bash
docker stack rm pgcluster
sleep 5
docker ps -aq | xargs -r docker rm -f
docker images -q | xargs -r docker rmi -f
docker volume ls -q | xargs -r docker volume rm -f
docker network prune -f
docker swarm leave --force
sudo rm -rf ~/pg-swarm-ha
```

---

## Troubleshooting Reference

| Symptom | Cause | Fix |
|---|---|---|
| Task stuck at pending forever | Bind mount volumes in Swarm on WSL2 | Use named volumes |
| Container exits code 1, empty logs | Wrong mount path for PostgreSQL 18 | Mount `/var/lib/postgresql` not `/data` |
| pg_basebackup fails with socket error | Using localhost or short hostname | Use `pgcluster_pg-primary` |
| pg_hba.conf path not found | PostgreSQL 18 uses different subdirectory | Use `SHOW data_directory` dynamically |
| primary_conninfo not in postgresql.conf | Normal behaviour | Check `postgresql.auto.conf` instead |
| Swarm init fails with IP error | WSL2 multiple network interfaces | Use `--advertise-addr $(hostname -I \| awk '{print $1}')` |
