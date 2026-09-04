# Clavem — a two-year plan: zero Go to a transactional distributed database

One project, two years, two identities.

**Year 1** builds a Redis: durable, Raft-replicated, RESP-compatible, verified.
**Year 2** replaces the substrate: LSM storage, MVCC, transactions, range partitioning, distributed commit.

Nothing from year one is thrown away. The RESP layer becomes one of two APIs.

**Phases are gated by exit criteria, not dates.** Week numbers are pacing estimates. Buffer is real and rolls forward.

---

## Naming

| Thing | Name |
|---|---|
| Server | `clavemd` |
| CLI / debugger | `clavem` (alias `clv`) |
| Redis-compatible protocol | CLASP — Clavem Serialization Protocol, package `clasp` |
| Transactional protocol | gRPC, package `clavempb` |
| Compatibility oracle | **Valkey 8** — a pinned version, not "Redis" in the abstract |
| Data directory | `.clavem/` |
| Cluster | the **ring** |
| Node | a **ward** |
| Partition | a **slot range** (Y1, hash) → a **range** (Y2, ordered) |

CLASP **v0** is RESP2 byte-for-byte plus a reserved `CLV.*` namespace. **v1** (Phase 5) adds request IDs, redirects, replication metadata, negotiated via `HELLO`.

---

# Debuggability — the cross-cutting thread

You cannot step through a bug that only appears when ward 3 is partitioned during a range split. Breakpoints are useless when the bug *is* the interleaving. So the debugging affordances live inside the system, and they get built alongside the features they inspect — never after.

**Four pillars, in order of power:**

**1. Deterministic replay (Phase 7).** A seed that reproduces a failure identically, on demand, converts an unreproducible distributed bug into an ordinary single-threaded one. This is the single most valuable debugging tool in the plan. Everything below is second best. But determinism is an architecture, not a feature: it is *designed for* from Phase 3 — every source of nondeterminism behind an injectable seam — and merely *built* in Phase 7. It cannot be retrofitted onto free-running goroutines.

**2. Correlation IDs (week 8).** One ID assigned at the client, propagated through routing, Raft, apply, commit, and stamped on every log line, metric exemplar and trace span. Without it a distributed log is noise. Costs an afternoon in week 8; costs weeks to retrofit.

**3. Introspection (every phase).** Internal state must be queryable from outside the process. Raft term and indices, per-peer match index and last contact, SSTable levels touched by a read, lock holders, range health. The CLI grows a `clavem debug` surface every phase — this is where it stops being a client and becomes the debugger.

**4. Loud invariant assertions (week 15 onward).** The worst bugs here corrupt state silently and surface hours later, when the cause is unrecoverable. Assertions at every layer boundary turn silent corruption into an immediate crash, a stack trace, and a dumped state file. Loud and early beats quiet and late. Keep them on in production; the cost is a branch.

**Supporting infrastructure, built early:**

- **Debug HTTP endpoint (week 12):** `net/http/pprof` plus custom pages. Grows into a status surface like etcd's or CockroachDB's.
- **Stall watchdog (week 16):** any request exceeding a threshold dumps all goroutine stacks. Finds deadlocks and lock convoys that logs never show.
- **Dynamic log levels (week 12):** raise verbosity on a running node without restarting. You will never reproduce the bug after a restart.
- **Structured errors (week 9):** every error carries ward ID, range ID, term, index. `errors.As` to extract structured context.
- **Event timeline (week 63):** render a simulation run as a readable timeline — who was leader, when messages dropped, when the invariant broke.

**The standing rule:**

> **If a bug took more than an hour to locate, the missing introspection is the real bug.** Fix the bug, then add the tool that would have found it in five minutes. Log both in the decision log.

---

## The `clavem debug` surface, by phase

| Phase | Commands added |
|---|---|
| 4 | `wal dump`, `keyspace stats`, `keys --with-ttl` |
| 5 | `repl status`, `repl trace <key>` — follow one write leader → replica |
| 6 | `raft state`, `raft log`, `raft peers`, `raft timeline` |
| 7 | `sim replay <seed>`, `sim timeline <seed>` |
| 8 | `lsm levels`, `lsm sst <file>`, `lsm explain <key>` — which files a read touched and why |
| 9 | `mvcc versions <key>`, `read --at <ts>`, `locks` |
| 10 | `ring topology`, `range <id>`, `ranges --problems` |
| 11 | `txn list`, `txn <id>`, `locks --waits`, `deadlock graph` |

---

# Operations at scale — the second cross-cutting thread

Operational knowledge is *time plus a running system*. You can't compress it, but you can start the clock early and let it accumulate in the background for eighteen months while you build.

**The standing commitment: a permanent staging ring from Phase 5.** Three to five wards on cheap VPS instances or containers, continuous synthetic load, alive for the rest of the project. **One hour per week**, every week: review dashboards, triage what broke, run a game day. This produces more operational knowledge than any single phase could.

