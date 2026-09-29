# Design: Spec-Driven Code Generator

## 1. Overview

This project demonstrates a spec-driven agentic code generation workflow.

A written behavioral specification serves as the source of truth for generating independent client implementations for multiple platforms.

The initial implementation generates:

* An iOS client using Swift and SwiftUI.
* An Android client using Kotlin and Jetpack Compose.

Both clients communicate with the same local messaging server and implement the same messaging protocol.

The primary deliverable is not the messaging application itself. The primary deliverable is the reusable generation workflow that can produce conforming clients from the specification.

---

# 2. Design Goals

The system has five primary goals.

## 2.1 Specification as the source of truth

Behavioral requirements should be defined in the specification rather than being implicitly defined by an existing implementation.

When behavior is ambiguous, the specification should be updated before implementation decisions are made.

The generated clients should be considered implementations of the specification rather than independent sources of product behavior.

---

## 2.2 Platform-independent behavior

The specification defines behavior that must remain consistent across all clients.

For example:

* Message structure
* Identity semantics
* Delivery semantics
* Offline queue behavior
* Retry behavior
* Acknowledgements
* Deduplication
* Reconnection

Platform-specific implementation details are deliberately excluded from the core protocol.

This allows Swift and Kotlin implementations to use different frameworks while still interoperating.

---

## 2.3 Regenerability

Generated client code must be disposable.

The repository explicitly identifies which code is generated and which code is maintained manually.

A clean workspace should be able to delete the generated clients and recreate them using the documented generation command.

Regeneration is treated as a first-class acceptance criterion rather than a theoretical capability.

---

## 2.4 Agentic development with deterministic boundaries

Claude Code is used as the implementation agent.

The agent is given:

* The authoritative specification
* Repository-level instructions
* Platform-specific constraints
* Validation commands
* Existing project context

The agent is responsible for implementing and validating generated code.

The generation harness is responsible for providing a repeatable workflow and preventing the agent from silently redefining the system's behavioral contract.

---

## 2.5 Validation over visual confidence

Generated code is not considered successful simply because it compiles or appears correct.

Validation must test:

1. Protocol conformance
2. Build correctness
3. Message sending
4. Message receiving
5. Offline queueing
6. Reconnection
7. Duplicate handling
8. Persistence
9. Cross-platform interoperability
10. Regeneration

---

# 3. System Architecture

The system consists of four primary layers:

```text
                 ┌──────────────────┐
                 │    SPEC.md       │
                 │                  │
                 │ Source of Truth  │
                 └────────┬─────────┘
                          │
                          ▼
                 ┌──────────────────┐
                 │ Generator       │
                 │                  │
                 │ Agentic Harness │
                 └────────┬─────────┘
                          │
             ┌────────────┴────────────┐
             ▼                         ▼
      ┌──────────────┐          ┌──────────────┐
      │ iOS Client   │          │ Android      │
      │              │          │ Client       │
      │ SwiftUI      │          │ Compose      │
      └──────┬───────┘          └──────┬───────┘
             │                         │
             └───────────┬─────────────┘
                         ▼
                  ┌─────────────┐
                  │   Server    │
                  │             │
                  │ Local       │
                  │ In-memory   │
                  └─────────────┘
```

The server is intentionally simple. The main engineering focus is the protocol and the client generation workflow.

---

# 4. Specification Architecture

The specification is divided conceptually into three areas.

## 4.1 Behavioral requirements

These describe what the system must do.

Examples:

* A user must identify themselves.
* Messages must be addressed to a recipient.
* Messages must survive temporary client disconnection.
* Received messages must be displayed.
* Duplicate deliveries must not create duplicate visible messages.

These requirements should not depend on a particular programming language.

---

## 4.2 Protocol requirements

These define communication between clients and the server.

The protocol uses JSON messages transmitted over WebSocket.

Protocol messages define:

* Message type
* Message identity
* Sender
* Recipient
* Payload
* Acknowledgement
* Errors

Protocol semantics are shared by all clients.

---

## 4.3 Platform extensions

Platform-specific implementation requirements are separated from the shared protocol.

For example:

### iOS

The implementation may use:

* Swift
* SwiftUI
* Swift concurrency
* URLSession WebSocket APIs
* A local persistence mechanism appropriate for iOS

