---
layout: default
title: Step-by-Step Usage Guide
nav_order: 4
---

# Step-by-Step Usage Guide

This guide walks through the complete flow: publishing data, discovering it, negotiating a contract, and transferring data. You can follow along using either the **dashboard UI** or **curl commands**.

{: .note }
> This guide assumes you have a running connector. See [Getting Started]({% link getting-started.md %}) for local setup or [Cloud VM Deployment]({% link deployment-cloud.md %}) for production.

---

## Setup: Environment Variables

Set these once before running the curl commands. The values differ depending on your deployment mode.

**Local development (Docker Compose):**

```bash
# Provider (participant-1)
PROVIDER_MGMT="http://localhost:19193"
P1_DSP="http://participant-1-controlplane:19194/protocol"
P1_DID="did:web:participant-1-identityhub%3A7093"

# Consumer (participant-2)
CONSUMER_MGMT="http://localhost:29193"
P2_DSP="http://participant-2-controlplane:29194/protocol"
P2_DID="did:web:participant-2-identityhub%3A7083"
```

**Cloud VM deployment:**

```bash
# Provider (VM A)
PROVIDER_MGMT="http://localhost:19193"    # run from VM A
P1_DSP="http://20.50.100.10:19194/protocol"
P1_DID="did:web:20.50.100.10%3A7093"

# Consumer (VM B)
CONSUMER_MGMT="http://localhost:19193"    # run from VM B
P2_DSP="http://20.50.100.20:19194/protocol"
P2_DID="did:web:20.50.100.20%3A7093"
```

---

## Provider Side: Publish Data

### Step 1: Create an Asset

Register a data source. This example exposes a public JSON API:

```bash
curl -X POST $PROVIDER_MGMT/management/v3/assets \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: password" \
  -d '{
    "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
    "@id": "sample-asset-1",
    "properties": {
      "name": "Sample Data Asset",
      "description": "A sample dataset for testing the dataspace",
      "contenttype": "application/json"
    },
    "dataAddress": {
      "type": "HttpData",
      "baseUrl": "https://jsonplaceholder.typicode.com/todos/1"
    }
  }'
```

The `dataAddress` tells the Data Plane where to fetch the actual data. It supports `HttpData` (any HTTP URL) and other types.

**Via dashboard:** Go to **Assets** page and click **Create Asset**.

### Step 2: Create a Policy Definition

Define access rules. This example is an open (permit-all) policy:

```bash
curl -X POST $PROVIDER_MGMT/management/v3/policydefinitions \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: password" \
  -d '{
    "@context": {
      "@vocab": "https://w3id.org/edc/v0.0.1/ns/",
      "odrl": "http://www.w3.org/ns/odrl/2/"
    },
    "@id": "open-policy",
    "policy": {
      "@context": "http://www.w3.org/ns/odrl.jsonld",
      "@type": "Set",
      "permission": [],
      "prohibition": [],
      "obligation": []
    }
  }'
```

In production you would add constraints (e.g., specific consumer IDs, time restrictions, purpose limitations).

**Via dashboard:** Go to **Policies** page and click **Create Policy**.

### Step 3: Create a Contract Definition

Link the asset to the policy. This makes the asset visible in your catalog:

```bash
curl -X POST $PROVIDER_MGMT/management/v3/contractdefinitions \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: password" \
  -d '{
    "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
    "@id": "sample-contract-def",
    "accessPolicyId": "open-policy",
    "contractPolicyId": "open-policy",
    "assetsSelector": []
  }'
```

An empty `assetsSelector` means **all assets** are included. You can add selectors to target specific assets.

**Via dashboard:** Go to **Contract Definitions** page and click **Create Contract Definition**.

---

## Consumer Side: Discover and Negotiate

### Step 4: Request the Provider's Catalog

Ask the provider what data they have. `counterPartyId` must be the provider's DID — this is required for DCP authentication.

