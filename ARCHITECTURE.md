# Architecture Deep Dive

This document explains the **design philosophy** and **architectural trade-offs** behind the Universal Retriever system. For a feature overview and guarantee validation, see [README.md](README.md).

---

## 🎯 Design Philosophy

### Principle 1: Correctness Over Performance
We chose explicit locks over lock-free structures because:
- **Correctness is non-negotiable** in systems handling real data
- Lock-free algorithms are harder to reason about and test
- The performance cost is acceptable (<5% overhead)
- **Trade-off:** Slightly slower, completely safe

### Principle 2: Determinism by Default
We use OrderedDict and canonical serialization to ensure:
- Same input always produces same output
- Enables testing, debugging, and replay
- Makes the system understandable (not "magical")
- **Trade-off:** Slightly more code, perfect reproducibility

### Principle 3: Explicit Over Implicit
Recovery logic is explicit (not automatic) because:
- Prevents silent data loss
- Makes recovery visible in code and logs
- Allows operators to understand what's happening
- **Trade-off:** More boilerplate, more confidence

### Principle 4: Auditability First
Event sourcing means every operation is recorded because:
- Enables complete audit trail
- Supports replay and debugging
- Proves the system did what it claims
- **Trade-off:** More storage, complete observability

---

## 🏗️ Architectural Layers

```
INPUT LAYER
  CLI interface / API / Programmatic access
         ↓
COORDINATION LAYER
  Multi-threaded orchestration
  Thread-safe memory management
  Retriever lifecycle control
         ↓
DATA ACQUISITION LAYER
  Multiple parallel retrievers
  Source-specific adapters
  Error isolation per source
         ↓
PERSISTENCE LAYER
  Event stream recording
  Checkpoint system
  Atomic writes
         ↓
ANALYSIS LAYER
  Event stream processing
  Semantic enrichment
  Classification engine
         ↓
OUTPUT LAYER
  Insights API
  CLI reporting
  Programmatic access
```

**Why This Structure?**
- **Separation of concerns:** Each layer has one responsibility
- **Testability:** Each layer can be tested independently
- **Scalability:** Layers can be replaced/upgraded independently
- **Maintainability:** Clear dependencies and interfaces

---

## 🔒 Thread Safety Strategy

### The Challenge
Multiple retrievers running in parallel, trying to update shared memory graph.

### Our Solution

**1. Explicit Locks**
- Lock acquired before every memory write
- Lock released immediately after update
- Prevents simultaneous writes

```python
# Pseudo-code
with memory_lock:
    memory_graph.record_event(...)  # Safe
```

**2. Atomic Operations**
- Checkpoint writes are atomic (all-or-nothing)
- No partial updates that could corrupt state
- Prevents "halfway written" files

**3. Lock Ordering**
- Always acquire locks in same order (prevents deadlock)
- Clear documentation of who holds which locks
- Stress tested with 100+ concurrent operations

### Why This Works
- ✅ Simple to reason about
- ✅ Easy to test (lock discipline verifiable)
- ✅ Prevents data corruption
- ✅ Performance cost acceptable

### What We Don't Do
- ❌ Lock-free algorithms (too complex to verify)
- ❌ Optimistic locking (requires careful conflict resolution)
- ❌ Distributed consensus (reserved for future scaling)

---

## 🔄 Event Sourcing Pattern

### What It Is
Every operation recorded as immutable event, then events replayed to reconstruct state.

### Why We Use It

**Auditability**
- Complete record of what happened
- Proves system behaved correctly
- Enables investigation of failures

**Reproducibility**
- Replay event stream → identical output
- Enables debugging without live system
- Supports regression testing

**Recovery**
- Checkpoint captures event stream position
- On crash: replay from checkpoint
- No data loss (all events already written)

### Example Flow
```
User: "Retrieve card_ids 1, 2, 3"
         ↓
System records event: {fetch_start, [1,2,3], timestamp}
         ↓
Retrievers work, record event per item: {fetch_complete, 1, latency, timestamp}
         ↓
System records event: {analysis_start, timestamp}
         ↓
Analysis complete, record: {analysis_complete, results, timestamp}
         ↓
Checkpoint created: {event_count: 5, hash: xxx, timestamp}
         ↓
(If crash happened here...)
         ↓
Recovery: Load checkpoint, replay events 1-5, continue from event 6
```

### Trade-offs
- ✅ Complete auditability
- ✅ Perfect reproducibility
- ✅ Lossless recovery
- ❌ Higher storage usage
- ❌ Slightly more complex code

---

## 🎯 Semantic Intelligence Design

### Problem
Raw metrics don't tell you what to do:
```
Error rate: 15%
Latency: 2.3s
Success: 85%
```

"Should I panic? Should I adjust concurrency? Should I skip this source?"

### Solution
Classify metrics into actionable states:

```
HEALTHY → "Everything fine, maintain current load"
DEGRADED → "Something's wrong, investigate"
UNSTABLE → "Probably temporary, increase retry"
FAILING → "Source is down, bypass or wait"
```

### Implementation

**1. Collect Metrics**
- Count successes and errors per retriever
- Track latency distribution
- Record error categories

