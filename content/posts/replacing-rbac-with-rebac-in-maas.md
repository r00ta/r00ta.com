+++
date = '2026-04-02T22:00:00+01:00'
draft = false
title = 'Replacing RBAC with ReBAC in MAAS: Embedding OpenFGA for Fine-Grained Access Control'
toc = true
+++

## Replacing RBAC with ReBAC in MAAS: Embedding OpenFGA for Fine-Grained Access Control

Hello everyone,

In this post I want to share the work I did to bring relationship-based access control (ReBAC) to [MAAS](https://maas.io/) by embedding [OpenFGA](https://openfga.dev/) directly into the region controller. This replaces the previous permission layer based on roles (RBAC) with a much more flexible, fine-grained model powered by OpenFGA — the open-source implementation of Google's [Zanzibar](https://research.google/pubs/zanzibar-googles-consistent-global-authorization-system/) authorization system.

MAAS 3.8 ships this as the built-in authorization system. The legacy Canonical RBAC integration is deprecated in 3.8 and fully removed in MAAS 4.0.

## Why Move from RBAC to ReBAC?

The old RBAC model in MAAS was built around static roles: **admin** and **user**. While simple, it was limiting:

- **No per-resource-pool permissions.** You were either an admin with full control or a regular user with limited global access.
- **External dependency on Canonical RBAC.** The RBAC service was a separate product that had to be deployed and maintained alongside MAAS.
- **Coarse-grained controls.** There was no way to say "this team can deploy machines only in pool X" without making them admins.

ReBAC solves all of these problems by modeling permissions as *relationships* between users, groups, and resources. The permission model becomes:

```
user → group → entitlement → resource
```

A user has a permission only if they belong to a group that holds the corresponding entitlement for that resource.

## Architecture: Embedding OpenFGA into MAAS

One of the key design decisions was to **embed OpenFGA directly into the MAAS region controller** rather than running it as a separate external service. Here is how it works:

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     MAAS Region Controller                       │
│                                                                  │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │   regiond    │  │ maasapiserver│  │  maastemporalworker    │  │
│  │  (Django)    │  │  (FastAPI)   │  │  (Temporal workflows)  │  │
│  │              │  │   V2/V3 API  │  │                        │  │
│  └──────┬───────┘  └──────┬───────┘  └────────────────────────┘  │
│         │                 │                                      │
│         │  permission     │  permission                          │
│         │  checks         │  checks                              │
│         │                 │                                      │
│         ▼                 ▼                                      │
│  ┌─────────────────────────────────┐                             │
│  │     OpenFGA Sync/Async Client   │                             │
│  │  (HTTP over Unix domain socket) │                             │
│  └──────────────┬──────────────────┘                             │
│                 │                                                │
│                 ▼                                                │
│  ┌─────────────────────────────────┐                             │
│  │        maas-openfga             │                             │
│  │  (Embedded OpenFGA server)      │                             │
│  │  Listening on Unix socket       │                             │
│  └──────────────┬──────────────────┘                             │
│                 │                                                │
└─────────────────┼────────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                      PostgreSQL Database                         │
│                                                                  │
│  ┌─────────────────────┐    ┌─────────────────────────────────┐ │
│  │   maasdb schema     │    │      openfga schema             │ │
│  │   (MAAS tables)     │    │   (OpenFGA stores, tuples,      │ │
│  │                     │    │    authorization models)         │ │
│  └─────────────────────┘    └─────────────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Process Topology on a Region Controller

Every MAAS region controller runs the following processes, managed by [Pebble](https://canonical-pebble.readthedocs-hosted.com/en/latest/):

```
┌─────────────────────────────────────────────────────────┐
│              MAAS Region Controller Node                 │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │   regiond    │  │ maasapiserver│  │ temporal-      │  │
│  │  (Django     │  │ (FastAPI,    │  │ worker         │  │
│  │   websocket  │  │  V2+V3 HTTP  │  │               │  │
│  │   + HTTP)    │  │  API server) │  │               │  │
│  └──────────────┘  └──────────────┘  └───────────────┘  │
│                                                         │
│  ┌─────────────────────────────────────────────────────┐│
│  │                 maas-openfga                        ││
│  │  (Go binary, embedded OpenFGA, Unix socket)        ││
│  └─────────────────────────────────────────────────────┘│
│                                                         │
└─────────────────────────────────────────────────────────┘
```

Each region controller in a multi-region HA setup runs its own `maas-openfga` process. Since they all point to the **same PostgreSQL database** (using a dedicated `openfga` schema), authorization state is consistent across all region controllers.

## The Database Schema Strategy: Avoiding Two-Phase Commits

This is perhaps the most interesting architectural decision. OpenFGA stores its data (authorization models, relationship tuples, stores) in the same PostgreSQL database as MAAS, but in a **separate dedicated schema** (`openfga`).

### Why this matters

When MAAS needs to update permissions — for example, when adding a user to a group or granting an entitlement — it needs to ensure atomicity: either both the MAAS state and the OpenFGA state are updated, or neither is.

If OpenFGA were running as a completely separate service with its own database, we would face the classic **two-phase commit (2PC) problem**: coordinating a distributed transaction across two different databases. 2PC is notoriously complex, slow, and fragile in practice.

By placing OpenFGA's data in a **different schema within the same database**, MAAS can **write directly to the OpenFGA tables within the same database transaction**:

```
┌────────────────────────────────────────────────────────┐
│              Single PostgreSQL Transaction              │
│                                                        │
│  1. INSERT INTO maasdb.maasserver_usergroup (...)       │
│  2. INSERT INTO openfga.tuple (...)                    │
│                                                        │
│  COMMIT;  ← both changes are atomic                   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

This eliminates the need for 2PC entirely, the need for compensating transactions, and gives us the full ACID guarantees of a single PostgreSQL transaction. The authorization model and the MAAS data model always stay consistent.

### How reads and writes flow

- **Permission checks** (reads) go through the OpenFGA HTTP client over a Unix socket. The `maas-openfga` process handles the evaluation of the authorization model (including relationship expansion, inheritance rules, etc.).
- **Permission mutations** (writes) — such as adding a user to a group or granting an entitlement — are written **directly to the `openfga.tuple` table** by MAAS Python code, bypassing the OpenFGA API for writes. This is what enables single-transaction atomicity.

```
                  ┌────────────┐
                  │   MAAS     │
                  │  (Python)  │
                  └─────┬──────┘
                        │
            ┌───────────┼───────────┐
            │           │           │
            ▼           │           ▼
    ┌───────────┐       │   ┌──────────────┐
    │  OpenFGA  │       │   │  PostgreSQL   │
    │  (Check)  │       │   │  direct write │
    │ via Unix  │       │   │  to openfga   │
    │  socket   │       │   │  .tuple table │
    └───────────┘       │   └──────────────┘
                        │
                   Reads go        Writes go
                   through         directly to
                   OpenFGA API     the database
```

## The Migration Story

Bringing OpenFGA into MAAS involved three types of migrations, all orchestrated from the `dbupgrade` management command:

1. **OpenFGA built-in migrations** (`maas-openfga-migrator`): Create the `openfga` schema and the core OpenFGA tables (stores, authorization models, tuples, changelog, etc.).
2. **MAAS application migrations** (`maas-openfga-app-migrator`): Set up the MAAS-specific authorization model in OpenFGA, defining the relationship types and permission rules.
3. **Alembic migrations**: Create the `maasserver_usergroup` table, the `maasserver_usergroup_members_view` (a SQL view that joins `openfga.tuple` with `auth_user` to expose group membership), and seed the default groups.

The OpenFGA built-in migrations run **before** the Alembic migrations, because some Alembic migrations (like the membership view) depend on the `openfga.tuple` table existing.

Existing users are automatically migrated to the two default groups: admins go into *Administrators* and regular users go into *Users*. These default groups come pre-loaded with entitlements that match the old RBAC behavior, ensuring full backward compatibility.

## The Access Control Model

### Core Concepts

The ReBAC model in MAAS is built around three entities:

- **Users**: Individual MAAS accounts.
- **Groups**: Collections of users. All permissions are granted to groups, never to individual users.
- **Entitlements**: A permission scoped to a specific resource. An entitlement says "group X can do Y on resource Z."

### Permission Scopes

Entitlements are scoped to two resource types:

| Resource Type | Resource ID | Scope |
|---------------|-------------|-------|
| `maas` | `0` | Global — applies across all resource pools |
| `pool` | Pool ID | Per-pool — applies to a specific resource pool |

### Permission Inheritance

The model defines two inheritance rules:

1. **Higher permissions imply lower ones.** For machines, the hierarchy is:

```
can_edit_machines
├── can_deploy_machines
│   └── can_view_machines
│       └── can_view_available_machines
└── can_view_machines
    └── can_view_available_machines
```

Other resources follow the same `can_edit_* → can_view_*` pattern.

2. **Global machine permissions cascade to pools.** If a group has `can_view_machines` on the `maas` resource, it automatically has `can_view_machines` on *every* resource pool.

### Available Entitlements

The system provides fine-grained permissions across many resource categories:

| Category | Entitlements |
|----------|-------------|
| Machines | `can_edit_machines`, `can_deploy_machines`, `can_view_machines`, `can_view_available_machines` |
| Global entities | `can_edit_global_entities`, `can_view_global_entities` |
| Controllers | `can_edit_controllers`, `can_view_controllers` |
| Identities | `can_edit_identities`, `can_view_identities` |
| Configurations | `can_edit_configurations`, `can_view_configurations` |
| Boot entities | `can_edit_boot_entities`, `can_view_boot_entities` |
| Notifications | `can_edit_notifications`, `can_view_notifications` |
| And more... | License keys, devices, IP addresses, DNS records |

## Using the CLI

Managing groups and permissions is straightforward with the MAAS CLI.

### Create a group and add members

```bash
# Create a new group
GROUP_ID=$(maas $PROFILE user-groups create \
    name=developers \
    description="Development team" | jq '.id')

# Add users to the group
maas $PROFILE user-group add-member $GROUP_ID username=alice
maas $PROFILE user-group add-member $GROUP_ID username=bob
```

### Grant per-pool deploy access

```bash
# Allow developers to deploy machines in resource pool 2
maas $PROFILE user-group add-entitlement $GROUP_ID \
    resource_type=pool \
    resource_id=2 \
    entitlement=can_deploy_machines
```

### Grant global read-only access

```bash
# Grant global view access
maas $PROFILE user-group add-entitlement $GROUP_ID \
    resource_type=maas \
    resource_id=0 \
    entitlement=can_view_machines

maas $PROFILE user-group add-entitlement $GROUP_ID \
    resource_type=maas \
    resource_id=0 \
    entitlement=can_view_global_entities
```

### Inspect a group's configuration

```bash
# List group members
maas $PROFILE user-group list-members $GROUP_ID

# List group entitlements
maas $PROFILE user-group list-entitlements $GROUP_ID
```

### Remove permissions

```bash
# Remove an entitlement
maas $PROFILE user-group remove-entitlement $GROUP_ID \
    resource_type=pool \
    resource_id=2 \
    entitlement=can_deploy_machines

# Remove a user from a group
maas $PROFILE user-group remove-member $GROUP_ID username=alice
```

## Implementation Timeline

The ReBAC feature was built incrementally over roughly a month. Here is a summary of the commits:

| Date | Description |
|------|-------------|
| Feb 16 | **OpenFGA infrastructure** — Embedded OpenFGA server (Go binary) listening on a Unix socket, two migrator binaries (`maas-openfga-migrator` and `maas-openfga-app-migrator`), service layer for tuple management, HTTP client for permission checks. |
| Feb 24 | **Build tooling** — Added `.gitignore` and updated Makefile for the `maasopenfga` Go module. |
| Feb 27 | **Replace built-in permission layer (v2)** — Introduced sync/async OpenFGA clients with context caching, migrated existing users to default groups, added `check_permission` decorator, signal handlers for automatic tuple management on user/resource pool changes. |
| Mar 3 | **Embed OpenFGA migrations** — Fixed migration embedding for Go binaries. |
| Mar 3 | **User groups endpoints (v2 + v3)** — CRUD for user groups, cascade delete of OpenFGA tuples, `maasserver_usergroup` table with default groups, Alembic migration. |
| Mar 4 | **Group membership endpoints** — SQL view joining `openfga.tuple` with `auth_user`, v2/v3 endpoints for managing group members. |
| Mar 5 | **Entitlement endpoints (POST/DELETE)** — v2/v3 endpoints to add/remove entitlements, `EntitlementsBuilderFactory` for validation. |
| Mar 6 | **Entitlement endpoints (GET)** — List entitlements for a group in both v2 and v3. |
| Mar 16 | **Replace v3 permission layer** — Fully switched the v3 API (FastAPI) to use OpenFGA for all permission checks. |
| Mar 17 | **Documentation** — Added ReBAC docs to the MAAS documentation. |
| Mar 18 | **Bug fix** — Handle edge case where default user groups are deleted. |

## Key Takeaways

1. **Embedding OpenFGA removes operational complexity.** No separate authorization service to deploy, monitor, or scale. It runs as a lightweight Go process on every region controller.

2. **Same database, different schema** is the key trick. It gives us single-transaction atomicity for permission changes without distributed coordination.

3. **Direct writes to OpenFGA tables** bypass the two-phase commit problem entirely. Permission mutations and MAAS data mutations are committed in a single PostgreSQL transaction.

4. **Unix socket communication** keeps permission checks fast and local. No network hops, no TLS overhead for authorization queries.

5. **Backward compatibility** is preserved through default groups and automatic user migration. Existing deployments upgrade seamlessly.

6. **The model is extensible.** Teams can create custom groups with fine-grained, per-pool permissions — something that was simply not possible with the old role-based system.

---

The full source code is available in the [MAAS repository on GitHub](https://github.com/canonical/maas). For the CLI reference, see the [user-group CLI docs](https://discourse.maas.io/t/user-group/15754) and the [how-to guide](https://discourse.maas.io/t/how-to-manage-users-groups-and-entitlements/15753).