**The ring is private from the first byte.** Wards talk over WireGuard or Tailscale; nothing listens on a public interface; minimal `AUTH` + TLS land before the ring does. An open RESP port on a public VPS is found by scanners and compromised in hours, not weeks — exposed Redis instances are among the most exploited things on the internet.

**Game days — the highest-value mechanism.** A script injects a fault into the running ring without telling you which. You diagnose it blind, timed. Record time-to-detect and time-to-diagnose, write a two-paragraph postmortem, file the fix. This is the closest available simulation of incident response — and it is also the *test* of your debug tooling. Either the introspection finds it in five minutes or it doesn't, and you'll know.

**Load generation, and the trap in it.** Build `clavem-bench` yourself (week 18) before adopting anything, because of **coordinated omission**: a closed-loop generator waits for each response before sending the next, so when the system stalls it stops sending — and your p99 looks best exactly when the system is worst. The fix is open-loop generation at a constant arrival rate, HDR histograms rather than averages, and *merged* histograms across workers (percentiles cannot be averaged). Learning that by building it is worth the two weeks.

Then adopt, rather than reinvent:

| Tool | Used for |
|---|---|
| `clavem-bench` (yours) | CLASP protocol, fault-correlated load, custom scenarios |
| **YCSB** | Workloads A–F, Zipfian distributions, comparability with published numbers |
| **k6** | Traffic-shape modelling (ramps, spikes, diurnal), thresholds as CI gates, gRPC for the Phase 9 API |
| **memtier_benchmark** | Third-party oracle for Redis-compatible numbers |

**Uniform random key distributions lie.** Real traffic is Zipfian with hot keys, and a hot key is what turns a healthy sharded ring into one overloaded range. None of that is visible until your generator models skew.

**What you measure:** the four golden signals — latency, traffic, errors, saturation — as SLOs with error budgets, not as vibes. p99 and p99.9, never averages.

---

# Redis compatibility — the third cross-cutting thread

"Redis-compatible" is a claim against a moving target of ~400 commands. Left undefined, it eats the plan. So define it:

**The oracle is pinned: Valkey 8.** Conformance means byte-identical behaviour against a named version with a BSD-licensed, reusable test suite — not "Redis" in the abstract. ("Redis" is also a trademark: describe compatibility with it, never brand with it.)

**Commands are tiered (week 13, revisited each phase):**

| Tier | Contents | When |
|---|---|---|
| 1 | Strings, expiry, hashes, lists, sets, sorted sets, `SCAN`, `MULTI`/`EXEC`/`WATCH`, pub/sub — plus everything real clients call implicitly on connect: `HELLO`, `INFO`, `CLIENT`, `CONFIG GET`, `SELECT 0` | Year 1, scheduled |
| 2 | Everything else worth having — bitmaps, streams, `OBJECT`, keyspace notifications | Background thread; never blocks a phase |
| Won't do | Modules, encoding fidelity (`OBJECT ENCODING`), databases beyond `SELECT 0` | Documented in the README |

**Clients are the real conformance test.** A command-diff harness can be green while go-redis fails on connect. From week 13 a **client matrix** runs real libraries against `clavemd` in CI: go-redis first, redis-py and Jedis by Phase 5, cluster-mode clients in Phase 10. Exact `MOVED`/`ASK` error formats, `CLUSTER SHARDS` reply shapes, graceful `HELLO 3` fallback — these only break under real clients.

**The positioning, decided now, because it settles three other arguments:** Clavem is **the Redis API for durable data** — the Kvrocks posture. Database-grade durability, quorum-write latency, honest about both. Consequences:

- **No `maxmemory`, no eviction.** Clavem is not a cache, and the README says so from `v0.1`. Users who need cache semantics should stay on Redis; users who need their data back should come here.
- **Compat mode keeps strict Redis Cluster semantics** — `CROSSSLOT` errors and all. It never silently exceeds what Redis can express: a client that works against Clavem works against Redis, and vice versa.
- **Everything Redis can't express** — cross-range transactions, MVCC reads, time travel — lives only in the reserved `CLV.*` namespace and the gRPC API. That is what the namespace is for.

**Tier 2 is the standing momentum task.** Each command is small, independently shippable, and oracle-verified by the conformance harness — ideal for stuck weeks and the month-8 and month-15 troughs. Like the ops hour: always available, never urgent.

---

# YEAR ONE — the Redis

## Phase 1 — Mechanical fluency
**~4 weeks.** Short on purpose.

