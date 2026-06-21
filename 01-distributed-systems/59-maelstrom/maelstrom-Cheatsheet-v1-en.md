# jepsen-io / maelstrom (Distributed Systems Verification & Fault-Tolerance Testbed)

## M1: Clojure Runtime & Kernel Topology

### Card 1: Clojure JVM Runtime
- **Core Mechanism**: Maelstrom is written in Clojure and runs on the JVM. It spawns target database executables as subprocesses and controls the virtual network topology between them.
- **Memory Trade-off**: JIT compilation speeds up execution but introduces large startup latencies and memory footprint (min 2GB heap recommended).
- **JVM Tuning**: Use `-Xmx4g` to limit maximum heap size and `-server` to optimize execution runtime JIT compiler path.

### Card 2: STDIN/STDOUT Protocol
- **Communication Protocol**: Inter-process communication between Maelstrom and nodes is mediated via standard input/output (stdin/stdout). Every message is a single-line JSON terminated with `\n`.
- **Deadlock Warning**: Testing nodes must never write unstructured debugging logs to stdout (e.g., `println`), as this corrupts JSON parse streams. All debug outputs must go to stderr.
- **Diagnostic Method**: Monitor stderr for crash dumps, or run `tail -f store/latest/node.log`.

### Card 3: Node-Node Communication
- **Virtual Network**: Maelstrom intercepts and redirects all node-to-node packets. Although nodes believe they write to standard socket interfaces, packets are routed in-memory by Maelstrom.
- **Message Schema**: Composed of `src`, `dest`, and `body` map containing `type` and `msg_id`.
- **Trade-off**: Zero socket network overhead enables scaling up to 100 nodes on a single machine, but misses physical OS network layer bugs (e.g., TCP connection timeouts).

### Card 4: Loop & State Concurrency
- **State Model**: State sharing inside Maelstrom leverages Clojure Agents, Refs, and Atoms. It processes incoming stdout byte streams asynchronously.
- **Synchronization**: Atom handles lock-free CAS state accumulations; Software Transactional Memory (STM) updates cluster topologies atomically.
- **Limit**: Concurrency is bounded by host CPU cores and JVM thread quotas. Limit node counts to 20 per host for scheduling precision.

### Card 5: JSON Schema RPC
- **RPC Protocol**: RPC requests specify `type` and `msg_id`. Response messages must reference `in_reply_to` pointing to the request's `msg_id`.
- **Errors**: Failures return standard JSON error blocks specifying `error_code` (e.g., 10: Not Supported, 14: Txn Conflict).
- **Trade-off**: Text-based JSON encoding incurs significant CPU serialization overhead, but guarantees language-agnostic integration for nodes (Python, Go, Rust, etc.).

---

## M2: Jepsen Engine & Test Harness

### Card 6: Knossos Linearizability
- **WGL Algorithm**: Knossos runs the Wing & Gong (WGL) algorithm to search for a valid total order of history events, checking if it complies with linearizability.
- **State Space Explosion**: High concurrency and write contention cause exponential search space growth during history verification.
- **Optimization**: Impose `--time-limit` to preempt search timeouts and prune non-overlapping concurrent segments to prevent JVM OutOfMemory (OOM) crashes.

### Card 7: Elle Transaction Checker
- **Dependency Graph**: Elle builds execution graphs containing read/write dependencies (WW, WR, RW edges) for multi-key transactions.
- **Cycle Detection**: Cycles in the dependency graph signify consistency violations (e.g., G1a: Dirty Write, G-single: Read Skew).
- **Feature**: Supports verification of complex multi-key isolation levels (e.g., Serializability) where Knossos (single-key) is inapplicable.

### Card 8: Generator Pipeline
- **Workload Pipeline**: Defines client operations generated during the test run (e.g., issuing random read/write transactions).
- **Concurrency**: Governed by client thread pools. Events are dispatched at a configurable rate (QPS).
- **Trade-off**: Custom workloads can target specific bottlenecks (e.g., hot key contention) but might miss edge cases found in random production traffic.

