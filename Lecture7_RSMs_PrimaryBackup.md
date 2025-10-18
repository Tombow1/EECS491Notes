# Lecture 7 – Replicated State Machines (RSMs) and Primary/Backup Replication

## 1. Recap: Logical Clocks and Causality

### Logical Clocks Overview
- **Goal:** Preserve the observable order of events across a distributed system.
- **Lamport Clock Rules:**
  1. Within a process, events occur in order: `A → B`.
  2. A message send must happen before its receive: `send → receive`.
  3. Transitive closure applies: if `A → B` and `B → C`, then `A → C`.

### Implementation Rules
1. Each process maintains a local clock `Ci`.
2. Before each local event, increment:  
   `Ci := Ci + 1`
3. When sending a message, attach `Ci`.
4. When receiving a message with timestamp `Cm`:  
   `Cj := max(Cj, Cm) + 1`
5. Break ties by **node ID** to form a global order `(timestamp, ID)`.

### Consequences
- Lamport “clock” doesn’t measure *real* time, only *causal* time.
- Clocks can drift arbitrarily if nodes don’t communicate.
- As long as the global order is consistent and deterministic, it’s valid.

---

## 2. Applying Logical Clocks to Replicated State Machines (RSMs)

### Problem Setup
- Each replica must apply all updates **in the same order**.
- Assume for now:
  - No node failures.
  - Reliable, in-order message delivery.

### RSM Rules Using Lamport Clocks
- Each operation `O` has a logical timestamp `T`.
- A replica can apply `O` once it knows:
  - All other replicas have advanced to at least `T`.
  - Therefore, all earlier updates `U → O` are known.

**How does a replica know others are caught up?**
- It has received messages from all other replicas with timestamps ≥ `T`.

---

## 3. Example: Two Replicas

| Replica | Operation | Timestamp | Knowledge of Peer |
|----------|------------|------------|------------------|
| R1 | Deposit $100 | 3.1 | Knows R2 ≥ 0 |
| R2 | Pay interest | 8.2 | Knows R1 ≥ 0 |

**Message Exchange:**
- R1 sends update to R2 (`send = 4.1`).
- R2 sends update to R1 (`send = 9.2`).
- On receive:
  - R1 updates to `C1 = 10.1`
  - R2 updates to `C2 = 10.2`

**Application Rule:**  
Apply the *head* of the pending update queue only when all known clocks ≥ its timestamp.

---

### Outcome
- R1 applies deposit first (11.1), then interest (12.1).
- R2 applies deposit later, but in the same order.
- Both replicas converge on the same sequence of state transitions.

---

## 4. Handling Waits and Heartbeats

### Problem: Waiting Indefinitely
- A replica can’t apply an update if it hasn’t heard from all peers.
- If a peer is idle or silent, progress stops.

### Solution: Heartbeat Messages
- Periodic “proof-of-life” messages exchanged among replicas.
- No data payload; just advances clocks to confirm activity.

### Optional Optimizations
- Allow temporary inconsistencies with **undo logs** for rollback.
- Used in systems where weak consistency is acceptable (e.g., social media feeds).

---

## 5. Why Lamport Clocks Alone Aren’t Enough

### Limitations
- If any node fails or stops responding, **no progress can be made**.
- The system becomes blocked until that node recovers.
- Failures can last an **unbounded** amount of time → system is unusable.

> Lamport clocks ensure consistency but not availability.

---

## 6. Toward a Better Solution: Primary/Backup Replication

### Motivation
We need a system that:
- Continues making progress even if one node fails.
- Still maintains external consistency (clients never see time “go backward”).

---

## 7. Primary/Backup Architecture

### Design
- **Primary:** Handles client requests.
- **Backup:** Receives updates from the primary.
- Client sees the system as a single logical server.

```
Client → Primary → Backup
          ↘︎ (sync)
```

### Goals
- Survive one node failure.
- Ensure updates visible to clients are reflected on both primary and backup.

---

## 8. Handling Failures

| Failure | Response |
|----------|-----------|
| **Primary fails** | Promote backup to new primary. |
| **Backup fails** | Recruit new backup and synchronize. |

**Redundancy Rule:**  
If either node fails, the other continues serving clients.

- If there are additional standby nodes, recovery is even faster.
- Availability improves exponentially with each additional replica.

---

## 9. Synchronization Timing

### Question: When should the primary synchronize?
Example from **MapReduce Manager** pseudocode:

```go
RegisterServer() {
  while (1) {
    receive msg and parse addr
    pick task to assign
    mark task as assigned
    respond with task assignment
  }
}
```

Options:
1. **Before responding to the worker:**  
   Ensures backup is consistent before acknowledgment.
2. **After responding:**  
   Improves latency but risks inconsistency if primary crashes.

---

## 10. Handling Message Loss and Retries

### Example: Worker → Manager
- Worker: “Done with task 1.”
- Primary assigns task 2.
- Primary fails before responding.
- Worker retries.

**Why this works:**  
- Even single-server systems must handle **duplicate requests**.
- Therefore, the replicated system just needs to remember past replies (idempotence).

---

## 11. External Consistency

**Rule:**  
Any state visible to the client must be replicated.

- **Before** an update becomes visible, it must exist on both primary and backup.
- **Temporary inconsistency** between primary and backup is okay if invisible externally.

**Informal definition:**
> Everything a client knows about the primary must also be known by the backup.

---

## 12. Vector Clocks (Preview for Next Lecture)

### Why Vector Clocks?
- Lamport clocks impose total order but over-order causally independent events.
- Vector clocks preserve **partial order** more precisely.

### Rules
1. Each process `i` maintains a vector `V[i]` of size `n` (one entry per process).
2. On each local event, increment `V[i]`.
3. On receiving message with vector `M`:
   - For all components `k`: `V[k] = max(V[k], M[k])`
   - Then increment local `V[i]`.

### Happens-Before Condition
`V(a) < V(b)` if and only if:
- For all `k`, `V(a)[k] ≤ V(b)[k]`
- And for at least one `k`, strict inequality holds.

### Concurrency Condition
Two events are concurrent if:
- `V(a)[i] < V(b)[i]` and `V(a)[j] > V(b)[j]` for some i, j.

---

## 13. Key Takeaways

| Concept | Insight |
|----------|----------|
| **Lamport Clocks** | Guarantee causal ordering, but not availability. |
| **RSM via Logical Clocks** | Works if messages never drop and nodes never fail. |
| **Primary/Backup** | Introduces fault tolerance by replication. |
| **External Consistency** | Clients never see state “go backward.” |
| **Vector Clocks** | Next evolution: capture causality precisely, avoid over-ordering. |

---

# Summary Diagram

```
                ┌────────────┐
                │   Client   │
                └─────┬──────┘
                      │
              ┌───────▼────────┐
              │    Primary     │
              │  (handles req) │
              └───────┬────────┘
                      │
             ┌────────▼────────┐
             │     Backup      │
             │ (keeps copy)    │
             └─────────────────┘
```

**Failure Recovery:**  
If primary fails → backup takes over.  
If backup fails → recruit new one.