| # | Focus | Deliverable |
|---|---|---|
| 1 | Types, slices, maps, `go mod` | `wc` clone |
| 2 | Structs, methods, receivers | Log-line parser aggregating by status code |
| 3 | Interfaces, `errors.Is`/`As`, wrapping | Refactor onto `io.Reader`; custom error types |
| 4 | `go test`, table-driven tests, `go vet` | In-memory KV with a stdin REPL. 80%+ coverage |

**Read:** A Tour of Go, Effective Go.
**Exit:** You can explain why a nil map read works but a nil map write panics.

---

## Phase 2 — Protocol, CLI, and the debug foundation
**~11 weeks.**

| # | Focus | Deliverable |
|---|---|---|
| 5 | `net`, `bufio`, `io.Reader` semantics | TCP echo server. Handle short reads — prove it with a `Reader` yielding 1 byte at a time |
| 6 | CLASP v0 decoder | All RESP2 types + inline commands. Fuzzing |
| 7 | Encoder, `clasp` API | Round-trip property tests. **First oracle:** `redis-cli PING` → `PONG` |
| 8 | `store`, dispatch, **correlation IDs** | `GET`/`SET`/`DEL`. Every request gets an ID at accept time, carried in `context`, stamped on every log line |
| 9 | Expiry, **structured errors** | `EXPIRE`/`TTL`, `SET` with `EX`/`NX`/`XX`. Error types carrying structured context, not strings. Clock behind an interface |
| 10 | Hashes, lists, sets | `HSET`/`HGETALL`, `LPUSH`/`LPOP`/`LRANGE`, `SADD`/`SREM`/`SMEMBERS`. `WRONGTYPE` everywhere |
| 11 | CLI, non-interactive | `flag`, not Cobra. Exit codes, `--json`, `--csv` |
| 12 | CLI interactive + **debug endpoint** | `x/term` raw mode, history, completion. Plus `/debug/pprof`, dynamic log levels, a status page |
| 13 | Conformance harness + **client matrix** | `make conformance`: drives `clavemd` and real `valkey-server`, diffs output. Command triage: tier the full command surface. go-redis smoke suite in CI — the first real client |
| 14 | `HELLO`, consolidation | Version negotiation with only v0 to negotiate. Reject `HELLO 3` gracefully and verify real clients fall back to RESP2 |

**Read:** *The Go Programming Language*. RESP2 spec.
**Exit:** `make conformance` green; the client matrix passes; you can trace one request by ID from accept to reply.
**Ship: `v0.1`.**

---

## Phase 3 — Concurrency
**~14 weeks.**

> **Design for Phase 7 now.** Deterministic replay cannot be retrofitted onto free-running goroutines. From week 15: every source of nondeterminism — clock, randomness, network, cross-actor scheduling — goes behind an injectable seam, and the server core stays runnable on a single-threaded scheduler. The seams cost minutes here; the rewrite they prevent costs months.

| # | Focus | Deliverable |
|---|---|---|
| 15 | Connection lifecycle + **assertions** | Graceful shutdown. Assertion helper that dumps state and crashes; invariants on every state transition |
| 16 | Leaks + **stall watchdog** | `goleak` everywhere. Any request over threshold dumps all goroutine stacks |
| 17 | Races | `-race` in CI. Introduce three races on purpose; understand why plain testing missed them |
| 18 | **`clavem-bench` v1** | Open-loop load generator at constant arrival rate. HDR histograms, merged across workers. Coordinated omission avoided by construction — prove it by stalling the server deliberately |
| 19 | Three keyspace architectures | Single `RWMutex`; sharded locks; owner goroutine fed by a channel |
| 20 | Benchmark them | Your generator + `memtier_benchmark` as oracle. p50/p99/p99.9 and allocations. Pick a winner with evidence |
| 21 | Background expiry | Sampling-based active expiry. Controllable clock — tests must not sleep |
| 22 | Pipelining + backpressure | Bound the queue; decide what a slow client does to you |
| 23 | Sorted sets | Skip list. `ZADD`/`ZRANGEBYSCORE`. Your first ordered iteration |
| 24 | Blocking commands | `BLPOP`. Waiting goroutines, per-client state, cancellation on disconnect |
| 25 | Pub/sub | Subscriptions, patterns, fan-out. Note what breaks under sharding |
| 26 | **`SCAN` + `MULTI`/`EXEC`/`WATCH`** | Cursor semantics under concurrent writes — guarantees, not vibes. `WATCH` is optimistic concurrency control: the single-node ancestor of Phase 9. Conformance-diff the error semantics; they are weirder than you think (a queued error does not abort the block) |
| 27 | **Debuggability review** | `CLIENT LIST` with per-connection state, slow log, command stats. First deliberate pass over your own tooling |
| 28 | Consolidation | Sustained load, leak-free, race-free. Write up week 20 |

