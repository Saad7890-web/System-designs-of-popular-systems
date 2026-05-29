# 🚦 Distributed Rate Limiter — System Design

A production-grade, distributed rate limiting system designed to handle **1 million requests/second** across **100 million daily active users**.

---

## Overview

This rate limiter controls how many requests a client can make within a specific timeframe, acting as a traffic controller for your API. Excess requests are rejected with `HTTP 429 Too Many Requests`, protecting backend services from abuse and ensuring fair usage across all clients.

---

## Functional Requirements

| Requirement           | Details                                                        |
| --------------------- | -------------------------------------------------------------- |
| Client identification | User ID, IP address, or API key                                |
| Limit enforcement     | Configurable rules (e.g., 100 requests/minute/user)            |
| Rejection response    | HTTP 429 with helpful headers (`X-RateLimit-*`, `Retry-After`) |

**Out of scope:** analytics/querying on rate limit data, long-term persistence of rate limit state.

---

## Non-Functional Requirements

| Requirement      | Target                                                |
| ---------------- | ----------------------------------------------------- |
| Latency overhead | < 10ms per request check                              |
| Availability     | High availability; eventual consistency is acceptable |
| Throughput       | 1M requests/second                                    |
| Scale            | 100M daily active users                               |

**Out of scope:** strong consistency guarantees across all nodes.

---

## Architecture

### Placement: API Gateway

The rate limiter runs at the **API Gateway layer** — every request is evaluated before reaching application servers. Blocked requests never touch the backend.

```
Client → [API Gateway + Rate Limiter] → Application Servers
                     ↕
               Redis Cluster
```

This is the most common production pattern. It provides centralized control without adding extra network hops inside the system.

### Client Identification

The rate limiter reads client identity directly from the HTTP request:

| Identifier | Source                       | Use Case              |
| ---------- | ---------------------------- | --------------------- |
| User ID    | `Authorization` header (JWT) | Authenticated APIs    |
| IP Address | `X-Forwarded-For` header     | Public/anonymous APIs |
| API Key    | `X-API-Key` header           | Developer APIs        |

In practice, **multiple rules are layered** and the most restrictive one wins:

- Per-user limit: `Alice can make 1,000 requests/hour`
- Per-IP limit: `This IP can make 100 requests/minute`
- Global limit: `API can handle 50,000 requests/second total`
- Endpoint-specific: `Search: 10 req/min — Profile updates: 100 req/min`

---

## Rate Limiting Algorithm: Token Bucket

Four common algorithms were evaluated:

| Algorithm              | Accuracy               | Memory  | Complexity | Burst Handling |
| ---------------------- | ---------------------- | ------- | ---------- | -------------- |
| Fixed Window Counter   | Low (boundary spikes)  | Low     | Low        | Poor           |
| Sliding Window Log     | High                   | High    | Medium     | Good           |
| Sliding Window Counter | Medium (approximation) | Low     | Medium     | Good           |
| **Token Bucket**       | **High**               | **Low** | **Low**    | **Excellent**  |

**Token Bucket** was selected. Each client has a bucket with a max capacity (burst limit) that refills at a steady rate. Requests consume tokens; when empty, requests are rejected.

> Used in production by companies like **Stripe** because it naturally handles bursty API traffic.

### State Storage: Redis

Token bucket state (`tokens`, `last_refill_time`) is stored in **Redis**, shared across all gateway instances.

**Request flow:**

1. Request arrives at a gateway for user `alice`
2. Gateway fetches Alice's bucket: `HMGET alice:bucket tokens last_refill`
3. Calculate tokens to add based on elapsed time since last refill
4. Atomically update bucket state:
   ```
   MULTI
   HSET alice:bucket tokens <new_count>
   HSET alice:bucket last_refill <timestamp>
   EXPIRE alice:bucket 3600
   EXEC
   ```
5. Allow if tokens ≥ 1; reject with `HTTP 429` otherwise

> **Race condition note:** The read-calculate-write sequence is wrapped in a **Lua script** for full atomicity, preventing double-spend when concurrent requests arrive for the same user.

---

## HTTP 429 Response Format

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640995200
Retry-After: 60
Content-Type: application/json