```bash
curl -s -X POST $CONSUMER_MGMT/management/v3/catalog/request \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: password" \
  -d "{
    \"@context\": { \"@vocab\": \"https://w3id.org/edc/v0.0.1/ns/\" },
    \"counterPartyAddress\": \"${P1_DSP}\",
    \"counterPartyId\": \"${P1_DID}\",
    \"protocol\": \"dataspace-protocol-http\"
  }" | jq .
```

In the response, find `dcat:dataset` → `odrl:hasPolicy` → `@id`. This is the **offer ID** you need for negotiation.

Extract it programmatically:

```bash
OFFER_ID=$(curl -s -X POST $CONSUMER_MGMT/management/v3/catalog/request \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: password" \
  -d "{
    \"@context\": { \"@vocab\": \"https://w3id.org/edc/v0.0.1/ns/\" },
    \"counterPartyAddress\": \"${P1_DSP}\",
    \"counterPartyId\": \"${P1_DID}\",
    \"protocol\": \"dataspace-protocol-http\"
  }" | jq -r '.["dcat:dataset"]["odrl:hasPolicy"]["@id"]')

echo "$OFFER_ID"
```

The offer ID looks like a Base64-encoded string, e.g. `c2FtcGxlLWNvbnRyYWN0LWRlZg==:c2FtcGxlLWFzc2V0LTE=:MjdlNDFh...`

**Via dashboard:** Go to **Catalog** page, select the remote connector (or enter the DSP endpoint + DID), and click **Fetch Catalog**.

### Step 5: Negotiate a Contract

Start a contract negotiation using the offer ID. The `assigner` must be the provider's DID:

```bash
NEGOTIATION_ID=$(curl -s -X POST $CONSUMER_MGMT/management/v3/contractnegotiations \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: password" \
  -d "{
    \"@context\": { \"@vocab\": \"https://w3id.org/edc/v0.0.1/ns/\" },
    \"counterPartyAddress\": \"${P1_DSP}\",
    \"counterPartyId\": \"${P1_DID}\",
    \"protocol\": \"dataspace-protocol-http\",
    \"policy\": {
      \"@context\": \"http://www.w3.org/ns/odrl.jsonld\",
      \"@id\": \"${OFFER_ID}\",
      \"@type\": \"Offer\",
      \"assigner\": \"${P1_DID}\",
      \"target\": \"sample-asset-1\",
      \"permission\": [],
      \"prohibition\": [],
      \"obligation\": []
    }
  }" | jq -r '.["@id"]')

echo "$NEGOTIATION_ID"
```

Poll until the state becomes `FINALIZED`:

```bash
curl -s $CONSUMER_MGMT/management/v3/contractnegotiations/$NEGOTIATION_ID \
  -H "X-Api-Key: password" | jq '{state, contractAgreementId}'
```

Once finalized, capture the `contractAgreementId`:

```bash
AGREEMENT_ID=$(curl -s $CONSUMER_MGMT/management/v3/contractnegotiations/$NEGOTIATION_ID \
  -H "X-Api-Key: password" | jq -r '.contractAgreementId')

echo "$AGREEMENT_ID"
```

**Via dashboard:** Click **Negotiate** on a catalog entry. The Negotiations page shows the state progression.

---

## Transfer Data

### Step 6: Pull Transfer (HttpData-PULL)

Request a data transfer using the contract agreement. Pull means the consumer fetches the data:

```bash
TRANSFER_ID=$(curl -s -X POST $CONSUMER_MGMT/management/v3/transferprocesses \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: password" \
  -d "{
    \"@context\": { \"@vocab\": \"https://w3id.org/edc/v0.0.1/ns/\" },
    \"counterPartyAddress\": \"${P1_DSP}\",
    \"counterPartyId\": \"${P1_DID}\",
    \"protocol\": \"dataspace-protocol-http\",
    \"contractId\": \"${AGREEMENT_ID}\",
    \"assetId\": \"sample-asset-1\",
    \"transferType\": \"HttpData-PULL\"
  }" | jq -r '.["@id"]')

echo "$TRANSFER_ID"
```

