# VeRAPAk Formal Verification

!!! abstract "Project Overview"
    **Objective:** Formally proved the termination and geometric correctness of the VeRAPAk neural network verification framework, mathematically guaranteeing the integrity of its N-dimensional partitioning engine.
    
    **Tech Stack:** `Dafny` `Python` `Atheris Fuzzer` `Formal Methods` `Z3 Theorem Prover`
    
    **Publication:** Case Study targeting FMCAD 2026

## System Architecture & The Formal Model

VeRAPAk operates by evaluating the robustness of neural networks across N-dimensional continuous input spaces. To guarantee the framework does not miss critical adversarial vulnerabilities or enter infinite loops during verification, the core partitioning engine required strict mathematical grounding. 

Instead of relying solely on empirical testing, I developed a complete formal model of the partitioning algorithm using the **Dafny** verification language. This model translates the Python-based execution logic into a strictly typed mathematical domain, establishing invariants for geometric validities, area preservation, and program termination.

![VeRAPAk Verification Architecture](/assets/images/verapak-architecture.png)
/// caption
VeRAPAk main verification logic flow
///

### Geometric Representation in Dafny

To represent the continuous search space, the model abstracts Python's NumPy arrays into heavily constrained Dafny datatypes. The input space is defined as a `Region`, consisting of two `NDArray` structures (a low and a high bound) that mathematically represent an N-dimensional hyper-rectangular search region. An illustration of this surrounding search space on an input is shown below. 

![Input Region](/assets/images/region.png)
/// caption
Illustrative visualization of an N-dimensional input region representing bounded search space around a nominal network input
///

Because Dafny lacks a native N -dim array comparable to Python, our solution is to formalize a custom generic datatype that isolates dimensions from the contents. Crucially, we represent the data as a flattened sequence rather than recursively nested sequences. Deeply nested structures in Dafny necessitate heavy use of nested quantifiers, which frequently cause excessive instantiations and divergence within the underlying Z3 SMT solver, rapidly leading to resource exhaustion and unpredictable proof timeouts.

```dafny title="Core Datatypes and Validity"
// Abstract representation of an N-dimensional array
datatype NDArray<T> = Array(shape: seq<nat>, data: seq<T>)

// Represents a hyper-rectangular search region defined by lower and upper bounds
datatype Region<T> = Region(low: NDArray<T>, high: NDArray<T>)

// Ensures a region's geometric bounds are structurally sound
predicate ValidRegion(region: Region<real>)
{
    |region.low.data| == |region.high.data|
    && region.low.shape == region.high.shape
    && (forall k :: 0 <= k < |region.low.data| ==> region.low.data[k] <= region.high.data[k])
}
```

---

## Core Subsystems & Mathematical Proofs

### Dimension Selection & Axiomatic Bridging

Before subdividing a region, VeRAPAk dynamically selects the most optimal dimension to partition. A significant challenge in verifying Python-based data science frameworks is their reliance on C-extensions (like NumPy) which are extremely difficult to formally verify.

To handle VeRAPAk's dimension ranking strategy, I integrated an axiomatic representation of `NumPy.argsort`. By defining the strict post-conditions of the sort—guaranteeing valid indices, distinct values, and sortedness—without evaluating the underlying C code, the Dafny model successfully proves the logic of the `LargestFirstDimSelection` heuristic. This ensures the engine consistently targets dimensions with the largest interval widths.

### Geometric Correctness & Z3 Solver Optimization

When VeRAPAk subdivides a continuous input region, it must guarantee that the new subregions cover the exact same mathematical volume as the parent—no overlaps, and no missed gaps. I modeled this through the `hierarchicalDimensionRefinement` method, which mathematically guarantees four critical properties for every partition operation:

1. **Cardinality:** The exact number of generated subregions is $d^M$.
2. **Spatial Containment:** Every subregion is strictly bounded by the parent region.
3. **Mutual Disjointness:** No two subregions share internal geometric volume.
4. **Volume Preservation:** The sum of all subregion volumes exactly equals the parent volume.

