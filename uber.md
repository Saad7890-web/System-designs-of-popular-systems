# Uber System Design

A deep-dive into designing a ride-sharing platform at scale — covering architecture, data modeling, API design, and key engineering tradeoffs.

---

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Core Entities](#core-entities)
- [API Design](#api-design)
- [High-Level Architecture](#high-level-architecture)
- [Deep Dives](#deep-dives)
- [Key Tradeoffs](#key-tradeoffs)

---

## Overview

Uber connects riders who need transportation with nearby drivers in real time. This document walks through designing the core platform: fare estimation, ride requesting, driver matching, and live ride management — at a scale of millions of concurrent users and drivers.

---

## Requirements

### Functional (In Scope)

| #   | Requirement                                                              |
| --- | ------------------------------------------------------------------------ |
| 1   | Riders can input a start location and destination to get a fare estimate |
| 2   | Riders can request a ride based on the estimated fare                    |
| 3   | Upon request, riders are matched with a nearby, available driver         |
| 4   | Drivers can accept or decline a request and navigate to pickup/drop-off  |

### Functional (Out of Scope)

- Post-trip ratings for riders and drivers
- Scheduled rides
- Multiple ride categories (X, XL, Comfort)

### Non-Functional (In Scope)

- **Low-latency matching** — match within 1 minute or fail gracefully
- **Strong consistency** — no driver assigned to multiple rides simultaneously
- **High throughput** — handle up to 100k concurrent requests from the same region

### Non-Functional (Out of Scope)

- GDPR / data privacy compliance
- CI/CD and zero-downtime deployments
- Monitoring, alerting, and observability

---

## Core Entities

| Entity       | Description                                                                           |
| ------------ | ------------------------------------------------------------------------------------- |
| **Rider**    | A registered user who requests rides. Stores personal info and payment methods.       |
| **Driver**   | A registered driver with vehicle details, preferences, and availability status.       |
| **Fare**     | An estimated fare for a proposed trip, including pickup/destination, price, and ETA.  |
| **Ride**     | A full trip record linking rider, driver, route, status, actual fare, and timestamps. |
| **Location** | Real-time driver coordinates (lat/lng + timestamp), used for matching and tracking.   |

---

## API Design

### Estimate Fare

```
POST /fare
Body: { pickupLocation, destination }
Returns: Fare
```

### Request a Ride

```
POST /rides
Body: { fareId }
Returns: Ride
```

### Update Driver Location

```
POST /drivers/location
Body: { lat, long }
Returns: 200 OK / Error
```

> `driverId` is extracted from the session/JWT — never passed in the body.

### Accept or Decline a Ride

```
PATCH /rides/:rideId
Body: { decision: "accept" | "decline" }
Returns: Ride
```

> **Security note:** Never trust client-supplied data for `userId`, `fareEstimate`, or timestamps. These must come from the server or session context.

---

## High-Level Architecture

### 1. Fare Estimation

```
Rider Client → API Gateway → Ride Service → Third-Party Mapping API
                                         ↓
                                      Database (Fare)
```

The Ride Service calls a mapping API (e.g., Google Maps) to compute distance and travel time, applies pricing logic, persists a `Fare` record, and returns it to the client.

### 2. Ride Request

When a rider confirms, the Ride Service creates a `Ride` record with status `requested` and triggers the matching flow.

### 3. Driver Matching

```
Driver Client → Location Service → Redis (Geospatial)
                                        ↑
Ride Service → Ride Matching Service ───┘
```

- Drivers continuously push location updates to the **Location Service**
- Locations are stored in **Redis** using `GEOADD` for O(log n) proximity queries
- The **Ride Matching Service** queries Redis with `GEOSEARCH` to rank nearby available drivers

### 4. Driver Accept / Decline

```
Ride Matching Service → Notification Service → APNs / FCM → Driver Client
                                                                  ↓
                                                           PATCH /rides/:rideId
                                                                  ↓
                                                           Ride Service (update status)
```

If a driver declines or doesn't respond, the next driver in the ranked list is notified.

---

## Deep Dives

### 1. Handling High-Frequency Location Updates

**Problem:** ~10M drivers sending updates every 5 seconds = ~2M writes/second. Postgres/DynamoDB can't handle this cost-effectively, and proximity queries on raw lat/lng are extremely slow.

**Solution: Redis Geospatial**

- Use `GEOADD` to store driver positions — each write overwrites the previous, so the store always holds the latest position
- Use `GEOSEARCH` for fast radius-based proximity queries (backed by geohashing into a sorted set)
- Run a periodic cleanup job using a companion timestamp sorted set to expire stale drivers (offline > 30s)
- Mitigate Redis data loss risk with AOF persistence and Redis Sentinel for automatic failover

> Recovery is fast: since drivers update every 5 seconds, the geo-state fully rebuilds within seconds of a failure.

---

### 2. Adaptive Location Update Frequency

**Problem:** Sending updates on a fixed 5-second interval wastes bandwidth and server resources when drivers are stationary.

**Solution:** On-device sensors determine the optimal update interval dynamically:

- Stationary or slow-moving → less frequent updates
- Fast-moving or changing direction → more frequent updates

This reduces unnecessary writes while maintaining location accuracy where it matters.

---

### 3. Preventing Double-Assignment of Drivers

**Problem:** Multiple instances of the Ride Matching Service could simultaneously send a ride request to the same driver.

**Solution: Redis Distributed Lock with TTL**

When a ride request is sent to a driver:

1. The Ride Matching Service acquires a Redis lock keyed on `driverId` with a 10-second TTL
2. No other service instance can claim that driver while the lock is held
3. If the driver accepts → the Ride record is updated and the lock is released
4. If the driver ignores → the lock expires automatically, the driver becomes available again

This guarantees mutual exclusion without relying on fragile in-memory state or manual lock cleanup.

---

### 4. Handling Dropped Requests During Peak Load

**Problem:** A sudden traffic spike (e.g., a concert ending) can overwhelm the Ride Matching Service, dropping requests. A service crash also loses any in-flight work.

**Solution: Message Queue with Geographic Partitioning**

- Ride requests are enqueued in Kafka, partitioned by geographic region
- Ride Matching Service consumers commit offsets only after a successful match
- If a service instance crashes mid-processing, Kafka replays the message to another instance
- Horizontal auto-scaling adds consumers when queue depth grows

---

### 5. Driver Timeout and Retry

**Problem:** A driver who doesn't respond should not block a rider indefinitely.

**Solution: Durable Execution with Temporal**

The matching flow is modeled as a Temporal workflow:

```
Send request to Driver #1
  → Wait up to 10 seconds
  → Accepted? → Done
  → Declined / Timeout? → Move to Driver #2
  → Repeat until matched or exhausted
```

Because Temporal persists workflow state externally, the process survives service crashes and restarts without losing progress — no manual delay queues or cron jobs needed.

---

### 6. Geographic Sharding for Global Scale

**Problem:** A single-region deployment introduces latency for distant users and creates a single point of failure.

**Solution:** Shard all services and data (Ride Service, Kafka, Redis, database) by geographic region. Most ride operations (matching, location updates, fare lookups) are inherently local — they never need to cross region boundaries. Cross-shard queries only occur during proximity searches near region borders, handled by targeted scatter-gather.

---

## Key Tradeoffs

| Decision                   | Chosen Approach                | Alternative Considered |
| -------------------------- | ------------------------------ | ---------------------- |
| Location store             | Redis (in-memory geospatial)   | PostgreSQL + PostGIS   |
| Consistency in matching    | Redis distributed lock (TTL)   | DB-level row locking   |
| Durability of driver state | Redis AOF + Sentinel           | Batch writes to DB     |
| Fault-tolerant matching    | Temporal durable workflows     | SQS delay queues       |
| Scaling strategy           | Geographic horizontal sharding | Vertical scaling       |
