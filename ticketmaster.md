# Ticketmaster System Design

A reference guide for designing a scalable ticket booking platform like Ticketmaster — covering requirements, core entities, APIs, high-level design, and deep dives.

---

## Table of Contents

- [Understanding the Problem](#understanding-the-problem)
- [Requirements](#requirements)
  - [Functional Requirements](#functional-requirements)
  - [Non-Functional Requirements](#non-functional-requirements)
- [Core Entities](#core-entities)
- [API Design](#api-design)
- [High-Level Design](#high-level-design)
  - [1. View Events](#1-users-should-be-able-to-view-events)
  - [2. Search Events](#2-users-should-be-able-to-search-for-events)
  - [3. Book Tickets](#3-users-should-be-able-to-book-tickets-to-events)
- [Deep Dives](#deep-dives)
  - [1. Reserving Tickets (Better Booking UX)](#1-how-do-we-improve-the-booking-experience-by-reserving-tickets)
  - [2. Scaling the View API](#2-how-is-the-view-api-going-to-scale-to-support-10s-of-millions-of-concurrent-requests)
  - [3. Real-Time Seat Map Updates](#3-how-will-the-system-ensure-a-good-user-experience-during-high-demand-events)
  - [4. Improving Search Latency](#4-how-can-you-improve-search-to-ensure-we-meet-our-low-latency-requirements)
  - [5. Caching Repeated Search Queries](#5-how-can-you-speed-up-frequently-repeated-search-queries)

---

## Understanding the Problem

**Ticketmaster** is an online platform that allows users to purchase tickets for concerts, sports events, theater, and other live entertainment.

---

## Requirements

### Functional Requirements

**In scope (top 3):**

1. Users should be able to **view events**
2. Users should be able to **search for events**
3. Users should be able to **book tickets** to events

**Out of scope (below the line):**

- Users should be able to view their booked events
- Admins or event coordinators should be able to add events
- Popular events should have dynamic pricing

---

### Non-Functional Requirements

**In scope:**

- The system should prioritize **availability** for searching & viewing events, but **consistency** for booking events (no double booking)
- The system should be **scalable** and able to handle high throughput — e.g., 10 million users for one event
- The system should have **low latency search** (< 500ms)
- The system is **read-heavy** and must support high read throughput (~100:1 read/write ratio)

**Out of scope (below the line):**

- GDPR / user data protection
- Fault tolerance
- Secure payment transactions
- CI/CD pipelines and testing
- Regular backups

---

## Core Entities

| Entity        | Description                                                                                                                                                                                                              |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Event**     | Stores event details: date, description, type, and the performer/team involved. Central point of information for each unique event.                                                                                      |
| **User**      | Represents the individual interacting with the system.                                                                                                                                                                   |
| **Performer** | Represents the individual or group performing. Includes name, description, and profile links. (General enough to cover artists, teams, companies, etc.)                                                                  |
| **Venue**     | Physical location of an event. Includes address, capacity, and a seat map (e.g., JSON structure defining sections, rows, seat numbers, and coordinates for UI rendering).                                                |
| **Ticket**    | Individual ticket for an event. Includes event ID, seat details (section, row, seat number), price, and status (`available` / `sold`). Created per seat when an event is added, using the venue's seat map.              |
| **Booking**   | Records a user's ticket purchase. Includes user ID, list of ticket IDs, total price, and booking status (`in-progress` / `confirmed`). A separate entity from Ticket to group multi-ticket orders under one transaction. |

---

## API Design

### View Event

```
GET /events/:eventId
→ Event & Venue & Performer & Ticket[]
```

Returns event details, venue info, performer data, and all tickets (used to render the interactive seat map on the client).

---

### Search Events

```
GET /events/search?keyword={keyword}&start={start_date}&end={end_date}&pageSize={page_size}&page={page_number}
→ Event[]
```

Returns a paginated list of events matching the search parameters.

---

### Book Tickets

```
POST /bookings/:eventId
Body: { "ticketIds": string[], "paymentDetails": ... }
→ bookingId
```

Initiates a ticket booking. Evolves into a two-step flow (reserve → confirm) in the deep dives below.

---

## High-Level Design

### 1. Users Should Be Able to View Events

**Components:**

- **Client** — Web or mobile app. All requests route through the API Gateway.
- **API Gateway** — Entry point for all client requests. Handles routing, authentication, rate limiting, and logging.
- **Event Service** — Fetches event, venue, and performer data and returns it to the client.
- **Events DB** — Stores `Events`, `Performers`, and `Venues` tables.

**Request flow:**

1. Client sends `GET /events/:eventId`
2. API Gateway routes to Event Service
3. Event Service queries Events DB and returns the result

---

### 2. Users Should Be Able to Search for Events

**Components added:**

- **Search Service** — Accepts search queries with keyword/date/location parameters, queries the Events DB, and returns matching events.

**Request flow:**

1. Client sends `GET /events/search?...`
2. API Gateway routes to Search Service
3. Search Service queries Events DB and returns matching events

> ⚠️ This naive approach uses `LIKE '%keyword%'` queries, which require full table scans and won't scale. See [Deep Dive #4](#4-how-can-you-improve-search-to-ensure-we-meet-our-low-latency-requirements) for improvements.

---

### 3. Users Should Be Able to Book Tickets to Events

To prevent double bookings, we need a database with **ACID transaction support**. We use **PostgreSQL** with row-level locking or Optimistic Concurrency Control (OCC).

**New components:**

- **Tickets table** — Stores ticket details: event ID, seat info, price, status, and a `bookingId` FK.
- **Bookings table** — Stores booking records: user ID, ticket IDs, total price, and status.
- **Booking Service** — Handles the ticket booking process, interfaces with Stripe, and updates ticket/booking status.
- **Payment Processor (Stripe)** — External service for payment transactions. Notifies the Booking Service via webhook on payment success.

**Request flow:**

1. User selects a seat and submits payment details
2. `POST /bookings` is sent with selected ticket IDs
3. Booking Service opens a transaction to:
   - Check ticket availability
   - Mark ticket as `booked`
   - Create a new `Booking` record
4. On success, returns a booking confirmation; on failure (ticket already taken), returns an error

> ⚠️ With this design, users can complete the entire payment form only to find out the ticket was taken. See [Deep Dive #1](#1-how-do-we-improve-the-booking-experience-by-reserving-tickets) for the fix.

> **Note on database sharing:** Multiple services share the same database here by design. The data is tightly coupled (bookings need tickets need events), ACID transactions are required, and splitting databases would add complexity without real benefit.

---

## Deep Dives

### 1. How Do We Improve the Booking Experience by Reserving Tickets?

**Problem:** Users fill out payment forms only to discover the ticket was taken, causing a poor experience.

**Solution: Distributed Lock with TTL (Redis)**

Implement a two-step booking flow:

1. **Reserve** — When a user selects a seat, lock that ticket in Redis with a **10-minute TTL**. Create a `Booking` record with status `in-progress`. Return `bookingId` and redirect to the payment page.
2. **Confirm** — User fills out payment details. Client uses Stripe.js to tokenize card data (server never sees raw card numbers — PCI compliant). Client sends the payment token + `bookingId` to the server, which creates a Stripe `PaymentIntent`.
3. **Webhook** — On successful payment, Stripe fires a webhook. The handler retrieves the `bookingId` from Stripe metadata, then atomically updates the Ticket status to `sold` and the Booking status to `confirmed`.
   - The webhook handler must be **idempotent** — Stripe can retry on failure. Use `bookingId` as an idempotency key and check current booking status before updating.

**If the user abandons checkout:** The Redis TTL expires after 10 minutes and the ticket is automatically released.

---

### 2. How Is the View API Going to Scale to Support 10s of Millions of Concurrent Requests?

**Problem:** When tickets go on sale, thousands of users simultaneously refresh the same event page. This extreme read load must be handled without overloading the database.

**Solution: Caching + Horizontal Scaling**

- **CDN / Edge Caching** — Cache static event assets (images, descriptions) at the edge.
- **Application-Level Cache (Redis)** — Cache event details, venue info, and seat maps. These change infrequently and are read constantly.
- **Horizontal Scaling** — Stateless Event Service instances behind a load balancer can be scaled out to handle spikes.
- **Read Replicas** — Offload read traffic from the primary database.

> **Pattern: Scaling Reads** — Aggressive caching of event details, venue information, and seating charts is essential for surviving high-traffic on-sale windows.

---

### 3. How Will the System Ensure a Good User Experience During High-Demand Events?

**Problem:** During popular on-sales, the seat map goes stale quickly. Users click on seats only to find they're already taken — a frustrating loop.

**Solution: Virtual Waiting Room**

Rather than pushing real-time updates to millions of clients (which is technically complex and expensive), introduce a **virtual waiting room**:

- Users are queued before accessing the booking page
- Only a controlled number of users are admitted at a time
- This naturally limits concurrency on the seat map and reduces stale-seat frustration

> **Note:** This is an example of solving a business problem without unnecessary technical complexity. The best solution isn't always the most technically sophisticated one.

> **Pattern: Real-time Updates** — For use cases where real-time push _is_ necessary (e.g., live score updates), consider Server-Sent Events (SSE), WebSockets, or long polling depending on directionality and frequency requirements.

---

### 4. How Can You Improve Search to Ensure We Meet Our Low Latency Requirements?

**Problem:** Wildcard `LIKE` queries require full table scans and are too slow at scale:

```sql
-- Slow — full table scan
SELECT * FROM Events
WHERE name LIKE '%Taylor%'
   OR description LIKE '%Taylor%';
```

**Solution: Dedicated Search Infrastructure**

Replace the naive DB query with a purpose-built search engine:

- **Elasticsearch / OpenSearch** — Index event data (name, description, performer, location, date). Supports full-text search, fuzzy matching, and filters with sub-100ms latency.
- **Sync strategy** — Use a CDC (Change Data Capture) pipeline or event-driven updates to keep the search index in sync with the primary DB.
- **Separate read path** — Search queries go directly to the search cluster; writes still go to PostgreSQL as the source of truth.

> **Target:** < 500ms p99 search latency, even as event count grows into the millions.

---

### 5. How Can You Speed Up Frequently Repeated Search Queries?

**Problem:** Popular searches (e.g., "Taylor Swift", "Lakers", "New York") are issued by thousands of users simultaneously, creating redundant load on the search cluster.

**Solution: Query Result Caching (Redis)**

- Cache search results by query string (normalized) in Redis with a short TTL (e.g., 1–5 minutes)
- On cache hit, return results instantly without hitting Elasticsearch
- On cache miss, query Elasticsearch, return results, and populate the cache

**Considerations:**

- TTL tuning: shorter TTL = fresher results but more cache misses; longer = better hit rate but slightly stale
- Cache invalidation: when a new event matching a popular query is added, optionally invalidate or proactively update relevant cache entries
- For geolocation-based queries, include location in the cache key

---

## Final Architecture Summary

```
Client
  └── API Gateway
        ├── Event Service       → Events DB (PostgreSQL)
        ├── Search Service      → Elasticsearch + Redis (query cache)
        └── Booking Service     → Events DB (PostgreSQL)
                                → Redis (distributed lock / ticket reservation)
                                → Stripe (payment processing via webhook)

Cross-cutting:
  - Redis: distributed locks (booking) + query cache (search) + app cache (event details)
  - Read replicas on Events DB for high read throughput
  - Horizontal scaling on all stateless services
  - CDN for static assets
```

---

_This design covers the core functional requirements (view, search, book) and addresses scalability, consistency, latency, and user experience through targeted deep dives._