``` Dafny title="Partitioning Contracts"
method hierarchicalDimensionRefinement(region: Region<real>, num_dimensions: nat, divisor: int) 
    returns (retVal: seq<Region<real>>)
  // Pre-conditions: Valid input geometry and valid partitioning parameters
  requires nonEmptyRegion(region) && ValidRegion(region) && divisor > 0
  requires num_dimensions <= |region.low.data| && num_dimensions <= |region.high.data|
  
  // 0. Validity: All generated subregions remain geometrically valid
  ensures (forall r :: r in retVal ==> ValidRegion(r))
  
  // 1. Cardinality: The exact number of generated subregions is d^M
  ensures |retVal| == power(divisor, num_dimensions)
  
  // 2. Spatial Containment: Every subregion is strictly bounded by the parent region
  ensures (forall r :: r in retVal ==> |r.low.data| == |region.low.data|)
  ensures (forall r :: r in retVal ==> |r.high.data| == |region.high.data|)
  ensures (forall r :: r in retVal ==> IsContained(r, region))
  
  // 3. Mutual Disjointness: No two subregions overlap in internal volume
  ensures (forall r1, r2 ::r1 in retVal && r2 in retVal && r1!=r2 ==> RegionsDisjoint(r1, r2))
  
  // 4. Volume Preservation: The sum of subregion volumes exactly equals the parent volume
  ensures SumArea(retVal) == GetArea(region)
```


#### Hiding Non-Linear Arithmetic from Z3

Proving volume preservation inherently requires calculating N-dimensional geometric volume (multiplication) and subdividing it (division). Standard theorem provers like Z3 often face heuristic instability and infinite loops when evaluating complex non-linear arithmetic.

To force the proof to complete efficiently, I heavily utilized Dafny's `{:opaque}` attribute. By wrapping the division logic inside an opaque function, I prevented the Z3 solver from continuously unrolling the math. I then designed a suite of explicit structural lemmas (such as `LemmaSumAreaDistributive` and `LemmaRegionAreaFactorization`) to manually guide the solver through the mathematical proofs without triggering infinite heuristic searches.

```dafny title="Opaque Function for Z3 Optimization"
// Opaque wrapper function to hide the non-linear arithmetic (multiplication/division) 
// from the global Z3 solver context, preventing heuristic instability.
function {:opaque} GetSplitOffset(total_diff: real, k: int, divisor: int): real
requires divisor > 0
requires total_diff >= 0.0
{
    total_diff * (k as real / divisor as real)
}
```

### Proving Termination Across Continuous Space

Because VeRAPAk evaluates a continuous real-number state space, proving that the verification loop will eventually terminate across all possible traces is a massive formal methods challenge.

To achieve this, I engineered a well-founded decreasing measure within Dafny. By mapping the continuous geometric regions into discrete computational budgets via a `WorkItem` datatype, the model assigns "fuel" (`splits_left`) to every region. Utilizing a recursive `TotalWeight` function, the proof guarantees that every single iteration of the partitioning engine strictly decreases the total mathematical weight of the system, mathematically forcing termination.

---

## Concrete Validation

While our Dafny model provides absolute mathematical guarantees over the abstract logic, the concrete execution of VeRAPAk relies on Python, external extensions, and discrete floating-point arithmetic. To bridge this semantic gap, we developed a contract-aware fuzzing test suite using Atheris, a coverage-guided fuzzing engine. Rather than treating the fuzzer as a black box, we leverage the Dafny specifications to directly dictate both input generation and output validation. 

![Atheris Fuzzing Pipeline](/assets/images/verificationDrivenFuzzing.drawio.png)
/// caption
Contract-constrained fuzzing pipeline utilizing Atheris to validate runtime Python execution against Dafny invariants.
///


### Precondition-Guided Generation

Standard mutation strategies rely on random byte modifications. When applied to complex $N$-dim geometric data, these random mutations can lead to deserialization failures or input invalidation. Additionally, unconstrained random mutation can cause inputs to be erroneous, failing to adhere to the functions precondition. 

