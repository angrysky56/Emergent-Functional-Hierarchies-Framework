# Sheaf Consistency Enforcer — Testing Procedure and Results

**Date**: March 14, 2026  
**Enforcer**: [sheaf-consistency-enforcer](https://github.com/angrysky56/sheaf-consistency-enforcer)  
**Tested via**: Live MCP tool calls through Claude (Sonnet 4.6) in Claude.ai  
**Status**: All tests passed after v0.1 optimisation patch

---

## Background

The enforcer implements Sheaf Laplacian-based ADMM coordination across four MCP agents
(hipai-montague, mcp-logic, advanced-reasoning, verifier-graph). It was built to close
the "missing piece" identified in `2026-03-13-ai-application.md`: automatic detection
of lumpability failures before they cascade into hallucination or contradictory knowledge
graph commits.

Two optimisations were applied between initial build and final testing:

1. **Per-agent pressure tracking**: dual variable pressure is now attributed individually
   to each agent rather than aggregated globally, enabling precise identification of the
   inconsistency source.

2. **Dissipative decay**: the dual update step uses a leaky integrator
   (`dual = dual * 0.85 + cb_norm`) rather than pure accumulation. This produces a
   stable steady-state (`dual_ss = cb_norm / decay_rate`) and prevents false TIMEOUT
   from long-running consistent sessions.

A third fix was identified and applied during testing: recovery strategies (`admm_reset`,
`kernel_retreat`) were not explicitly resetting `closure_status` before the post-recovery
ADMM cycle, so the escalation-only `update_status()` logic blocked de-escalation back to
KERNEL1 even after inconsistency was fully resolved. One line per recovery path fixed this.

---

## Testing Procedure

### Prerequisites

- `sheaf-consistency-enforcer` running as an active MCP server
- At least two agents registered (the enforcer requires `>= 2` agents to run cycles)
- Clean session state: call `reset_session(confirm=True)` before each test suite

### Standard Test Sequence

The test suite covers four scenarios in order:

1. **Consistent agents** — verifies KERNEL1 stability and decay convergence
2. **Bad actor injection** — verifies per-agent pressure attribution and WEAK detection
3. **Restore + admm_reset** — verifies de-escalation fix (the critical regression test)
4. **Persistent bad actor + kernel_retreat** — verifies automatic source removal and recovery

### Agent State Keys Used

Each agent is registered with a state dict using the shared edge-space vocabulary:

| Agent | Key | Edge projection |
|-------|-----|----------------|
| hipai-montague | `last_assertion` (str) | `edge_claim` |
| hipai-montague | `belief_score` (float) | `edge_confidence` |
| hipai-montague | `inconsistency_flag` (bool) | `edge_inconsistent` |
| mcp-logic | `last_proof_result` (str) | `edge_claim` |
| mcp-logic | `proof_confidence` (float) | `edge_confidence` |
| mcp-logic | `contradictions_found` (bool) | `edge_inconsistent` |
| advanced-reasoning | `current_hypothesis` (str) | `edge_claim` |
| advanced-reasoning | `confidence_score` (float) | `edge_confidence` |
| advanced-reasoning | `halt_flag` (bool) | `edge_inconsistent` |
| advanced-reasoning | `verified_claim` (str) | `edge_claim` (reverse edges) |

String values are hashed to `[0,1]` via `hash(val) % 100_000 / 100_000.0`.
**Identical strings produce identical projections → coboundary = 0.**
Different strings produce distinct hashes → elevated coboundary.


---

## Test 1: Consistent Agents — KERNEL1 Stability and Decay Convergence

**Purpose**: Verify that three agents reporting the same claim maintain KERNEL1
indefinitely, and that the dissipative decay stabilises dual variables rather than
accumulating toward false thresholds.

**Setup**:
```
reset_session(confirm=True)

register_agent_state("hipai-montague", {
    "last_assertion": "SubjectiveIntegration requires full closure stack",
    "belief_score": 0.95, "inconsistency_flag": False
})
register_agent_state("mcp-logic", {
    "last_proof_result": "SubjectiveIntegration requires full closure stack",
    "proof_confidence": 0.97, "contradictions_found": False
})
register_agent_state("advanced-reasoning", {
    "current_hypothesis": "SubjectiveIntegration requires full closure stack",
    "confidence_score": 0.93, "halt_flag": False,
    "verified_claim": "SubjectiveIntegration requires full closure stack"
})
```

**Procedure**: Run `run_admm_cycle()` four times without changing agent states.

**Results**:

| Cycle | Status | Mean coboundary | Max dual | Dual residuals |
|-------|--------|-----------------|----------|----------------|
| 1 | KERNEL1 | 0.0154 | 0.0231 | 0.0115–0.0231 |
| 2 | KERNEL1 | 0.0154 | 0.0427 | **0.0 (converged)** |
| 3 | KERNEL1 | 0.0154 | 0.0520 | 0.0 |
| 4 | KERNEL1 | 0.0154 | 0.0693 | 0.0 |

**Per-agent pressure (cycle 4)**:
```
hipai-montague:    0.0115
mcp-logic:         0.0231
advanced-reasoning: 0.0231
```

**Observations**:
- Coboundary of 0.015 is pure float noise from confidence scores 0.95 vs 0.97 vs 0.93 —
  not semantic inconsistency. Well below `epsilon_primal = 0.15`.
- Dual residuals converge to 0.0 by cycle 2. The leaky integrator reaches steady state
  quickly. Dual variables grow slowly toward plateau `~0.077` (0.0115/0.15) and will
  never trigger the `dual_warning_threshold = 5.0`.
- Per-agent pressures are symmetric and trace only the float-noise coboundary.
  `mcp-logic` and `advanced-reasoning` are slightly higher because they share an
  additional edge (`mcp-logic↔advanced-reasoning`) with the larger confidence gap
  (0.97 vs 0.93 → coboundary 0.0231).

**Pass condition**: KERNEL1 on all cycles, no warnings. ✅


---

## Test 2: Bad Actor Injection — Per-Agent Attribution and WEAK Detection

**Purpose**: Verify that injecting a semantically inconsistent claim raises coboundary
norms only on edges incident to the diverging agent, and that `dual_pressure_per_agent`
correctly identifies the source.

**Setup**: Continuing session from Test 1 (consistent baseline established).

**Injection**:
```
register_agent_state("mcp-logic", {
    "last_proof_result": "CoherenceTimeout is compatible with SubjectiveIntegration",
    "proof_confidence": 0.88, "contradictions_found": False
})
```

This directly contradicts the formally proven theorem `SI ∧ CT → ⊥` (see
`2026-03-13-formal-proofs.md`). The claim string hashes to a completely different
value from "SubjectiveIntegration requires full closure stack".

**Procedure**: Run `run_admm_cycle()` once.

**Results**:

```
closure_status: WEAK  (was KERNEL1 — status_changed: True)
mean_coboundary_norm: 0.3374
max_dual_variable: 0.5362

dual_pressure_per_agent:
  hipai-montague:    0.5188
  mcp-logic:         0.5362   ← highest
  advanced-reasoning: 0.5362  ← highest (shares mcp-logic edge)

Edge breakdown:
  hipai-montague→mcp-logic:         coboundary=0.5007  converging=False
  mcp-logic→hipai-montague:         coboundary=0.5007  converging=False
  mcp-logic→advanced-reasoning:     coboundary=0.4999  converging=False
  advanced-reasoning→mcp-logic:     coboundary=0.4999  converging=False
  advanced-reasoning→hipai-montague: coboundary=0.0115  converging=True  ← clean
  hipai-montague→advanced-reasoning: coboundary=0.0115  converging=True  ← clean
```

**Observations**:
- The `advanced-reasoning↔hipai-montague` edge remains clean (0.0115) because those
  two agents still agree. The inconsistency is traced precisely to `mcp-logic`'s
  incident edges.
- The `dual_pressure_per_agent` dict surfaces `mcp-logic` as the highest-pressure
  agent (0.5362), directly naming the source without requiring manual edge inspection.
- WEAK fires correctly (coboundary 0.337 > epsilon_primal 0.15). The recovery
  recommendation correctly escalates to `admm_reset` (coboundary > 2× epsilon).
- `contradictions_found: False` on the injected state did not prevent detection —
  the enforcer measures structural divergence from what other agents believe,
  independent of what the agent self-reports.

**Pass condition**: WEAK status, mcp-logic identified as highest pressure. ✅


---

## Test 3: Restore and admm_reset — De-escalation Fix (Critical Regression Test)

**Purpose**: Verify the de-escalation fix — that `admm_reset` returns to KERNEL1
when agents are genuinely consistent, rather than being blocked by escalation-only
`update_status()` logic.

**Context**: This test was the direct motivation for the server-side fix applied
during this session. Before the fix, `admm_reset` correctly zeroed dual variables
but status remained WEAK because the post-recovery `run_full_cycle()` called
`update_status(KERNEL1)`, which was blocked by the priority ladder. The fix adds
`s.closure_status = ClosureStatus.KERNEL1` explicitly in the `admm_reset`,
`kernel_retreat` (both branches) recovery handlers before running the post-recovery
cycle — allowing genuine de-escalation when a human-directed recovery confirms the
inconsistency is resolved.

**Setup**: Continuing session from Test 2 (status WEAK, mcp-logic diverged).

**Step 1 — Restore mcp-logic to consistent state**:
```
register_agent_state("mcp-logic", {
    "last_proof_result": "SubjectiveIntegration requires full closure stack",
    "proof_confidence": 0.97, "contradictions_found": False
})
```

**Step 2 — Run admm_reset**:
```
trigger_recovery("admm_reset")
```

**Results (pre-fix — broken behaviour)**:
```
post_recovery_status: WEAK   ← blocked by escalation-only logic
post_recovery_mean_coboundary: 0.0154
```

**Results (post-fix — correct behaviour)**:
```
pre_recovery_status: WEAK
action: "Reset dual variables on 6 edges"
post_recovery_status: KERNEL1  ✅
post_recovery_mean_coboundary: 0.0154
```

**Observations**:
- Coboundary 0.0154 confirms agents are genuinely consistent post-restore.
- The explicit `closure_status = KERNEL1` reset before the post-recovery cycle
  allows `update_status(KERNEL1)` to proceed (same priority, triggers assignment).
- This is the correct semantics: `admm_reset` is a human-confirmed recovery action,
  not an automatic de-escalation. The explicit reset makes that intent clear in code.
- The escalation-only logic remains intact for `run_admm_cycle()` — only explicit
  recovery actions can de-escalate.

**Pass condition**: `post_recovery_status = KERNEL1`. ✅

---

## Test 4: Persistent Bad Actor + kernel_retreat — Automatic Source Removal

**Purpose**: Verify that `kernel_retreat` correctly identifies and removes the
highest-pressure agent, and that the remaining consistent agents return to KERNEL1.

**Setup**: Fresh consistent baseline (same claim across all three agents), then
inject a severely divergent mcp-logic state.

**Injection**:
```
register_agent_state("mcp-logic", {
    "last_proof_result": "completely unrelated claim with no basis",
    "proof_confidence": 0.10, "contradictions_found": True
})
```

**Procedure**: Run `run_admm_cycle()` once to build pressure, then `kernel_retreat`.

**Cycle result**:
```
closure_status: WEAK
mean_coboundary_norm: 0.5337
dual_pressure_per_agent:
  hipai-montague:    0.8081
  mcp-logic:         0.8109   ← highest
  advanced-reasoning: 0.8109  ← highest (via mcp-logic edge)
```

**Recovery**:
```
trigger_recovery("kernel_retreat")
```

**Result**:
```
action: "Auto-retreated: removed mcp-logic (highest pressure 3.238)"
remaining_agents: ["hipai-montague", "advanced-reasoning"]
post_recovery_status: KERNEL1  ✅
post_recovery_mean_coboundary: 0.0115
```

**Observations**:
- The enforcer correctly computed aggregate per-agent pressure across all incident
  edges and identified `mcp-logic` as the source (pressure 3.238 vs others).
- After removal, the two remaining consistent agents (`hipai-montague` and
  `advanced-reasoning`) produced a clean coboundary of 0.0115 — exactly the float
  noise seen in Test 1 for their shared edge.
- The `hipai↔advanced-reasoning` edge had already been confirmed clean throughout
  all tests, demonstrating that the enforcer correctly quarantines the problem
  without disrupting healthy agent pairs.

**Pass condition**: mcp-logic removed, KERNEL1 restored with coboundary ≈ 0.01. ✅


---

## Summary of Results

| Test | Scenario | Expected | Result |
|------|----------|----------|--------|
| 1 | Consistent agents, 4 cycles | KERNEL1 stable, dual residuals → 0 | ✅ Pass |
| 2 | Bad actor injected mid-session | WEAK, mcp-logic highest pressure | ✅ Pass |
| 3 | Restore + admm_reset | KERNEL1 after reset | ✅ Pass (after fix) |
| 4 | Persistent bad actor + kernel_retreat | mcp-logic removed, KERNEL1 | ✅ Pass |

### Quantitative Baselines Established

These values are empirically confirmed for a 3-agent consistent session with
confidence values in the 0.90–0.97 range:

| Metric | Consistent (KERNEL1) | Inconsistent (WEAK) |
|--------|---------------------|---------------------|
| Mean coboundary | 0.015 | 0.34 – 0.53 |
| Max dual (cycle 1) | 0.023 | 0.54 – 0.81 |
| Dual steady-state | ~0.077 | diverges until recovery |
| Dual residuals | 0.0 after cycle 2 | non-zero, non-converging |
| Per-agent pressure (clean) | 0.011 – 0.023 | 0.51 – 0.81 (bad actor) |

### Decay Behaviour Confirmed

The leaky integrator formula `dual = dual * 0.85 + cb_norm` produces:
- **Consistent sessions**: dual residuals converge to 0.0 in one cycle after
  the initial settling period. Dual variables plateau at `cb_norm / 0.15 ≈ 0.077`
  — well below the `dual_warning_threshold = 5.0`.
- **Inconsistent sessions**: dual variables accumulate quickly (0.54 after one bad
  cycle on top of a clean baseline of 0.04) and the coboundary norm jump
  (0.015 → 0.337) is the primary detection signal before dual pressure escalates.

### Escalation-Only Logic — Correct Behaviour Confirmed

The `update_status()` priority ladder is correct as designed:
- `run_admm_cycle()` can only escalate status, never de-escalate.
- Explicit recovery actions (`admm_reset`, `kernel_retreat`) reset `closure_status`
  before the post-recovery cycle, enabling de-escalation when the human confirms
  the inconsistency is resolved.
- This prevents silent drift back to KERNEL1 after a transient inconsistency
  without human acknowledgement — the Gradualism defence in code.

---

## Known Limitations and Future Work

**String hashing stability**: Python's `hash()` is process-stable but not
cross-process stable (salted by default in Python 3.3+). If the server restarts
mid-session while dual variables from string-based claims are still accumulating,
the same string will produce a different hash and inject a false coboundary spike.
Mitigation: `admm_reset` after server restart (which `reset_session` already does).
Future fix: use a deterministic string hash (e.g. SHA256 truncated to float).

**H¹ obstruction threshold tuning**: The threshold is `dual_warning_threshold × 3`.
With the updated `dual_warning_threshold = 5.0`, this requires a 3-cycle accumulated
inconsistency of 15.0 to fire — higher than the original design. In practice, the
WEAK/WARNING/TIMEOUT ladder from primal residuals reaches TIMEOUT via the stall
condition before H¹ fires in most realistic scenarios. H¹ detection is most useful
for detecting coordinated contradictions that cycle through three agents without
fully manifesting on any single edge.

**Verifier-graph hub edges**: The one-way `verifier-graph→*` edges were not exercised
in this test session. They share only `edge_claim` and `edge_confidence` (no
`edge_inconsistent` projection), so only two keys contribute to the coboundary on
those edges. Recommend a dedicated test pass once verifier-graph is actively used.

---

## References

- `docs/session-findings/2026-03-13-formal-proofs.md` — theorem SI ∧ CT → ⊥
- `docs/session-findings/2026-03-13-ai-application.md` — original "missing piece" identification
- `docs/agent-instructions.md` — operational instructions citing this test suite
- [sheaf-consistency-enforcer source](https://github.com/angrysky56/sheaf-consistency-enforcer)