### Android

The implementation may use:

* Kotlin
* Jetpack Compose
* Kotlin coroutines
* An appropriate WebSocket library
* A local persistence mechanism appropriate for Android

The choice of implementation technology does not change protocol semantics.

---

# 5. Message Delivery Model

The system uses **at-least-once delivery**.

This is intentional.

Distributed systems cannot safely assume that failure to receive an acknowledgement means that the server did not process a message.

For example:

```text
Client                  Server

  SEND ------------------>

       <------------------ ACCEPTED

Connection fails
```

The client may receive the message acceptance or may lose it due to the connection failure.

Therefore, a client must retain enough information to safely retry.

---

# 6. Idempotency

Outgoing messages use a client-generated `clientMessageId`.

The ID remains unchanged across retries.

The server uses the combination of sender identity and `clientMessageId` to determine whether an incoming send request represents a new logical message or a retry of an existing message.

If the message already exists, the server returns the previously assigned `serverMessageId` rather than creating a duplicate.

This allows clients to retry safely.

---

# 7. Why Two Message IDs Exist

The system deliberately distinguishes between:

### Client message ID

Generated by the sender.

Its purpose is to identify the logical outgoing operation from the sender's perspective.

### Server message ID

Generated by the server.

Its purpose is to identify the canonical accepted message.

This separation allows the sender to safely retry a message while allowing recipients to deduplicate server-delivered messages.

---

# 8. Client State

The client maintains two major categories of durable state.

## 8.1 Outgoing state

Messages waiting for server acceptance.

```text
PENDING
   │
   ▼
SEND
   │
   ├── failure ──► PENDING
   │
   ▼
ACCEPTED
   │
   ▼
REMOVE FROM OUTBOX
```

A pending message must survive application termination.

---

## 8.2 Incoming state

Messages accepted and delivered by the client.

```text
SERVER MESSAGE
      │
      ▼
DEDUPLICATE
      │
      ├── already seen → ignore
      │
      ▼
PERSIST
      │
      ▼
DISPLAY
      │
      ▼
ACK
```

Persistence occurs before acknowledgement to prevent the server from believing a message was successfully processed when the client has not durably stored it.

---

# 9. Connection State

The client does not rely solely on operating-system network connectivity to determine whether it can communicate with the server.

The relevant connection states are:

```text
DISCONNECTED
      │
      ▼
CONNECTING
      │
      ▼
CONNECTED
      │
      ▼
DISCONNECTED
```

A device may have an active network connection while the WebSocket connection itself is unusable.

Therefore, the WebSocket connection is the authoritative source for application-level connectivity.

---

# 10. Reconnection

Reconnection is treated as a normal state transition rather than an exceptional one.

After establishing a new connection, the client must:

1. Re-identify itself.
2. Resume pending outgoing messages.
3. Receive pending incoming messages.
4. Continue normal message processing.

The user should not need to manually resend a message because of temporary connectivity loss.

---

# 11. Server Responsibilities

The server exists primarily to provide a deterministic local environment in which independent clients can communicate.

It is intentionally not production-oriented.

The server is responsible for:

* Tracking registered users
* Tracking active connections
* Accepting messages
* Assigning server message IDs
* Deduplicating outgoing submissions
* Retaining undelivered messages
* Delivering messages after reconnection
* Processing client acknowledgements

The server may maintain state entirely in memory.

---

# 12. Client Responsibilities

Clients are responsible for:

* Managing local identity
* Persisting outgoing messages
* Retrying unacknowledged messages
* Persisting received messages
* Deduplicating received messages
* Managing connection lifecycle
* Rendering the current message state
* Conforming to the shared protocol

---

# 13. Agentic Generation Architecture

The generator is designed as a reusable agentic workflow rather than a single large prompt.

Conceptually:

```text
SPEC.md
   │
   ▼
Generation Instructions
   │
   ▼
Claude Code
   │
   ├── Inspect workspace
   │
   ├── Plan implementation
   │
   ├── Implement
   │
   ├── Build
   │
   ├── Test
   │
   └── Repair failures
   │
   ▼
Generated Client
```

