# Apache Cassandra Demo

CS 5614 topic demonstration: a 3-node Apache Cassandra cluster running in Docker.
You will start the cluster, create a replicated table, and then stop nodes to see
how replication and consistency levels behave.

## What you need

| Tool | Version | Notes |
|---|---|---|
| Docker Desktop (includes Compose) | 4.37.1 or newer (4.96.x tested by the team) | Linux users can use Docker Engine with the Compose plugin |
| Git | any recent version | Or download the repo as a ZIP |
| RAM | 6 GB allocated to Docker | Docker Desktop: Settings > Resources |
| Free port | 9042 | Stop any local Cassandra first |
| Internet | first run only | Pulls the Cassandra image (a few hundred MB) |

VS Code is optional. Any terminal works.

## Quick start

```bash
git clone https://github.com/dtsmedt/Apache-Cassandra-Demo
cd Apache-Cassandra-Demo
docker compose up -d
```

The nodes start one after another, so the first start takes a few minutes. Check
progress with:

```bash
docker compose ps
docker exec cass1 nodetool status
```

The cluster is ready when `nodetool status` shows three rows beginning with `UN`
(Up, Normal), all in `Datacenter: dc1`. A row beginning with `UJ` is still
joining, so wait and run the command again.

## Smoke test

Open a CQL shell on the first node:

```bash
docker exec -it cass1 cqlsh
```

```sql
CREATE KEYSPACE demo WITH replication = {'class':'NetworkTopologyStrategy','dc1':3};
USE demo;
CREATE TABLE t (id text PRIMARY KEY, v int);
INSERT INTO t (id, v) VALUES ('a', 1);
SELECT * FROM t;
```

In a second terminal, see which nodes hold that row:

```bash
docker exec cass1 nodetool getendpoints demo t a
```

You should see three IP addresses, one per node. With replication factor 3 on a
3-node cluster, every node holds a full copy.

## Failure test

In `cqlsh` on `cass1`, run `CONSISTENCY QUORUM;`, then:

1. `docker stop cass3`. Reads and writes still work (2 of 3 replicas are up).
2. `docker stop cass2`. Queries at `QUORUM` now fail with an `Unavailable` error
   because only 1 of 3 replicas is left.
3. In `cqlsh`, run `CONSISTENCY ONE;`. The same queries work again.
4. `docker start cass2 cass3`, then run `docker exec cass1 nodetool status` until
   both nodes show `UN`.

Do not stop `cass1`. It is the only node that exposes port 9042.

## Reset

```bash
docker compose down -v
```

This removes the containers and all data. Run `docker compose up -d` to start fresh.

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| A container exits with code 137 | Out of memory. Raise Docker's RAM allocation. |
| `cass2` or `cass3` never starts | The previous node is not healthy yet. Run `docker logs cass1`. |
| `cqlsh` says connection refused | The node is still starting. Wait until `docker compose ps` shows `healthy`. |
| Datacenter shows `datacenter1` instead of `dc1` | The snitch setting did not apply. Reset with `docker compose down -v` and start again. |

## Files

- `docker-compose.yml`: the 3-node cluster definition.
- `README.md`: this file.
- More still need to be added for the full demo