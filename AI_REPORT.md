# AI Collaboration Report

**Project Name:** Hybrid LABS Solver (Phase 1)
**Team Name:** Stern42
**Date:** February 1, 2026

## 1. The Workflow

As a solo developer, I utilized AI agents to simulate a hierarchical engineering team structure. This separation of concerns prevented context pollution and ensured rigorous quality control.

* **Architectural Logic (Claude 3.5 Sonnet / ChatGPT-4o):** I used Large Language Models (LLMs) as the "Chief Physicist." Their primary role was to translate the mathematical definition of the LABS Hamiltonian (specifically the interaction terms G2 and G4) from the provided academic literature into algorithmic pseudocode.
* **Code Generation (Cursor IDE):** I used the IDE-integrated agent to translate the pseudocode into executable Python syntax and CUDA-Q kernels. This agent was responsible for handling boilerplate code, such as the Trotterization loop structure.
* **Human Reviewer (The "Gatekeeper"):** I assumed the role of the Lead Engineer. My responsibility was not to write the initial code, but to write the *tests* that the code must pass. I strictly enforced a "Trust but Verify" protocol where no AI-generated code was integrated without passing a pre-defined validation logic.

## 2. Verification Strategy

To validate the code created by AI, I employed a **Test-Driven Generation** approach. I recognized that AI models often hallucinate indices when dealing with combinatorial physics problems. Therefore, I implemented the following specific Unit Tests before accepting the AI's output:

* **Unit Test 1: Ground Truth Sanity Check**
    * **Objective:** Validate that the energy calculation logic matches manual mathematical derivation.
    * **Implementation:** I manually calculated the energy for a trivial sequence N=3, S=[1, 1, -1]. The expected autocorrelation energy is exactly 1.0.
    * **Validation:** I wrote an assertion `assert verify_energy([1, 1, -1]) == 1.0`. The AI's initial code failed this check due to an off-by-one error in the loop range, which I subsequently corrected.

* **Unit Test 2: Physical Symmetry Invariance**
    * **Objective:** Ensure the Hamiltonian construction respects the fundamental symmetries of the LABS problem (Ising symmetry).
    * **Implementation:** I implemented a programmatic check asserting that the energy of a sequence $S$ must equal the energy of its bit-flipped negation $-S$ and its spatial reversal $S_{reverse}$.
    * **Validation:** `assert energy(S) == energy(-S)` and `assert energy(S) == energy(S[::-1])`. This test caught a critical error where the AI initially generated 4-body terms that did not correctly collapse into 2-body terms when indices overlapped.

## 3. The "Vibe" Log

### Win: Automated Combinatorial Indexing
**Scenario:** Implementing the `get_interactions(N)` function.
**Details:** Deriving the explicit indices for the term $\sum_{k} (\sum_{i} s_i s_{i+k})^2$ involves expanding a square, resulting in four nested indices ($i, j, i+k, j+k$). Manually managing the bounds for these loops is tedious and highly error-prone.
**AI Contribution:** The AI agent correctly identified the expansion logic and separated the terms into "Overlapping" (2-body) and "Distinct" (4-body) lists within seconds. This saved approximately 2 hours of manual derivation and debugging time.

### Learn: Context-Aware Prompting
**Scenario:** Implementing the Hybrid Workflow (Exercise 6).
**Initial Mistake:** I prompted the AI to "write the code to run the sampling" within an isolated chat window.
**Result:** The AI generated code that used variables (like `thetas` and `dt`) that were defined in previous cells but not passed to the function, resulting in `NameError`.
**Strategy Shift:** I altered my prompting strategy to include the full initialization context. I explicitly provided the definitions of hyperparameters (N, T, dt) in the prompt. This "Context Dump" allowed the AI to generate a self-contained executable block that ran without scope errors.

### Fail: Over-Optimized Hallucination
**Scenario:** Attempting to optimize the Classical Tabu Search.
**Failure:** I asked the AI to "make the energy calculation faster." The AI suggested using a function `cudaq.kernels.numpy_interop` to offload the calculation to the GPU.
**Correction:** This function does not exist in the current version of the CUDA-Q library. The AI hallucinated a library feature based on the variable name. I fixed this by rejecting the complex optimization and instructing the AI to focus on a standard, readable Python implementation using `numpy` arrays, ensuring stability for the Phase 1 submission.

### Context Dump
Below is an excerpt of the prompt structure used to generate the Hamiltonian decomposition, demonstrating the requirement for separation of G2 and G4 terms:

> **System Prompt:** You are a Quantum Physicist specializing in Ising Hamiltonians.
> **User Prompt:** I need to write a Python function `get_interactions(N)` for the LABS problem.
> The energy is defined as $E = \sum_{k=1}^{N-1} C_k^2$, where $C_k$ is the autocorrelation.
> Please expand the square term $C_k^2$.
> **Constraint:** You must return two separate lists:
> 1. `G2`: A list of pairs `[i, j]` for terms that involve only 2 spins (where indices overlap).
> 2. `G4`: A list of quads `[i, j, k, l]` for terms that strictly involve 4 distinct spins.
> Do not use any external libraries, just standard Python lists.