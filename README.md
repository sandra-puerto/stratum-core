# Stratum-Core — Multi-Tenant Zero-Trust Infrastructure & Island Isolation Architecture

> *"Isolation by design, not by discipline. Built for multi-tenant and multi-organization enterprise scale."*

[![Architecture](https://img.shields.io/badge/Architecture-Stratum%20Core%20v1.2%20(Multi--Tenant)-4f46e5?style=for-the-badge&logo=docker&logoColor=white)](https://github.com/sandrapuerto/stratum-core)
[![Author](https://img.shields.io/badge/Architect-Sandra%20Puerto-f59e0b?style=for-the-badge&logo=linkedin&logoColor=white)](https://sandrapuerto.com)
[![Multi-Org Ingress](https://img.shields.io/badge/Ingress-Multi--Organization%20Tunnels-0284c7?style=for-the-badge&logo=cloudflare&logoColor=white)](gateway/README.md)
[![Security](https://img.shields.io/badge/Security-Strict%20Cap--Drop%20%7C%20DMZ%20Isolated-emerald?style=for-the-badge)](SECURITY.md)
[![License](https://img.shields.io/badge/License-MIT%20Custom%20(Commercial%20Ready)-yellow.svg?style=for-the-badge)](LICENSE)

**Stratum-Core** is an enterprise-grade, cloud-agnostic infrastructure orchestration pattern designed for deploying **multi-tenant, multi-organization, zero-trust containerized workloads** on single or hybrid virtual private servers (VPS).

Engineered by **Sandra Gabriela Puerto Torres**, Stratum solves the friction between high operational cloud costs (OPEX) and enterprise-grade multi-organization isolation. It enables independent enterprise clients and diverse workloads to coexist securely on a unified physical/virtual host without port collisions, public surface exposure, or operational cross-contamination.

---

## 1. Executive Summary & Engineering Motivation

Traditional single-server Docker deployments and multi-client hosting suffer from three structural failure modes:
1. **Network Sprawl & Port Collisions:** Multiple applications and client domains competing for host ports (`80`, `443`, `3306`), leading to brittle port-mapping workarounds.
2. **Excessive Cloud OPEX:** Over-reliance on expensive managed cloud services (e.g., dedicated load balancers, multi-VPC gateways, managed database instances) for modest to medium enterprise workloads.
3. **Implicit Lateral Movement & Tenant Cross-Contamination:** Containers sharing default bridge networks without segmentation, allowing a compromised client container to query internal databases and neighbor workloads.

**Stratum-Core** eliminates these failure modes through **Topological Island Isolation & Multi-Organization Tunnels**:
* **-60% to -80% OPEX Reduction:** Consolidates multiple isolated enterprise organizations onto self-managed infrastructure with zero cloud-vendor lock-in.
* **Multi-Organization Zero-Trust Ingress:** Supports $N$ independent Cloudflare Zero-Trust Tunnels concurrently. Distinct client organizations connect their own Cloudflare accounts, WAF rules, and SSO/IdP policies without credential sharing.
* **100% Ingress Cloaking:** Zero inbound open ports on the physical host. All ingress traffic is terminated exclusively via authenticated outbound Cloudflare Tunnels.
* **Boundary Client Pattern:** Persistence engines exist in air-gapped internal networks (`internal: true`) and can only be reached via hardened Layer 4 TCP stream boundary clients.

---

## 2. High-Level Architecture

### 2.1 Multi-Organization Platform Topology

```mermaid
graph TB
    subgraph MultiOrgWAN ["🌐 Multi-Organization Cloudflare Edge (Zero Trust)"]
        ClientA["🏢 Org A Users<br/>(org-a.com)"] --> CF_A["Cloudflare Account A<br/>(WAF & Access Policies)"]
        ClientB["🏢 Org B Users<br/>(org-b.com)"] --> CF_B["Cloudflare Account B<br/>(WAF & Access Policies)"]
        ClientPri["👤 Primary Org Users<br/>(sandrapuerto.com)"] --> CF_Pri["Primary Cloudflare Account<br/>(WAF & Access Policies)"]
    end

    subgraph Host ["🖥️ Virtual Private Server (Host Level - Zero Open Ingress Ports)"]
        subgraph GatewayIsland ["🚪 Layer 1: Multi-Tenant Access Tier (gateway/)"]
            Tunnel_A["cloudflared-org-a<br/>(Token Org A)"]
            Tunnel_B["cloudflared-org-b<br/>(Token Org B)"]
            Tunnel_Pri["cloudflared-primary<br/>(Token Primary)"]
            
            NPM["Nginx Proxy Manager<br/>(Internal Routing & SSL)"]

            Tunnel_A -->|HTTP Forward| NPM
            Tunnel_B -->|HTTP Forward| NPM
            Tunnel_Pri -->|HTTP Forward| NPM
        end

        subgraph SharedDMZ ["🛡️ Layer 2: Shared Demilitarized Zone (dmz/)"]
            DMZ_Net(("stratum_dmz<br/>(External Bridge Network)"))
        end

        subgraph ConsumerApps ["📦 Layer 3: Application Consumers (Isolated Workloads)"]
            App1["landing-sgpt-web<br/>(sandrapuerto.com)"]
            App2["overleaf-sharelatex<br/>(LaTeX Collaborative Studio)"]
            App3["client-b-erp<br/>(Enterprise Client App)"]
        end

        subgraph DatabaseIsland ["🗄️ Layer 4: Data Layer Island (database/)"]
            DB_Proxy["stratum-database-nginx<br/>(Boundary TCP Proxy)"]
            DB_Net(("stratum_database_internal<br/>(Air-Gapped: internal=true)"))
            
            Postgres[("PostgreSQL 16 Engine<br/>Relational Data")]
            MongoDB[("MongoDB 7.0 Engine<br/>Document Store")]
            Redis[("Redis Engine<br/>Cache & Pub/Sub")]

            DB_Proxy --> DB_Net
            DB_Net --> Postgres
            DB_Net --> MongoDB
            DB_Net --> Redis
        end
    end

    CF_A -.->|Encrypted Outbound Tunnel A| Tunnel_A
    CF_B -.->|Encrypted Outbound Tunnel B| Tunnel_B
    CF_Pri -.->|Encrypted Outbound Tunnel Primary| Tunnel_Pri

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

    class ClientA,ClientB,ClientPri,CF_A,CF_B,CF_Pri edge;
    class Tunnel_A,Tunnel_B,Tunnel_Pri,NPM gateway;
    class DMZ_Net dmz;
    class App1,App2,App3 consumer;
    class DB_Proxy,Postgres,MongoDB,Redis database;
```

---

## 3. Core Component Matrix

Stratum is modularized into three autonomous components, each maintaining an independent Compose project, lifecycle, configuration, and security perimeter:

| Component | Responsibility | Provisioned Network | Isolation Level |
| :--- | :--- | :--- | :--- |
| [`dmz/`](dmz/) | Shared network provisioning & persistence anchor | `stratum_dmz` | Inter-island convergence bridge |
| [`gateway/`](gateway/) | Multi-organization ingress tunnels & reverse proxying | `stratum_gateway_internal` | Multi-tenant egress-only access tier |
| [`database/`](database/) | Multi-engine persistence & boundary stream proxy | `stratum_database_internal` | Air-gapped internal data tier |

---

## 4. Multi-Organization Traffic Flow Model

```mermaid
sequenceDiagram
    autonumber
    actor UserA as Org A Client (app.org-a.com)
    participant CF_A as Cloudflare Edge (Org A Account)
    participant Tunnel_A as cloudflared-org-a (Gateway)
    participant NPM as Nginx Proxy Manager
    participant DMZ as stratum_dmz Network
    participant App as Org A Consumer Web App
    participant DBProxy as stratum-database-nginx Boundary
    participant MongoDB as MongoDB 7.0 (Air-Gapped)

    UserA->>CF_A: HTTPS Request (e.g. app.org-a.com)
    CF_A->>Tunnel_A: Encrypted Stream (Outbound Tunnel A)
    Tunnel_A->>NPM: Forward HTTP via stratum_gateway_internal
    NPM->>DMZ: Route by Host Header (app.org-a.com) via stratum_dmz
    DMZ->>App: Process Application Request
    opt Persistence Query
        App->>DMZ: Query stratum-database-nginx:27017
        DMZ->>DBProxy: Terminate at Boundary TCP Proxy
        DBProxy->>MongoDB: Stream TCP via stratum_database_internal
        MongoDB-->>DBProxy: Return Query Result
        DBProxy-->>App: Stream Response to App
    end
    App-->>NPM: HTTP 200 OK Response
    NPM-->>Tunnel_A: Pipe Output
    Tunnel_A-->>CF_A: Encrypted Response
    CF_A-->>UserA: Rendered Secure Web Response
```

---

## 5. Security & Hardening Baseline

Every container deployed across Stratum-Core adheres strictly to the **Stratum Hardening Specification**:

```
                         ┌────────────────────────────────────┐
                         │   STRATUM HARDENING BASELINE       │
                         ├────────────────────────────────────┤
                         │ • read_only: true (Immutable root) │
                         │ • cap_drop: [ALL]                  │
                         │ • no-new-privileges: true          │
                         │ • tmpfs for volatile paths         │
                         │ • internal: true for data bridges  │
                         │ • Multi-organization profiles      │
                         │ • Memory, CPU & PID constraints    │
                         │ • Structured log-rotation (10MBx3) │
                         └────────────────────────────────────┘
```

### 5.1 Defense-in-Depth Attributes

* **No Inbound Public Ports:** The VPS does not listen on ports `80`, `443`, or database ports. Cloudflare handles edge TLS termination and forwards requests through authenticated outbound tunnels.
* **Tenant Ingress Separation:** Organizations do not share Cloudflare API keys or tunnel credentials. Each organization operates a discrete, isolated `cloudflared` process.
* **Zero Trust Lateral Movement:** Compromising a web application gives an attacker access only to `stratum_dmz`. The database storage engines (`postgres`, `mongodb`, `redis`) reside on `stratum_database_internal` (`internal: true`), completely unreachable without credentials through the TCP proxy.
* **Static Host Integrity:** Containers cannot write to their own root filesystem (`read_only: true`). Temporary execution files reside in capped RAM mounts (`tmpfs`).

---

## 6. Deployment & Orchestration Runbook

The components must be initialized sequentially to satisfy Docker network dependency graphs.

```mermaid
graph LR
    Step1["1. dmz/<br/>(Provision DMZ Network)"] --> Step2["2. gateway/<br/>(Launch Multi-Org Tunnels & NPM)"]
    Step2 --> Step3["3. database/<br/>(Deploy Persistence & Proxy)"]
    Step3 --> Step4["4. Consumer Apps<br/>(Attach to stratum_dmz)"]

    style Step1 fill:#0f172a,stroke:#f59e0b,stroke-width:2px,color:#fff
    style Step2 fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#fff
    style Step3 fill:#0f172a,stroke:#ec4899,stroke-width:2px,color:#fff
    style Step4 fill:#0f172a,stroke:#10b981,stroke-width:2px,color:#fff
```

### 6.1 Provisioning Commands

```bash
# 1. Provision the Shared DMZ Network
cd dmz/
cp .env.example .env && chmod 600 .env
docker compose up -d

# 2. Deploy the Ingress Access Layer
cd ../gateway/
cp .env.example .env && chmod 600 .env

# Standard (Primary Organization only):
docker compose up -d

# Or Multi-Organization with Profiles (e.g. Org A + Primary):
docker compose --profile org-a up -d

# Or all configured organizations:
docker compose --profile all up -d

# 3. Deploy the Data Layer & Boundary Proxy
cd ../database/
cp .env.example .env && chmod 600 .env
docker compose up -d
```

### 6.2 State Verification

```bash
# Verify container health across all Stratum tiers
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Networks}}"

# Verify network segmentation
docker network ls | grep stratum
```

Expected Network Inventory:
* `stratum_dmz` (Bridge / External Convergence)
* `stratum_gateway_internal` (Bridge / Multi-Tunnel Ingress)
* `stratum_database_internal` (Bridge / Air-Gapped `internal: true`)

---

## 7. Repository Structure

```
stratum-core/
├── LICENSE                 # Enhanced MIT License (Commercial & Multi-Tenant Permitted)
├── README.md               # Master Architecture & Technical Specification
├── SECURITY.md             # Responsible Vulnerability Disclosure Protocol
├── CONTRIBUTING.md         # Multi-Tenant Contribution Guidelines
├── .gitignore              # Multi-tier exclusion rules
├── dmz/                    # Shared DMZ Network Anchor & Topology
│   ├── docker-compose.yml  # Network lifecycle container
│   ├── .env.example        # Network parameter definitions
│   └── README.md           # DMZ component manual & subnet specs
├── gateway/                # Multi-Tenant Ingress & Access Tier
│   ├── docker-compose.yml  # Multi-Tunnel cloudflared + Nginx Proxy Manager
│   ├── .env.example        # Multi-org tokens & configuration
│   └── README.md           # Gateway routing manual & TLS administration
└── database/               # Data Persistence & Boundary Tier
    ├── docker-compose.yml  # PostgreSQL + MongoDB + Redis + TCP Proxy
    ├── .env.example        # Root credentials & memory configurations
    ├── README.md           # Data operations, backup & rotation manual
    └── nginx/
        └── nginx.conf      # Layer 4 TCP Stream Boundary configuration
```

---

## 8. Author & Commercial Architecture Provenance

**Stratum-Core** was conceived, architected, and implemented by **Sandra Gabriela Puerto Torres**.

* **Role:** Backend Developer & Technical Infrastructure Director
* **Core Philosophy:** *Systems must be financially viable, structurally isolated, and resilient to operational turbulence without requiring oversized cloud budgets.*
* **Commercial Applications:** SaaS Platforms, Managed Service Providers (MSP), Agency Multi-Client Infrastructure, Enterprise Private Clouds.
* **Specialization:** PHP/Laravel, Linux Systems, Network Topology, Proxmox VE, Cloudflare Edge Architecture, Container Hardening.
* **Website:** [https://sandrapuerto.com](https://sandrapuerto.com)
* **Contact:** [contacto@sandrapuerto.com](mailto:contacto@sandrapuerto.com)

---

## 9. License

This repository is distributed under an enhanced MIT License with a mandatory private security vulnerability disclosure requirement. Commercial and multi-tenant hosting use is fully permitted. See [LICENSE](LICENSE) for full legal text.