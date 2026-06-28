# Mock Interview Script — Chat System
*For the interviewer. Do not share with candidate.*

## Pre-Session Setup
- Problem: Design a real-time chat system (WhatsApp/Messenger scale)
- Target level: Apple ICT3 / Meta E5 / Google L5
- Time: 45 minutes
- Note the minute at each phase transition.

## Opening (read aloud, verbatim)
"Design a real-time chat application similar to WhatsApp or Facebook Messenger. It should support 1:1 messaging and group chats. You have 45 minutes. Please start by asking clarifying questions."

*Start timer. Say nothing else.*

## Phase 1: Requirements Clarification (0–8 min)

**Answers to give if asked:**
| Question | Answer |
|----------|--------|
| How many users? | 500 million DAU |
| 1:1 only or groups? | Both — groups up to 500 members |
| Delivery/read receipts? | Yes |
| Offline delivery? | Yes — message must arrive when user comes back online |
| Media (images/video)? | Focus on text for now; mention media architecture briefly |
| Multi-device? | Yes — user on phone and laptop should see same messages |
| End-to-end encryption? | Mention as an important consideration, but no deep dive needed |
| Message retention? | 7 days on server; permanent on device |
| Message ordering? | Consistent for all participants |

**Red flags:**
- Doesn't ask about offline delivery
- Doesn't ask about group chat vs 1:1
- Assumes stateless servers from the start

## Phase 2: Estimation (8–13 min)

**Expected:**
- Messages/sec: 500M × 40 / 86,400 ≈ 231K/sec
- WebSocket connections: 50M concurrent
- Storage: ~2 TB/day text only

**If stuck:** "500M DAU, each user sends and receives about 40 messages per day on average."

## Phase 3: High-Level Design (13–28 min)

**Expected components:**
- WebSocket-based Chat Servers (stateful per connection)
- User Presence Service (Redis: user_id → server_id)
- Message Store (Cassandra)
- Kafka fan-out for group messages
- Push notification path (APNs/FCM) for offline users
- Delivery receipt event flow

**Critical question at 18 min:** "Alice is on Server A. Bob is on Server B. Alice sends Bob a message. Walk me through exactly how that message gets from Alice's WebSocket to Bob's WebSocket."

*If they can't explain routing to Bob's specific server → they're missing User Presence Service.*

**Critical question at 23 min:** "Bob's phone dies. Alice sends him a message. What happens?"

*If they have no offline path → note it. Ask: "How does Bob's app know there's a new message when it reconnects?"*

**If they propose HTTP long polling instead of WebSocket:**
"What's the latency of long polling vs WebSocket for message delivery? And what's the connection overhead difference at 50M concurrent users?"

## Phase 4: Deep Dive (28–40 min)

**Pick ONE based on candidate gaps:**

**Option A — Message Ordering:**
"Two users in a group chat send messages simultaneously. User C sees them in one order; User D sees them in a different order. How does your system handle this? What ordering guarantees do you provide?"

**Option B — Group Chat Fan-Out:**
"Walk me through your group chat architecture in detail. Group has 500 members, 200 are online, 300 are offline. One message is sent. What are all the operations that happen? What's the total work per message?"

**Option C — Multi-Device Sync:**
"Alice is logged in on her iPhone and her MacBook. She sends a message on her iPhone. How does her MacBook know about this message? When she opens her MacBook and sees the conversation, what does she see?"

**Option D — Delivery Receipts:**
"Explain the complete flow for a delivery receipt. From the moment Bob's device receives Alice's message to the moment Alice sees the double checkmark. What are all the events, what goes through what system, what is written to storage?"

## Phase 5: Failure & Scaling Probe (40–44 min)

Ask TWO:
1. "A Chat Server handling 10,000 connections crashes. What happens to those users? How does your system recover?"
2. "The User Presence Redis cluster goes down. What is the impact? Does the system degrade gracefully or fail hard?"
3. "Cassandra is experiencing high latency (writes taking 200ms instead of 2ms). How does this affect the 500ms latency SLA? What can you do?"
4. "A celebrity sends a message in a group with 1,000 members and it goes viral — they forward it to their own groups. How does this fan-out propagate? What breaks first?"

## Closing (44–45 min)
"Last question: what is the weakest point in your design, and what would you fix with more time?"

## Scoring Checklist

| Dimension | Evidence of Excellence | Score (1-5) |
|-----------|----------------------|-------------|
| Requirements | Asked about offline, groups, multi-device, receipts | |
| Estimation | Derived messages/sec; sized WebSocket connections; storage | |
| Connection management | WebSocket + User Presence Service + routing via server_id | |
| Offline path | Push notification → reconnect → message sync | |
| Persistence | Cassandra with correct partition key; write-before-deliver | |
| Group chat | Kafka fan-out; cost analysis; async delivery | |
| Failure handling | Server crash recovery; Redis failure degradation | |
