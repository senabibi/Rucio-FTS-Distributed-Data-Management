# Rucio / FTS Distributed Data Management — Local Exploration

## Overview

This repository documents a hands-on, local exploration of [Rucio](https://rucio.cern.ch/)—CERN's open-source scientific data management platform—and its integration with [FTS](https://fts.web.cern.ch/) (File Transfer Service). The primary objective was not to deploy a production-grade cluster, but to spin up Rucio's official local development environment to gain a deep, practical understanding of grid-scale data management concepts. This includes Rucio Storage Elements (RSE) topology, X.509 authentication, data cataloging, and policy-driven replication using real scientific datasets.

This exploration was conducted to build a solid working knowledge of the Worldwide LHC Computing Grid (WLCG) data management stack in preparation for DevOps and systems engineering roles in scientific computing environments.

> [!NOTE]  
> **Scope & Environment:** This is a local, single-node exploration utilizing Rucio's official Docker-based development environment (`rucio/rucio`, `etc/docker/dev`). No production infrastructure or live grid resources were used. The focus was entirely on architectural understanding and hands-on command-line interaction.

---

## Learning Objectives

- **Orchestration vs. Execution:** Understand the functional boundary between Rucio (policy, metadata, and routing decisions) and FTS (physical file transfer execution).
- **Grid Security Standards:** Experience why grid systems rely on X.509 proxy certificates and PKI rather than traditional password/token-based logins.
- **Storage Topology Modeling:** Learn how Rucio maps heterogeneous storage endpoints using Rucio Storage Elements (RSEs) and protocols.
- **Declarative Data Replication:** Observe how Rucio automates data replication using high-level policies (rules) rather than manual transfer commands.

---

## Environment Setup

The local environment was built using the official Rucio developer profile, which orchestrates a multi-service stack via Docker Compose:

* **Core Services:** Rucio Server (Python/Flask) + PostgreSQL (Catalog Database)
* **Transfer Broker:** FTS (File Transfer Service) + MySQL Database
* **Message Broker:** ActiveMQ (for event and state notifications)
* **Monitoring Stack:** Elasticsearch, InfluxDB, Graphite
* **Simulated Storage Nodes:** 
  * 5× XRootD Storage Elements (`XRD1` to `XRD5`)
  * 1× WebDAV Storage Element (`WEB1`)
  * 1× SSH/SFTP Storage Element (`SSH1`)

The stack was initialized using:
```bash
docker compose --file etc/docker/dev/docker-compose.yml --profile storage up -d
```

---

## Step-by-Step Exploration & Findings

### 1. Database Initialization & Schema Migrations
Although the Rucio server container spun up successfully, running `rucio whoami` initially failed due to an uninitialized database schema. To resolve this, I ran Rucio's Alembic-based database migration and initialization tool:

```bash
tools/reset_database.py
```
This successfully initialized the PostgreSQL schema, resolving the connection and confirming the CLI could communicate with the API server:
* **Account:** `root`
* **Status:** `ACTIVE`

---

### 2. Conceptualizing RSEs (Rucio Storage Elements)
To understand how Rucio abstracts physical storage, I manually registered a custom RSE and bound a protocol configuration to it:

```bash
# Registering the Storage Element
rucio-admin rse add ATLAS-DEMO-RSE

# Binding the WebDAV protocol configuration
rucio-admin rse add-protocol \
  --rse ATLAS-DEMO-RSE \
  --hostname web1 \
  --scheme https \
  --prefix /rucio/ \
  --port 443 \
  --impl rucio.rse.protocols.gfal.Default \
  --domains-upload '{"wan": 1, "lan": 1}' \
  --domains-download '{"wan": 1, "lan": 1}'
```
This experiment reinforced a fundamental concept: **an RSE is a logical abstraction**. It represents a named endpoint in the database and has no physical storage capabilities until protocols (specifying scheme, port, hostname, and path prefix) are attached to it.

---

### 3. Diagnosing Grid Authentication Failures (HTTP 401)
Attempts to upload files to the manually configured RSE failed with an `HTTP 401: Authentication Error`. 

This served as a practical introduction to Grid security: storage endpoints in the WLCG expect **X.509 proxy certificates** for authn/authz. Because no proxy credential had been generated on the client container, the storage server rejected the write request before it ever reached Rucio's core logic.

---

### 4. Bootstrapping the Official Test Topology
To bypass manual certificate generation and routing setup, I used Rucio’s bootstrap script to generate a fully credentialed, multi-RSE environment:

```bash
bash tools/docker_activate_rses.sh
```

This automated script:
1. Initialized an X.509 user proxy using XRootD tools (`xrdgsiproxy init`).
2. Registered the pre-defined test RSEs (`XRD1` to `XRD5`, `WEB1`, `SSH1`).
3. Configured **network distances and routing weights** between RSEs to simulate WAN/LAN transfer costs (which Rucio uses to determine optimal transfer paths).
4. Set account storage quotas and data scopes.
5. Delegated client certificates to FTS (`https://fts:8446`) to enable FTS to perform transfers on behalf of the user.

---

### 5. Cataloging a Real ATLAS Open Data File
To make the transfer scenarios realistic, I downloaded a public dataset metadata file (`metadata.csv`, ~270 KB) from the official [ATLAS Open Data Portal](https://opendata.atlas.cern/) and uploaded it to Rucio:

```bash
rucio upload --rse XRD1 --scope test metadata.csv
```

I then verified that Rucio successfully cataloged the file, recorded its size, generated an Adler32 checksum, and mapped its physical path (LFN to PFN mapping) on `XRD1`:

```
+---------+--------------+------------+-----------+---------------------------------------------------------+
| SCOPE   | NAME         | FILESIZE   | ADLER32   | RSE: REPLICA                                             |
|---------+--------------+------------+-----------+---------------------------------------------------------|
| test    | metadata.csv | 270.820 kB | 35295918  | XRD1: root://xrd1:1094//rucio/test/86/c5/metadata.csv    |
+---------+--------------+------------+-----------+---------------------------------------------------------+
```

---

### 6. Defining a Replication Policy
I declared a replication rule requesting Rucio to maintain at least one replica of the dataset on `XRD2`:

```bash
rucio add-rule test:metadata.csv 1 XRD2
```

After registering the rule, its state remained stuck in `REPLICATING`. This prompted a debugging phase to identify the bottleneck.

---

### 7. Troubleshooting FTS Service Downtime
By checking container statuses from the host shell, I discovered that the FTS daemon container (`dev-fts-1`) had crashed and exited:

```
dev-fts-1   docker.io/rucio/test-fts:latest   Exited (0) 17 minutes ago
```

Because FTS was offline, Rucio's conveyor daemons could not submit jobs to `https://fts:8446` (throwing resolution errors). I restarted the service:

```bash
docker compose --file etc/docker/dev/docker-compose.yml start fts
```

---

### 8. Manually Triggering Conveyor Daemons
In Rucio's development environment, background conveyor daemons do not run continuously and must be executed manually to progress the replication pipeline:

```bash
# 1. Submit pending transfer rules as jobs to FTS
rucio-conveyor-submitter --run-once

# 2. Poll FTS to monitor transfer completion state
rucio-conveyor-poller --run-once --older-than 0

# 3. Finalize and update metadata states in Rucio catalog
rucio-conveyor-finisher --run-once
```

Once the daemons executed, the replication status updated successfully:

```
rucio rule-info a28de71d833a4975ad6cd08ba903644c
State: OK
Locks OK/REPLICATING/STUCK: 1/0/0
```

Querying the replica catalog confirmed the file was now physically present on both storage sites under the same logical name and checksum:

```
+---------+--------------+------------+-----------+---------------------------------------------------------+
| SCOPE   | NAME         | FILESIZE   | ADLER32   | RSE: REPLICA                                             |
|---------+--------------+------------+-----------+---------------------------------------------------------|
| test    | metadata.csv | 270.820 kB | 35295918  | XRD1: root://xrd1:1094//rucio/test/86/c5/metadata.csv    |
| test    | metadata.csv | 270.820 kB | 35295918  | XRD2: root://xrd2:1095//rucio/test/86/c5/metadata.csv    |
+---------+--------------+------------+-----------+---------------------------------------------------------+
```

---

## Key Architecture Takeaways

1. **Rucio does not move data:** It operates purely as the control and metadata plane. It tracks dataset locations, manages user policies, and delegates actual byte copying to external transfer systems (like FTS).
2. **Replication is Declarative:** Instead of invoking an imperative copy command, users declare a target state via rules (e.g., *"keep 2 replicas of dataset X in Europe"*). The system continuously works asynchronously to reconcile the current state with the target policy.
3. **Identity & Delegation:** Grid-scale file transfers rely heavily on X.509 proxy delegation. For Rucio and FTS to transfer files on a user's behalf, user proxy certificates must be delegated to the transfer middleware.
4. **Topology Aware Routing:** Distances and weights configured between RSEs allow Rucio to dynamically calculate the most efficient path for moving massive datasets across heterogeneous storage systems globally.

---

## Disclaimer

This project is an independent, educational, and local exploration using Rucio's open-source development tools. It is not affiliated with CERN's production Rucio instance, nor does it interface with any active grid infrastructure or production scientific datasets. All storage nodes, certificate environments, and transfer operations were simulated locally.
