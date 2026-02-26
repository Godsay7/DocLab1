# Laboratory Work 1
## Designing a Messaging System

### Variant 6 — Message Editing & Deletion

#### Part 1 — Component Diagram

```mermaid
graph LR
  Client[Client App Web/Mobile] --> API[API Gateway]
  API --> Auth[Auth Service]
  Auth[Auth Service] --> MessageService[Message Service]
  
  MessageService --> DB[(Main DB)]
  MessageService --> AuditDB[(Audit Log DB)]
  
  MessageService --> RealtimeService[Realtime / WebSocket Service]
  RealtimeService --> Client
```

### Part 2 — Sequence Diagram

```mermaid
sequenceDiagram
  participant A as User A
  participant Client
  participant API
  participant Msg as Message Service
  participant AS as Auth Service
  participant DB as Main DB
  participant Audit as Audit DB
  participant RT as Realtime Service
  participant B as User B

  A->>Client: Changes the message text
  Client->>API: PUT /messages/{id} (new_text)
  API->>Msg: editMessage(id, new_text, user_id)
  
  Msg->>AS: Author and time limit verification
  Msg->>Audit: Keeping old text (INSERT)
  Msg->>DB: Text updates and is_edited=true (UPDATE)
  
  Msg->>RT: Publish event "message_edited"
  API-->>Client: 200 OK
  
  RT-->>B: Event "message_edited" (if online)
```

### Part 3 — State Diagram

```mermaid
stateDiagram-v2
  [*] --> Active
  
  state "Message is alive (Active)" as Active {
    [*] --> Sent
    Sent --> Delivered
    Delivered --> Read
    
    Sent --> Edited
    Delivered --> Edited
    Read --> Edited
  }
  
  Active --> DeletedForEveryone
  Active --> DeletedForMe
```

### Part 4 — ADR (Architecture Decision Record)

```markdown
# ADR-001: Strategy for saving edited messages (Mutable State + Audit Log)

## Status
Accepted

## Context
Users should be able to edit sent messages. However, for security and moderation reasons, the system should retain the original message text. (Audit trail). 
Client applications need to load message history quickly without complex calculations.

## Decision
For the `Messages` table, use a mutable approach: when editing, the record is updated with new text (UPDATE) and the `is_edited = true` flag is set. Old versions of messages are stored as immutable records (append-only) in the additional `Message_Audit_Log` table.

## Alternatives
- **Event Sourcing (Immutable only)**: All messages and their changes are stored as events. The client or backend must "glue" the final state. (Deprecated: too difficult to read large chat histories, performance drops).
- **Soft-delete and create new ID**: When editing, the old message is marked as deleted, and a new one is created with the new ID. (Deprecated: breaks the threads of replies that refer to the original ID).

## Consequences
+ **Read Speed**: Clients quickly get the current chat state from a single table without additional JOINs.
+ **Auditability**: Full change history is securely stored for support/moderation purposes.
- **Write Complexity**: Each edit requires two database writes within a single transaction (update `Messages` + insert into `Message_Audit_Log`).
```
