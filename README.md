# WhatsApp Group Chat – System Design (HLD & LLD)

A system design of a WhatsApp-like group messaging system that allows users to send messages to groups, delivers messages to online members in real time, and ensures offline members receive their missed messages when they reconnect.

## Table of Contents

1. [Functional Requirements](#1-functional-requirements)
2. [Non-Functional Requirements](#2-non-functional-requirements)
3. [High-Level Design (HLD)](#3-high-level-design-hld)
4. [Key Components](#4-key-components)
5. [Message Flow](#5-message-flow)
6. [Q&A – When Should We Show the Single Tick?](#6-qa--when-should-we-show-the-single-tick)

---

## 1. Functional Requirements

These describe what our WhatsApp Group Chat system should do.

- **Send Messages:** Users should be able to send messages to a group.
- **Message Delivery:** Messages should be delivered to all eligible members of the group.
- **Offline Message Delivery:** Members who are offline should receive their missed messages when they reconnect to the internet.

## 2. Non-Functional Requirements

These describe how well our system should work.

- **Low Latency:** Messages should be delivered to group members with minimal delay.
- **Scalability:** The system should support millions of concurrent connections and messages, and scale to billions of users as the platform grows.

## 3. High-Level Design (HLD)

The following diagram represents the overall architecture of our WhatsApp Group Chat system.

![WhatsApp Group Chat HLD](https://github.com/user-attachments/assets/3573b43e-e56b-438d-b459-14dc1d3ccb45)

## 4. Key Components

- **API Gateway + Load Balancer:** Handles incoming client requests, manages routing, and distributes traffic across service instances.
- **WebSocket:** Maintains a persistent, bidirectional connection between clients and the server for real-time communication.
- **Chat Service:** Performs lightweight validations, such as checking whether the group ID is valid and whether the sender is a member of the group.
- **Kafka:** Acts as a message queue and decouples message creation from asynchronous processing.
- **Worker Service:** Consumes messages from Kafka and stores them in Cassandra DB using batch writes.
- **Fanout Service:** Distributes messages to all eligible group members.
- **Group Service:** Retrieves group details and the list of group members. Uses caching to reduce repeated database queries.
- **Presence Service:** Checks whether group members are online or offline. Uses caching to maintain user presence information.
- **Notification Service:** Sends push notifications to offline members.
- **Sync Service:** Fetches missed messages from the database when an offline user reconnects. Uses pagination to retrieve messages in batches.
- **Cassandra DB:** Stores chat messages and supports efficient message retrieval.

## 5. Message Flow

Let's understand how the system works when User A sends a message to a group.

1. **Client → API Gateway:** User A sends a message through a WebSocket connection.
2. **API Gateway → Chat Service:** The request is forwarded to the Chat Service.
3. **Chat Service:** Performs lightweight validations, such as checking whether the group exists and whether User A is a member.
4. **Chat Service → Kafka:** Publishes the message event to Kafka for asynchronous processing.
5. **Kafka → Worker Service:** The Worker Service consumes the message and stores it in Cassandra DB.
6. **Kafka → Fanout Service:** The Fanout Service processes the message event and starts distributing the message.
7. **Fanout Service → Group Service:** Retrieves all eligible members of the group.
8. **Fanout Service → Presence Service:** Checks which members are online and which are offline.
9. **Online Members:** Messages are delivered through their active WebSocket connections.
10. **Offline Members:** Push notifications are sent through the Notification Service. Their actual missed messages are retrieved when they reconnect.

### Offline Message Synchronization

When an offline user reconnects:

1. The user reconnects to the internet.
2. The client establishes a WebSocket connection through the API Gateway.
3. The request is forwarded to the Sync Service.
4. The Sync Service fetches missed messages from Cassandra DB.
5. Messages are retrieved using pagination.
6. The missed messages are sent back to the client through WebSocket.

This ensures that offline users can receive messages they missed while disconnected.

## 6. Q&A – When Should We Show the Single Tick?

**Question: When should we display the single tick after a user sends a message?**

There are two possible approaches:

### Approach 1: After publishing the message to Kafka

Once the Chat Service successfully publishes the message to Kafka and receives an acknowledgement, we display the single tick.

**Advantages:**
- Lower latency.
- The user gets an acknowledgement quickly.
- We don't need to wait for the Worker Service to finish database operations.

**Disadvantage:**

The message might still be in Kafka and may not have been stored in Cassandra DB yet.

For example:

1. User A sends a message.
2. Kafka successfully acknowledges the message.
3. We display the single tick.
4. User B is offline and reconnects immediately.
5. The Sync Service checks Cassandra DB, but the message has not been stored yet.
6. The message might not appear in the synchronization response.

This creates a temporary consistency issue between Kafka and Cassandra.

### Approach 2: After storing the message in the database

We display the single tick only after the Worker Service successfully stores the message in Cassandra DB.

**Advantages:**
- The message is already persisted before acknowledging it.
- Offline users can retrieve the message from the database.
- Provides stronger persistence guarantees before showing the acknowledgement.

**Disadvantage:**
- Higher acknowledgement latency.
- If the Worker Service or database is slow, the user has to wait longer for the single tick.

### Our Design Decision

For now, we choose **Approach 1 – Show the single tick after Kafka successfully acknowledges the message**.

The main reason is to prioritize low latency and provide a faster acknowledgement to the sender.

However, this tick means that the backend has accepted the message, not that it has been stored in Cassandra or delivered to recipients.

The Kafka acknowledgement should be configured for appropriate durability. We also need to handle the temporary Kafka-to-Cassandra delay so that the Sync Service does not permanently miss messages that have not yet been persisted.

---
