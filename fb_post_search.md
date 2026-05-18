# 📰 Facebook News Feed — System Design

A deep-dive into designing a scalable, highly available social media news feed at the scale of 2 billion users. This document covers requirements, API design, data modeling, and solutions to the classic fan-out problem.

---

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Core Entities](#core-entities)
- [API Design](#api-design)
- [High-Level Design](#high-level-design)
- [Deep Dives](#deep-dives)
- [Key Tradeoffs](#key-tradeoffs)

---

## Overview

Facebook's News Feed shows users a reverse-chronological stream of posts from the people they follow. This design uses **uni-directional follow relationships** (as opposed to bi-directional friendships) and focuses on the core challenges of **fan-out**, **hot key reads**, and **eventual consistency at massive scale**.

---

## Requirements

### Functional Requirements

| #   | Requirement                                                      |
| --- | ---------------------------------------------------------------- |
| 1   | Users can create posts                                           |
| 2   | Users can follow other users                                     |
| 3   | Users can view a feed of posts from followed users, newest first |
| 4   | Users can page through their feed                                |

> **Out of scope:** likes, comments, post privacy/visibility controls.

### Non-Functional Requirements

| Requirement   | Target                                                     |
| ------------- | ---------------------------------------------------------- |
| Availability  | Highly available; prioritize availability over consistency |
| Consistency   | Eventual — up to 1 minute of post staleness is acceptable  |
| Latency       | < 500ms for posting and feed retrieval                     |
| Scale         | 2 billion users                                            |
| Follow limits | Unlimited followers and followees                          |

---

## Core Entities

```
User     — A registered user in the system
Follow   — A uni-directional link between two users (follower → followed)
Post     — Content created by a user, surfaced in followers' feeds
```

---

## API Design

### Create a Post

```http
POST /posts
Content-Type: application/json

{
  "content": { }
}
```

```json
// 200 OK
{ "postId": "abc123" }
```

### Follow a User

```http
PUT /users/{id}/follow
```

```json
// 200 OK
```

> Uses `PUT` for idempotency — clicking "Follow" twice won't error. Unfollow would use `DELETE`.

### Get Feed

```http
GET /feed?pageSize={size}&cursor={timestamp}
```

```json
// 200 OK
{
  "items": [
    /* Post[] */
  ],
  "nextCursor": "2024-01-15T10:30:00Z"
}
```

> Cursor-based pagination using a timestamp. Each page returns `N` posts older than the cursor, supporting infinite scroll.

---

## High-Level Design

### 1. Post Creation

A horizontally scaled **Post Service** sits behind an API gateway/load balancer. Each host is stateless and writes to a **DynamoDB** table. Scaling is as simple as adding more hosts.

```
Client → API Gateway → Post Service (stateless, N hosts) → DynamoDB (Posts table)
```

**Posts Table Schema**

```
PK: postId
Attributes: creatorId, content, createdAt
GSI: creatorId (PK) + createdAt (SK)  ← enables fetching posts by user in order
```

---

### 2. Follow Graph

The Follow relationship is stored in a DynamoDB table with a GSI for reverse lookups:

**Follow Table Schema**

```
PK: userFollowing
SK: userFollowed

GSI:
  PK: userFollowed
  SK: userFollowing
```

This supports three efficient query patterns:

- **Is A following B?** → Query by `(userFollowing, userFollowed)`
- **Who does A follow?** → Query by `userFollowing`
- **Who follows A?** → Query GSI by `userFollowed`

---

### 3. Feed Generation (Naive)

A dedicated **Feed Service** assembles feeds on read:

```
1. Query Follow table → get all users that requester follows
2. Query Post GSI     → get posts from each followed user (filtered by cursor)
3. Merge & sort       → return top N posts by timestamp
```

> ⚠️ **Known limitation:** This fans out to potentially thousands of DB queries for users following many people. See [Deep Dives](#deep-dives) for solutions.

---

## Deep Dives

### 1. Fan-Out on Read — Users Following Many People

**Problem:** A user following 10,000 accounts triggers 10,000 Post table queries per feed load.

**Solution: Fan-Out on Write with a Precomputed Feed Table**

Rather than assembling the feed at read time, precompute it at write time. When a post is created, insert its ID into the feed of every follower.

**PrecomputedFeed Table Schema**

```
PK: userId
Value: [postId1, postId2, ..., postId200]  ← sorted reverse-chronological, capped at ~200
```

- Feed reads become a **single key lookup** instead of N queries
- Storage cost: `~2KB per user × 2B users = ~4TB` — very reasonable
- For users who scroll past 200 posts, fall back to the naive query approach

> **Product lever:** Setting a max follow count (Facebook caps friends at 5,000) avoids the worst fan-out cases entirely and is common in production systems.

---

### 2. Fan-Out on Write — Users With Many Followers

**Problem:** A celebrity with 10M followers creating a post requires 10M feed writes.

**Solution: Async Queue-Based Fan-Out**

```
Post Created → Message Queue → Fan-Out Workers (parallel) → PrecomputedFeed Table
```

- Workers process follower batches in parallel
- Our non-functional requirements allow up to **1 minute of staleness**, giving plenty of time for async fan-out
- For extreme cases (mega-celebrities), a hybrid approach works: precompute for normal users, query on read for accounts exceeding a follower threshold

---

### 3. Hot Key Reads — Viral Posts

**Problem:** DynamoDB requires even load distribution across its keyspace. A viral post receiving 500 req/s while others get 0 creates a "hot key" and can overwhelm a single partition.

**Solution: Caching Layer**

Add a **distributed cache** (e.g. Redis/Elasticache) in front of the Posts table:

```
Feed Service → Cache (Redis) → Post Table (on cache miss)
```

- Posts are read-heavy and rarely edited — ideal for caching
- A short TTL (e.g. 60s) balances freshness with cache hit rate
- For truly viral content, cache hit rates will be extremely high

---

## Key Tradeoffs

| Decision          | Chosen Approach                | Alternative        | Reason                                 |
| ----------------- | ------------------------------ | ------------------ | -------------------------------------- |
| Consistency model | Eventual (1 min staleness)     | Strong consistency | Enables async fan-out, simpler scaling |
| Graph storage     | DynamoDB + GSI                 | Neo4j or graph DB  | No traversals needed; simpler ops      |
| Feed assembly     | Precomputed (fan-out on write) | Computed on read   | Lower read latency at scale            |
| Pagination        | Cursor (timestamp)             | Offset-based       | Stable under concurrent inserts        |
| Follow limits     | Potentially capped             | Unlimited          | Bounds worst-case fan-out              |

---

## Architecture Summary

```
                        ┌─────────────────────────────────────────┐
                        │              API Gateway                │
                        └──────────┬────────────┬────────────────┘
                                   │            │
                          ┌────────▼──┐   ┌─────▼──────┐   ┌──────────────┐
                          │  Post     │   │  Follow    │   │  Feed        │
                          │  Service  │   │  Service   │   │  Service     │
                          └────────┬──┘   └─────┬──────┘   └──────┬───────┘
                                   │            │                  │
                        ┌──────────▼──────────  │  ────────────────▼──────┐
                        │                    DynamoDB                      │
                        │   Posts Table    Follow Table   Feed Table       │
                        └──────────────────────────────────────────────────┘
                                                    │
                                          ┌─────────▼──────────┐
                                          │   Message Queue    │
                                          │  (Fan-out workers) │
                                          └────────────────────┘
```
