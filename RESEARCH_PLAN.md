# MorphClass Research Plan

## Working title

**MorphClass: Budget-Enforced, Semantics-Preserving Reconfiguration of OVS
MegaFlow Classifiers**

The project uses "budget-enforced" rather than "SLO-bounded": observing a tail
latency violation and aborting afterward is SLO-aware behavior, not a bound.

## Research question

Can a live Open vSwitch (OVS) userspace datapath replace the physical
representation of its MegaFlow classifier while traffic, flow insertion,
deletion, revalidation, and action updates continue, without changing
forwarding semantics or exceeding explicit CPU, memory, and update-disruption
budgets?

MorphClass targets `dpif-netdev`/`dpcls`, not the priority-ordered OpenFlow policy
classifier. Its semantic oracle is the flow and associated action returned by
upstream generic `dpcls` for the same canonical datapath-flow state. OpenFlow
priority bounds and first-match directory semantics are therefore out of scope.

## Phase 0: OVS-native kill study

Implementation of migration or another classifier backend starts only after a
six-week measurement study in current OVS. Instrument coupled traffic and
control-plane workloads to record, per analysis window:

- classifier cycles and subtables visited;
- lookup and flow-hit distributions;
- flow insertion, deletion, revalidation, mask, and action changes;
- Exact Match Cache (EMC), Signature Match Cache (SMC), and `dpcls` hit rates;
- PMD utilization; and
- the performance of each candidate backend used statically.

For backend \(b\) and window \(w\), measure static-backend regret against the
best backend in that window. The project proceeds only if all of these conditions
hold:

1. the winning backend changes across a meaningful fraction of windows;
2. the best fixed backend incurs approximately 15--20% regret in those windows;
3. typical phase residence time is several times migration break-even time; and
4. classification remains a material CPU or latency cost with realistic caches
   enabled.

Failure of either of the first two conditions stops or redirects the project
before substantial backend engineering.

## Deliberately constrained first system

The first implementation has:

- upstream generic OVS `dpcls` as baseline and semantic oracle;
- one structurally distinct CPU backend selected from measured bottlenecks;
- fixed ownership using OVS's natural mask-subtable structure;
- source-authoritative delta replay; and
- deterministic admission and resource isolation.

It does **not** include a third backend, general partition split/merge, a global
priority-bound directory, or learning-based control. TupleMerge remains a
microbenchmark and historical baseline rather than the runtime substrate.

## State model

MorphClass separates two independently changing identifiers:

- **logical epoch \(L\):** the committed set of datapath flows and actions; and
- **physical generation \(G\):** the index representation used for lookup.

Ordinary updates advance \(L\). Representation migration changes \(G\) without
changing logical semantics.

A canonical flow object contains a stable flow identifier, key and mask, current
action-version pointer, stable counter pointer, and lifecycle metadata. Physical
indexes reference canonical objects instead of copying mutable actions and
counters.

## Migration state machine

The explicit states are:

```text
IDLE -> ADMITTED -> BUILDING -> REPLAYING -> QUIESCING -> VALIDATING
                                                            |       |
                                                            v       v
                                                     PUBLISHED   ABORTING
                                                            |       |
                                                            v       v
                                                      RETIRING  ROLLED_BACK
                                                            |       |
                                                            +---> IDLE <---+
```

The protocol is:

1. **Admit.** Check CPU, memory, log capacity, convergence, and predicted
   amortization.
2. **Snapshot.** Capture canonical state at logical epoch \(L_0\).
3. **Build.** Construct the target on reserved resources while the source serves
   packets.
4. **Log.** Commit each update batch to the authoritative source and append an
   ordered, idempotent delta record.
5. **Replay.** Apply deltas asynchronously to the isolated target.
6. **Catch up.** Continue until lag is below a small configured threshold.
7. **Barrier.** Briefly quiesce the relevant writer, drain the tail, and establish
   common watermark \(L_c\).
8. **Validate.** Check exact canonical state, backend invariants, and shadow
   lookup equivalence.
