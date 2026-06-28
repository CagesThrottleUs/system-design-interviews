# Lessons Learned — Chat System

## Real Interview Stories

**Story 1: Meta/Facebook E5 — The Stateless Server Mistake**
Company: Meta | Level: E5 | Round: System Design

What happened: Candidate correctly chose WebSocket for real-time delivery. Then said: "I'll put the chat servers behind an HTTP load balancer — they're stateless, so any server can handle any request." Interviewer asked: "If Alice is connected to Server A and I need to deliver a message to her, how does the server know to send it to Server A?"

Where it went wrong: WebSocket connections create statefulness even if the servers themselves don't persist data. The connection is pinned to one server for its lifetime. A "stateless" load balancer would route an internal delivery request to Server B, which has no connection to Alice. Alice would never receive the message.

Consequence: Candidate was asked to redesign the routing mechanism mid-interview. Lost 12 minutes and never got to the deep dive on group chat.

What to do instead: Say explicitly: "Chat servers are stateful in the sense that each user's WebSocket connection is pinned to one server. To route messages, I need a User Presence Service — a Redis map from user_id to server_id. When Server A needs to deliver to Alice, it looks up Alice's server in Redis and sends via internal gRPC."

---

**Story 2: Google L5 — The Missing Offline Path**
Company: Google | Level: L5 | Round: System Design

What happened: Candidate designed a clean 1:1 chat system with WebSocket connections, a message store, and delivery receipts. Interviewer asked: "Bob's phone is off. Alice sends him a message. What happens?"

Where it went wrong: Candidate's system had no offline delivery path. The User Presence Service would show Bob as offline, but the candidate hadn't designed what happens next. They improvised: "We poll his mailbox when he comes online." Interviewer pressed: "How does Bob know to poll? Who tells his app that there are new messages?"

Consequence: Candidate couldn't explain the end-to-end offline flow (push notification → app opens → WebSocket established → message sync). Scored "below expectation" on completeness.

What to do instead: Design the offline path explicitly. When User Presence shows offline: (1) message is already persisted in Cassandra, (2) trigger push notification via APNs/FCM, (3) user opens app, WebSocket connects, (4) client sends last_message_id, server streams all messages newer than that ID, (5) client sends delivered ack.

---

**Story 3: Apple ICT3 — Group Chat Fan-Out Underestimated**
Company: Apple | Level: ICT3 | Round: System Design (iMessage-like system)

What happened: Candidate designed 1:1 chat correctly. For group chat, proposed: "Same as 1:1, but deliver to all N members." Interviewer asked: "Your group has 500 members. Alice sends one message per second. What is your fan-out rate?"

Where it went wrong: 1 message/sec × 500 members = 500 delivery operations/second per group. In a product with thousands of active groups, this is millions of delivery operations/second. The candidate's design routed all of this through the same path as 1:1 messages, creating a hot path.

Consequence: Candidate could not propose a scalable fan-out mechanism. Cascaded into confusion about Kafka and message queues.

What to do instead: Recognize group chat fan-out as a fundamentally different problem. Introduce Kafka as the fan-out broker: sender writes one message to Kafka, fan-out workers consume and deliver to each member in parallel. This decouples the sender from the delivery latency of all members.

## General Pattern Mistakes

**Mistake 1: Treating Chat Servers as Stateless**
What happens: Candidate puts chat servers behind a round-robin load balancer, declaring them "stateless."
Why wrong: WebSocket connections are long-lived and bound to one server. A message for a user must be routed to their specific server. Treating servers as stateless means you have no routing mechanism.
Fix: Introduce User Presence Service (Redis: user_id → server_id). Internal messages use gRPC to deliver to the correct server.

**Mistake 2: No Offline Delivery Design**
What happens: Design handles online delivery but has no answer for offline users.
Why wrong: In mobile chat, users are frequently offline. WhatsApp was designed for 2G networks in India where connections drop constantly. Zero offline design = lost messages.
Fix: Always design the offline path. Push notification (APNs/FCM) alerts the app. On reconnect, client sends last_ack_id, server streams missed messages.

**Mistake 3: Using a Relational DB for Message Storage**
What happens: Candidate proposes PostgreSQL or MySQL for the message table.
Why wrong: Chat is a time-series append workload with massive write volume (231K messages/second at scale). Relational databases are not optimized for time-series appends. At chat scale, Cassandra or ScyllaDB are purpose-built — partitioned by conversation_id, clustered by time.
Fix: Cassandra with partition key = conversation_id, clustering key = message_id (ULID). All messages in a conversation co-located. Append-only writes. TTL built-in.

**Mistake 4: Delivery Receipt as a DB Poll**
What happens: Candidate implements delivery receipts as: "sender polls the message table periodically to check if status changed to 'delivered'."
Why wrong: Polling creates unnecessary load and adds latency. In a chat system with 50M concurrent connections, polling on message status creates 50M × polling_frequency read operations.
Fix: Receipts are acknowledgment events that flow through the system like messages. Recipient's client sends a "delivered" event via WebSocket when message is received. Server routes this back to the sender's Chat Server, which pushes to sender's WebSocket. No polling.

**Mistake 5: Fan-Out on Write for Large Groups Without Async**
What happens: Candidate fans out group messages synchronously — deliver to all 500 members before ACKing to the sender.
Why wrong: The sender's latency becomes dependent on the slowest recipient delivery. One offline user triggering APNs delays the sender's ACK.
Fix: ACK to sender after Cassandra write + Kafka enqueue. Fan-out to members happens asynchronously via Kafka workers. Sender's latency is decoupled from recipient count.
