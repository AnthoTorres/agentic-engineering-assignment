# CLAUDE.md

# Spec-Driven Code Generator

This repository is an agentic code-generation system.

The primary objective is not to manually build a messaging application.

The primary objective is to create a reusable workflow in which an agent can read an authoritative specification and generate independent, working client implementations that conform to that specification.

The current example system is a local messaging application with:

* A local messaging server
* An iOS client
* An Android client

The clients must independently implement the same externally observable behavior and protocol.

---

# 1. Source of Truth

The repository has an explicit source-of-truth hierarchy.

## Priority 1 — SPEC.md

`SPEC.md` is authoritative for required system behavior.

When implementation behavior conflicts with `SPEC.md`, the implementation is wrong.

Do not silently change required behavior because another implementation approach appears easier.

Do not infer new product behavior when the specification already defines the behavior.

---

## Priority 2 — DESIGN.md

`DESIGN.md` explains architectural intent, design principles, tradeoffs, and generation strategy.

Use it to understand why the repository is structured as it is.

If `DESIGN.md` and `SPEC.md` appear to conflict:

* `SPEC.md` controls behavioral requirements.
* `DESIGN.md` controls architectural intent.
* Do not silently resolve a contradiction by inventing a new requirement.

---

## Priority 3 — CLAUDE.md

This file defines how the coding agent should operate.

It does not override behavioral requirements in `SPEC.md`.

---

## Priority 4 — Existing implementation

Existing code is evidence of the current implementation, not the authoritative definition of required behavior.

Do not assume existing code is correct simply because it already exists.

When existing code conflicts with the specification, bring the implementation into conformance with the specification.

---

# 2. Core Agent Principles

The agent must follow these principles throughout the task.

## 2.1 Specification before implementation

Before writing substantial code:

1. Read `SPEC.md`.
2. Read `DESIGN.md`.
3. Inspect the relevant repository structure.
4. Identify applicable requirements.
5. Form an implementation plan.
6. Implement only after understanding the behavioral contract.

Do not begin by blindly generating files.

---

## 2.2 Do not invent protocol behavior

The protocol is shared by independent clients.

A behavior that affects interoperability must not be invented casually.

Examples include:

* Message field names
* Message types
* Message IDs
* Acknowledgement semantics
* Retry semantics
* Delivery semantics
* Deduplication rules
* Ordering rules
* Connection lifecycle
* Error behavior

If the specification is insufficient to determine an interoperability-critical behavior, stop and identify the ambiguity rather than silently creating a platform-specific interpretation.

If the behavior can be resolved without changing the public contract, make the smallest reasonable implementation decision and document it.

If the ambiguity changes the contract, the specification should be updated before proceeding.

---

# 3. Generated vs Hand-Written Code

Generated code is intentionally disposable.

The generated client boundary is:

```text
clients/ios/
clients/android/
```

Code inside those directories should be treated as generated unless explicitly marked otherwise.

Do not move application behavior outside the generation boundary merely to make generation easier.

Do not manually patch generated code as a substitute for fixing the generator, prompts, specification, or generation workflow.

If generated code requires a correction:

1. Determine why the generated implementation was incorrect.
2. Correct the appropriate source of the problem.
3. Regenerate.
4. Validate again.

The desired workflow is:

```text
Specification
     ↓
Generation workflow
     ↓
Generated implementation
     ↓
Validation
```

not:

```text
Specification
     ↓
Generated implementation
     ↓
Manual patch
     ↓
"It works"
```

---

# 4. Platform Independence

The iOS and Android clients are independent implementations.

Do not share client source code between platforms.

The clients must independently implement the protocol.

The following must remain semantically consistent:

* User identity
* Message representation
* Client message IDs
* Server message IDs
* Sending
* Acknowledgement
* Offline queueing
* Retry behavior
* Receiving
* Deduplication
* Reconnection
* Error behavior

The following may differ between platforms:

* UI framework
* Persistence framework
* WebSocket implementation
* Concurrency model
* Dependency injection
* Project structure
* Testing framework
* Platform lifecycle integration

A platform-specific implementation decision must not silently change shared protocol semantics.

---

# 5. Protocol Discipline

The protocol defined by `SPEC.md` is a public contract between independent implementations.

Treat protocol changes as API changes.

Before modifying protocol-related code, verify:

1. The behavior exists in `SPEC.md`.
2. The wire representation matches the specification.
3. The other platform can understand the same message.
4. Error handling remains compatible.
5. Existing scenarios remain valid.

Do not introduce undocumented fields or message types solely for convenience unless they are explicitly internal and cannot affect interoperability.

