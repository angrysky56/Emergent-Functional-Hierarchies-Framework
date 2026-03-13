# Emergent Functional Hierarchies Framework (EFHF)

## Overview

This project explores the synthesis of three interrelated theoretical frameworks for understanding complex systems, emergence, and consciousness:

1. **Emergent Functional Hierarchies (EFH)** — A computational mechanics framework that formalizes when macroscopic processes achieve genuine "software" status through causal and computational closure.
2. **Persistence Theory & The Dual Kernel Model** — A thermodynamic extension that treats emergence as an active survival strategy, with two domains: Kernel 1 (reversible, information-preserving computation) and Kernel 2 (irreversible hardware).
3. **The Triadic Kernel** — A cognitive architecture mapping (Differentiated State, Autonomous Boundary, Teleological Action) that operationalizes closure modalities as agent-theoretic axioms.

## Key Concepts

### Closure Modalities (EFH)
- **Informational Closure**: The macro-level is self-predictive; micro-data adds zero predictive power.
- **Causal Closure**: Macro-interventions are sufficient for system control; ε-machine ≅ υ-machine.
- **Computational Closure**: The macro-automaton is a coherent coarse-graining of the micro-automaton (commuting diagram holds).

### Key Theorems (EFH)
- **Theorem 1**: For spatial coarse-grainings, Informational Closure ↔ Causal Closure.
- **Theorem 2**: Informational Closure → Computational Closure (for spatial coarse-grainings).

### Dual Kernel Model (Persistence Theory)
- **Kernel 1**: Domain of reversible, information-preserving computation. Mapped to computationally closed levels in EFH.
- **Kernel 2**: Domain of irreversible hardware where software has collapsed.
- **Coherence Timeout**: The transition boundary where Λ(K_T(t)) > τ_{T,m} — the causal diameter of the Reciprocal Coherence Kernel exceeds the relativistic coherence window.
- **Fluctuating Tip**: The entropic edge where the ε-machine first contacts unbufferable environment.

### Triadic Kernel Axioms
- **Differentiated State** → Maps to Informational Closure
- **Autonomous Boundary** → Maps to Causal Closure
- **Teleological Action** → Maps to Computational Closure
- **Subjective Integration** → Recursively integrates all three (consciousness threshold)

### Strong vs Weak Lumpability
- **Strong Lumpability**: Markov property persists regardless of initial distribution. Property of the transition kernel itself. Required for genuine Causal/Informational Closure.
- **Weak Lumpability**: Markov property holds only for specific initial distributions. Sensitive to perturbation.

## Formally Proven Results (Prover9)

### Theorem: Consciousness Requires Full Closure Stack
```
SubjectiveIntegration(x) → Kernel1(x) ∧ ComputationalClosure(x) ∧ CausalClosure(x) ∧ InformationalClosure(x)
```
**Proof**: Via dependency chain SI → TA → AB → DS → IC → {CC, CompC} → K1. Confirmed by Prover9 in 0.00 seconds.

### Theorem: Consciousness and Coherence Timeout are Logically Incompatible
```
SubjectiveIntegration(x) ∧ CoherenceTimeout(x) → ⊥ (contradiction)
```
**Proof**: SI implies CompC via the chain. CT implies ¬CompC via ¬K1 and the K1↔CompC biconditional. The conjunction is formally **unsatisfiable**. This means the K1→K2 transition is a discrete logical phase boundary, not a smooth degradation.

### Structural Finding: Triadic Kernel Generalizes EFH
The EFH framework proves IC↔CC for spatial coarse-grainings. The Triadic Kernel correctly identifies that Differentiated State and Autonomous Boundary are NOT equivalent in general — a system can have divergent internal states without an autonomous boundary (e.g., a thermostat). The Triadic Kernel is the more general framework; EFH provides the special-case mathematical proof for physical systems.

## Application to AI: Weak vs Strong Lumpability

Current AI systems (LLMs) operate in **weak lumpability** — coherent within training distribution but failing under distribution shift. Hallucination IS lumpability failure: the macro-level ε-machine generating outputs that don't commute with micro-level reality.

**Strong lumpability** (knowledge as kernel property, not distribution-dependent) requires a composite architecture:

| Layer | Function | EFHF Analog | Tools |
|-------|----------|-------------|-------|
| 1. Statistical Substrate | Raw stochastic material, weak lumpability | Microscopic substrate X | LLM (transformer attention) |
| 2. Knowledge Graph | Distribution-independent structural relationships | Transition kernel | HiPAI-Montague (world model) |
| 3. Logic Verification | Validates macro-level inferences commute with structure | Lumpability checker | Prover9/Mace4 (mcp-logic) |
| 4. Meta-cognitive Monitor | Tracks coherence, detects boundary approach | Coherence window | Advanced-reasoning (confidence/quality) |

## MCP Servers Used

This project demonstrates the integration of custom MCP servers for formal reasoning:

- **hipai-montague**: Graph-based world model for ontology construction and semantic queries
- **mcp-logic**: Prover9/Mace4 interface for first-order logic proofs and counterexample search
- **advanced-reasoning**: Meta-cognitive reasoning chains with hypothesis tracking, confidence scoring, and persistent memory
- **verifier-graph**: Reasoning provenance tracking
- **cognitive-diagram-nav**: Formal diagram navigation and rewriting

## Project Structure

```
docs/
  Emergent-Functional-Hierarchies-Framework.md  — Core EFHF document
  session-findings/
    2026-03-13-formal-proofs.md     — Formal logic proofs and structural mappings
    2026-03-13-ai-application.md    — AI application analysis
```

## How to Build a System Around These Servers

You only need a **system prompt** that instructs the LLM to coordinate the MCP servers in the correct sequence. The core loop is:

1. **Formulate** hypotheses using LLM knowledge (Layer 1)
2. **Build** explicit world model relationships using hipai-montague (Layer 2)
3. **Verify** structural consistency using mcp-logic proofs (Layer 3)
4. **Monitor** reasoning quality using advanced-reasoning meta-cognition (Layer 4)
5. **Iterate**: Use proof results to refine the world model and generate new hypotheses

A system prompt encoding this loop — with the MCP server URLs and the coordination protocol — is sufficient to create an autonomous reasoning agent that maintains computational closure over its knowledge domains.

## References

- Rosas, F.E., et al. "Software in the natural world: A computational approach to emergence in complex multi-level systems." arXiv:2402.09090 (2024).
- [Is Our Reality an Event Horizon We’re Falling Through? Medium Article by Bill Giannakopoulos](https://medium.com/@bill.giannakopoulos/is-our-reality-an-event-horizon-were-falling-through-fa1d666ca7f9)
- Persistence Theory and Dual Kernel Model synthesis documents (project knowledge).
- Formal proofs conducted via Prover9 (LADR package), March 13, 2026.

## License

Research documentation. Theoretical frameworks referenced are attributed to their respective authors.