### Card 9: Checker Engine
- **Workflow**: Maelstrom saves all execution histories to local files and triggers the Checker once the test phase completes.
- **Output**: Generates `results.edn` containing pass/fail metrics, anomaly analysis, and counterexample sequences.
- **Manual Debug**: Parse history records directly using Clojure tools: `clojure -m jepsen.store`.

### Card 10: Client Proxy Concurrency
- **Thread Mapping**: Every client proxy in Maelstrom maps to an independent OS thread.
- **RPC Timeouts**: Defaults to 5000ms. Unanswered requests are logged as `info` (indeterminate status) and evaluated by checker logic to see if they took effect.
- **Trade-off**: High thread counts increase load but escalate context-switching overhead, degrading timer resolution.

---

## M3: Consistency Models & Verifications

### Card 11: Causal Consistency
- **Partial Order**: Demands that causally related writes are observed in the same order by all nodes. Concurrent writes with no causal relation may be observed in different orders.
- **Implementation**: Utilizes vector clocks or Lamport logical timestamps attached to messages.
- **Trade-off**: Avoids global coordination and remains responsive during network partitions, but allows reading stale values.

### Card 12: Read Uncommitted / Committed
- **Isolation Distinctions**:
  - **Read Uncommitted**: Allows transactions to read uncommitted changes (dirty reads).
  - **Read Committed**: Restricts transactions to only read committed values.
- **Elle Check**: Cycle detection flags $G1a$ (dirty writes) and $G1b$ (dirty reads) anomalies.
- **Cost**: Read Committed requires write locks or visibility checking using Active Transaction Lists in MVCC.

### Card 13: Repeatable Read
- **Read Guarantee**: Successive reads of the same key within a transaction must return the same value (prevents non-repeatable reads).
- **Phantom Reads**: Maelstrom maps phantom reads through multi-key range tracking since it only hooks explicit key operations.
- **Cost**: Transactions must hold shared locks or execute under Snapshot Isolation (SI) epochs.

### Card 14: Serializability
- **Definition**: The execution of transactions behaves as if they were executed in some global serial order.
- **Elle Analysis**: Execution dependency graphs must not contain cycles with any mixture of RW, WR, and WW dependency edges (known as $G2$ anomalies).
- **Trade-off**: Highest consistency level, but causes transaction abort storms and high retries under heavy concurrent write loads.

### Card 15: Monotonic Read/Write
- **Session Guarantees**:
  - **Monotonic Reads**: If a client reads version V1, subsequent reads will never return an older version.
  - **Monotonic Writes**: Guarantees a client's writes are processed in the order they were submitted.
- **Implementation**: Pin client sessions to specific replicas, or maintain a local timestamp watermark on the client.

---

## M4: Distributed Storage & Services

### Card 16: Lin-KV Strong Consistency
- **Interface**: Maelstrom exposes a shared key-value store named `lin-kv`.
- **Mechanism**: Nodes leverage this HTTP/JSON endpoint to persist state. `lin-kv` behaves as a linearizable repository running on Jepsen's consensus layer.
- **Trade-off**: Extremely useful for building stateless services, but introduces high network roundtrip latency.

### Card 17: Seq-KV Sequential Consistency
- **Definition**: Guarantees that all writes appear in a single global order observed by all nodes, but does not guarantee real-time visibility.
- **Trade-off**: Offers lower write latencies than `lin-kv`, but reads can return stale data during transient partitions.
- **Best Use**: Ideal for global configurations where millisecond-level propagation delay is acceptable.

### Card 18: Lww-Register Conflict Resolution
- **CRDT Merge**: Last-Write-Wins Register is a state-based CRDT. Concurrent write conflicts are resolved by keeping the write with the highest physical timestamp.
- **Clock Skew Danger**: If physical clocks drift between nodes, logically later writes can be overwritten by stale writes with skewed forward timestamps.
- **Mitigation**: Use logical or hybrid logical clocks (HLC) instead of raw physical system time.