**Read:** "Go Concurrency Patterns" (Pike). The Go Memory Model. `testing/synctest` (virtual time for concurrency tests, stable since Go 1.25 — it directly serves the no-sleep rule).
**Exit:** You can state from your own numbers where the channel keyspace loses to sharded mutexes — with p99.9, not averages.
**Ship: `v0.2`.**

---

## Phase 4 — Durability, performance, observability
**~12 weeks.**

| # | Focus | Deliverable |
|---|---|---|
| 29 | Write-ahead log | Length-prefixed framing + CRC32. **Every on-disk format carries a versioned header from its first byte** — the week-114 rolling upgrade is designed here, not there. fsync policies: always / everysec / never. Measure each |
| 30 | Recovery | Replay on start; truncate a torn tail write; corruption detection |
| 31 | Compaction | Rewrite from live state. Atomic swap via rename |
| 32 | Bitcask | Keydir in memory, values on disk, hint files. *(Replaced by an LSM in Y2 — building both is the point)* |
| 33 | Crash testing | `kill -9` under load at random points, restart, verify invariants. 1000×. Plus the ugly cases: disk full, `ENOSPC` mid-write, and fsync failure — an fsync error is **not retryable** (fsyncgate); prove Clavem crashes rather than trusts a poisoned page cache |
| 34 | **Realistic workloads** | Zipfian key distribution, hot keys, read/write mixes, value size distributions. YCSB A–F running against Clavem. Compare against your uniform-random numbers and watch them collapse |
| 35 | `pprof` | CPU, heap, block, mutex profiles *under realistic load*. Fix your top three bottlenecks |
| 36 | Allocation reduction | Escape analysis, `sync.Pool`, kill `[]byte`↔`string` copies. Before/after `-benchmem` |
| 37 | **Golden signals + SLOs** | `log/slog` with correlation IDs, Prometheus metrics with exemplars, `INFO`. Define SLOs and error budgets; alert on burn rate, not on thresholds |
| 38 | **Saturation and degradation** | Find the cliff. Load-shedding, admission control, bounded queues. Does it degrade gracefully or fall over? Make it the former |
| 39 | **`clavem debug` v1** | `wal dump`, `keyspace stats`, `keys --with-ttl`. Inspect on-disk state without starting the server |
| 40 | Backups | Point-in-time snapshot, restore, verify. Non-blocking `BGSAVE` |

**Read:** Bitcask paper. *DDIA* ch. 3. "Files are hard" (Aphyr). PostgreSQL's fsyncgate postmortem. "How NOT to Measure Latency" (Tene). Google SRE book ch. 4, 6.
**Exit:** You can state your single-node capacity with a headroom curve, and the system sheds load rather than collapsing at 110% of it.
**Ship: `v0.3`.**

## Phase 5 — Replication, tracing, and the staging ring
**~9 weeks + 1 buffer.**

| # | Focus | Deliverable |
|---|---|---|
| 41 | **Fault injection harness** | Partitionable `net.Conn`: drop, delay, reorder, one-way partition. Disk stalls. Clock skew. Built *before* the code it will break |
| 42 | Replication I | `REPLICAOF`, handshake, full sync via snapshot transfer |
| 43 | Replication II | Stream the command log, offsets, lag, partial resync |
| 44 | **Determinism** | Why replicas diverge: `SPOP`, `TIME`, expiry-on-read. Propagate effects, not commands. The concept everything later rests on |
| 45 | **Distributed tracing** | OpenTelemetry. First cross-node request. Correlation ID becomes a trace ID |
| 46 | **Divergence detection** | Continuous checksum comparison leader vs replica. Detect divergence in seconds, not hours. `repl trace <key>` follows one write across nodes |
| 47 | Follower reads | Stale reads from replicas. Quantify the staleness |
| 48 | **Stand up the staging ring** | 3 wards on real hosts — **private network only** (WireGuard/Tailscale), minimal `AUTH` + TLS before the first byte of exposure. Continuous k6 load with diurnal shape. Grafana dashboards, alerting on SLO burn. **The weekly ops hour starts now and never stops** |
| 49 | **First game day** | Script injects a blind fault: kill a replica, stall a disk, partition a link, skew a clock. Diagnose timed. Record MTTD/MTTR. Write the postmortem. Fix the tooling gap it exposes |

**Exit:** Partition a replica for 10 minutes under load; it resyncs and your checksum monitor proves it. You diagnosed a blind fault in under 15 minutes.
**Ship: `v0.4`.**

## Phase 6 — Raft
**~11 weeks + 2 buffer.** No `hashicorp/raft`. From the paper.