To ensure the mutation of the input is varied to provide increased coverage while still enforcing our preconditions of the Dafny model, we create a custom, precondition-guided mutator. The preconditions defined in our Dafny model directly govern this mutation logic. It operates on the deserialized `Region` objects, applying a randomly selected geometric transformation such as boundary sliding, expansion, or shrinking. Crucially, the mutator is contract-aware, forcing the adherence to the preconditions prior to execution. One such specification for the partitioning procedure is the lower bounds never exceed the upper bounds and no dimension is 0. By embedding these preconditions into the generation phase, we guarantee the fuzzer only produces valid inputs for downstream validation. 

### Postcondition Oracles

Once a valid input is generated, it is executed against the native Python codebase function of interest. To evaluate correctness of the resulting concrete outputs, we construct test oracles derived directly from the Dafny postconditions. However, translating continuous, infinite-precision real constraints into bit-precise Python assertions yields false positives due to IEEE 754 floating-point rounding errors. To bridge this mathematical domain mismatch, our oracles evaluate constraints using hardware-appropriate tolerances (e.g., relative $R_{tol}=1e^{-4}$ and absolute $A_{tol}=1e^{-7}$ via `np.isclose`). This strictly enforces the formal intent while accommodating permissible precision drift.

Translating Dafny's formal postconditions into executable Python oracles requires mapping symbolic quantifiers to highly optimized array operations. To verify properties like spatial containment and mutual disjointness across $N$ dimensions, the oracle avoids slow, deeply nested loops. Instead, these first-order logic statements are translated into vectorized NumPy operations, utilizing combinatorial iterators and element-wise boolean arrays aggregated via `np.all`. This direct translation ensures the oracle remains mathematically faithful and computationally efficient during high-throughput fuzzing.

---

## Evaluation & Performance Metrics

### Annotation Overhead & Z3 Optimization

Formalizing continuous-domain logic requires a significant engineering tradeoff between core execution logic and proof scaffolding. While the primary Dafny execution logic—including datatypes, dimension selection axioms, and the `hierarchicalDimensionRefinement` loops—is highly compact at approximately 100 lines, mathematically isolating the non-linear arithmetic over $N$-dimensional space required an additional 300 lines of auxiliary lemmas, ghost-state witnessing, and strict loop invariants. 

Despite this heavy annotation overhead, these structural mitigation strategies successfully rendered the continuous-domain proof tractable. By explicitly guiding the Z3 solver's existential quantification and encapsulating computationally expensive operations, the entire formalized framework verifies in just **2 minutes and 8 seconds**. Without these proactive solver management techniques, the Z3 solver suffered from unpredictable heuristic timeouts exceeding 20 minutes. 

### Fuzzing Results & Operational Constraints

Evaluating the Python codebase against these formal guarantees required managing significant computational complexity. The Atheris precondition-guided mutator generated **100,000 $N$-dimensional permutations** for both the partitioning and dimension-ranking engines. 

To manage combinatorial overhead during the partitioning tests, the maximum number of generated child subregions was clamped to 1,000. This constraint directly aligns with VeRAPAk's intended operational behavior, which favors iterative refinement over massive single-step partitioning. Enforcing this bound allowed the engine to process all 100,000 geometric samples in approximately 9 minutes, while the dimension ranking function evaluated in 90 minutes.

Across all 200,000 combined iterations, the fuzzer recorded **zero semantic violations, false positives, or contract failures**. In the context of formal verification, this absence of crashes yields two critical insights:
1. **Execution Parity:** It provides high confidence that VeRAPAk's core Python implementation faithfully mirrors the rigorous specifications proven in the Dafny model.
2. **Domain Bridging:** It validates the testing methodology, demonstrating that Dafny's infinite-precision mathematical real domain can be safely evaluated against Python's hardware-limited floating-point domain without triggering false logic violations from floating-point rounding errors.

---

## Source Code

This formal grounding ensures VeRAPAk operates with mathematical certainty when evaluating neural network robustness, eliminating the risk of infinite loops and verification gaps. The complete Dafny modeling, non-linear arithmetic lemmas, and empirical fuzzing results are actively being compiled into a case study paper for the FMCAD 2026 conference.

- **Research Deliverable:** Packaged as a reproducible Docker-based research artifact for peer review.
- **Framework Documentation:** [verapak.net](https://www.verapak.net/)

[View Dafny Proof Repository](https://github.com/formal-verification-research/VeRAPAkTerminationProof){ .md-button .md-button--primary }