{
  "error": "Rate limit exceeded",
  "message": "You have exceeded 100 requests/minute. Retry in 60 seconds."
}
```

---

## Scaling to 1M Requests/Second

A single Redis instance handles ~50,000–100,000 rate limit checks/second. At 1M req/s, Redis must be sharded.

### Redis Sharding via Consistent Hashing

Client identifiers are hashed to determine which Redis shard owns their bucket state. This ensures:

- All requests from one client always hit the same shard
- Load is evenly distributed across shards

With **10 Redis shards** each handling ~100K ops/s, the system reaches 1M req/s capacity.

In production, **Redis Cluster** automates this via 16,384 hash slots distributed across nodes — no custom routing logic required in the gateway.

```
Gateway → hash(alice_user_id) % num_shards → Redis Shard N
```

---

## High Availability & Fault Tolerance

Each Redis shard is a critical component. Failure mode options:

| Mode            | Behavior                           | When to Use                                     |
| --------------- | ---------------------------------- | ----------------------------------------------- |
| **Fail-closed** | Reject all requests (HTTP 503/429) | When uncontrolled access is worse than downtime |
| Fail-open       | Allow all requests through         | When availability > protection                  |

**This system uses fail-closed.** During traffic spikes (when Redis failures are most likely), failing open risks cascading backend overload — brief request rejection is preferable to complete platform failure.

### Redis Master-Replica Replication

Each shard runs with one or more read replicas. On master failure, Redis Cluster automatically promotes a replica. This provides:

- Continued operation during node failures
- No manual intervention required
- Minimal replication lag

---

## Latency Optimizations

| Optimization                   | Impact | Notes                                               |
| ------------------------------ | ------ | --------------------------------------------------- |
| **Connection pooling**         | High   | Eliminates TCP handshake overhead per request       |
| **Geographic distribution**    | High   | Deploy Redis close to users (region-local clusters) |
| Redis pipelining / Lua scripts | Medium | Reduces round trips; adds some complexity           |
| Local caching                  | Risky  | Stale state can cause incorrect enforcement         |

Target: **< 10ms overhead per request**, typically achievable with sub-millisecond Redis operations + pooled connections in the same region.

---

## Handling Hot Keys

Hot key scenarios arise from both abuse and legitimate high-volume clients.

### Legitimate High-Volume Clients

- **Client-side rate limiting:** Encourage SDK-level throttling to smooth bursts
- **Request batching:** Batch multiple operations into single requests
- **Premium tiers:** Higher limits with dedicated infrastructure for power users

### Abusive Traffic

- **Automatic blocking:** Temporarily blocklist clients that hit limits repeatedly (e.g., 10 violations/minute)
- **DDoS protection:** Upstream services like Cloudflare or AWS Shield absorb attacks before they reach the rate limiter

---

## Dynamic Rule Configuration

Rate limits need to change without code deploys. Two approaches:

### Poll-Based (Recommended for most systems)

Gateways poll a configuration database every ~30 seconds and cache rules locally.

- ✅ Simple to implement
- ✅ Works for 99% of operational changes
- ⚠️ Up to 30-second propagation delay

### Push-Based (ZooKeeper / Redis Pub-Sub)

Configuration changes are pushed to all gateways in real-time via ZooKeeper or Redis pub/sub.

- ✅ Near-instant propagation (seconds)
- ⚠️ Significant operational complexity
- ✅ Justified for security incidents or high-frequency trading environments

---

## System Interface

```typescript
isRequestAllowed(
  clientId: string,   // user ID, IP address, or API key
  ruleId: string      // identifies the applicable rate limit rule
) -> {
  passes: boolean,
  remaining: number,
  resetTime: timestamp
}
```

Response values map directly to `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers.

---

## Core Entities

| Entity       | Description                                                                    |
| ------------ | ------------------------------------------------------------------------------ |
| **Rules**    | Policies defining limits per client type, endpoint, and time window            |
| **Clients**  | Entities being rate limited (users, IPs, API keys) with associated usage state |
| **Requests** | Incoming HTTP requests carrying client identity, endpoint, and timestamp       |

---

## Key Design Decisions Summary

| Decision       | Choice             | Rationale                                         |
| -------------- | ------------------ | ------------------------------------------------- |
| Placement      | API Gateway        | Centralized; blocks before backend; no extra hops |
| Algorithm      | Token Bucket       | Low memory, burst-friendly, production-proven     |
| State store    | Redis Cluster      | Sub-ms latency, atomic ops, automatic sharding    |
| Failure mode   | Fail-closed        | Prevents cascading failures during traffic spikes |
| Atomicity      | Lua scripting      | Eliminates read-modify-write race conditions      |
| Scaling        | Consistent hashing | Even shard distribution, client affinity          |
| Config updates | Poll-based         | Sufficient for most use cases; simpler to operate |