The generator should provide Claude with enough context to implement the specification without embedding the messaging application's behavior directly into the generator itself.

---

# 14. Generation Boundary

The generated clients are disposable artifacts.

The intended boundary is:

```text
clients/ios/
clients/android/
```

Everything inside these directories is generated unless explicitly documented otherwise.

The generator and specification remain hand-maintained.

Generated files should identify their generated status where practical.

The README must document how to remove and regenerate the generated clients.

---

# 15. Validation Strategy

Validation occurs at multiple levels.

## Level 1 — Static validation

Check that:

* Required files exist.
* Generated boundaries are respected.
* Protocol definitions are consistent.
* Required message types exist.

## Level 2 — Build validation

Each client must compile successfully using its platform's build tooling.

## Level 3 — Unit validation

Client logic should test:

* Message encoding/decoding
* Queue behavior
* Retry behavior
* Deduplication
* Persistence
* Connection state transitions

## Level 4 — Integration validation

The clients must communicate with the same server.

## Level 5 — Scenario validation

The required Alice/Bob scenarios must be exercised.

## Level 6 — Regeneration validation

Generated clients are deleted and recreated using the documented generator workflow.

The regenerated clients must pass the same validation suite.

---

# 16. Agent Validation Loop

The generator should favor a feedback loop over a single generation attempt.

Conceptually:

```text
Generate
   │
   ▼
Build
   │
   ├── FAIL ───────┐
   │               │
   ▼               │
Test               │
   │               │
   ├── FAIL ───────┤
   │               │
   ▼               │
Validate           │
   │               │
   ├── FAIL ───────┘
   │
   ▼
SUCCESS
```

When a failure occurs, the agent should inspect the failure and make a targeted correction rather than restarting generation from scratch.

The goal is to make generation robust to ordinary implementation mistakes.

---

# 17. Specification Evolution

The generator should not be hard-coded around the initial messaging feature set.

Future versions of the specification may introduce capabilities such as:

* Attachments
* Reactions
* Group conversations
* Message editing
* Read receipts
* Typing indicators
* Additional protocol messages

The preferred evolution model is:

```text
SPEC.md changes
      │
      ▼
Generator remains stable
      │
      ▼
Agent interprets new requirements
      │
      ▼
Clients regenerate
      │
      ▼
Validation exposes missing implementation
```

The generator may require new generic capabilities over time, but application behavior should primarily enter the system through the specification rather than through messaging-specific conditionals in the generator.

---

# 18. Design Tradeoffs

## WebSocket vs HTTP polling

WebSocket was selected because the assignment requires clients to receive messages and reconnect after temporary disconnection.

HTTP polling could implement the behavior but would add unnecessary polling logic and make the real-time protocol less direct.

## At-least-once vs exactly-once delivery

At-least-once delivery was selected because it provides a practical failure model.

Exactly-once semantics across independently failing clients and networks would require substantially more infrastructure and would obscure the primary purpose of the assignment.

Idempotency and client-side deduplication provide the required user-visible behavior without pretending network delivery is perfectly reliable.

## In-memory server vs persistent server

An in-memory server was selected because server persistence is explicitly outside the assignment's scope.

Client persistence is more important because offline behavior is a core requirement.

## Native platform implementations vs shared client code

Separate Swift and Kotlin implementations were selected to demonstrate that the specification actually defines an interoperable protocol rather than simply generating multiple wrappers around shared implementation code.

---

# 19. Non-Goals

The design intentionally does not attempt to solve:

* Production authentication
* Production authorization
* End-to-end encryption
* Cloud deployment
* Horizontal scaling
* DDoS protection
* Production observability
* Production database architecture
* Push notification infrastructure
* Advanced messaging features

These could be addressed in future specifications.

---

# 20. Success Criteria

The design is considered successful if:

1. The specification completely describes required observable behavior.
2. Both clients independently implement the same protocol.
3. The clients can exchange messages through the server.
4. Offline messages survive client restarts.
5. Temporary connection failures do not lose messages.
6. Duplicate network delivery does not create duplicate visible messages.
7. The clients can be deleted and regenerated.
8. Regenerated clients pass the same validation scenarios.
9. The generator can accommodate meaningful future specification extensions without being rewritten specifically for the current messaging application.
