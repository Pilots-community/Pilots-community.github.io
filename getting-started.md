---
layout: default
title: Getting Started
nav_order: 2
---

# Getting Started

This guide gets you running with two connector participants on a single machine using Docker Compose — the fastest way to see the full dataspace in action.

## Prerequisites

| Dependency | Version | Install |
|---|---|---|
| Java (JDK) | 17+ | `sudo apt install openjdk-17-jdk` |
| Docker + Compose | v2+ | [docs.docker.com/get-docker](https://docs.docker.com/get-docker/) |
| Python 3 | 3.8+ | `sudo apt install python3` |
| `cryptography` (Python) | any | `pip install cryptography` |
| `jq` | any | `sudo apt install jq` |
| `curl` | any | `sudo apt install curl` |

{: .note }
> **Windows users:** The setup scripts require a bash shell. Use [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) (recommended) or Git Bash.

## One-Command Setup

Clone the repository and run the setup script:

```bash
git clone https://github.com/Pilots-community/pilots-dataspace.git
cd pilots-dataspace
./setup.sh
```

This single command:
1. Checks all prerequisites are installed
2. Generates cryptographic keys (data plane tokens, VC issuer keys)
3. Builds Docker images from source (`./gradlew dockerize`)
4. Starts all services via Docker Compose
5. Waits for all containers to become healthy (~60-120 seconds)
6. Seeds identity data (participant DIDs, Verifiable Credentials, STS secrets)

Once complete, you have **two full connector stacks** running locally, simulating two independent organizations.

## What's Running

After setup, you have these services:

| Service | Participant 1 | Participant 2 |
|---|---|---|
| Dashboard (Web UI) | [localhost:3000](http://localhost:3000) | [localhost:3001](http://localhost:3001) |
| Management API | localhost:19193 | localhost:29193 |
| DSP Protocol | localhost:19194 | localhost:29194 |
| Data Plane Public | localhost:38185 | localhost:48185 |
| IdentityHub DID | localhost:7093 | localhost:7083 |
| Vault UI | [localhost:8200](http://localhost:8200) | [localhost:8201](http://localhost:8201) |

## Verify It Works

Run the automated end-to-end test:

```bash
./test-e2e.sh
```

This runs 20 steps with 23 assertions covering:
- Forward direction: participant-1 owns data, participant-2 requests and fetches it (pull + push)
- Reverse direction: participant-2 owns data, participant-1 requests and fetches it (pull + push)

Exit code 0 means everything is working.

## Restarting Without Rebuilding

After a reboot or `docker compose down`, restart without rebuilding:

```bash
./start.sh
```

Use `--clean` to wipe all data (database, vault) and start fresh:

```bash
./setup.sh --clean
```

## Next Steps

- [**Step-by-Step Usage Guide**]({% link step-by-step-guide.md %}) — walk through creating assets, negotiating contracts, and transferring data manually
- [**Deploy to a Cloud VM**]({% link deployment-cloud.md %}) — deploy a connector to production
- [**Architecture**]({% link architecture.md %}) — understand how the components work together
