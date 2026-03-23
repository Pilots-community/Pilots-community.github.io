---
layout: default
title: Cloud VM Deployment
parent: Deployment
nav_order: 2
---

# Deploy to a Cloud VM

Deploy a self-contained connector stack to a cloud VM (Azure, AWS, GCP, etc.) or any server with a public IP. Each organization runs one connector on their own infrastructure.

## Architecture

Each VM is fully independent — it generates its own keys, signs its own VCs, and serves its own issuer DID document.

```
Organization A (VM: 20.50.100.10)        Organization B (VM: 20.50.100.20)
+-----------------------------+          +-----------------------------+
| identityhub        7090-96  |          | identityhub        7090-96  |
| controlplane       18181    |          | controlplane       18181    |
|   mgmt:19193 DSP:19194      |  <---->  |   mgmt:19193 DSP:19194      |
| dataplane          38181    |          | dataplane          38181    |
|   public:38185              |          |   public:38185              |
| dashboard          3000     |          | dashboard          3000     |
| vault              8200     |          | vault              8200     |
| postgres          15432     |          | postgres          15432     |
+-----------------------------+          +-----------------------------+
```

All VMs use the same ports — no conflicts since they're on different hosts.

---

## Prerequisites

| Dependency | Version | Install |
|---|---|---|
| Docker + Compose | v2+ | [docs.docker.com/get-docker](https://docs.docker.com/get-docker/) |
| Python 3 | 3.8+ | `apt install python3` |
| `cryptography` (Python) | any | `pip3 install cryptography` |
| `jq` | any | `apt install jq` |
| `curl` | any | `apt install curl` |
| `openssl` | 1.1+ | `apt install openssl` |
| JDK | 17+ | `apt install openjdk-17-jdk` (build only) |

{: .tip }
> VMs that don't build from source can load pre-built Docker images instead (see [Distributing Docker Images](#distributing-docker-images) below).

---

## Step-by-Step Deployment

### 1. Provision a VM

Create a VM with a public IP. Recommended minimum: **2 vCPUs, 4 GB RAM**.

**Example (Azure CLI):**
```bash
az vm create \
  --resource-group myResourceGroup \
  --name edc-connector \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys
```

### 2. Open Firewall Ports

**Ports that must be reachable from other connectors:**

| Port | Service | Why |
|---|---|---|
| 7091 | IdentityHub Credentials API | VP/VC presentation requests |
| 7093 | IdentityHub DID endpoint | Participant DID document resolution |
| 9876 | DID Server (nginx) | Issuer DID document resolution |
| 19194 | Control Plane DSP | Catalog, negotiation, transfer callbacks |
| 38185 | Data Plane public | Data fetch via EDR token (pull transfers) |

**Optional — restrict to your IP only:**

| Port | Service | Why |
|---|---|---|
| 3000 | Dashboard | Web UI |
| 19193 | Management API | REST API |
| 4000 | http-receiver | Push transfer test destination |

**Example (Azure CLI):**
```bash
az vm open-port --resource-group myResourceGroup --name edc-connector \
  --port 7091,7093,9876,19194,38185,3000 --priority 1000
```

### 3. Install Dependencies

```bash
ssh azureuser@<PUBLIC_IP>

sudo apt update && sudo apt install -y \
  openjdk-17-jdk python3 python3-pip jq curl git docker.io docker-compose-plugin

sudo pip3 install cryptography
sudo usermod -aG docker $USER
newgrp docker
```

### 4. Clone and Generate Keys

```bash
git clone https://github.com/Pilots-community/pilots-dataspace.git
cd pilots-dataspace
./generate-keys.sh
```

This creates:
- `config/certs/private-key.pem` and `public-key.pem` (data plane token keys)
- `deployment/assets/issuer_private.pem` and `issuer_public.pem` (VC issuer keys)

### 5. Build Docker Images

```bash
./gradlew dockerize
```

Produces three local images: `controlplane:latest`, `dataplane:latest`, `identityhub:latest`.

### 6. Configure Environment

```bash
cd deployment/connector
cp .env.example .env
```

Edit `.env` — set `MY_PUBLIC_HOST` to the VM's **public IP or DNS name**:

```bash
MY_PUBLIC_HOST=20.50.100.10
```

This value gets injected into:
- **DID identifiers**: `did:web:20.50.100.10%3A7093`
- **DSP callback URLs**: `http://20.50.100.10:19194/protocol`
- **Data plane public URL**: `http://20.50.100.10:38185/public`
- **Issuer DID document**: `did:web:20.50.100.10%3A9876`

{: .tip }
> You can use a DNS name instead of an IP (e.g. `my-connector.westeurope.cloudapp.azure.com`).

### 7. Start the Stack

```bash
docker compose up -d
```

Wait for all containers to be healthy (30-60 seconds):

```bash
docker ps    # all 8 containers should show (healthy)
```

### 8. Seed Identity Data

```bash
MY_PUBLIC_HOST=20.50.100.10 ./seed.sh
```

The seed script:
1. Generates a MembershipCredential VC (signed with this VM's issuer key)
2. Creates a participant context in IdentityHub
3. Publishes the participant's DID document
4. Stores the STS client secret in the Control Plane's vault
5. Stores the MembershipCredential in IdentityHub
6. Updates the issuer DID document on the DID server (nginx)
7. Registers this VM's issuer as a trusted issuer in the Control Plane

### 9. Open the Dashboard

The dashboard is available at **http://\<PUBLIC_IP\>:3000**.

Key pages:
- **Assets** — create and view data assets
- **Policies** — define access and contract policies
- **Contract Definitions** — link policies to assets
- **Catalog** — browse remote connectors' catalogs (trusted issuers appear as quick-select buttons)
- **Negotiations** — track contract negotiation state
- **Transfers** — initiate and monitor data transfers, fetch data via EDR
- **Trusted Issuers** — add/edit/remove trusted issuer DIDs with connector details

### 10. Share Your Connector Details

After setup, share these three values with partner organizations:

| What | Value | Example |
|---|---|---|
| **Issuer DID** | `did:web:<MY_PUBLIC_HOST>%3A9876` | `did:web:20.50.100.10%3A9876` |
| **DSP Endpoint** | `http://<MY_PUBLIC_HOST>:19194/protocol` | `http://20.50.100.10:19194/protocol` |
| **Participant DID** | `did:web:<MY_PUBLIC_HOST>%3A7093` | `did:web:20.50.100.10%3A7093` |

Partners will register these on their connector via the Trusted Issuers page.

---

## Connecting Two Connectors

See [Connecting Connectors]({% link connecting-connectors.md %}) for the full guide on establishing trust and exchanging data between two deployed connectors.

---

## Distributing Docker Images

If a VM doesn't have the source code or JDK to build images, transfer pre-built images:

```bash
# On the build machine
docker save controlplane:latest dataplane:latest identityhub:latest | gzip > edc-images.tar.gz

# Copy to the VM
scp edc-images.tar.gz azureuser@20.50.100.10:~/

# On the VM
docker load < edc-images.tar.gz
```

The tarball is typically ~300-400 MB. VMs that load images this way still need:
- The repository files (for config, `seed.sh`, `docker-compose.yml`, `generate-keys.sh`)
- Python 3 + `cryptography` + `jq` + `curl` (for `seed.sh`)
- OpenSSL (for `generate-keys.sh`)

---

## Resetting a VM

To wipe all state and start fresh:

```bash
cd deployment/connector
docker compose down -v    # stops containers AND deletes volumes
```

Then re-run from [step 7](#7-start-the-stack). Keys don't need to be regenerated unless you want new ones.

---

## Networking Notes

### Bidirectional Connectivity Required

The DSP protocol requires both connectors to reach each other. Even if you only want to consume data, the provider must reach your connector to verify your identity (DID resolution, VP requests) and send negotiation callbacks.

### Machines Behind NAT

If your machine doesn't have a public IP (e.g., a laptop behind a home router):

- **[Tailscale](https://tailscale.com/)** — creates a private WireGuard mesh. Install on both machines, run `tailscale up`, and use the Tailscale IPs as `MY_PUBLIC_HOST`. Free, no port forwarding needed.
- **Router port forwarding** — forward the required ports to your machine's LAN IP and use your public IP as `MY_PUBLIC_HOST`.
- **Deploy to a cloud VM** — even a small VM (~$4/month) avoids all NAT issues.
