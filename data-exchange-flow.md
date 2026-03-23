---
layout: default
title: Data Exchange
parent: Architecture
nav_order: 2
---

# Data Exchange — How It Works

Data exchange has three phases: **catalog discovery**, **contract negotiation**, and **data transfer**. Each phase uses DSP (Dataspace Protocol) over HTTP and is authenticated via DCP (see [Identity & Trust]({% link identity-flow.md %})).

---

## Component Roles

```
┌──────────────────────────────────────────────────────────────┐
│                  Control Plane (19193 / 19194)               │
│                                                              │
│  Management API (19193)          DSP Protocol (19194)        │
│  ┌────────────────────────┐      ┌─────────────────────────┐ │
│  │ Your commands:         │      │ Machine-to-machine:     │ │
│  │  - create asset        │      │  - catalog exchange     │ │
│  │  - create policy       │      │  - negotiation messages │ │
│  │  - start negotiation   │      │  - transfer coordination│ │
│  │  - start transfer      │      │  - callbacks            │ │
│  │  - get EDR             │      │                         │ │
│  └────────────────────────┘      └─────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                  Data Plane (38181 / 38185)                  │
│                                                              │
│  Control API (38182)             Public API (38185)          │
│  ┌────────────────────────┐      ┌─────────────────────────┐ │
│  │ From Control Plane:    │      │ Consumer fetches here:  │ │
│  │  - "prepare source X"  │      │  - GET /public          │ │
│  │  - "push data to Y"    │      │  - Authorization: Bearer│ │
│  └────────────────────────┘      │    <EDR token>          │ │
│                                  └─────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Catalog Discovery

The consumer asks: "What data do you have, and under what terms?"

```
Consumer                                            Provider
────────                                            ────────
POST /management/v3/catalog/request
body: { counterPartyAddress,
        counterPartyId }
       │
       │           DCP authentication
       │           (SI token + credential exchange)
       │
       │  DSP: POST /protocol/catalog/request
       │──────────────────────────────────────────→│
       │                                           │
       │                                     Build DCAT catalog:
       │                                      - assets → datasets
       │                                      - contracts → policies
       │                                           │
       │              Catalog response             │
       │←──────────────────────────────────────────│
```

The catalog response contains **datasets** (assets) with **policies** (offers). The offer `@id` from `odrl:hasPolicy` is what you need for negotiation.

### What the Provider Set Up Beforehand

The provider creates three things via the Management API:

1. **Asset** — "I have this data" (name, data source URL)
2. **Policy Definition** — "Under these terms" (access rules)
3. **Contract Definition** — "Link assets to terms" (connects assets to policies)

---

## Phase 2: Contract Negotiation

The consumer says: "I want asset X under offer Y." This is an asynchronous state machine:

```
Consumer CP                                          Provider CP
───────────                                          ───────────

POST /management/v3/contractnegotiations
  │
  │  State: REQUESTING
  │
  │  DSP: POST /protocol/negotiations/request
  │──────────────────────────────────────────────→│
  │                                                │
  │                                          Validate offer
  │                                          (asset exists?
  │                                           policy matches?)
  │                                                │
  │                                          State: AGREEING
  │  DSP: agreement                                │
  │←──────────────────────────────────────────────│
  │
  │  State: AGREED
  │
  │  DSP: verification
  │──────────────────────────────────────────────→│
  │                                                │
  │                                          State: FINALIZED
  │  DSP: finalized                                │
  │←──────────────────────────────────────────────│
  │
  │  State: FINALIZED
  │  contractAgreementId = <AGREEMENT_ID>
```

Poll `GET /management/v3/contractnegotiations/<ID>` until `state` = `FINALIZED`, then extract the `contractAgreementId`.

Each DSP callback goes through DCP authentication — both sides verify identity on every message.

---

## Phase 3a: Pull Transfer (HttpData-PULL)

The consumer asks the provider to prepare data for download. The consumer then fetches directly from the provider's Data Plane.

```
Consumer CP              Provider CP              Provider DP
───────────              ───────────              ───────────

