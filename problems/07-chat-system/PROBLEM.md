# Chat System (WhatsApp / Messenger)
**Difficulty:** Intermediate
**Category:** Real-Time / Fan-Out / State Management
**Time Box:** 45 min
**Key Concepts:** WebSocket, Message Ordering, Fan-Out, Offline Storage, Delivery Receipts
**Asked at:** Facebook/Meta, Google, Uber, Airbnb, Snap, Apple, LinkedIn

## Problem Statement

You are designing a chat application that supports real-time 1:1 messaging and group conversations. Think WhatsApp or Facebook Messenger. Users expect messages to appear within seconds of being sent, regardless of whether the recipient is currently active on their phone. If a recipient is offline, the message should be reliably delivered the next time they open the app.

The system must handle users across the globe, across multiple devices per user, and group conversations of up to 500 members. Sent, delivered, and read receipts are first-class features — users trust the double-tick to know their message arrived. The business cannot afford to lose a single message.

## Actors
| Actor | Description |
|-------|-------------|
| Sender | User composing and sending messages |
| Recipient | User receiving messages; may be online or offline |
| Group Member | Participant in a group chat (up to 500 members) |
| Push Notification System | APNs/FCM for offline delivery |

## Functional Requirements
**Core (MVP):**
- FR1: Send and receive text messages in 1:1 conversations
- FR2: Messages persist; users can scroll back through history
- FR3: Delivery receipt: show when message is delivered to recipient's device
- FR4: Read receipt: show when message is opened by recipient
- FR5: Offline users receive messages when they come online
- FR6: Group chat: up to 500 members, all members see all messages

**Secondary:**
- FR7: Multi-device support — user logged in on phone and laptop sees the same messages
- FR8: Media messages (images, video) — out of scope for deep dive, mention architecture
- FR9: Message search within a conversation

## Non-Functional Requirements
| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Latency | < 500ms message delivery (sender → recipient, online) | Beyond this, chat feels broken |
| Durability | Zero message loss | Lost messages destroy user trust; WhatsApp's core promise |
| Availability | 99.99% | Chat is a core product surface |
| Scale | 2 billion users (WhatsApp scale); design for 500M DAU | |
| Group chat | Up to 500 members | WhatsApp raised group limit from 256 to 1024 in 2022 |
| Message storage | 7 days on server; permanent on device | Reduce server cost while preserving user history |

## Capacity Estimation Hints
- DAU: 500 million
- Messages per user per day: 40 (send + receive)
- Total messages per day: 500M × 40 = 20 billion/day
- Average message size: 100 bytes (text only)
- Total storage per day: 20B × 100 bytes = 2 TB/day
- Online users at peak: ~50M concurrent WebSocket connections
- Group chat members: average 50, max 500

## Good Clarifying Questions to Ask

**Scale:**
- How many DAU? How many concurrent online users?
- What is the maximum group size?
- How many devices can one user be logged into simultaneously?

**Features:**
- 1:1 only, or group chat too?
- Do we need delivery receipts and read receipts?
- What happens if the recipient is offline?
- Do we need message search?

**Consistency:**
- Must messages appear in the same order for all participants?
- What happens to messages if the sender loses connectivity mid-send?

**Constraints:**
- How long do we retain messages on the server?
- Media (images, video) in scope?
- End-to-end encryption in scope?

## Key Components
<details><summary>Reveal components</summary>

- **Chat Service**: Stateful WebSocket servers; each server holds open connections for a set of users
- **Message Store**: Cassandra for persistent message history (time-series, append-heavy, high write throughput)
- **User Presence Service**: Tracks which WebSocket server each user is connected to; Redis-backed
- **Message Queue (Kafka)**: Fan-out broker for group messages; decouples sender from all recipients
- **Push Notification Service**: Wrapper around APNs/FCM for offline delivery
- **Delivery/Read Receipt Handler**: Processes acknowledgment events and updates message status
- **Service Discovery (ZooKeeper/etcd)**: Chat servers register their connected user list for routing
</details>
