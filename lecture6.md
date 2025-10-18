# Lecture 6 – Logical Clocks and Ordering in Distributed Systems

## 1. Recap: MapReduce and Stateless Computation

- **MapReduce** is a simple but powerful model that splits computation into:
  - **Map:** Convert inputs into intermediate key–value pairs.
  - **Reduce:** Combine values with the same key into a single output.
- Many large-scale problems fit this framework because:
  - The system provides **scalability** and **fault tolerance** automatically.
  - Workers are **stateless**: each job’s output depends *only* on its input.
- **Constraint:** All `map` tasks must finish before any `reduce` starts.

**Key properties of MapReduce:**
- Stateless workers.
- No concurrent interdependencies.
- Built-in load balancing and failure recovery.

---

## 2. Stateful Computations and Finite State Machines (FSMs)

- Some computations depend on **past inputs** — these are **stateful**.
- Such systems can be modeled as **deterministic finite state machines (FSMs)**:
  - Computation = states + transitions.
  - Each transition is deterministic (same next state for same input/state).

### Replication for Fault Tolerance
- To tolerate failures, we replicate the FSM across nodes.
- Replicas must communicate and stay **consistent**, but:
  - Communication is **delayed** and **asynchronous**.
  - Hence, replicas are **temporally separated**.

### Goal
Ensure all replicas **eventually agree** on the same sequence of updates.

---

## 3. The Need for Ordering

To maintain consistency, replicas must:
- See **the same updates in the same order**.
- Use a consistent ordering mechanism independent of real clocks.

### Why Not Use Wall-Clock Time?
- **Clock drift** violates monotonicity (time should always move forward).
- Synchronizing clocks is approximate — not guaranteed.
- Real-time ordering can be ambiguous due to network delay.

---

## 4. Causality and Observability

- **Causality:** Event A *causes* event B if A could influence B’s input.
  - Example: “Deposit $100” must occur before “Check balance.”
- **Concurrency:** If A cannot influence B and vice versa, they are concurrent.

**We only care about the order that an *observer* can detect.**

### Example
- You deposit money → system confirms.
- Any later query to *any replica* must reflect that deposit.
- If it doesn’t, time appears to “go backward.”

---

## 5. Logical Clocks (Lamport Clocks)

We track event order using **logical clocks**, not real time.

### Properties
- Preserve **causality**: if A → B, then `T(A) < T(B)`.
- May over-order events (that’s fine — it doesn’t affect correctness).

### Definition
For events A, B:
- If A happened before B (A → B), then `T(A) < T(B)`.
- If `T(A) < T(B)`, it doesn’t necessarily mean A → B.

---

## 6. Formal “Happens-Before” Relation

A “happens-before” relation (→) is defined by:

1. **Process order:**  
   Within the same process, if A occurs before B, then A → B.

2. **Message order:**  
   If event A is the **send** of a message and B is the **receive**, then A → B.

3. **Transitivity:**  
   If A → B and B → C, then A → C.

This ensures a **partial order** over all events.

---

## 7. Assigning Logical Clock Values

Each process maintains its own **clock `C_i`**.

Rules:
1. **Initialization:** `C_i = 0` at process start.
2. **Internal events:**  
   For each event, increment the local clock:  
   `C_i := C_i + 1`
3. **Send event:**  
   When sending a message, attach current `C_i` to it.
4. **Receive event:**  
   On receiving a message with timestamp `t_m`:  
   `C_i := max(C_i, t_m) + 1`

Result: For any A → B, `C(A) < C(B)`.

---

## 8. Example Walkthrough

Process 1 and Process 2:
- `A (1)` → internal → `B (2)` → send → `C`
- Message from B to C carries timestamp 2.
- Receiver sets `C := max(local, received) + 1`.

Thus, timestamps progress causally across processes.

These timestamps are **Lamport (logical) clocks**.

---

## 9. Interpreting Concurrent Events

Two events can be **concurrent** if:
- Neither can causally affect the other.
- Order doesn’t matter — observers can’t tell the difference.

Example:
- Event F (timestamp 3) occurs before B (timestamp 2) in wall-clock time.  
  Still fine if F and B are independent — **no causal violation**.

---

## 10. Key Takeaways on Observability

- Distributed systems only need to satisfy **observable correctness**.
- It doesn’t matter what “really happened,” only what observers can tell.

> **Rule:** If no external observer can tell the difference, both orders are valid.

---

## 11. Replicated State Machines (RSMs)

- Each replica runs the same deterministic state machine.
- To ensure consistency:
  - Apply **the same ordered sequence of events** at every replica.
- If events A and B are concurrent:
  - Either order is valid, but **all replicas must choose the same one**.

---

## 12. Generating a Total Order

To get a **total order** from Lamport clocks:

1. No two events in the same process can share a timestamp.
2. For events in different processes with equal timestamps:
   - Break ties using a unique **process ID**.

### Combined Timestamp
`T = (C_i, ID_i)`  
Order lexicographically:
- Compare `C_i` first.
- If equal, compare `ID_i`.

This ensures a **global, deterministic ordering** across replicas.

---

## 13. Unique Identifiers (Process IDs)

- Each process gets a unique ID (e.g., via hardware **MAC address**).
- This guarantees tie-breaking consistency.
- Every event can be represented as `(timestamp, process_id)`.

---

## 14. Applying Global Order

- Each operation is tagged with its **Lamport timestamp**.
- Replicas apply operations in order of these timestamps.
- Ensures all replicas see updates in the same sequence.

---

## 15. Why Logical Time Matters

Logical clocks guarantee:
- All **causally related** events are ordered correctly.
- All **replicas** maintain the same causal history.
- No observer sees “time reversal” or inconsistent updates.

---

## 16. Real-World Example: Social Media Scenario

- User posts “Block manager” → later posts “I hated my internship.”
- If server processes the second event before the first, results differ.
- Respecting Lamport clocks ensures causal order is preserved:
  - Block action (`A`) happens before post (`C`).

---

## 17. Reflection: What Makes Distributed Systems Hard

- The challenge is not knowing the *real order*, but enforcing a *consistent one*.
- **Key insight:** “If no one can observe a difference, it’s not wrong.”

> “The way to solve problems in distributed systems is to decide what you don’t need to know.”

---

## 18. Philosophical Takeaway

> “In times of great change, learners inherit the earth, while the learned find themselves beautifully equipped for a world that no longer exists.”

Distributed systems challenge our intuition:
- Events don’t have one universal order.
- “Correctness” is about *observable causality*, not *absolute time*.

---

# Summary Table

| Concept | Definition / Rule | Example |
|----------|------------------|----------|
| Stateless Worker | Output depends only on input | MapReduce mapper |
| Stateful System | Depends on past messages | Key-value store |
| Happens-Before (→) | A influences B | Message send → receive |
| Concurrency | A ↮ B, neither influences other | Two independent tasks |
| Logical Clock Rule | Increment, send, receive rules | `C_i := max(C_i, recv)+1` |
| Lamport Clock | Logical time preserving causality | Used in RSM ordering |
| Total Order | `(timestamp, process_id)` | Break ties deterministically |
