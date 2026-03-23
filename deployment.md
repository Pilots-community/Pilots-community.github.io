---
layout: default
title: Deployment
nav_order: 3
has_children: true
---

# Deployment

The Pilots Dataspace connector supports two deployment modes:

| Mode | Use case | Guide |
|---|---|---|
| **[Local Development]({% link deployment-local.md %})** | Two participants on one machine for development and testing | Uses root `docker-compose.yml` |
| **[Cloud VM (Standalone)]({% link deployment-cloud.md %})** | Production — one connector per VM, each with independent keys | Uses `deployment/connector/` |

The **cloud VM deployment** is the target for real-world use: each organization runs one connector on their own infrastructure with their own keys, identity, and trust configuration.

The **local development** setup runs two complete participants on a single machine, simulating two independent organizations for testing.
