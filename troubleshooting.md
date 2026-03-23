---
layout: default
title: Troubleshooting
nav_order: 7
---

# Troubleshooting

Common issues and how to fix them.

---

## 401 Unauthorized on Catalog/Negotiation

**Symptoms:** Catalog request or negotiation returns 401 or an empty/error response.

**Causes and fixes:**

1. **Trusted issuer not registered** — The remote connector's issuer DID must be in your trusted issuer registry.
   - Check the **Trusted Issuers** page in your dashboard
   - Make sure the issuer DID, DSP endpoint, and participant DID are all correct

2. **Issuer DID document not accessible** — Your connector needs to resolve the remote issuer's DID:
   ```bash
   curl http://<remote-host>:9876/.well-known/did.json
   ```
   This should return a JSON document with a `verificationMethod`. If it fails, check the remote VM's firewall.

3. **Wrong `counterPartyId`** — The `counterPartyId` in your catalog/negotiation request must be the provider's **participant DID** (not the issuer DID). It's used as the JWT audience.

4. **`MY_PUBLIC_HOST` misconfigured** — If this value doesn't match the address other connectors use to reach you, DID resolution will fail. Check your `.env` file.

---

## DID Resolution Failures

**Symptoms:** Logs show errors about DID resolution or "could not resolve DID".

**Fixes:**

1. Verify the IdentityHub DID endpoint is accessible:
   ```bash
   curl http://<host>:7093/.well-known/did.json
   ```

2. Check that `edc.iam.did.web.use.https=false` is set in the config files. EDC defaults to HTTPS for `did:web`, but this project uses HTTP.

3. If running across different networks, ensure port 7093 is open in the firewall.

---

## Connection Refused

**Symptoms:** curl returns "Connection refused" when hitting a remote connector.

**Fixes:**

1. Check the required ports are open in your cloud provider's NSG/firewall:
   - 7091, 7093, 9876, 19194, 38185 (for other connectors)
   - 3000, 19193 (for your own access)

2. Check local firewall rules on the VM:
   ```bash
   sudo iptables -L -n
   sudo ufw status
   ```

3. Test basic connectivity:
   ```bash
   nc -zv <host> 19194
   ```

4. Verify the container is running and healthy:
   ```bash
   docker ps
   ```

---

## Seed Script Fails

**Symptoms:** `./deployment/seed.sh` (or `./seed.sh` in the connector directory) exits with errors.

**Fixes:**

1. **Containers not healthy** — Wait for all containers to be healthy before seeding:
   ```bash
   docker ps    # all should show (healthy)
   curl http://localhost:7090/api/check/health
   ```

2. **Missing issuer key** — Run `./generate-keys.sh` from the project root:
   ```bash
   ls deployment/assets/issuer_private.pem    # should exist
   ```

3. **Missing Python cryptography library**:
   ```bash
   python3 -c "from cryptography.hazmat.primitives import serialization"
   ```
   If this fails: `pip3 install cryptography`

4. **Missing jq**:
   ```bash
   jq --version    # should print version
   sudo apt install jq
   ```

---

## Transfer Stays in REQUESTING / Never Reaches STARTED

**Symptoms:** A pull transfer never transitions from `REQUESTING` to `STARTED`.

**Causes:**

1. **Bidirectional connectivity issue** — The provider needs to reach the consumer for DSP callbacks. Even as a consumer, your connector must be reachable. Check firewall rules.

2. **Data Plane not healthy** — Check the Data Plane container:
   ```bash
   curl http://localhost:38181/api/check/health
   docker logs <dataplane-container>
   ```

3. **Contract agreement invalid** — Verify the agreement ID is correct and the negotiation reached `FINALIZED`.

---

## EDR Token Returns 401

**Symptoms:** Fetching data with the EDR token returns 401.

**Causes:**

1. **Token expired** — EDR tokens are short-lived. Request a fresh one:
   ```bash
   curl -s http://localhost:19193/management/v3/edrs/$TRANSFER_ID/dataaddress \
     -H "X-Api-Key: password" | jq .
   ```

2. **Wrong Data Plane port** — When using Docker Compose, the EDR `endpoint` shows the Docker service name. Use `localhost` with the mapped port from your host terminal:
   - Participant-1 Data Plane: `http://localhost:38185/public`
   - Participant-2 Data Plane: `http://localhost:48185/public`

---

## "MY_PUBLIC_HOST variable is not set" on Docker Compose Down

This warning is harmless. Docker Compose reads `.env` on `down` to resolve variable references, but the values don't matter for stopping containers.

---

## Health Check Endpoints

Use these to verify individual services are running:

```bash
# Control Plane
curl http://localhost:18181/api/check/health

# Data Plane
curl http://localhost:38181/api/check/health

# IdentityHub
curl http://localhost:7090/api/check/health
```

All should return a JSON response with `isSystemHealthy: true`.
