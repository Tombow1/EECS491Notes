# EECS 491 — Lecture 06: Logical Clocks

> A little “not knowing” makes some things easier — we only need to preserve orders that **matter** (causal), and we can choose convenient orders for events that **don’t**.

---

## Learning objectives
- Understand **happens-before** (→) and **concurrency** (∥) in distributed systems.  
- Construct and reason about **Lamport (logical) clocks** that preserve causality.  
- Extend logical time to a **total order** usable by replicas.  
- See how logical clocks support **replicated state machines (RSMs)** and user-facing consistency.

---

## Last time (recap)
- **MapReduce**: stateless workers; same input ⇒ same output; all `map()` must finish before any `reduce()` start.  
- **Replicated state**: stateful computations depend on message **plus** (some set of) past messages; replicas must eventually agree ⇒ need a **single order** of updates.  
- **Wall-clock is unreliable** for ordering: drift, non-monotonicity, and message delays.

---

## What is a distributed system?
A collection of distinct processes that: (1) are spatially separated, (2) communicate by **messages**, (3) have non-negligible, variable **delays**, and (4) **do not share fate**.

---

## Motivating example (project partners)
- Me: “I’ll fix `foo()` if you fix `bar()`.” You: “OK.”  
- You later: “I fixed `bar()`.”  
- Me: “Great, I fixed `foo()`.”  
Key point: You **can’t tell** whether I fixed `foo()` *before* or *after* your “I fixed bar” message—only that when I *say* it’s fixed, you may **rely** on it thereafter.

---

## Happens-before and concurrency
- **Happens-before (A → B)**: It was **possible** for A’s effects to influence B (directly or transitively).  
- **Concurrent (C ∥ D)**: Neither C → D nor D → C; their relative order **doesn’t affect** externally observable correctness.
- We want timestamps **T(e)** such that: if **A → B**, then **T(A) < T(B)**. (The **converse need not hold**.)

### Three event types
1. Internal event (within a process).
2. **Send** of a message.
3. **Receive** of a message (paired with exactly one send, assuming reliable delivery here).

### Formal rules of →
1. **Process order**: within one process, earlier event a precedes later event b ⇒ **a → b**.  
2. **Message order**: send b precedes its receive c ⇒ **b → c**.  
3. **Transitivity**: if **a → b** and **b → c**, then **a → c**.

---

## Lamport (logical) clocks
Each process **i** keeps an integer clock **Cᵢ**.

**Algorithm (per event e at process i):**
1. **Before** executing any event at i: `Cᵢ ← Cᵢ + 1`  
2. **When sending** message m: include timestamp `ts = Cᵢ(m_send)`.  
3. **When receiving** message m with timestamp `ts`:  
   `Cᵢ ← max(Cᵢ, ts) + 1`  
   (Then timestamp the receive event with this new `Cᵢ`.)

**Clock condition:** If **a → b**, then **C(a) < C(b)** (by construction).  
**Not conversely:** `C(x) < C(y)` **does not imply** `x → y` (x and y may be concurrent).

---

## From partial to total order
Logical clocks give a **partial order** (respecting causality). For RSMs, we often need a **total order**:
- Tie-break equal logical times by **node ID**.  
- Stamp each event e at process i as a pair `(C(e), i)` and order lexicographically:
  - `(t₁, i) < (t₂, j)` if `t₁ < t₂`, or (`t₁ == t₂` **and** `i < j`).  
- Note: Two events **within one process** can’t share the same `C(e)`, so ties only arise **across** processes.

This yields a **global total order** that (a) **respects causality** and (b) **decides** an order for concurrent events deterministically.

---

## Applying to Replicated State Machines (RSMs)
- Deterministic FSM: state + input ⇒ next state. Replicas must apply **the same transitions in the same order**.  
- With logical clocks:
  - Assign each operation **O** a time **T(O)**.  
  - A replica can safely apply O once it knows **all operations U with U → O** have been (or will be) placed **before** O: i.e., it has seen messages confirming peers’ clocks ≥ **T(O)**.  
  - **Heartbeats** (or any message) help disseminate clock advancement, even when no user ops are in flight.

**Why this matters (user example):**  
A: “Block manager.” B: “Done.” C: “Post: I hated my internship.”  
Respecting logical time ensures **A → B → C**, so the post won’t be visible to the manager if A preceded C.

---

## Subtleties & caveats
- Logical clocks **preserve causality** but may still impose an arbitrary order on **concurrent** events—acceptable because observers **cannot detect** a difference.  
- They do **not** measure real time; they measure **event precedence**.  
- To model failures, partitions, and losses, we’ll build on this foundation (e.g., vector clocks, consensus).

---

## Worked mini-example (message receive)
Suppose process P₁ sends to P₂ while both start at 0:  
- P₁: internal event → `C₁=1`; send m → `C₁=2` (message carries `ts=2`)  
- P₂: receives m with `ts=2` while `C₂=0` ⇒ `C₂ ← max(0,2)+1 = 3` (receive event time 3)  
Thus **send(2) → recv(3)**, satisfying the clock condition.

---

## Pseudocode (per process i)
```text
C_i := 0

on_internal_event():
  C_i := C_i + 1
  timestamp(event) := C_i

on_send(msg):
  C_i := C_i + 1
  msg.ts := C_i
  send(msg)

on_receive(msg):
  C_i := max(C_i, msg.ts) + 1
  timestamp(receive_event) := C_i
```

---

## Key takeaways
- **Causality first**: Only preserve orders that external observers could detect.  
- **Lamport clocks**: Simple, local, and guarantee `a → b ⇒ C(a) < C(b)`.  
- **Total order**: Break ties with node IDs to drive identical ordering at all replicas.  
- This is the backbone for **ordering** in fault-tolerant, replicated systems.

---

### References
- EECS 491, Lecture 06 slides and in-class transcript.
