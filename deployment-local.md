---
layout: default
title: Local Development
parent: Deployment
nav_order: 1
---

# Local Development Setup

Run two complete connector participants on a single machine using Docker Compose. This simulates two independent organizations without needing cloud VMs.

## Architecture

```
Single dev machine
┌───────────────────────────────────────────────────────────────────────┐
│                                                                       │
│ Participant 1                         Participant 2                   │
│ ─────────────────────                 ─────────────────────           │
│ identityhub        7090-96            identityhub        7080-86      │
│ controlplane       18181              controlplane       28181        │
│   mgmt:19193 DSP:19194                 mgmt:29193 DSP:29194           │
│ dataplane          38181              dataplane          48181        │
│   public:38185                          public:48185                  │
│ dashboard          3000               dashboard          3001         │
│ vault              8200               vault              8201         │
│ did-server         9876               did-server         9877         │
│                                                                       │
│ postgres  15432 (shared — separate databases per participant)         │
│ http-receiver  4000 (shared — push transfer test endpoint)            │
└───────────────────────────────────────────────────────────────────────┘
```

Each participant has its own Vault and DID server with independently generated issuer keys and VCs, matching the cloud deployment's multi-issuer trust model.

## Quick Start

From the **project root**:

```bash
./setup.sh
```

Or step by step:

```bash
# 1. Generate keys
./generate-keys.sh

# 2. Build Docker images
./gradlew dockerize

# 3. Start all services
docker compose up -d

# 4. Wait for all containers to be healthy (~60-120s)
docker compose ps

# 5. Seed identity data for both participants
./deployment/seed.sh
```

## Dashboards

| Dashboard | URL | Participant |
|---|---|---|
| Participant 1 | [http://localhost:3000](http://localhost:3000) | Provider in the forward direction |
| Participant 2 | [http://localhost:3001](http://localhost:3001) | Consumer in the forward direction |

## Environment Variables for API Calls

When making API calls (e.g., catalog requests), you need to use Docker service names for container-to-container communication:

```bash
P1_DSP="http://participant-1-controlplane:19194/protocol"
P1_DID="did:web:participant-1-identityhub%3A7093"
P2_DSP="http://participant-2-controlplane:29194/protocol"
P2_DID="did:web:participant-2-identityhub%3A7083"
```

## Running the E2E Test

```bash
./test-e2e.sh
```

Runs 20 steps with 23 assertions across both directions (forward + reverse, pull + push).

## Stopping and Restarting

```bash
# Stop services (data preserved)
docker compose down

# Stop and wipe all data (database, vault)
docker compose down -v

# Restart without rebuilding
./start.sh

# Full rebuild + restart
./setup.sh

# Either command supports --clean to wipe volumes first
./start.sh --clean
```

## Running Natively (Without Docker)

You can also run the runtimes directly with `java -jar`. This requires local Vault and PostgreSQL instances.

### Start Vault

```bash
vault server -dev -dev-root-token-id=root-token
```

### Start PostgreSQL

```bash
sudo -u postgres psql <<'SQL'
CREATE USER edc WITH PASSWORD 'edc';
CREATE DATABASE participant_1_controlplane OWNER edc;
CREATE DATABASE participant_2_controlplane OWNER edc;
CREATE DATABASE participant_1_dataplane OWNER edc;
CREATE DATABASE participant_2_dataplane OWNER edc;
SQL
```

### Start Runtimes

In separate terminals:

```bash
# Participant-1 Control Plane
java -Dedc.fs.config=config/controlplane-participant-1.properties \
     -jar runtimes/controlplane/build/libs/controlplane.jar

# Participant-2 Control Plane
java -Dedc.fs.config=config/controlplane-participant-2.properties \
     -jar runtimes/controlplane/build/libs/controlplane.jar

# Participant-1 Data Plane
java -Dedc.fs.config=config/dataplane-participant-1.properties \
     -jar runtimes/dataplane/build/libs/dataplane.jar

# Participant-2 Data Plane
java -Dedc.fs.config=config/dataplane-participant-2.properties \
     -jar runtimes/dataplane/build/libs/dataplane.jar
```

When running natively, use `localhost` URLs for environment variables:

```bash
P1_DSP="http://localhost:19194/protocol"
P1_DID="did:web:localhost%3A7093"
P2_DSP="http://localhost:29194/protocol"
P2_DID="did:web:localhost%3A7083"
```
