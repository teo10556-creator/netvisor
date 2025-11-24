# NetVisor Architecture Overview

**Version:** 0.4.3
**Last Updated:** November 2025

---

## Table of Contents

1. [High-Level Architecture](#high-level-architecture)
2. [Component Overview](#component-overview)
3. [Backend Architecture](#backend-architecture)
4. [Frontend Architecture](#frontend-architecture)
5. [Database Schema](#database-schema)
6. [Communication Flow](#communication-flow)
7. [Discovery System](#discovery-system)
8. [Security Architecture](#security-architecture)
9. [Deployment Models](#deployment-models)
10. [Technology Stack](#technology-stack)

---

## High-Level Architecture

NetVisor follows a **distributed client-server architecture** with three main components:

```
┌─────────────────────────────────────────────────────────────────┐
│                         NetVisor System                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐         ┌──────────────┐                    │
│  │  Web Browser │◄────────┤    Server    │                    │
│  │    (User)    │  HTTP   │  (Central)   │                    │
│  └──────────────┘         └───────┬──────┘                    │
│                                    │                            │
│                                    │ HTTP API                   │
│                                    │                            │
│                    ┌───────────────┼───────────────┐           │
│                    │               │               │           │
│              ┌─────▼────┐    ┌────▼─────┐   ┌────▼─────┐     │
│              │ Daemon 1 │    │ Daemon 2 │   │ Daemon N │     │
│              │ (VLAN 1) │    │ (VLAN 2) │   │  (Edge)  │     │
│              └────┬─────┘    └────┬─────┘   └────┬─────┘     │
│                   │               │               │           │
│              ┌────▼──────────────▼───────────────▼────┐      │
│              │     Physical Network Infrastructure    │      │
│              └───────────────────────────────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Architecture Principles

1. **Distributed Scanning**: Multiple daemons can scan from different network vantage points
2. **Centralized Control**: Single server manages all data and orchestrates discovery
3. **Agent-Based**: Lightweight daemons deployed where needed
4. **Scalable**: Add daemons to scan additional network segments
5. **Self-Hosted**: Complete data sovereignty and privacy

---

## Component Overview

### 1. Server (Central Hub)

**Purpose:** Central coordination, data storage, API, and web UI

**Responsibilities:**
- Store all discovered network data
- Serve web UI to users
- Provide REST API for daemons and UI
- Generate network topology visualizations
- Manage user authentication and authorization
- Schedule recurring discoveries
- Coordinate multiple daemons

**Technology:** Rust (Axum framework) + PostgreSQL + Svelte UI

**Location:** Single instance (can be anywhere)

---

### 2. Daemon (Discovery Agent)

**Purpose:** Network scanning and service discovery

**Responsibilities:**
- Scan network subnets for active hosts
- Detect running services (50+ definitions)
- Query Docker API for container info
- Report discoveries back to server
- Execute scheduled scans
- Self-report capabilities to server

**Technology:** Rust (async runtime)

**Location:** One or more, deployed on different network segments

---

### 3. Database (PostgreSQL)

**Purpose:** Persistent data storage

**Stores:**
- User accounts and sessions
- Networks, subnets, hosts
- Services and their relationships
- Discovery history
- Topology configurations
- Daemon registrations

**Technology:** PostgreSQL 14+

**Location:** Co-located with server

---

### 4. Web UI (Frontend)

**Purpose:** User interface for visualization and management

**Features:**
- Interactive network topology
- Host and service management
- Discovery scheduling
- User authentication
- Real-time progress updates (SSE)

**Technology:** Svelte + TypeScript + TailwindCSS

**Location:** Served by server (static files)

---

## Backend Architecture

### Directory Structure

```
backend/src/
├── bin/
│   ├── server.rs          # Server entry point
│   └── daemon.rs          # Daemon entry point
├── server/               # Server-specific code
│   ├── auth/            # Authentication & authorization
│   ├── users/           # User management
│   ├── organizations/   # Multi-tenancy
│   ├── networks/        # Network entities
│   ├── hosts/           # Host management
│   ├── services/        # Service definitions & detection
│   ├── subnets/         # Subnet management
│   ├── groups/          # Service grouping
│   ├── daemons/         # Daemon registration & management
│   ├── discovery/       # Discovery orchestration
│   ├── topology/        # Topology generation
│   ├── api_keys/        # API key management
│   ├── billing/         # Billing (future cloud service)
│   └── shared/          # Shared server utilities
│       ├── handlers/    # HTTP handlers
│       ├── services/    # Business logic
│       ├── storage/     # Database access layer
│       └── types/       # Common types
├── daemon/              # Daemon-specific code
│   ├── discovery/       # Discovery execution
│   │   ├── service/
│   │   │   ├── network.rs    # Network scanning
│   │   │   ├── docker.rs     # Docker discovery
│   │   │   └── self_report.rs # Capability reporting
│   │   └── manager.rs   # Discovery coordination
│   ├── runtime/         # Daemon runtime & state
│   ├── utils/           # Platform-specific utilities
│   │   ├── scanner.rs   # Port scanning
│   │   ├── linux.rs     # Linux-specific code
│   │   ├── macos.rs     # macOS-specific code
│   │   └── windows.rs   # Windows-specific code
│   └── shared/          # Shared daemon utilities
└── lib.rs               # Library root
```

### Layer Architecture

```
┌─────────────────────────────────────────┐
│         HTTP Handlers (Axum)            │  ← Routes & request handling
├─────────────────────────────────────────┤
│        Services (Business Logic)        │  ← Core business rules
├─────────────────────────────────────────┤
│      Storage (Repository Pattern)       │  ← Database abstraction
├─────────────────────────────────────────┤
│           Database (sqlx)               │  ← PostgreSQL access
└─────────────────────────────────────────┘
```

### Key Design Patterns

1. **Repository Pattern**: Storage layer abstracts database access
2. **Service Layer**: Business logic separated from HTTP handlers
3. **Trait-Based**: Extensive use of traits for abstraction
4. **Generic Storage**: `GenericPostgresStorage<T>` for CRUD operations
5. **Type Safety**: Strong typing with Rust's type system

---

## Frontend Architecture

### Directory Structure

```
ui/src/
├── routes/                    # SvelteKit routes
│   ├── +layout.svelte        # Root layout
│   └── +page.svelte          # Landing page
├── lib/
│   ├── features/             # Feature modules
│   │   ├── auth/            # Authentication
│   │   ├── topology/        # Network visualization
│   │   ├── hosts/           # Host management
│   │   ├── services/        # Service management
│   │   ├── subnets/         # Subnet management
│   │   ├── groups/          # Group management
│   │   ├── daemons/         # Daemon management
│   │   ├── discovery/       # Discovery scheduling
│   │   ├── networks/        # Network management
│   │   ├── users/           # User management
│   │   ├── organizations/   # Organization management
│   │   └── api_keys/        # API key management
│   ├── shared/              # Shared components
│   │   ├── components/      # Reusable UI components
│   │   │   ├── layout/     # Layout components
│   │   │   ├── data/       # Data display components
│   │   │   ├── forms/      # Form components
│   │   │   └── feedback/   # Feedback components
│   │   ├── stores/         # Svelte stores
│   │   └── utils/          # Utility functions
│   └── templates/          # Template files
└── main.ts                  # Entry point
```

### State Management

**Svelte Stores** for reactive state:
- `auth.ts` - User authentication state
- `discovery.ts` - Discovery sessions
- `topology.ts` - Topology visualization state
- `hosts.ts`, `services.ts`, etc. - Entity stores

**Server-Sent Events (SSE)** for real-time updates:
- Discovery progress updates
- Live scan results

### Component Architecture

```
┌─────────────────────────────────────────┐
│           Page Components               │  ← Route-level components
├─────────────────────────────────────────┤
│        Feature Components               │  ← Feature-specific UI
├─────────────────────────────────────────┤
│         Shared Components               │  ← Reusable UI elements
├─────────────────────────────────────────┤
│            Svelte Stores                │  ← State management
├─────────────────────────────────────────┤
│             API Layer                   │  ← HTTP client
└─────────────────────────────────────────┘
```

---

## Database Schema

### Core Entities

```
┌──────────────┐
│     users    │
├──────────────┤
│ id (PK)      │
│ email        │
│ password_hash│
│ org_id (FK)  │
└──────┬───────┘
       │
       ▼
┌──────────────┐      ┌──────────────┐
│organizations │      │   networks   │
├──────────────┤      ├──────────────┤
│ id (PK)      │◄─────┤ id (PK)      │
│ name         │      │ org_id (FK)  │
│ plan         │      │ name         │
└──────────────┘      └──────┬───────┘
                             │
                    ┌────────┴────────┐
                    │                 │
              ┌─────▼──────┐   ┌─────▼──────┐
              │   daemons  │   │   subnets  │
              ├────────────┤   ├────────────┤
              │ id (PK)    │   │ id (PK)    │
              │ network_id │   │ network_id │
              │ name       │   │ cidr       │
              │ api_key    │   │ name       │
              └────────────┘   └─────┬──────┘
                                     │
                               ┌─────▼──────┐
                               │   hosts    │
                               ├────────────┤
                               │ id (PK)    │
                               │ network_id │
                               │ interfaces │
                               │ ports      │
                               └─────┬──────┘
                                     │
                               ┌─────▼──────┐
                               │  services  │
                               ├────────────┤
                               │ id (PK)    │
                               │ host_id    │
                               │ name       │
                               │ port       │
                               │ definition │
                               └────────────┘
```

### Key Tables

- **users**: User accounts and authentication
- **organizations**: Multi-tenant isolation
- **networks**: Logical network groupings
- **daemons**: Registered discovery agents
- **subnets**: Network segments (CIDR blocks)
- **hosts**: Discovered devices
- **services**: Services running on hosts
- **groups**: Logical service groupings
- **discoveries**: Discovery session history
- **api_keys**: API authentication tokens

---

## Communication Flow

### 1. User Authentication Flow

```
Browser                Server               Database
   │                      │                     │
   │──Login Request──────▶│                     │
   │   (email/password)   │                     │
   │                      │──Query User────────▶│
   │                      │                     │
   │                      │◄───User Data────────│
   │                      │                     │
   │                      │──Verify Password    │
   │                      │   (Argon2)          │
   │                      │                     │
   │◄─Session Cookie──────│                     │
   │   (JWT)              │                     │
```

### 2. Discovery Execution Flow

```
UI              Server          Daemon         Network
│                 │                │              │
│─Start Scan────▶│                │              │
│                 │                │              │
│                 │──Trigger────▶  │              │
│                 │   Discovery    │              │
│                 │                │              │
│                 │                │──Scan───────▶│
│                 │                │   Hosts      │
│                 │                │              │
│                 │                │◄─Results─────│
│                 │                │              │
│                 │◄─Report────────│              │
│                 │   Discoveries  │              │
│                 │                │              │
│◄─SSE Updates────│                │              │
│   (Progress)    │                │              │
│                 │                │              │
│                 │──Store────────▶│              │
│                 │   Results      Database      │
│                 │                │              │
│─View Results───▶│──Query────────▶│              │
│                 │   Hosts        │              │
│◄─Host Data──────│◄───────────────│              │
```

### 3. Daemon Registration Flow

```
Daemon                 Server              Database
  │                      │                     │
  │──Register───────────▶│                     │
  │   (capabilities)     │                     │
  │                      │──Create Daemon─────▶│
  │                      │                     │
  │                      │◄───Daemon ID────────│
  │                      │                     │
  │◄─Registration────────│                     │
  │   Confirmation       │                     │
  │                      │                     │
  │──Heartbeat──────────▶│                     │
  │   (every 30s)        │                     │
  │                      │──Update Last Seen──▶│
```

### 4. Topology Generation Flow

```
UI              Server             Database
│                 │                    │
│─Request────────▶│                    │
│  Topology       │                    │
│                 │──Query All─────────▶│
│                 │   Entities         │
│                 │   (hosts,          │
│                 │    services,       │
│                 │    subnets)        │
│                 │                    │
│                 │◄───Data─────────────│
│                 │                    │
│                 │──Generate          │
│                 │   Topology         │
│                 │   (Graph alg)      │
│                 │                    │
│◄─Topology───────│                    │
│  JSON           │                    │
│  (nodes+edges)  │                    │
```

---

## Discovery System

### Discovery Types

NetVisor supports three discovery types:

#### 1. Self-Report Discovery

**Purpose:** Daemon reports its own capabilities

**Process:**
1. Daemon detects network interfaces
2. Checks Docker socket availability
3. Reports capabilities to server
4. Server updates daemon record

**Data Collected:**
- Network interfaces and subnets
- Docker socket access (yes/no)
- Operating system info

#### 2. Network Scan Discovery

**Purpose:** Scan IP ranges for hosts and services

**Process:**
```
1. Get subnet CIDR (e.g., 192.168.1.0/24)
2. Generate all IPs in range (192.168.1.1 - 192.168.1.254)
3. For each IP (parallel):
   a. TCP SYN scan on common ports
   b. If port open, grab banner
   c. HTTP(S) request to detect web services
   d. Match against service definitions
   e. Reverse DNS lookup for hostname
   f. ARP table lookup for MAC address
4. Report all discoveries to server
```

**Service Detection:**
- 50+ pre-defined service patterns
- Port-based detection
- Banner grabbing
- HTTP endpoint matching
- Response header analysis

**Example Service Definitions:**
- Plex: Port 32400, HTTP response contains "X-Plex"
- Home Assistant: Port 8123, HTTP response contains "Home Assistant"
- Proxmox: Port 8006, HTTPS with specific cert patterns
- Docker: Port 2375/2376, API endpoint detection

#### 3. Docker Discovery

**Purpose:** Discover containers via Docker API

**Process:**
```
1. Connect to Docker socket (/var/run/docker.sock)
2. List all containers
3. For each container:
   a. Get container metadata (name, image, labels)
   b. Get network info (networks, IPs, ports)
   c. Exec into container to scan internal ports
   d. Match services running inside
4. Map container→host relationships
5. Report to server
```

**Special Features:**
- Container-level service detection
- Docker network topology
- Container→VM relationships (Proxmox)

### Discovery Manager

**Coordination Logic:**

```rust
// Simplified discovery flow
async fn execute_discovery(session: DiscoverySession) {
    match session.type {
        SelfReport => {
            // Report daemon capabilities
            let interfaces = get_network_interfaces();
            let has_docker = check_docker_socket();
            report_to_server(capabilities);
        }

        NetworkScan(subnets) => {
            // Scan IP ranges
            for subnet in subnets {
                let ips = generate_ip_range(subnet.cidr);

                // Parallel scan
                let hosts = scan_hosts_concurrent(ips, max_concurrent);

                for host in hosts {
                    let services = detect_services(host);
                    report_to_server(host, services);
                }
            }
        }

        DockerScan => {
            // Docker API discovery
            let containers = docker_client.list_containers();

            for container in containers {
                let services = scan_container(container);
                report_to_server(container, services);
            }
        }
    }
}
```

---

## Security Architecture

### Authentication & Authorization

**Authentication Methods:**
1. **Username/Password** (Argon2id hashing)
2. **OIDC** (OAuth2/OpenID Connect)
3. **API Keys** (for daemons)

**Session Management:**
- Server-side sessions (PostgreSQL storage)
- HttpOnly cookies
- SameSite=Lax protection
- Configurable secure flag for HTTPS

**Authorization Levels:**
1. **Owner** - Full control
2. **Admin** - Manage resources
3. **Member** - View and basic operations
4. **Viewer** - Read-only (future)

### Network Security

**Daemon Authentication:**
- API key based (Bearer token)
- Keys generated by server
- Expiration support
- Per-network isolation

**Communication:**
- HTTP(S) between components
- No external dependencies
- All communication within trust boundary

**Privilege Requirements:**
- **Server**: Normal user (or root for port 80)
- **Daemon**: CAP_NET_RAW + CAP_NET_ADMIN (for scanning)
  - Or privileged mode in Docker

### Data Security

**Sensitive Data:**
- Passwords: Argon2id hashed
- Sessions: Encrypted in database
- API keys: Hashed for storage

**Network Data:**
- Stored plaintext (network info is not secret)
- Access controlled by authentication
- Multi-tenant isolation via organizations

---

## Deployment Models

### Model 1: Docker Compose (Default)

```yaml
services:
  daemon:
    network_mode: host    # Access to LAN
    privileged: true      # Network scanning

  server:
    networks:
      - netvisor          # Internal bridge
    ports:
      - "60072:60072"

  postgres:
    networks:
      - netvisor
```

**Use Case:** Simple deployment, everything on one host

---

### Model 2: Multi-VLAN with Multiple Daemons

```
Server (VLAN 1)
   │
   ├─ Daemon 1 (VLAN 1) ──▶ Scans VLAN 1
   │
   ├─ Daemon 2 (VLAN 2) ──▶ Scans VLAN 2
   │
   └─ Daemon 3 (VLAN 3) ──▶ Scans VLAN 3
```

**Use Case:** Complex networks with isolated VLANs

---

### Model 3: Native Installation (Ubuntu)

```
/opt/netvisor/
├── bin/
│   ├── netvisor-server
│   └── netvisor-daemon
└── static/

/etc/netvisor/
├── server.env
└── daemon.env

Systemd Services:
├── netvisor-server.service
└── netvisor-daemon.service
```

**Use Case:** Production deployments, no Docker available

---

### Model 4: Cloud Service (Future)

```
NetVisor Cloud (SaaS)
   │
   ├─ Customer Daemon 1 ──▶ Scans Customer Network 1
   ├─ Customer Daemon 2 ──▶ Scans Customer Network 2
   └─ Customer Daemon N ──▶ Scans Customer Network N
```

**Use Case:** Managed service, no self-hosting

---

## Technology Stack

### Backend

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Language | Rust 1.90+ | Performance, safety, concurrency |
| Web Framework | Axum 0.8 | HTTP server, routing |
| Async Runtime | Tokio | Async I/O, concurrency |
| Database | PostgreSQL 14+ | Relational data storage |
| Database Client | sqlx 0.8 | Type-safe SQL queries |
| Serialization | serde + serde_json | JSON encoding/decoding |
| HTTP Client | reqwest 0.12 | HTTP requests (daemon→server) |
| Authentication | argon2 + tower-sessions | Password hashing, sessions |
| Network Scanning | pnet + custom scanner | Raw sockets, port scanning |
| Docker Client | bollard | Docker API client |

### Frontend

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Framework | SvelteKit | Reactive UI framework |
| Language | TypeScript | Type-safe JavaScript |
| Styling | TailwindCSS | Utility-first CSS |
| Visualization | D3.js (custom) | Network topology rendering |
| State Management | Svelte Stores | Reactive state |
| Real-time Updates | SSE (Server-Sent Events) | Live progress updates |
| HTTP Client | fetch API | REST API communication |

### Infrastructure

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Containerization | Docker | Packaging and deployment |
| Orchestration | Docker Compose | Multi-container management |
| Service Manager | systemd | Native Linux services |
| Reverse Proxy | Nginx/Caddy (optional) | HTTPS termination |
| Database | PostgreSQL | Data persistence |

---

## Performance Characteristics

### Scalability

**Network Scanning:**
- 10-15 concurrent host scans (default)
- ~1-2 seconds per host (depends on open ports)
- Small network (/24, 254 hosts): 5-10 minutes
- Large network (/16, 65k hosts): Several hours (use multiple daemons)

**Database:**
- Handles 10,000+ hosts easily
- Indexes on key columns (id, network_id, etc.)
- PostgreSQL JSONB for flexible schemas

**UI Rendering:**
- Topology: Handles 1000+ nodes
- Real-time updates via SSE (no polling)
- Lazy loading for large lists

### Resource Usage

**Server:**
- CPU: Minimal (mostly I/O bound)
- RAM: 256MB base + ~1MB per 1000 hosts
- Disk: Depends on data (estimate 1GB for 10k hosts)

**Daemon:**
- CPU: High during scans (multi-threaded)
- RAM: 50-200MB (depends on concurrent scans)
- Network: Generates scan traffic (be mindful of rate limits)

---

## Future Architecture Considerations

### Planned Enhancements

1. **Real-time Monitoring**
   - Continuous port monitoring
   - Alert on topology changes
   - Service uptime tracking

2. **Advanced Analytics**
   - Network traffic analysis
   - Service dependency mapping
   - Historical trend analysis

3. **Automation**
   - Auto-remediation workflows
   - Integration with IaC tools
   - API-first architecture

4. **Cloud Service**
   - Multi-tenant SaaS
   - Customer-hosted daemons
   - Centralized management

---

## Architecture Decision Records (ADRs)

### ADR-001: Why Rust for Backend?

**Decision:** Use Rust instead of Go/Python/Node.js

**Rationale:**
- Performance: Near-C performance for network scanning
- Safety: Memory safety without garbage collection
- Concurrency: Tokio async runtime for handling many connections
- Type Safety: Strong type system catches bugs at compile time

**Trade-offs:**
- Longer compile times
- Steeper learning curve
- Smaller ecosystem vs. JavaScript

### ADR-002: Why PostgreSQL?

**Decision:** PostgreSQL instead of MongoDB/MySQL

**Rationale:**
- JSONB support (flexible schemas when needed)
- Strong relational model (hosts→services relationships)
- Excellent performance
- ACID guarantees

**Trade-offs:**
- More complex setup vs. SQLite
- Requires separate service

### ADR-003: Server-Daemon Architecture

**Decision:** Agent-based with multiple daemons

**Rationale:**
- Scan from multiple network segments
- Distributed scanning for large networks
- Scalability (add daemons as needed)
- Network isolation (VLANs, firewalls)

**Trade-offs:**
- More complex than monolithic
- Requires coordination between components

---

## Conclusion

NetVisor's architecture is designed for:

✅ **Flexibility** - Deploy anywhere (Docker, native, cloud)
✅ **Scalability** - Add daemons for larger networks
✅ **Security** - Strong authentication, multi-tenant isolation
✅ **Performance** - Rust backend, efficient scanning
✅ **Maintainability** - Clean layer separation, type safety

The distributed agent-based model allows NetVisor to scale from home networks to enterprise environments while maintaining simplicity for basic deployments.

---

**Architecture Version:** 1.0
**Document Status:** Complete
**Last Review:** November 2025