When protocol extensions are required, prefer explicit versioned or extensible structures rather than ad-hoc behavior.

---

# 6. Message Identity

The system distinguishes between:

```text
clientMessageId
serverMessageId
```

Do not collapse these concepts.

## clientMessageId

Generated by the sending client.

It identifies the logical outgoing message from the sender's perspective.

It must remain stable across retries.

## serverMessageId

Generated by the server.

It identifies the canonical accepted message.

It is used by receiving clients for deduplication.

A retry of an existing outgoing message must not create a new logical server message.

---

# 7. Delivery Semantics

The system uses at-least-once delivery.

Never assume that:

```text
send succeeded locally
```

means:

```text
server accepted the message
```

Likewise, never assume:

```text
acknowledgement was not received
```

means:

```text
server did not receive the message
```

Network failure may occur between any two protocol operations.

Therefore:

* Durable outgoing messages must survive connection loss.
* Unacknowledged outgoing messages must be retryable.
* Retries must preserve `clientMessageId`.
* Received messages must be deduplicated using `serverMessageId`.

Do not implement fake exactly-once semantics by simply assuming network operations cannot partially succeed.

---

# 8. Persistence Rules

The client must persist durable state before relying on that state across failures.

For outgoing messages:

```text
Create message
     ↓
Persist locally
     ↓
Attempt transmission
```

Not:

```text
Create message
     ↓
Attempt transmission
     ↓
Persist only after success
```

For incoming messages:

```text
Receive message
     ↓
Deduplicate
     ↓
Persist
     ↓
Display
     ↓
Acknowledge
```

The client should not acknowledge successful processing before the message is safely persisted.

The exact persistence technology is platform-specific.

Do not force both platforms to use the same persistence technology merely for superficial symmetry.

---

# 9. Connection Management

Application-level connectivity is defined by the WebSocket connection, not merely by the operating system reporting network availability.

Treat connection state as an explicit state machine.

At minimum:

```text
DISCONNECTED
CONNECTING
CONNECTED
```

A connection can fail even while the device has Wi-Fi or cellular connectivity.

After reconnection the client must:

1. Re-establish the WebSocket.
2. Re-identify itself.
3. Resume pending outgoing messages.
4. Process pending incoming messages.
5. Return to normal operation.

Reconnection must not require the user to manually resend durable messages.

---

# 10. Agent Workflow

For substantial implementation tasks, use the following workflow.

## Phase 1 — Understand

Read:

```text
SPEC.md
DESIGN.md
CLAUDE.md
```

Then inspect the repository.

Identify:

* Applicable requirements
* Existing implementation
* Generation boundary
* Platform constraints
* Relevant tests
* Known limitations

---

## Phase 2 — Plan

Before making broad changes, create a concise implementation plan.

The plan should identify:

* Files/components to create or modify
* Protocol considerations
* State management
* Persistence requirements
* Testing strategy
* Validation commands
* Potential failure points

Do not produce a plan that merely restates the specification.

The plan should explain how the requirements will become implementation.

---

## Phase 3 — Implement

Implement the smallest coherent change that satisfies the requirements.

Prefer:

* Clear boundaries
* Small components
* Explicit state
* Testable logic
* Deterministic behavior
* Platform-native idioms

Avoid:

* Unnecessary abstractions
* Premature frameworks
* Large amounts of duplicated logic
* Hidden protocol behavior
* Hard-coded behavior specific to the current test scenario

---

## Phase 4 — Validate

After implementation:

1. Build.
2. Run unit tests.
3. Run protocol tests.
4. Run integration tests.
5. Exercise required scenarios.
6. Inspect failures.
7. Fix root causes.
8. Repeat validation.

Never declare success because the code "looks correct."

---

# 11. Validation Hierarchy

Validation should progress from cheap checks to expensive checks.

## Level 1 — File and structure validation

Verify required files and directories exist.

## Level 2 — Static validation

Verify protocol structures and generated boundaries.

## Level 3 — Build validation

Compile the relevant client.

## Level 4 — Unit tests

Test isolated logic such as:

* Serialization
* Deserialization
* Queue state
* Retry behavior
* Deduplication
* Persistence
* Connection state

## Level 5 — Integration tests

Verify client/server communication.

## Level 6 — Cross-platform scenarios

Verify that an iOS client can communicate with the server and an Android client can communicate with the same server.

## Level 7 — Regeneration

Delete generated clients.

Run the documented generator.

Repeat validation.

Regeneration is not optional for declaring the generator successful.

---

# 12. Failure Handling

When a build or test fails, diagnose the failure before making changes.

Classify the failure as one of:

### Specification problem