| # | Focus | Deliverable |
|---|---|---|
| 50 | Read Raft properly | Extended paper + Ongaro ch. 3–4. Design doc. **No code** |
| 51 | **Raft introspection first** | Before the algorithm: state dump, log inspection, per-peer view, event log of every state transition. *Build the instruments before the engine* |
| 52 | Leader election | Terms, `RequestVote`, randomized timeouts. Tested against the week-41 fault-injection harness |
| 53 | Log replication | `AppendEntries`, `nextIndex`/`matchIndex`, `commitIndex`, divergent-log resolution |
| 54 | Persistence | State durable *before* responding. Restart under crash injection |
| 55 | State machine | Apply committed entries. Leader redirect over CLASP v1; CLI follows it |
| 56 | Linearizable reads | ReadIndex or lease reads. Demonstrate why naive leader reads are unsafe |
| 57 | Idempotent requests | Request IDs, dedup table. Retry across leader failure without double-applying |
| 58 | Snapshots | `InstallSnapshot`, log truncation, transfer to a lagging follower |
| 59 | Fencing | Stale leaders, fencing tokens, the partitioned old leader returning |
| 60 | Membership | Single-server add/remove (Ongaro ch. 4 — avoid joint consensus). Include the 2015 raft-dev fix: a new leader commits a no-op entry in its own term before any membership change |

**Read:** Raft extended paper; Ongaro ch. 3, 4, 6. The 2015 raft-dev thread on the single-server membership bug.
**Exit:** A 5-node group survives rolling restarts, partitions and leader kills under continuous write load with no lost acknowledged write — and `raft timeline` shows exactly what happened during each election. **Game day:** kill a leader mid-write blind; diagnose from dashboards alone in under 10 minutes.
**Ship: `v0.5` — end of year one.**

> **Why introspection comes first here.** Debugging Raft without visibility into term, commit index and per-peer state is close to impossible. One week up front saves several later. This is the clearest case in the plan for tools before features.

---

# YEAR TWO — the database

## Phase 7 — Deterministic replay
**~7 weeks + 2 buffer.** The most powerful debugging tool you will build. It comes *before* the year-two rewrites deliberately — and it only fits in seven weeks because the seams were cut in Phase 3.

| # | Focus | Deliverable |
|---|---|---|
| 61 | Deterministic simulation | Seed-driven: virtual clock, virtual network, single-threaded scheduler. A failing seed replays identically |
| 62 | Simulation coverage | Thousands of seeds in CI. Partitions, loss, reordering, clock skew, restarts |
| 63 | **Event timeline** | Render a run as a readable timeline: leadership, message drops, the moment an invariant broke. `sim timeline <seed>` |
| 64 | Linearizability checking | Record histories including **indeterminate** results; check with `porcupine` |
| 65 | Minimization | Shrink a failing seed to the smallest reproducing case. A 10,000-event trace becomes twelve events |
| 66–67 | Chaos soak, fix, re-verify | It will find real bugs. That's the point |

**Read:** Aphyr's Jepsen analyses. FoundationDB's simulation testing talk.
**Exit:** 10,000 seeds pass in CI; any failure replays deterministically and minimizes to one screen.
**Ship: `v0.6` — verified.**

---

## Phase 8 — LSM storage engine
**~10 weeks.** Bitcask can't do ordered scans, and ordered scans are what transactions and range partitioning both need.

| # | Focus | Deliverable |
|---|---|---|
| 68 | MemTable | Reuse the Phase 3 skip list. Immutable rotation, WAL tie-in |
| 69 | SSTable format | Sorted blocks, restart points, block index, footer. Versioned header, per-block checksums |
| 70 | Bloom filters | Per-SSTable, false positive tuning. Measure the read amplification saved |
| 71 | Block cache | LRU over decompressed blocks. Hit rate under *Zipfian* load, not uniform |
| 72 | Leveled compaction | L0 overlap, level sizing, compaction picking, write amplification |
| 73 | Tiered compaction | Implement it too. Benchmark both. Write up the RUM tradeoff |
| 74 | Iterators | Merging iterator across memtable + levels. Ordered, prefix, reverse scans |
| 75 | **`lsm explain`** | For any read: which files were touched, which bloom filters fired, which were false positives. Compaction history and level stats |
| 76 | **Endurance run** | Multi-day soak under sustained write load. Compaction debt, write stalls, L0 pileup, memory fragmentation, disk growth. **Only visible over days — this is why the staging ring exists** |
| 77 | Migration + verification | LSM and Bitcask side by side under the Phase 7 harness. Byte-identical or a bug |

