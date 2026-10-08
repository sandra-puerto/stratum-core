# Stratum-Core — Self-Hosted Multi-Organization Container Infrastructure with Zero-Trust Ingress

> *"Isolation by design, not by discipline."*

[![Architecture](https://img.shields.io/badge/Architecture-Stratum%20Core%20v1.2-4f46e5?style=for-the-badge&logo=docker&logoColor=white)](https://github.com/sandra-puerto/stratum-core)
[![Author](https://img.shields.io/badge/Author-Sandra%20Puerto-f59e0b?style=for-the-badge&logo=linkedin&logoColor=white)](https://sandrapuerto.com)
[![Ingress](https://img.shields.io/badge/Ingress-Multi--Organization%20Tunnels-0284c7?style=for-the-badge&logo=cloudflare&logoColor=white)](gateway/README.md)
[![Security](https://img.shields.io/badge/Security-Cap--Drop%20%7C%20Isolated%20Data%20Tier-emerald?style=for-the-badge)](SECURITY.md)
[![License](https://img.shields.io/badge/License-MIT--based%20Custom-yellow.svg?style=for-the-badge)](LICENSE)

**Stratum-Core** is a self-hosted infrastructure orchestration pattern for running containerized workloads of **several independent organizations** on a single Docker host (or a small set of hosts) without publishing any application port to the public interface.

Designed and implemented by **Sandra Gabriela Puerto Torres**, Stratum addresses the cost and complexity of hosting multiple organizations' applications on managed cloud services. It lets independent organizations share one host while keeping their **ingress paths, credentials and data tier separated**, and without port collisions.

---

## 1. Summary & Motivation

Single-server Docker deployments that host more than one client tend to run into three structural problems:

1. **Network sprawl and port collisions:** several applications and domains compete for host ports (`80`, `443`, `3306`), which leads to brittle port-mapping workarounds.
2. **Recurring cost of managed services:** load balancers, multi-VPC gateways and managed database instances can be disproportionate for small and medium workloads.
3. **Flat networks:** containers on default bridge networks can reach each other and the databases, with no defined boundary around the data tier.

Stratum-Core addresses them through **topological island isolation and per-organization tunnels**:

* **Self-managed, lower recurring cost:** several organizations run on one self-managed host instead of separate managed services.
* **Per-organization ingress:** each organization runs its own `cloudflared` process with its own tunnel token. Organizations connect their own Cloudflare accounts, WAF rules and access policies without sharing credentials.
* **No application ports published on the host:** all web ingress arrives through authenticated outbound Cloudflare Tunnels.
* **Boundary client pattern:** the persistence engines live on a network with `internal: true` (no external egress) and are reached only through a Layer 4 TCP proxy.

> **Dependency note:** the pattern runs on any Docker host, but its ingress layer depends on Cloudflare Tunnel.

---

## 2. Architecture

### 2.1 Platform Topology

```mermaid
graph TB
    subgraph Edge ["🌐 Cloudflare Edge (one account per organization)"]
        UsersPri["🏢 Primary Org Users<br/>(primary-org.com)"] --> CF_Pri["Primary Org<br/>Cloudflare Account"]
        UsersA["🏢 Org A Users<br/>(org-a.com)"] --> CF_A["Org A<br/>Cloudflare Account"]
    end

    subgraph Host ["🖥️ Docker Host (no application ports published)"]
        subgraph GatewayIsland ["🚪 Layer 1: Access Tier (gateway/)"]
            Tunnel_Pri["cloudflared-primary<br/>(Primary token)"]
            Tunnel_A["cloudflared-org-a<br/>(Org A token)"]
            NPM["Nginx Proxy Manager<br/>(internal routing & SSL)"]

            Tunnel_Pri -->|HTTP| NPM
            Tunnel_A -->|HTTP| NPM
        end

        subgraph SharedDMZ ["🛡️ Layer 2: Shared DMZ (dmz/)"]
            DMZ_Net(("stratum_dmz<br/>(shared bridge network)"))
        end

        subgraph ConsumerApps ["📦 Layer 3: Application Workloads"]
            App1["primary-portal<br/>(primary-org.com)"]
            App2["overleaf-stratum<br/>(collaborative LaTeX editor)"]
            App3["org-a-app<br/>(org-a.com)"]
        end

        subgraph DatabaseIsland ["🗄️ Layer 4: Data Tier (database/)"]
            DB_Proxy["stratum-database-nginx<br/>(Layer 4 TCP boundary proxy)"]
            DB_Net(("stratum_database_internal<br/>(internal: true)"))

            Postgres[("PostgreSQL 16")]
            MongoDB[("MongoDB 7.0")]
            Redis[("Redis")]

            DB_Proxy --> DB_Net
            DB_Net --> Postgres
            DB_Net --> MongoDB
            DB_Net --> Redis
        end
    end

    CF_Pri -.->|Outbound tunnel| Tunnel_Pri
    CF_A -.->|Outbound tunnel| Tunnel_A

    NPM --> DMZ_Net
    DMZ_Net --> App1
    DMZ_Net --> App2
    DMZ_Net --> App3
    DMZ_Net --> DB_Proxy

    classDef edge fill:#f59e0b,stroke:#b45309,stroke-width:2px,color:#000;
    classDef gateway fill:#1e293b,stroke:#38bdf8,stroke-width:2px,color:#fff;
    classDef dmz fill:#0f172a,stroke:#f59e0b,stroke-width:2px,color:#fff;
    classDef consumer fill:#1e293b,stroke:#10b981,stroke-width:2px,color:#fff;
    classDef database fill:#1e293b,stroke:#ec4899,stroke-width:2px,color:#fff;

    class UsersPri,UsersA,CF_Pri,CF_A edge;
    class Tunnel_Pri,Tunnel_A,NPM gateway;
    class DMZ_Net dmz;
    class App1,App2,App3 consumer;
    class DB_Proxy,Postgres,MongoDB,Redis database;
```

---

## 3. Components

Stratum is split into three components. Each one is an independent Compose project with its own lifecycle, configuration and network:

| Component | Responsibility | Network | Isolation |
| :--- | :--- | :--- | :--- |
| [`dmz/`](dmz/) | Shared network provisioning | `stratum_dmz` | Convergence bridge between tiers |
| [`gateway/`](gateway/) | Per-organization tunnels and reverse proxy | `stratum_gateway_internal` | Egress-only access tier |
| [`database/`](database/) | PostgreSQL, MongoDB, Redis and TCP boundary proxy | `stratum_database_internal` | Internal network (`internal: true`) |

---

## 4. Request Flow

```mermaid
sequenceDiagram
    autonumber
    actor UserA as Org A User (app.org-a.com)
    participant CF_A as Cloudflare Edge (Org A account)
    participant Tunnel_A as cloudflared-org-a (Gateway)
    participant NPM as Nginx Proxy Manager
    participant DMZ as stratum_dmz
    participant App as Org A Web App
    participant DBProxy as stratum-database-nginx
    participant MongoDB as MongoDB 7.0 (internal network)

    UserA->>CF_A: HTTPS request (app.org-a.com)
    CF_A->>Tunnel_A: Encrypted stream (outbound tunnel)
    Tunnel_A->>NPM: Forward HTTP via stratum_gateway_internal
    NPM->>DMZ: Route by Host header via stratum_dmz
    DMZ->>App: Application request
    opt Persistence query
        App->>DMZ: Connect to stratum-database-nginx:27017
        DMZ->>DBProxy: Terminate at TCP boundary proxy
        DBProxy->>MongoDB: Stream over stratum_database_internal
        MongoDB-->>DBProxy: Query result
        DBProxy-->>App: Stream response
    end
    App-->>NPM: HTTP response
    NPM-->>Tunnel_A: Response
    Tunnel_A-->>CF_A: Encrypted response
    CF_A-->>UserA: Rendered response
```

---

## 5. Security Baseline

The hardening specification below is applied to the containers in each tier **where the image supports it**. Some images (for example Nginx Proxy Manager or the database engines) require writable paths, which are handled through volumes or `tmpfs`.

```
                         ┌────────────────────────────────────┐
                         │   STRATUM HARDENING BASELINE       │
                         ├────────────────────────────────────┤
                         │ • read_only: true (where possible) │
                         │ • cap_drop: [ALL]                  │
                         │ • no-new-privileges: true          │
                         │ • tmpfs for volatile paths         │
                         │ • internal: true for the data tier │
                         │ • Memory, CPU & PID constraints    │
                         │ • Log rotation (10 MB x 3)         │
                         └────────────────────────────────────┘
```

### 5.1 What the design provides

* **No application ports on the host:** the host does not publish `80`, `443` or database ports. Cloudflare terminates TLS at the edge and forwards requests through authenticated outbound tunnels.
* **Per-organization ingress separation:** organizations do not share Cloudflare API keys or tunnel credentials. Each one runs a separate `cloudflared` process.
* **Defined data-tier boundary:** the database engines (`postgres`, `mongodb`, `redis`) sit on `stratum_database_internal` (`internal: true`), with no external egress. Applications reach them only through the TCP boundary proxy, and still need valid database credentials.
* **Immutable root filesystem where supported:** with `read_only: true`, containers cannot modify their own root filesystem. Temporary files go to capped `tmpfs` mounts.

### 5.2 Scope and limitations

* **Applications share `stratum_dmz`.** Isolation between organizations is enforced at the ingress layer (separate tunnels and credentials) and at the data tier (credentials per database or user). It is **not** enforced between application containers: any container on `stratum_dmz` can reach the other applications and the database proxy. A compromised application therefore reaches the DMZ, not only itself.
* **Stronger tenant isolation** requires a separate network per organization and separate database credentials with minimal privileges. This is a recommended extension and is not part of the current baseline.
* **Ingress depends on Cloudflare Tunnel.** The host does not publish ports, but availability and access policies at the edge depend on that service.

---

## 6. Deployment Runbook

The components must be started in order to satisfy the Docker network dependencies.

```mermaid
graph LR
    Step1["1. dmz/<br/>(provision DMZ network)"] --> Step2["2. gateway/<br/>(tunnels + Nginx Proxy Manager)"]
    Step2 --> Step3["3. database/<br/>(engines + TCP proxy)"]
    Step3 --> Step4["4. Applications<br/>(attach to stratum_dmz)"]

    style Step1 fill:#0f172a,stroke:#f59e0b,stroke-width:2px,color:#fff
    style Step2 fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#fff
    style Step3 fill:#0f172a,stroke:#ec4899,stroke-width:2px,color:#fff
    style Step4 fill:#0f172a,stroke:#10b981,stroke-width:2px,color:#fff
```

### 6.1 Commands

```bash
# 1. Provision the shared DMZ network
cd dmz/
cp .env.example .env && chmod 600 .env
docker compose up -d

# 2. Deploy the access tier
cd ../gateway/
cp .env.example .env && chmod 600 .env

# Primary organization only:
docker compose up -d

# Primary + Org A (Compose profile):
docker compose --profile org-a up -d

# All configured organizations:
docker compose --profile all up -d

# 3. Deploy the data tier and boundary proxy
cd ../database/
cp .env.example .env && chmod 600 .env
docker compose up -d
```

### 6.2 Verification

```bash
# Container health and networks across tiers
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Networks}}"

# Network segmentation
docker network ls | grep stratum
```

Expected networks:
* `stratum_dmz` (bridge, shared between tiers)
* `stratum_gateway_internal` (bridge, tunnel ingress)
* `stratum_database_internal` (bridge, `internal: true`)

---

## 7. Repository Structure

```
stratum-core/
├── LICENSE                 # MIT-based custom license (commercial use permitted)
├── README.md               # Architecture and technical specification
├── SECURITY.md             # Vulnerability disclosure protocol
├── CONTRIBUTING.md         # Contribution guidelines
├── .gitignore              # Exclusion rules
├── dmz/                    # Shared DMZ network
│   ├── docker-compose.yml
│   ├── .env.example
│   └── README.md
├── gateway/                # Per-organization tunnels + reverse proxy
│   ├── docker-compose.yml
│   ├── .env.example
│   └── README.md
└── database/               # Data tier and boundary proxy
    ├── docker-compose.yml
    ├── .env.example
    ├── README.md
    └── nginx/
        └── nginx.conf      # Layer 4 TCP stream configuration
```

---

## 8. Author

**Stratum-Core** was designed and implemented by **Sandra Gabriela Puerto Torres**.

* **Role:** Backend Developer (PHP/Laravel) and infrastructure
* **Approach:** *Systems should be financially viable, structurally separated and resilient, without requiring oversized cloud budgets.*
* **Intended use cases:** self-hosted platforms for small organizations, agencies hosting several clients, and managed-service scenarios on private infrastructure.
* **Stack:** PHP/Laravel, Linux, Docker, Proxmox VE, Cloudflare Tunnel, Nginx
* **Website:** [https://sandrapuerto.com](https://sandrapuerto.com)
* **Contact:** [contacto@sandrapuerto.com](mailto:contacto@sandrapuerto.com)

---

## 9. License

Distributed under an MIT-based custom license that requires private disclosure of security vulnerabilities. Commercial and multi-organization hosting use is permitted. See [LICENSE](LICENSE) for the full text.