The specification is ambiguous, incomplete, or contradictory.

### Generator problem

The generation workflow failed to provide the agent with necessary context or constraints.

### Implementation problem

The generated code does not correctly implement the specification.

### Test problem

The validation incorrectly represents the specification.

### Environment problem

The failure is caused by the local build/runtime environment rather than the implementation.

Do not automatically modify implementation code for every failure.

Fix the correct layer.

---

# 13. Do Not Hide Failures

Never:

* Disable a failing test merely to achieve green output.
* Remove validation because it exposes an implementation problem.
* Catch errors and ignore them without justification.
* Replace real behavior with mocks solely to make integration tests pass.
* Modify acceptance criteria to fit the current implementation.
* Claim a scenario works without executing or otherwise verifying it.

If a limitation prevents validation, document the limitation explicitly.

---

# 14. Testing the Required Scenarios

The required Alice/Bob scenarios are acceptance criteria.

At minimum, validation must cover:

1. Alice sends a message to Bob.
2. Bob replies to Alice.
3. Alice goes offline and queues a message.
4. Bob goes offline and queues a message.
5. Alice reconnects and transmits her queued message.
6. Alice disconnects again.
7. Bob reconnects and receives Alice's message.
8. Bob's queued message is transmitted.
9. Alice receives Bob's message.
10. Duplicate delivery does not create duplicate visible messages.

The test should verify behavior rather than simply checking that a network request returned HTTP/WebSocket success.

---

# 15. Regeneration Requirements

The generator must be capable of producing the generated clients from a clean generation boundary.

Before declaring the project complete:

1. Record the current working state.
2. Delete generated clients.
3. Run the documented generation command.
4. Build the regenerated clients.
5. Run tests.
6. Run the required messaging scenarios.
7. Verify cross-platform interoperability.

If regeneration produces different behavior from the original implementation, investigate why.

Do not manually repair regenerated output without addressing the underlying generation problem.

---

# 16. Reproducibility

The generation workflow should be reproducible.

Avoid relying on:

* Undocumented manual steps
* Local-only configuration
* Personal machine state
* Undocumented environment variables
* External services
* Uncommitted files
* Manual edits after generation

If an environment requirement is unavoidable, document it.

The evaluator should be able to understand:

```text
What to install
       ↓
What command to run
       ↓
What agent/tool is invoked
       ↓
What gets generated
       ↓
How generation is validated
```

---

# 17. No Third-Party Messaging SDKs

Do not use turnkey messaging or chat SDKs.

General-purpose technologies are allowed, including:

* WebSocket libraries
* HTTP libraries
* JSON libraries
* Local persistence libraries
* Standard UI frameworks
* Testing libraries

Do not introduce a library whose primary purpose is to hide the messaging protocol or offline synchronization behavior being evaluated.

The protocol and offline behavior must be implemented by the generated clients.

---

# 18. Server Scope

The server should remain intentionally simple.

Do not over-engineer:

* Authentication
* Authorization
* Cloud infrastructure
* Distributed storage
* Horizontal scaling
* DDoS protection
* Production monitoring
* Complex deployment infrastructure

The server exists to provide a deterministic environment for testing the generated clients.

Spend engineering effort primarily on:

* Protocol correctness
* Client behavior
* Offline handling
* Reconnection
* Deduplication
* Generation reliability
* Validation

---

# 19. UI Scope

The UI is not the primary evaluation target.

Prefer a simple UI that makes system behavior observable.

The UI should make it easy to observe:

* Current identity
* Connection state
* Messages
* Pending outgoing messages
* Send failures
* Reconnection

Do not spend significant implementation time on visual polish unless it directly improves validation or observability.

---

# 20. Specification Evolution

The generator must remain as generic as practical.

Do not hard-code the current messaging feature set into the generation workflow.

Avoid logic such as:

```text
if feature == "messaging":
    generate messaging-specific code
```

Prefer generic generation capabilities such as:

```text
interpret requirements
define domain models
define protocol contracts
implement state machines
implement persistence
implement transport
generate platform adapter
generate tests
validate behavior
```

When the specification evolves, prefer updating `SPEC.md` and allowing the generation workflow to interpret the new requirements.

If the generator itself needs additional generic capability, improve the generator rather than adding one-off logic for a single specification revision.

---

# 21. Future Specification Extensions

The architecture should remain open to requirements such as:

* Attachments
* Reactions
* Group conversations
* Message editing
* Message deletion
* Read receipts
* Typing indicators
* Additional protocol messages
* New client platforms

When extending the system, determine:

1. What is globally shared?
2. What is protocol-level behavior?
3. What is platform-specific?
4. What must persist?
5. What changes the state machine?
6. What new failure modes exist?
7. What new validation scenarios are required?

Do not implement a new feature solely by modifying the UI.

Trace the requirement through:

```text
Specification
     ↓
Protocol
     ↓
Domain/state
     ↓
Persistence
     ↓
Transport
     ↓
UI
     ↓
Tests
```

---

# 22. Code Quality

Generated code should be production-minded even though the application itself is intentionally minimal.

Prefer:

* Meaningful names
* Small focused components
* Explicit error handling
* Testable business logic
* Clear state transitions
* Dependency boundaries
* Minimal coupling
* Platform-native conventions

Avoid:

* Giant files
* Giant functions
* Global mutable state
* Hidden singletons without justification
* Magic strings for protocol semantics
* Duplicated protocol definitions
* UI-driven business logic
* Swallowing errors
* Excessive abstractions

Do not introduce architecture merely to make the project appear sophisticated.

Architecture should solve a demonstrated problem.

---

# 23. Documentation

When adding a significant architectural component, document:

* What it does
* Why it exists
* Which requirement it satisfies
* How it is validated

Documentation should explain decisions rather than merely describing obvious code.

If an implementation intentionally differs between platforms, document why the difference does not change protocol behavior.

---

# 24. Generated File Attribution

Generated files should make their provenance clear where practical.

Use a generated-file header such as:

```text
GENERATED FILE
Source of truth: SPEC.md
Generation workflow: generator/
Do not edit manually.
```

Do not claim that generated code was manually authored.

The repository should clearly distinguish:

* Hand-written specification
* Hand-written generator infrastructure
* Hand-written server
* Agent-generated clients
* Agent-generated tests, if applicable
* Human-authored tests or validation infrastructure

---

# 25. Git Discipline

Changes should be committed in coherent increments.

Prefer commits that represent meaningful milestones, such as:

```text
Add messaging client specification
Add generator architecture
Add local messaging server
Add generation workflow
Generate initial iOS client
Generate initial Android client
Add protocol conformance tests
Add offline delivery tests
Fix reconnect idempotency
Document regeneration workflow
```

Do not collapse the entire assignment into a single final commit.

Commit history should make the engineering process understandable.

---

# 26. Before Declaring Completion

Do not declare the project complete until all applicable checks have been considered.

### Specification

* [ ] Requirements are explicit.
* [ ] Protocol is defined.
* [ ] Offline behavior is defined.
* [ ] Failure behavior is defined.
* [ ] Deduplication is defined.
* [ ] Platform-specific boundaries are clear.

### Generator

* [ ] Generation workflow is documented.
* [ ] Generated boundary is clear.
* [ ] Generator is not hard-coded to one implementation.
* [ ] Claude Code has sufficient context.
* [ ] Generation is repeatable.

### Clients

* [ ] iOS client builds.
* [ ] Android client builds.
* [ ] Both implement the same protocol.
* [ ] Both support offline queues.
* [ ] Both reconnect.
* [ ] Both deduplicate messages.
* [ ] Client state survives application restart.

### Server

* [ ] Server starts locally.
* [ ] Users can identify themselves.
* [ ] Messages can be accepted.
* [ ] Messages can be delivered.
* [ ] Offline recipients receive pending messages.
* [ ] Duplicate submissions are handled idempotently.

### Validation

* [ ] Unit tests pass.
* [ ] Integration tests pass.
* [ ] Required Alice/Bob scenarios pass.
* [ ] Cross-platform interoperability is verified.
* [ ] Regeneration has been tested.

---

# 27. Agent Completion Report

At the end of a substantial generation or implementation task, report:

```text
## Completed

- [major implementation changes]

## Validation

- [build result]
- [unit test result]
- [integration result]
- [scenario result]

## Known Limitations

- [limitations]

## Specification Changes Needed

- [ambiguities or missing requirements]

## Generated Artifacts

- [generated directories/files]

## Manual Follow-Up

- [anything requiring human action]
```

Do not report successful validation that was not actually performed.

---

# 28. Final Principle

The goal is not:

> Generate code quickly.

The goal is:

> Generate code that can be independently reproduced, validated, regenerated, and evolved from a written specification.

When forced to choose between:

```text
fast implementation
```

and:

```text
reproducible implementation
```

prefer reproducibility.

When forced to choose between:

```text
clever implementation
```

and:

```text
specification-conforming implementation
```

prefer specification conformance.

When uncertain about a behavioral requirement:

```text
Do not invent.
Inspect the specification.
Identify the ambiguity.
Resolve the contract.
Then implement.
```