Poll until state becomes `STARTED`:

```bash
curl -s $CONSUMER_MGMT/management/v3/transferprocesses/$TRANSFER_ID \
  -H "X-Api-Key: password" | jq '{state, type}'
```

### Step 7: Fetch Data via EDR

Once the transfer is `STARTED`, retrieve the Endpoint Data Reference (EDR). This contains a short-lived access token for the data plane:

```bash
curl -s $CONSUMER_MGMT/management/v3/edrs/$TRANSFER_ID/dataaddress \
  -H "X-Api-Key: password" | jq .
```

Extract the token and fetch the data:

```bash
TOKEN=$(curl -s $CONSUMER_MGMT/management/v3/edrs/$TRANSFER_ID/dataaddress \
  -H "X-Api-Key: password" | jq -r '.authorization')

curl -s http://localhost:38185/public \
  -H "Authorization: Bearer $TOKEN" | jq .
```

{: .note }
> When running with Docker Compose, the EDR `endpoint` shows the Docker service name (e.g., `http://participant-1-dataplane:38185/public`). From your host terminal, use `localhost` with the mapped port instead.

A successful response returns the actual data:

```json
{"userId": 1, "id": 1, "title": "delectus aut autem", "completed": false}
```

This confirms the full pipeline is working: DCP authentication, credential verification, contract enforcement, EDR token validation, and data proxying.

**Via dashboard:** Go to **Transfers** page, click **Pull Transfer** on a contract agreement, then click **Fetch Data** when the transfer starts.

---

### Step 8: Push Transfer (HttpData-PUSH)

Instead of the consumer pulling data, the provider can **push** data directly to an HTTP endpoint. Reuse the same contract agreement:

```bash
PUSH_TRANSFER_ID=$(curl -s -X POST $CONSUMER_MGMT/management/v3/transferprocesses \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: password" \
  -d "{
    \"@context\": { \"@vocab\": \"https://w3id.org/edc/v0.0.1/ns/\" },
    \"counterPartyAddress\": \"${P1_DSP}\",
    \"counterPartyId\": \"${P1_DID}\",
    \"protocol\": \"dataspace-protocol-http\",
    \"contractId\": \"${AGREEMENT_ID}\",
    \"assetId\": \"sample-asset-1\",
    \"transferType\": \"HttpData-PUSH\",
    \"dataDestination\": {
      \"type\": \"HttpData\",
      \"baseUrl\": \"http://http-receiver:4000/\"
    }
  }" | jq -r '.["@id"]')

echo "$PUSH_TRANSFER_ID"
```

Push transfers reach `COMPLETED` once the data has been delivered (unlike pull which stays `STARTED`):

```bash
curl -s $CONSUMER_MGMT/management/v3/transferprocesses/$PUSH_TRANSFER_ID \
  -H "X-Api-Key: password" | jq '{state, type}'
```

Verify the data arrived:

```bash
curl -s http://localhost:4000/ | jq .
```

---

## Pull vs Push Summary

| | Pull (HttpData-PULL) | Push (HttpData-PUSH) |
|---|---|---|
| **Who fetches data** | Consumer, via EDR token | Provider's Data Plane |
| **Consumer needs** | Nothing (calls out) | A reachable HTTP endpoint |
| **Final state** | `STARTED` (can fetch repeatedly) | `COMPLETED` (one-time delivery) |
| **Best for** | On-demand access | Event-driven delivery |

---

## Reverse Direction

The flow is fully bidirectional. Any connector can be both provider and consumer. To test the reverse direction (participant-2 owns data, participant-1 requests it):

1. Create an asset, policy, and contract definition on **participant-2's** management API
2. From **participant-1**, request participant-2's catalog, negotiate, and transfer

The E2E test script (`./test-e2e.sh`) exercises both directions automatically.