POST /management/v3/
  transferprocesses
  (HttpData-PULL)
       │
       │  DSP: transfer request
       │────────────────────────→│
       │                         │
       │                   Validate contract
       │                         │
       │                   Tell DP: prepare
       │                         │──────────→│
       │                         │           │
       │                   Create EDR:       │
       │                    endpoint URL     │
       │                    + auth token     │
       │                         │           │
       │  DSP: start (EDR)       │           │
       │←────────────────────────│           │
       │                         │           │
  State: STARTED                 │           │
       │                         │           │
  GET /edrs/<ID>/dataaddress     │           │
  → { endpoint, authorization }  │           │
       │                         │           │
       │  GET <endpoint>         │           │
       │  Auth: Bearer <token>   │           │
       │──────────────────────────────────→│
       │                         │           │
       │            actual data  │      Verify token,
       │←──────────────────────────────── fetch from source
```

### The EDR (Endpoint Data Reference)

The EDR contains:
- **endpoint** — the Data Plane's public API URL (port 38185)
- **authorization** — a JWT signed with the Data Plane's private key, proving the consumer has a valid contract for this asset

---

## Phase 3b: Push Transfer (HttpData-PUSH)

The consumer tells the provider: "Send the data to this URL." The provider's Data Plane fetches the data and delivers it.

```
Consumer CP              Provider CP              Provider DP
───────────              ───────────              ───────────

POST /management/v3/
  transferprocesses
  (HttpData-PUSH,
   dataDestination:
     consumer:4000)
       │
       │  DSP: transfer request
       │────────────────────────→│
       │                         │
       │                   Tell DP: push
       │                         │──────────→│
       │                         │           │
       │                         │      Fetch data
       │                         │      from source
       │                         │           │
       │                         │      POST to
  http-receiver ←───────────────────── consumer
  (port 4000)                    │           │
       │                         │           │
       │  DSP: completion        │           │
       │←────────────────────────│           │
       │
  State: COMPLETED
```

---

## Pull vs Push Comparison

| | Pull (HttpData-PULL) | Push (HttpData-PUSH) |
|---|---|---|
| **Who fetches data** | Consumer, via EDR token | Provider's Data Plane |
| **Who delivers data** | Provider's Data Plane serves it | Provider's Data Plane POSTs it |
| **Consumer needs** | EDR endpoint + token | A reachable HTTP endpoint |
| **Final state** | `STARTED` (consumer can fetch repeatedly) | `COMPLETED` (one-time delivery) |
| **Token involved** | EDR JWT (DP key pair) | None for delivery |

---

## Full End-to-End Sequence

```
Provider                                                Consumer
────────                                                ────────

 1. Create asset          "I have this data"
 2. Create policy         "Under these terms"
 3. Create contract def   "Link assets to terms"

                          ← 4. Request catalog ──────────
 Return DCAT catalog ───────────────────────────────────→

                          ← 5. Negotiate contract ───────
 Validate, agree ───────────────────────────────────────→
 ← Verify ──────────────────────────────────────────────
 Finalize ──────────────────────────────────────────────→
                               contractAgreementId ✓

                          ← 6. Start transfer ───────────

 ┌─── PULL ─────────┐     ┌─── PUSH ──────────────┐
 │ Prepare DP,      │     │ DP fetches data,       │
 │ send EDR         │     │ POSTs to consumer      │
 │ Consumer fetches │     │ Transfer completes     │
 │ from DP public   │     │ automatically          │
 └──────────────────┘     └────────────────────────┘

                               7. Data received ✓
```

## Ports Used in Data Exchange

| Port | Service | Role |
|---|---|---|
| 19193 | CP Management API | Your commands: create assets, start negotiations, get EDRs |
| 19194 | CP DSP Protocol | Machine-to-machine: catalog, negotiation, transfer messages |
| 19192 | CP Control API | Internal: CP tells DP to register |
| 38182 | DP Control API | Internal: CP tells DP to prepare/push data |
| 38185 | DP Public API | Consumer fetches data here (pull transfers) |
| 4000 | http-receiver | Test endpoint for push transfers |
