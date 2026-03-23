---
layout: default
title: Home
nav_order: 1
---

# Pilots Dataspace

An open-source dataspace connector that lets organizations **share data with each other while keeping control over who can access what**. Built on [Eclipse Dataspace Components (EDC)](https://github.com/eclipse-edc/Connector), it implements the [Dataspace Protocol (DSP)](https://docs.internationaldataspaces.org/ids-knowledgebase/dataspace-protocol) with decentralized identity using DIDs and Verifiable Credentials.

---

## What Is a Dataspace?

A dataspace is a way for multiple organizations to share data without a central broker or data lake. Each organization runs its own **connector** on their own infrastructure. Connectors communicate peer-to-peer using standard protocols:

- **No central data store** — data stays with its owner until explicitly shared
- **Contracts before data** — every transfer requires a negotiated agreement
- **Decentralized identity** — each connector proves who it is using cryptographic credentials, not shared passwords

```
 Organization A                Organization B                Organization C
 ┌────────────┐               ┌────────────┐               ┌────────────┐
 │ Connector  │◄─────────────►│ Connector  │◄─────────────►│ Connector  │
 │ (own data) │  DSP protocol │ (own data) │  DSP protocol │ (own data) │
 └────────────┘               └────────────┘               └────────────┘
      Each organization runs their own connector on their own VM
```

## What Does This Connector Do?

Each connector is a self-contained stack of services that handles everything needed to participate in the dataspace:

| Capability | How it works |
|---|---|
| **Publish data** | Register datasets as assets with access policies |
| **Discover data** | Browse other connectors' catalogs to find available datasets |
| **Negotiate access** | Automated contract negotiation with policy enforcement |
| **Transfer data** | Pull (consumer fetches via token) or push (provider delivers to endpoint) |
| **Prove identity** | DCP with `did:web` DIDs and signed Verifiable Credentials |
| **Manage trust** | Dynamic trusted issuer registry — add or remove partners without restarts |
| **Web dashboard** | UI for managing assets, policies, catalogs, transfers, and trusted issuers |

## Architecture at a Glance

Each organization deploys this stack on a cloud VM (or any server with a reachable IP):

```
┌─────────────────────────────────────────────┐
│  Control Plane         (ports 19193, 19194) │  Catalog, negotiation, transfer management
│  Data Plane            (ports 38181, 38185) │  Actual data transfer (pull & push)
│  Identity Hub          (ports 7090–7096)    │  DID management, VC wallet, STS
│  DID Server (nginx)    (port 9876)          │  Serves issuer DID document
│  Dashboard             (port 3000)          │  Web UI
│  Vault                 (port 8200)          │  Secret storage
│  PostgreSQL            (port 15432)         │  Persistent state
└─────────────────────────────────────────────┘
```

## How Organizations Connect

1. Each organization **deploys their own connector** on a cloud VM
2. They **generate cryptographic keys** and a self-signed Membership Credential
3. They **share three values** with partners: issuer DID, DSP endpoint, and participant DID
4. Partners **register each other as trusted issuers** via the dashboard
5. They can now **discover, negotiate, and transfer data** with full identity verification

No central authority is required. Trust is managed directly between participants.

## Get Started

- [**Getting Started**]({% link getting-started.md %}) — prerequisites and one-command setup
- [**Deploy to a Cloud VM**]({% link deployment-cloud.md %}) — production deployment guide
- [**Step-by-Step Usage Guide**]({% link step-by-step-guide.md %}) — create assets, negotiate contracts, transfer data
- [**Architecture**]({% link architecture.md %}) — how the components work together