### Card 19: PN-Counter State Merge
- **CRDT Counter**: Positive-Negative Counter allows local, lock-free additions (P) and subtractions (N).
- **Merge Logic**: Replicas broadcast local P/N arrays. State merge performs an element-wise maximum check: `max(P[i], P_remote[i])`.
- **Benefit**: Replicas automatically converge post-partition without acquiring distributed locks.

### Card 20: Replicated Log
- **Replication Mode**: Nodes implementing custom storage must manage log synchronization. Choose between synchronous (slow, safe) and asynchronous (fast, stale).
- **Split-Brain Mitigation**: Logs must obtain a majority quorum (i.e., `N/2 + 1`) consensus before committing to state machines.

---

## M5: Network Chaos Injection & Fault Tolerance

### Card 21: Network Partition
- **Failure Model**: Nemesis splits the network into isolated zones (e.g., partitioning a 5-node cluster into `{n1, n2}` and `{n3, n4, n5}`).
- **Log Signature**: Appears in output logs as `:info :nemesis :start :partition`.
- **Validation Goal**: Verifies that the majority partition accepts writes while the minority partition rejects or blocks writes.

### Card 22: Packet Drop Simulation
- **Packet Discard**: Maelstrom drops packets between nodes based on a configured probability.
- **Parameter**: `--drop-rate 0.1` discards 10% of inter-node messages at random.
- **Test Goal**: Verifies that nodes implement timeouts and retries, failing which packet loss results in deadlocks or state loss.

### Card 23: Latency Inflation & Reordering
- **Congestion Simulation**: Simulates queue delays by inflating message delivery time.
- **Parameters**: `--latency 100` adds 100ms base latency; `--delta 50` adds jitter, resulting in packet reordering.
- **Challenge**: Nodes must handle delayed, out-of-order packets safely (typically using unique message IDs).

### Card 24: Nemesis Coordinator
- **Fault Manager**: The central orchestrator inside Jepsen that schedules failure injections.
- **Strategy**: Configured via time-intervals (e.g., partition for 5s, restore for 10s).
- **Command**: Run test with `--nemesis partition` and `--nemesis-interval 10` to configure chaos schedules.

### Card 25: Recovery & Reconnection
- **Route Restoration**: When Nemesis heals a partition, Maelstrom restores the routing tables, allowing direct communication.
- **State Recovery**: Reconnected minority nodes must pull logs from the Leader (catch-up) and discard uncommitted/aborted locks.

---

## M6: Diagnostics & Performance Tuning

### Card 26: Gnuplot Latency & Throughput Graphs
- **Reporting**: Maelstrom triggers Gnuplot to render performance graphs after each run.
- **Key Files**:
  - `latency-raw.png`: Scatter plot tracking response times over the test duration (blue represents success, red denotes failures).
  - `throughput.png`: Time-series graph of transaction commit rates.
- **Debugging**: Extended red clusters during partition indicate recovery or timeout tuning failures.

### Card 27: History Log Analysis
- **Log Path**: Stored under `store/latest/history.txt`.
- **Format**: Tracks client actions with `:process`, `:type` (e.g., `:invoke`, `:ok`, `:fail`), `:f` (function called), and `:value`.
- **Debugging Tip**: When linearizability checks fail, trace values in `history.txt` to find which process read a stale value.

### Card 28: CLI Parameter Reference
- **Execution Command**:
  `./maelstrom test -w txn-list --bin ./my_node --node-count 5 --time-limit 30 --rate 100 --nemesis partition`
- **Core Parameters**:
  - `--node-count`: Number of simulated nodes.
  - `--rate`: Target requests per second.
  - `--concurrency`: Number of client threads.
  - `--key-count`: Set size of target keys (smaller size increases conflict rate).