**2. Compute Statistics**
- Success rate = successes / (successes + errors)
- Average latency = sum(latencies) / count
- Error proportion = errors / total

**3. Apply Classification Rules**
```
IF reliability >= 95% AND latency < 1.0s:
    state = "STABLE"
ELIF reliability >= 85% AND latency < 2.0s:
    state = "HEALTHY"
ELIF reliability >= 70%:
    state = "DEGRADED"
ELIF reliability >= 50%:
    state = "UNSTABLE"
ELSE:
    state = "FAILING"
```

**4. Generate Recommendations**
```
IF state == "FAILING":
    alert = "🔴 CRITICAL"
    recommendation = "Source is down, bypass it"
ELIF error_rate_increasing:
    alert = "🟠 WARNING"
    recommendation = "Error spike detected, reduce concurrency"
ELIF latency_increasing:
    alert = "🟡 INFO"
    recommendation = "Latency trend up, consider backoff"
```

### Why This Works
- Moves from observation to action
- Reduces manual decision-making
- Works with real data from real systems

---

## 🔐 Determinism Guarantees

### Challenge
Multi-threaded systems are inherently non-deterministic. How do we make ours deterministic?

### Solution

**1. OrderedDict**
- Python dict insertion order is only guaranteed since 3.7
- OrderedDict is explicit about ordering
- Ensures consistent serialization

**2. Canonical Serialization**
- JSON keys always in same order
- No floating-point representation variance
- Same input always produces same JSON

**3. Single-Writer Event Log**
- Only one thread appends to event log
- Prevents interleaved writes
- Enables reliable checkpointing

### Validation
Test strategy: 
1. Run retrieval N times
2. Compute hash of output each time
3. Verify all hashes identical
4. Tests prove determinism

---

## 💾 Checkpoint & Recovery Design

### Problem
System crashes mid-operation. How do we recover without losing data?

### Solution

**Checkpoint Manifest**
```json
{
  "checkpoint_id": "cp_12345",
  "timestamp": "2026-01-29T14:32:01Z",
  "event_count": 1250,
  "hash": "a3b2c1d0...",
  "retriever_states": {
    "scryfall_01": {"status": "healthy", "last_event": 1250}
  },
  "recovery_info": {"resume_from_event": 1250}
}
```

**Recovery Process**
1. Load checkpoint manifest
2. Verify hash of events up to checkpoint
3. Start from `resume_from_event`
4. Continue normal operation

**Why It Works**
- ✅ Atomic write (manifest all-or-nothing)
- ✅ Hash validation (detect corruption)
- ✅ Explicit resume point (no guessing)
- ✅ No data loss (all events already persisted)

### Trade-offs
- ✅ Lossless recovery
- ✅ Verifiable correctness
- ❌ Requires explicit checkpoint calls
- ❌ Slightly more complex code

---

## 🚀 Scalability Roadmap

### Current: Single-Node + Small Federation
- All coordination in-process
- Memory graph on local disk
- Suitable for 1-10 retrievers

### Future: Distributed System
**Option 1: Message Queue**
- Add RabbitMQ/Kafka for inter-process communication
- Coordinator becomes stateless (horizontal scalable)
- Memory graph moves to distributed storage

**Option 2: Event Stream**
- Export events to streaming platform
- Multiple consumers can process independently
- Event log becomes system of record

**Option 3: Hybrid**
- Local memory graph for fast queries
- Distributed event log for durability
- Best of both worlds

### Extensibility Points
The current design enables future scaling because:
- Event log is already designed to be distributed
- Coordinator state is already isolated
- Memory graph operations are already atomic
- No implicit dependencies on single-node execution

---

## 🧪 Testing Strategy

### Unit Tests
- Individual components in isolation
- Mock external dependencies
- Focus: correctness of logic

### Integration Tests
- Multiple components together
- Real event flow
- Focus: interactions between layers

### Stress Tests
- Concurrent operations
- High throughput scenarios
- Focus: thread safety, no data loss

### Determinism Tests
- Run same scenario multiple times
- Compute hash of outputs
- Verify identical results
- Focus: reproducibility

### Edge Case Tests
- One retriever fails → others continue
- All retrievers fail → graceful degradation
- Crash during checkpoint write → recovery works
- Config schema violation → early rejection
- Focus: robustness

---

## 🎯 Design Trade-offs Summary

| Design Choice | Benefit | Cost | Why We Chose It |
|---------------|---------|------|-----------------|
| Explicit locks | Thread safety | Slight perf | Correctness > speed |
| Event sourcing | Auditability + replay | Storage | Debugging + recovery |
| OrderedDict | Determinism | Code | Reproducibility |
| Explicit recovery | Safety | Boilerplate | No silent failures |
| Semantic classification | Actionable insights | Complexity | Ops simplification |
| Private code | Responsible disclosure | Access control | Professional judgment |

---

## 🔗 Related

- **Feature Overview:** [README.md](README.md) — What it does and why
- **Proof & Evidence:** See README.md "Proof & Evidence" section
- **Implementation:** Private portfolio repo (available on request)

---

**Philosophy:** Architecture should be understandable, testable, and defensible. Every design choice should have a reason.

