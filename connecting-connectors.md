---
layout: default
title: Connecting Connectors
nav_order: 6
---

# Connecting Connectors

This guide explains how to establish trust between two separately deployed connectors so they can discover and exchange data.

{: .note }
> Both connectors must be fully deployed and seeded before connecting them. See [Cloud VM Deployment]({% link deployment-cloud.md %}) for setup instructions.

---

## 1. Exchange Connector Details

Each organization shares three values with their partners:

| What | How to find it | Example |
|---|---|---|
| **Issuer DID** | `did:web:<MY_PUBLIC_HOST>%3A9876` | `did:web:20.50.100.10%3A9876` |
| **DSP Endpoint** | `http://<MY_PUBLIC_HOST>:19194/protocol` | `http://20.50.100.10:19194/protocol` |
| **Participant DID** | `did:web:<MY_PUBLIC_HOST>%3A7093` | `did:web:20.50.100.10%3A7093` |

Replace `<MY_PUBLIC_HOST>` with the value from your `.env` file (the public IP or DNS name).

---

## 2. Register Each Other as Trusted Issuers

Trust is **mutual** — both sides must register the other. This tells your connector: "I trust Verifiable Credentials signed by this issuer."

### Via Dashboard

On **your** dashboard (http://\<your-ip\>:3000):

1. Go to **Trusted Issuers** page
2. Click **Add Issuer**
3. Fill in the partner's details:

| Field | Value |
|---|---|
| DID | `did:web:20.50.100.20%3A9876` |
| Name | Organization B |
| Organization | Org B |
| DSP Endpoint | `http://20.50.100.20:19194/protocol` |
| Participant DID | `did:web:20.50.100.20%3A7093` |

4. Click **Save**

The partner does the same on their dashboard with your details.

### Via API

```bash
curl -X POST http://localhost:19193/management/v1/trusted-issuers \
  -H "Content-Type: application/json" \
  -H "x-api-key: password" \
  -d '{
    "did": "did:web:20.50.100.20%3A9876",
    "name": "Organization B",
    "organization": "Org B",
    "dspEndpoint": "http://20.50.100.20:19194/protocol",
    "participantDid": "did:web:20.50.100.20%3A7093"
  }'
```

{: .tip }
> No restart needed. Trust changes take effect immediately. You can add or remove trusted issuers at any time via the dashboard or API.

---

## 3. Verify Connectivity

Before trying to exchange data, verify that both connectors can reach each other's endpoints:

```bash
# Issuer DID document (should return JSON with a verificationMethod)
curl http://<OTHER_HOST>:9876/.well-known/did.json

# Participant DID document (should return JSON)
curl http://<OTHER_HOST>:7093/.well-known/did.json

# DSP endpoint (should return a JSON error, NOT connection refused)
curl http://<OTHER_HOST>:19194/protocol
```

If any of these fail with "connection refused" or timeout, check your firewall rules (see [Cloud VM Deployment — Firewall Ports]({% link deployment-cloud.md %}#2-open-firewall-ports)).

---

## 4. Browse the Catalog

### Via Dashboard

Go to the **Catalog** page. The remote connector you registered appears as a **clickable button** — click it and hit **Fetch Catalog**.

### Via API

```bash
curl -s -X POST http://localhost:19193/management/v3/catalog/request \
  -H "Content-Type: application/json" \
  -H "x-api-key: password" \
  -d '{
    "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
    "counterPartyAddress": "http://20.50.100.20:19194/protocol",
    "counterPartyId": "did:web:20.50.100.20%3A7093",
    "protocol": "dataspace-protocol-http"
  }' | jq .
```

If the catalog returns datasets, the connection is working. You can now proceed to negotiate and transfer data — see the [Step-by-Step Usage Guide]({% link step-by-step-guide.md %}).

---

## Adding More Connectors

To add a new organization to the dataspace:

1. The new organization deploys and seeds their connector ([Cloud VM Deployment]({% link deployment-cloud.md %}))
2. On **each existing** connector's dashboard, go to **Trusted Issuers** and add the new connector's details
3. On the **new** connector's dashboard, add each existing connector's details

This is an O(n) process — each new participant exchanges details with every existing participant.

---

## Removing Trust

To stop trusting a connector:

1. Go to **Trusted Issuers** on your dashboard
2. Click **Delete** on the issuer you want to remove

The connector can no longer authenticate to yours. Existing contract agreements remain in the database but no new negotiations will succeed.