9. **Publish.** Atomically replace the active physical-generation pointer.
10. **Retire.** Reclaim the old generation after an RCU/epoch grace period and
    configurable rollback window.
11. **Abort.** Discard an unpublished target if any admission, convergence,
    resource, or correctness condition fails.

Synchronous dual application is retained only as an experimental baseline.

## Admission and interference control

A migration is admitted only when estimates satisfy:

\[
\widehat{\mu}_{\mathrm{replay}} >
\widehat{\lambda}_{\mathrm{update}}(1+\epsilon),
\]

\[
M_{\mathrm{source}} + M_{\mathrm{target}} + M_{\Delta}
\leq M_{\mathrm{budget}},
\]

and

\[
T_{\mathrm{phase}}^{\mathrm{pred}} > T_{\mathrm{build}} +
T_{\mathrm{catchup}} + T_{\mathrm{break-even}}.
\]

Construction and replay run on a dedicated core or bounded work queue with
NUMA-aware placement. Dynamic batch caps limit memory-bandwidth and LLC
interference. If observed conditions invalidate an admission estimate before
publication, migration aborts; this does not retroactively claim a latency SLO
was bounded.

## Correctness contract

- **Logical-state exactness:** after an acknowledged update batch's specified
  linearization point, later lookups reflect its post-update logical state.
- **Batch atomicity:** a lookup observes either all or none of a multi-flow
  transaction.
- **Physical transparency:** changing \(G\) preserves the lookup result for \(L\).
- **Target isolation:** incomplete or lagging targets never serve production
  packets.
- **Cutover equivalence:** publication requires identical canonical flow IDs,
  keys, masks, and action versions at \(L_c\).
- **Action continuity:** lookup returns the correct action version, not only the
  correct flow identifier.
- **Counter continuity:** counters reside outside indexes or follow a separately
  specified consistency rule.
- **Snapshot lookup:** one lookup uses one physical generation throughout.
- **Atomic publication:** the active generation pointer changes atomically.
- **Safe retirement:** reclamation waits for all possible old-generation readers.
- **Rollback safety:** the source remains authoritative before publication; any
  post-publication rollback uses a retained valid source or canonical rebuild.

Validation combines exact canonical-state comparison, backend structural checks,
OVS-autovalidator-style shadow execution, history-based linearizability checking
or model checking, and fault injection at every state transition. Tests report
wrong-flow and wrong-action results separately.

## Coupled workloads

The primary generator must record packets and the control-plane evolution that
caused datapath state, including flow insertion/deletion, cache invalidation,
action replacement, and expiration timestamps. Candidate scenarios include:

- pod or VM creation/deletion and tenant churn;
- service endpoint and NetworkPolicy/security-group rollout;
- DDoS mitigation update bursts;
- high-cardinality short-lived flows and hotspot movement; and
- mask proliferation, action replacement, expiration, and revalidation.

ClassBench-ng remains useful for rule geometry. CAIDA and MAWI traces are
secondary packet-distribution sensitivity studies and must not be paired with
unrelated synthetic rule evolution as primary evidence.

## Evaluation

Use a current stable OVS release as the artifact baseline and verify mainline
compatibility before submission. The principal testbed is OVS-DPDK with 100-Gb/s
NICs, pinned PMD cores, explicit NUMA placement, 64-byte and IMIX traffic,
offered-load sweeps, and both one- and multi-PMD configurations. Report EMC, SMC,
and `dpcls` hit fractions; classifier-only microbenchmarks are secondary.

### Codex Cloud and hardware validation

Codex Cloud is suitable for implementation, functional CI, deterministic replay,
model checking, fault injection, and classifier microbenchmarks. It is **not** a
substitute for a controlled 10/40/100-Gb/s testbed. A shared virtual machine does
not normally provide exclusive physical NIC queues, stable PMD core isolation,
controllable NUMA placement, or repeatable access to PCIe and memory bandwidth.
Virtual-NIC rate limits, noisy neighbors, host scheduling, and unavailable DPDK
offloads can dominate throughput and tail-latency measurements.

