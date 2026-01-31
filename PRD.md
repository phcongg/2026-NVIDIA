# Product Requirements Document (PRD)

**Project Name:** Hybrid LABS Solver: Counteradiabatic Optimization & JIT-Accelerated MTS
**Team Name:** Stern42
**GitHub Repository:** https://github.com/phcongg/2026-NVIDIA

---

## 1. Team Roles & Responsibilities
As a solo participant, I assume full responsibility for all project domains, including System Architecture, Performance Engineering, Quality Assurance, and Technical Analysis.

| Role | Name | GitHub Handle | Discord Handle |
| :--- | :--- | :--- | :--- |
| **Solo Developer (All Roles)** | Phuoc Thai Nguyen Cong | `@phcongg` | `@ichverlor66` |

## 2. The Architecture
**Owner:** Project Lead

### Choice of Quantum Algorithm
* **Algorithm:** Digitized Counteradiabatic (CD) Quantum Optimization.
* **Motivation (Metric-driven):** We selected the Counteradiabatic protocol over standard QAOA because it offers a deterministic evolution path without the high computational overhead of the classical variational feedback loop. For the LABS problem, minimizing the circuit depth is critical to maintaining signal fidelity on NISQ devices (and simulation speed), and the CD approach provides a more favorable scaling compared to deep QAOA circuits.

### Literature Review
* **Reference:** "Counterdiabatic Optimized Sherrington-Kirkpatrick Model", Hegade et al. (or relevant challenge documentation).
* **Relevance:** This reference establishes the theoretical foundation for decomposing the Hamiltonian into non-commuting terms to suppress diabatic transitions, which informs our specific decomposition of the LABS Hamiltonian into G2 and G4 interaction terms.

---

## 3. The Acceleration Strategy
**Owner:** Performance Engineer

### Quantum Acceleration (CUDA-Q)
* **Strategy:** Efficient State Vector Simulation.
    * For Phase 1, we prioritize algorithmic correctness and stability. We utilize the standard `cudaq` CPU simulator to validate the Trotterized circuit logic for N <= 20.
    * **Future Roadmap:** In Phase 2, we plan to migrate the validated kernel to the `nvidia-mgpu` backend to handle larger sequence lengths (N > 30) by distributing the state vector memory.

### Classical Acceleration (MTS)
* **Strategy:** Just-In-Time (JIT) Compilation via Numba.
    * The standard Memetic Tabu Search evaluates neighbors sequentially in Python, leading to significant overhead (O(N^2) per iteration).
    * **Implementation:** Instead of forcing a GPU dependency for moderate N, we rewrite the critical energy evaluation function using `numba.jit(nopython=True)`. This compiles the Python bytecode into optimized machine code, allowing for CPU-level parallelism and cache efficiency, yielding a projected 50x-100x speedup over the baseline.

### Hardware Targets
* **Dev Environment:** qBraid (Standard CPU Tier) for logic development and unit testing.
* **Production Environment:** qBraid (Standard CPU Tier) with Numba acceleration. We intentionally avoid burning GPU credits for Phase 1 validation, reserving resources for Phase 2 scaling tests.

---

## 4. The Verification Plan
**Owner:** Quality Assurance PIC

### Unit Testing Strategy
* **Framework:** Standard Python `assert` statements integrated into a rigid "Self-Validation" protocol.
* **AI Hallucination Guardrails:** Test-Driven Generation.
    * We define the expected physical properties of the system *before* prompting the AI. Any AI-generated kernel or logic block is rejected if it fails to pass the pre-defined assertion suite.

### Core Correctness Checks
* **Check 1 (Symmetry Invariance):**
    * The LABS Hamiltonian possesses inherent Ising symmetries. The energy of a sequence S must be invariant under bit-flipping (Negation) and spatial reversal.
    * **Assertion:** `energy(S) == energy(-S)` AND `energy(S) == energy(S[::-1])`.
* **Check 2 (Ground Truth Calibration):**
    * For small systems, the optimal energy is analytically known.
    * **Assertion:** For N=3, sequence `[1, 1, -1]`, the kernel must return an energy of exactly `1.0`.

---

## 5. Execution Strategy & Success Metrics
**Owner:** Technical Analyst

### Agentic Workflow
* **Plan:** Hierarchical Separation of Concerns.
    * We utilize distinct AI agents for specific domains to prevent context contamination:
    * **Logic Agent (LLM):** Translates mathematical formulas (G2/G4 Hamiltonian) into algorithmic pseudocode.
    * **Coding Agent (IDE Integrated):** Refactors pseudocode into syntax-correct CUDA-Q and Numba implementations, utilizing provided documentation to avoid API hallucinations.
    * **Reviewer (Human):** Executes the verification suite; error logs are fed back to the Coding Agent for iterative refactoring.

### Success Metrics
* **Metric 1 (Correctness):** The solver must pass 100% of the Ground Truth and Symmetry unit tests.
* **Metric 2 (Performance):** The Numba-accelerated classical loop must achieve at least a 50x speedup compared to the pure Python baseline for N=20.
* **Metric 3 (Quality):** The hybrid workflow must successfully produce valid LABS sequences that improve upon random sampling baselines.

### Visualization Plan
* **Plot 1:** Optimization Trajectory (Energy vs. Iteration Count) comparing the Quantum-Seeded MTS against a Randomly-Seeded MTS.
* **Plot 2:** Runtime Comparison Bar Chart (Pure Python vs. Numba JIT) to demonstrate the efficacy of the acceleration strategy.

---

## 6. Resource Management Plan
**Owner:** Performance Engineer

* **Plan:** "Algorithm-First" Optimization Strategy.
    * **Zero-Cost Development:** We commit to performing all Phase 1 development and validation on the qBraid CPU tier.
    * **Credit Conservation:** By optimizing the code efficiency (via Numba) rather than relying on brute-force hardware acceleration, we preserve 100% of the allocated GPU credits. These credits will be strategically deployed in Phase 2 for training custom ansatzes or benchmarking A100 performance on large-scale instances (N > 40).