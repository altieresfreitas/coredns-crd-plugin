# CoreDNS CRD Plugin - Agent Guide & Architecture Context

## Overview

`coredns-crd-plugin` (`k8s_crd`) is a CoreDNS plugin developed for [k8gb](https://k8gb.io) (Kubernetes Global Balancer). It acts as an authoritative DNS server backed by Kubernetes `DNSEndpoint` and `ZoneDelegation` Custom Resources (CRDs).

---

## Plugin Pipeline Architecture

Requests processed by `k8s_crd` flow through an internal service container pipeline (`service.Container` in `service/container.go`):

```
DNS Query (Client)
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│                       k8s_crd Plugin                        │
│                                                             │
│  1. Gateway (service/gateway)                               │
│     • Matches zone against DNSEndpoint CRDs in K8s          │
│     • Hit: Writes A/TXT records into container writer       │
│     • Miss: Writes SOA + NXDOMAIN into container writer     │
│                                                             │
│  2. WRR (service/wrr) [Optional]                            │
│     • Applies Weight-Round-Robin / Geo shuffling on answers │
│                                                             │
│  3. Reactive (service/reactive) [Optional]                  │
│     • Triggers only on local misses (NXDOMAIN) for Type A   │
│     • Queries peer CoreDNS servers in sibling clusters      │
│     • Synthesizes and caches aggregated A records           │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
DNS Response (Client)
```

### Container Writer (`service/writer.go`)
- `containerResponseWriter` wraps `dns.ResponseWriter`.
- `WriteMsg()` captures the latest response message `m` without sending it directly to the socket, allowing subsequent handlers in the pipeline to inspect or replace it.
- `Request()` returns the original `req *dns.Msg` received from the network, preserving sections like `Extra` (EDNS0 `OPT` records) across sub-services.

---

## Reactive Cross-Cluster Resolution & Loop Guard

### Context
In multi-cluster k8gb deployments, multiple independent clusters are authoritative for the same delegated zone. When an application is deployed in Cluster B but a client's recursive resolver queries Cluster A, Cluster A initially produces a local miss (`NXDOMAIN`).

With `reactive on`:
1. Cluster A queries the peer CoreDNS instances discovered from the `ZoneDelegation` CRD status (`dnsServers`).
2. The query asks for `<host>` and stops on the first responding peer (early return), since each authoritative cluster serves the aggregated GSLB target set.
3. The result is cached with positive (`reactivecachettl`) and negative (`reactivenegttl`) TTLs.

### Loop Prevention Strategy (RFC 6891 EDNS0 Private Option)
If a domain genuinely does not exist in any cluster, Cluster A queries Cluster B, and Cluster B must **not** re-query Cluster A (preventing ping-pong loops).

- **Implementation:**
  - Probes sent by `reactive.Querier` attach an **EDNS0 Private Option** (RFC 6891 §9.1):
    - `OptionCode: 65001` (in the designated Private / Local Use range `65001-65534` / `0xFDE9-0xFFFE`).
    - `OptionData: "k8gb-reactive"`.
  - Probes keep `RecursionDesired = false` and leave all DNS header flags untouched.
  - When `reactive.ServeDNS` receives a query, it checks whether the original request carries this EDNS0 option via `isReactiveProbe()`. If present, it immediately short-circuits without initiating peer queries.

### ⚠️ Critical Rule for Agents & Contributors
> **NEVER use standard DNS header flags (such as `CheckingDisabled`/CD, `AD`, `DO`, `RD`, `RA`) for loop detection or internal IPC.**
> 
> *Reason:* `CheckingDisabled` (CD bit) is a standard DNSSEC flag (RFC 4035 §3.2.2). Legitimate DNSSEC-validating resolvers (e.g. `dig +cd`, `unbound`, `bind`, `systemd-resolved`) send queries with `CD=1`. Overloading `CD` causes legitimate queries to be mistaken for internal probes, breaking reactive resolution.

---

## Key Files & Directories

| Path | Purpose |
| :--- | :--- |
| `setup.go` | Caddy / CoreDNS plugin directive parsing (`k8s_crd`). |
| `k8scrd.go` | Plugin entry point implementing `plugin.Handler`. |
| `service/container.go` | Orchestrates the sequential pipeline of sub-services. |
| `service/writer.go` | `containerResponseWriter` intercepting DNS messages and preserving original requests. |
| `service/gateway/` | Authoritative lookups on local Kubernetes `DNSEndpoint` objects. |
| `service/wrr/` | Weighted Round Robin and GeoIP shuffling. |
| `service/reactive/reactive.go` | Reactive service handler, singleflight coalescing, and caching. |
| `service/reactive/querier.go` | Queries peer CoreDNS servers. |
| `service/reactive/probe.go` | EDNS0 loop probe helper functions (`setReactiveProbe`, `isReactiveProbe`). |
| `service/reactive/peers.go` | Dynamic informer watching `ZoneDelegation` CRDs. |
| `service/reactive/cache.go` | TTL-based concurrent cache for resolved targets. |

---

## Development & Verification

### Running Unit Tests
```bash
go test ./...
```

### Building the Binary
```bash
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o coredns cmd/coredns.go
```