Cloud results therefore must not be presented as line-rate NIC results or used to
substantiate the budget-enforcement hypothesis. The evaluation is split into
three tiers:

1. **Every cloud change:** compile and run unit, differential, concurrency, state
   machine, replay, and fault-injection tests. Compare every lookup and action
   against the generic `dpcls` oracle.
2. **Dedicated bare-metal development:** run CPU-only `dpcls`/`testpmd` replay,
   pin cores, control NUMA placement, and report relative classifier costs. These
   experiments can reject poor designs, but cannot establish wire-rate behavior.
3. **Publication hardware:** use a dedicated traffic generator and device under
   test with physical 10/40/100-Gb/s NICs. Re-run tagged release commits with
   fixed firmware, driver, DPDK, BIOS, CPU-frequency, core-pinning, queue, and
   NUMA configurations.

Access to the third tier is a project dependency, not an optional final polish.
It must be arranged during the Phase-0 study and confirmed before migration
engineering begins. If dedicated hardware cannot be secured, narrow the claims
to classifier correctness and CPU microbenchmarks and target a venue appropriate
for that evidence; do not claim 100-Gb/s end-to-end performance or bounded
datapath interference.

Primary same-hardware baselines are:

1. upstream generic `dpcls`;
2. each backend used statically;
3. the best static backend selected with the full trace;
4. an offline per-window oracle with future knowledge;
5. periodic global rebuild/A-B replacement;
6. synchronous dual application;
7. delta replay without admission;
8. admission without phase-amortization checks; and
9. full MorphClass, plus shadow-validation and isolation ablations.

Published systems are direct performance baselines only when their artifacts can
be reproduced fairly on the same platform.

Measure end-to-end throughput, goodput, loss, p50/p99/p99.9 latency, PMD cycles,
classifier cycles per exercised lookup, update visibility latency, writer-barrier
duration, build and catch-up time, delta backlog, transient memory, migration CPU
and memory bandwidth, cache interference, success/rollback rate, controller
precision, amortized-migration fraction, static-backend regret, and wrong-flow and
wrong-action counts. "Durable" is reserved for state persisted across process or
machine failure.

Fault injection covers construction and allocation failure, replay
non-convergence, log overflow, validation failure, crashes immediately before and
after publication, delayed grace periods, stalled workers, controller
oscillation, and backend corruption detected by shadow validation.

## Falsifiable hypotheses

- **H1 — Phase variation:** no one backend wins all coupled OVS workload windows,
  and static regret is operationally significant.
- **H2 — Correctness:** MorphClass preserves generic `dpcls` results and actions
  under concurrent traffic, updates, and injected failures.
- **H3 — Delta-log advantage:** replay reduces foreground update-tail overhead
  relative to synchronous dual application.
- **H4 — Budget enforcement:** admission and isolation keep tail inflation and
  transient memory within configured budgets more reliably than uncontrolled
  rebuilds.
- **H5 — Economic adaptation:** MorphClass captures a substantial fraction of the
  offline oracle's benefit while rejecting migrations that will not amortize.

## Schedule

| Period | Outcome |
| --- | --- |
| Weeks 1--6 | OVS instrumentation, coupled generator, and regret kill study |
| Weeks 7--12 | Exact semantics, dual epochs, canonical flows, shadow validation |
| Weeks 13--20 | Delta log, writer barrier, cutover, and RCU retirement |
| Weeks 21--28 | Convergence estimates, admission, isolation, and rollback |
| Weeks 29--35 | One measured, structurally distinct backend |
| Weeks 36--42 | 100-Gb/s evaluation, ablations, and fault injection |
| Weeks 43--48 | Artifact, paper, model-checking appendix, result verification |

For a six-month deadline, scope is reduced to current OVS plus one backend, fixed
ownership, delta replay, resource admission, one coupled workload, and 100-Gb/s
evaluation. Even this scope is aggressive.