**Read:** LSM-tree paper (O'Neil). RocksDB tuning guide. Pebble and Badger source.
**Exit:** A 72-hour write-heavy soak shows no unbounded growth, no write stall over 500ms, and `lsm explain` accounts for every file a slow read touched.
**Ship: `v0.7`.**

---

## Phase 9 — MVCC and transactions
**~10 weeks.** The largest jump in intellectual content in the plan. Time becomes a first-class concept — and, usefully, a debugging one.

| # | Focus | Deliverable |
|---|---|---|
| 78 | Versioned keys | Encoding as `(user_key, timestamp)` descending. Reads take a read-timestamp. **Decide the timestamp bit-layout now — physical + logical bits, HLC-shaped** — because Phase 11 replaces the allocator, and must not replace the encoding on every SSTable |
| 79 | Snapshot isolation | Transaction begins at a timestamp, sees a consistent snapshot. Write conflicts detected at commit |
| 80 | Write intents and locks | Uncommitted writes as intents. Conflict handling: wait, abort, or wound-wait |
| 81 | Single-node commit | Atomic commit of a write set. Rollback. Crash recovery mid-transaction |
| 82 | Serializability | SI's write-skew anomaly with a real example. Then SSI or pessimistic locking to close it |
| 83 | **Time travel** | `mvcc versions <key>` and `read --at <ts>`. MVCC gives you a debugging superpower for free: *what did this key look like when it broke?* |
| 84 | Garbage collection | Reclaim versions past the oldest active read timestamp. Interaction with compaction. **Watch GC debt accumulate under a long-running read** |
| 85 | gRPC transactional API | Protobuf schema, `Begin`/`Get`/`Scan`/`Put`/`Commit`. k6 drives it natively |
| 86 | **Redis API onto MVCC** | `MULTI`/`EXEC`/`WATCH` re-implemented as real transactions, with Redis's error semantics preserved exactly. The week the two identities meet |
| 87 | Isolation verification | Beyond linearizability: Elle-style anomaly detection |

**Read:** *DDIA* ch. 7. "A Critique of ANSI SQL Isolation Levels". Elle paper (Kingsbury & Alvaro).
**Exit:** Your checker finds write skew under SI and confirms its absence under SSI — and you can reconstruct any key's full history at any timestamp.
**Ship: `v0.8` — transactional single node.**

---

## Phase 10 — Cluster and range partitioning
**~15 weeks + 3 buffer.**

| # | Focus | Deliverable |
|---|---|---|
| 88 | Hash slots first | 16384 slots, `CRC16 mod 16384`, hash tags, strict `CROSSSLOT` errors. Redis-compatible mode, kept forever. Cluster-mode clients join the client matrix |
| 89 | Gossip I | SWIM: ping, indirect ping, suspicion, refutation |
| 90 | Gossip II | Propagate ownership and epochs. Conflict resolution when two wards claim a partition |
| 91 | Client routing | CLI caches the map, hashes locally, talks to owners. `MOVED` refresh-and-retry — byte-exact error format, verified against real cluster clients |
| 92 | Ranges | Ordered ranges instead of slots. Range descriptors, routing layer, metadata in its own Raft group |
| 93 | Multi-Raft | Many groups per ward. Shared heartbeat batching, shared WAL, scheduler fairness. **The real engineering problem at scale** |
| 94 | Range split | Split at a key under live traffic. Atomic descriptor update, no lost writes |
| 95 | Range merge | Merge underfull adjacent ranges. Harder than split — both sides must agree |
| 96 | **Hot ranges** | Zipfian load with a single scorching key. Watch one range saturate while the ring idles. Split-by-load, not just by size. **The lesson uniform benchmarks hide** |
| 97 | **Cluster introspection** | `ranges --problems`: which ranges lack quorum, are mid-split, are behind, are hot |
| 98 | **Capacity planning** | Model ring capacity from single-node numbers. Predict, then measure, then explain the gap. Scale the ring under sustained load and verify the prediction |
| 99 | Rebalancing policy | Hotspot detection, load-based placement, avoiding thrash, rack awareness |
| 100 | Migration verification | Linearizability and isolation checking *with splits and merges active* |
| 101 | Sharded pub/sub | Fan-out across the ring. Why Redis added `SSUBSCRIBE` |
| 102 | Cluster chaos | Full-ring soak: partitions during split, node loss mid-merge, split brain |

**Read:** SWIM paper. Redis Cluster spec. TiKV placement driver docs. *DDIA* ch. 6. Google SRE book ch. 21 (load shedding).
**Exit:** Add a ward under sustained skewed load, rebalance every range, kill a node mid-split — no lost writes, still linearizable, `ranges --problems` names the degraded range in seconds, and your capacity model predicted the result within 20%.
**Ship: `v0.9`.**

---

## Phase 11 — Distributed transactions
**~9 weeks.** The hardest and most interesting part of the plan.

| # | Focus | Deliverable |
|---|---|---|
| 103 | Time across machines | Lamport clocks, vector clocks, then **hybrid logical clocks**. What Spanner buys with atomic clocks, and what you can't have without them |
| 104 | Timestamp oracle | Monotonic allocation. **The naive version is a single point of failure** — make it a Raft group, batch allocations, measure the latency cost under load. TSO vs HLC is an architecture choice (TiDB vs CockroachDB) — pick one, log why |
| 105 | Percolator 2PC I | Prewrite across ranges. Primary lock, secondary locks, conflict detection |
| 106 | Percolator 2PC II | Commit phase. Async secondary cleanup. The commit point, and why it's exactly one write |
| 107 | Failure recovery | Coordinator dies mid-commit. Lock resolution by a later reader. Why the primary lock is the source of truth |
| 108 | Cross-range reads | Consistent snapshot across ranges. Interaction with splits mid-read |
| 109 | Contention | Deadlock detection or avoidance. Retry policy, backoff, starvation. **Benchmark under deliberately high contention — a hot row, many writers** |
| 110 | **Transaction introspection** | `txn list`, `txn <id>`, `locks --waits`, `deadlock graph`. Trace one transaction across every range it touched |
| 111 | Verification | Elle-style checking on cross-range transactions under chaos. Kill the coordinator mid-commit, repeatedly |

**Read:** Percolator paper. Spanner paper. HLC paper (Kulkarni et al.). TiKV transaction docs.
**Exit:** Transactions spanning four ranges keep **snapshot isolation** while nodes die, ranges split, and the coordinator is killed mid-commit — Percolator buys SI, not serializability; write down what distributed serializability would additionally require (read refreshes, timestamp caches, or 2PL) and why you didn't build it — and when a transaction hangs, you can name the lock it's waiting on in under a minute.
**Ship: `v0.10` — a distributed transactional database.**

---

## Phase 12 — Running it for real
**~9 weeks.** The phase that converts eighteen months of staging-ring hours into something you could put on a CV without flinching.

| # | Focus | Deliverable |
|---|---|---|
| 112 | Multi-tenancy | Namespaces, per-tenant quotas. **Noisy neighbours:** one tenant's Zipfian burst degrading another. Measure it, then stop it |
| 113 | Admission control | Per-tenant rate limiting, priority queuing, graceful shedding under overload. Prove the SLO holds for well-behaved tenants while a bad one is throttled |
| 114 | Rolling upgrade | Version bump across a live ring under load. Mixed-version cluster correctness — the week-29 format headers pay off here. Zero dropped requests — measured, not assumed |
| 115 | Disaster recovery drill | Destroy the ring. Restore from backup. **Time it.** Write the RTO/RPO numbers down. Repeat until the numbers stop embarrassing you |
| 116 | Change data capture | Durable ordered change stream per range. Resumable consumer offsets |
| 117 | Lua scripting | Embed a Lua VM. **The real lesson:** a script calling `TIME` diverges replicas. Determinism in a replicated state machine, learned the hard way |
| 118 | Client-side caching | RESP3 push invalidation. Distributed cache coherence in miniature |
| 119 | Security hardening | ACLs, per-command permissions, transaction-level authorization. (TLS and `AUTH` have protected the ring since week 48 — this week is depth, not table stakes) |
| 120 | **Runbook** | For every failure mode you've hit in eighteen months of game days: the symptom, the dashboard that shows it, the command that confirms it, the fix. This document is the artefact of your operational experience |

**Read:** Google SRE book (chapters on SLOs, overload, incident response). *Database Reliability Engineering*.
**Exit:** A blind game day combining two simultaneous faults, diagnosed from the runbook in under 20 minutes.

---

## Phase 13 — The month, and the write-up
**~5 weeks, overlapping.** Run and write at the same time — the ring needs elapsed time, not attention.

| # | Deliverable |
|---|---|
| 121 | **Start the month.** Production-shaped load on the ring, continuously, for 30 days. Weekly game days. An incident log with a postmortem per event |
| 122 | Architecture README; design docs published |
| 123 | Benchmarks vs. real Redis, Redis Cluster, TiKV. Honest numbers, including where you lose badly. Latency distributions, not averages |
| 124 | **The operations report:** SLO attainment over 30 days, every incident, MTTD and MTTR trends across eighteen months of game days |
| 125 | A post per phase. Tag `v1.0` |

**Exit:** Thirty days of continuous operation with measured SLO attainment and a written incident history.

---

## Pacing

| | Phases | Weeks | Buffer |
|---|---|---|---|
| **Year 1** | 1–6 | 60 | 3 |
| **Year 2** | 7–11 | 51 | 5 |
| **Year 3 (partial)** | 12–13 | 14 | 2 |

**~135 weeks ≈ 2 years 8 months** at 12–15 h/week, plus the standing **1 hour per week** on the staging ring from week 48 onward.

That's the honest number with debuggability, operations and Redis compatibility threaded through. I'd rather show it than round it down to something tidier.

Estimates reliably break at: Raft (52–60), range split and merge (94–95), and Percolator recovery (106–107).

**If you need it shorter:**

1. Phase 12 weeks 116–119 (CDC, Lua, client caching, deep security) — saves 4
2. Tiered compaction (73) — saves 1
3. Sharded pub/sub (101) — saves 1
4. Multi-tenancy and admission control (112–113) — saves 2

That lands near 2 years 5 months with everything essential intact. Hash-slot mode is **no longer cuttable** — Redis Cluster compatibility is a goal, not a convenience.

**Do not cut debug or ops weeks to save time.** They pay for themselves inside the same phase — a game day that exposes a missing introspection command saves more hours than the game day cost.

**Never cut:** conformance harness + client matrix (13), correlation IDs (8), assertions (15), `clavem-bench` (18), realistic workloads (34), fault injection (41), determinism (44), staging ring (48), Raft introspection (51), deterministic simulation (61), linearizability checking (64), minimization (65), isolation checking (87), disaster recovery drill (115).

---

## Reading other people's code

One session a month, ~2 hours, with notes. Read them *after* building your own version — before, they're intimidating; after, they're a conversation.

| From | Read |
|---|---|
| Phase 4 | `boltdb/bolt` — small enough to read completely in a weekend |
| Phase 6 | `etcd/raft` — compare its structure to yours, after yours works |
| Phase 8 | `cockroachdb/pebble`, `dgraph-io/badger` — two LSM philosophies in Go |
| Phase 9 | `apache/kvrocks` — the closest existing thing to Clavem's positioning: the Redis API on a durable engine |
| Phase 10 | `hashicorp/memberlist` — SWIM in production |
| Phase 11 | TiKV's transaction layer (Rust, but readable) |
| Phase 12 | Google SRE book, and `grafana/k6` source for how open-loop scheduling is done |

Pay attention to their *debug* surfaces specifically. CockroachDB's problem-ranges page and etcd's raft status endpoint exist because someone spent a bad night without them.

And find review, not just reading: post the Phase 2 parser and Phase 3 keyspace for critique (Gophers Slack, r/golang), or land one small patch in pebble or etcd to feel real review standards. Two and a half years solo entrenches habits at exactly the age they're cheapest to fix.

---

## Momentum

**Month 8** — Raft half-works and nothing feels finished. Ship `v0.4` before starting it; keep the CLI improving in parallel.

**Month 15** — the LSM and MVCC rewrites produce nothing demoable for weeks. The real danger point. `v0.7` sits immediately after the LSM for exactly this reason.

- **Ship at every phase gate.** A tagged release you'd show someone.
- **Write the post before moving on.** Explaining it is how you find out you didn't understand it.
- **Keep a decision log.** One paragraph per non-obvious choice. In month twenty you will not remember why the keydir is sharded.
- **Stuck more than a week? Switch layers.** Go improve a debug command. The bug will still be there, and you'll see it differently — sometimes the new tool finds it.
- **Tier-2 commands are momentum too.** Small, shippable, oracle-verified — a green conformance diff on a week when nothing else moves.
- **Demo to someone every phase.** Even if they don't care. Especially then.
- **The staging ring is a momentum device too.** On weeks when the code won't move, the ring still generates real problems to solve. A dashboard that looks wrong is a task you can start in five minutes.

---

## Non-negotiable habits

- `-race` on every test run from week 17. No exceptions.
- `goleak` in every test that starts a goroutine.
- No `time.Sleep` in tests after week 20. You have a clock interface.
- Every chaos-found bug gets a deterministic regression test with its seed.
- `make conformance` before any protocol change merges — and the client matrix stays green.
- One design doc per phase, written *before* the code.
- **Every log line carries the correlation ID.** From week 8, without exception.
- **Every source of nondeterminism sits behind an injectable seam.** From week 15 — Phase 7 is an architecture, not a phase.
- **Every persistent and wire format carries a versioned header.** From week 29 — mixed-version clusters are built two years before they run.
- **Durability and crash tests run on Linux.** Windows and WSL2 have different fsync and rename semantics than the machines the ring runs on; numbers measured there are fiction.
- **Assertions stay on in production.** A crash with a state dump beats silent corruption every time.
- **If a bug took over an hour to find, add the tool that would have found it in five minutes.** The missing introspection is the real bug.
- **One ops hour per week, every week, from week 48.** Dashboards, incidents, one game day. Non-negotiable — it is the only way elapsed time turns into experience.
- **Never report an average latency.** p99 and p99.9, from merged HDR histograms, under open-loop load.
- **Never benchmark with a uniform key distribution.** Zipfian with hot keys, or the numbers are fiction.

---

## What "hero" means at the end

Not that you know Go's syntax. That you can look at a distributed system, form a hypothesis about how it fails, build a harness that provokes that failure, and fix it with evidence rather than intuition.

Two and a half years in, you'll have built the thing most engineers only read about — proved it correct rather than hoped, made it explain itself when it breaks, and run it long enough to know how it behaves at 3am.

Clavem is the vehicle. That is the point.
