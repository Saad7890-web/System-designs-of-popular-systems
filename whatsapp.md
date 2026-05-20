# Distributed Chat System Design

### A WhatsApp-style messaging system supporting billions of users

---

## Overview

This document walks through the design of a large-scale real-time messaging system modeled after WhatsApp. It covers the full design lifecycle — from requirements and core entities, through the high-level architecture, to deep dives on scalability, fault tolerance, and edge cases.

---

## Table of Contents

- [Functional Requirements](#functional-requirements)
- [Non-Functional Requirements](#non-functional-requirements)
- [Core Entities](#core-entities)
- [API Design](#api-design)
- [High-Level Design](#high-level-design)
- [Deep Dives](#deep-dives)
- [Architecture Diagram Summary](#architecture-diagram-summary)

---

## Functional Requirements

**In scope:**

- Users can create group chats with up to 100 participants
- Users can send and receive messages in real time
- Users receive messages sent while offline (retained up to 30 days)
- Users can send and receive media attachments

**Out of scope:** audio/video calling, business integrations, registration/profile management

---

## Non-Functional Requirements

| Requirement              | Target                                      |
| ------------------------ | ------------------------------------------- |
| Message delivery latency | < 500ms                                     |
| Durability               | Messages guaranteed to reach all recipients |
| Scale                    | Billions of users, high throughput          |
| Storage                  | Messages retained only as long as necessary |
| Resilience               | Tolerant of individual component failures   |

---

## Core Entities

| Entity    | Description                                                 |
| --------- | ----------------------------------------------------------- |
| `User`    | A person using the system                                   |
| `Chat`    | A conversation with 2–100 participants                      |
| `Message` | A text or media message within a chat                       |
| `Client`  | A specific device belonging to a user (phone, laptop, etc.) |

---

## API Design

The system uses **WebSocket connections over TLS** for bidirectional, low-latency communication. Clients exchange commands with the server; most outbound server commands require an explicit `ACK` from the client to confirm delivery.

### Client → Server

```
createChat          { participants, name }           → { chatId }
sendMessage         { chatId, message, attachments } → { status, messageId }
modifyChatParticipants { chatId, userId, operation } → SUCCESS | FAILURE
getAttachmentTarget {}                               → { presignedUrl }
getLastSeen         { targetUserId, requestingUserId }
```

### Server → Client

```
chatUpdate          { chatId, participants }          → RECEIVED
newMessage          { chatId, userId, message, attachments } → RECEIVED
updateLastSeen      { targetUserId, reporter, lastSeen }
```

> All server-push commands expect a `RECEIVED` acknowledgement. This ACK drives inbox cleanup and confirms end-to-end delivery.

---

## High-Level Design

### 1. Creating a Chat

A user sends `createChat` to the Chat Server, which writes to two DynamoDB tables:

- **Chat** table — keyed by `chatId`
- **ChatParticipant** table — composite key `(chatId, participantId)`, with a GSI on `(participantId, chatId)` to support both "who's in this chat?" and "what chats is this user in?" queries

For chats near the 100-participant limit, writes are batched to stay within DynamoDB transaction limits.

### 2. Sending & Receiving Messages

The Chat Server maintains an in-memory map of `userId → WebSocket connection`. On `sendMessage`:

1. Look up chat participants via `ChatParticipant` table
2. Write the message to the **Message** table
3. Write an entry to each recipient's **Inbox** table
4. Push `newMessage` to each connected recipient
5. On `ACK`, delete the entry from the Inbox

For **offline users**, messages wait in the Inbox (TTL: 30 days). On reconnect, the server syncs pending Inbox entries to the client and clears them on ACK.

> Write throughput estimate: ~200M DAU × 20 messages/day ≈ **40K messages/second**. With fan-out to Inbox, ~100K writes/second — well within DynamoDB's capacity.

### 3. Media Attachments

Attachments are handled outside the Chat Server to avoid saturating its bandwidth:

1. Client requests a **pre-signed upload URL** via `getAttachmentTarget`
2. Client uploads directly to **blob storage** (e.g. S3)
3. Client sends the resulting URL in `sendMessage` as an opaque reference
4. Recipients fetch attachments directly from blob storage using a pre-signed download URL

Attachments are stored with a 30-day TTL matching message retention.

---

## Deep Dives

### Scaling to Billions of Users

A single Chat Server can't hold connections for hundreds of millions of users. The solution is **Redis Pub/Sub**, sharded by `userId`:

- On connect, the Chat Server subscribes to the Pub/Sub channel for that user
- On message send, the server publishes to the target user's channel
- Any Chat Server subscribed to that channel forwards the message to the live WebSocket

This decouples message routing from any individual server. Redis Pub/Sub is lightweight (no disk persistence, no consumer offsets) and can sustain **100K+ events/second on a single node** — and is horizontally sharded in practice.

**Why not Kafka?** Kafka carries ~50KB overhead per topic. At 1B users, that's 50TB just for topic metadata — impractical.

**Pub/Sub channel granularity:** Channels are per-user (not per-chat) because WhatsApp is dominated by 1:1 conversations. For large chats (e.g. >25 members), an adaptive strategy can switch to per-chat channels to reduce fan-out overhead.

---

### Multiple Devices Per User

Each user may have several active clients. Changes to support this:

- New **Clients** table keyed by `userId` (capped at ~3 per account)
- **Inbox** entries are per-client, not per-user
- On message send, fan out to all clients for each participant
- Pub/Sub channels remain per-user; Chat Servers deliver to all active connections for that user

---

### Handling Failed WebSocket Connections

TCP keepalives can take minutes to detect a dead connection. Two mechanisms address this:

**ACK timeouts:** The server waits up to ~500–2000ms for a client ACK after delivering a message. On repeated failure, it closes the socket and forces a reconnect + Inbox sync.

**Heartbeats:** The server sends a `ping` every 10–30 seconds. If no `pong` is received within ~5 seconds, the connection is closed. This guarantees detection of dead connections within ~15 seconds.

---

### Handling Redis Pub/Sub Failures

Redis Pub/Sub is "at most once" — dropped messages are possible. Durability is handled separately:

- Messages are written to the **Message** and **Inbox** tables _before_ publishing to Pub/Sub
- If Pub/Sub drops a message, it will be delivered when the client reconnects

For faster detection of missed messages on live connections, a **sequence number** approach is used:

1. Each message to a user increments an atomic counter (via Redis `INCR`)
2. The current sequence number is included in server heartbeat pings
3. If the client's local sequence is behind, it immediately requests a sync

This catches missed messages within one heartbeat interval with minimal additional load.

---

### Message Ordering

Rather than enforcing strict ordering (which requires buffering and reordering logic), messages are stamped with a **server-received timestamp** (synchronized via NTP across Chat Servers). Clients sort messages by this timestamp for display.

This means a late-arriving message may occasionally appear above a newer one in the UI — a tradeoff users find acceptable in exchange for low latency.

---

### "Last Seen" Status

A dedicated **LastSeen** table stores the most recent disconnect timestamp per user.

- Written on disconnect (conditional update to avoid races between servers)
- On `getLastSeen` request:
  1. Server reads the LastSeen table → returns last disconnect time
  2. Server forwards the request to the target user's Pub/Sub channel
  3. If the target is online, their Chat Server publishes an `ONLINE` response
- The requesting client merges responses: `ONLINE` wins; otherwise, show the timestamp

This requires only **one record per user** and one **write per disconnect** — far more efficient than heartbeat-driven timestamp updates.

---

## Architecture Diagram Summary

```
Clients (WebSocket / TLS)
        │
        ▼
  L4 Load Balancer
        │
        ▼
  Chat Servers ◄──────► Redis Pub/Sub (sharded by userId)
        │
        ├──► DynamoDB
        │     ├── Chat
        │     ├── ChatParticipant (+ GSI)
        │     ├── Message (TTL: 30d)
        │     ├── Inbox (per-client, TTL: 30d)
        │     ├── Clients
        │     └── LastSeen
        │
        └──► Blob Storage (S3)
               └── Attachments (pre-signed URLs, TTL: 30d)
```

---

## Key Design Decisions

| Decision           | Choice                 | Rationale                                   |
| ------------------ | ---------------------- | ------------------------------------------- |
| Transport          | WebSocket over TLS     | Bidirectional, low-latency                  |
| Load balancer      | L4                     | WebSockets don't need L7 routing            |
| Primary database   | DynamoDB               | Fast key/value, scalable, TTL support       |
| Real-time fan-out  | Redis Pub/Sub          | Lightweight, no persistence overhead        |
| Attachment storage | S3 + pre-signed URLs   | Offloads bandwidth from Chat Servers        |
| Ordering           | Server timestamp (NTP) | Simple, low-latency, acceptable UX tradeoff |
| Message durability | Inbox table + ACKs     | Guarantees delivery independent of Pub/Sub  |
