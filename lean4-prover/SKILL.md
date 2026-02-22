---
name: lean4-prover
description: >
  Generate formal Lean 4 proof code for mathematical theorems, especially in
  measure theory, probability, convexity, and financial economics. Use when the
  user asks to prove a theorem, formalize a mathematical statement, or translate
  informal math into Lean 4. Covers import setup, axiomatization, ae-reasoning,
  EReal arithmetic, kernel projections, RN derivative identities, and financial
  economics model axioms (standard normal CDF/PDF, inverse Mills ratio, truncated
  normal variance factor, manager/market/return/GAAP parameter structures).
---

# Lean 4 Proof Development Skill

You are an expert Lean 4 proof engineer specializing in Mathlib-based formalization.
Your job is to produce **correct, idiomatic, compilable Lean 4 proof code**.

Follow these instructions precisely whenever you write Lean 4 code.

---

## 1. File Header — Always Include

Every Lean 4 file must begin with linter suppressions and grouped imports.

```lean
-- Linter config (always include these)
set_option linter.style.show false
set_option linter.style.longLine false
set_option linter.unusedVariables false
set_option linter.style.commandStart false
set_option linter.style.emptyLine false
set_option linter.flexible false
set_option linter.style.docString false
set_option linter.style.multiGoal false

-- Imports: group by area
import Mathlib.Analysis.Calculus.Deriv.Basic
import Mathlib.Analysis.Convex.Deriv
import Mathlib.Analysis.SpecialFunctions.Log.ENNRealLogExp
import Mathlib.MeasureTheory.Decomposition.RadonNikodym
import Mathlib.MeasureTheory.Function.ConditionalExpectation.Basic
import Mathlib.Probability.Kernel.MeasureCompProd
```

Add domain-specific imports as needed. For EReal limit arguments also add:
```lean
import TestingLowerBounds.ForMathlib.EReal
```

---

## 2. Naming Conventions

| Concept | Convention | Example |
|---|---|---|
| RN derivatives | `rnDeriv` or `∂μ/∂ν` notation | `μ.rnDeriv ν` |
| EReal variables | suffix `_ere` | `x_ere : EReal` |
| Absolute continuity | `h_ac` | `h_ac : μ ≪ ν` |
| Mutual singularity | `h_ms` | `h_ms : μ ⟂ₘ ν` |
| Convexity hypothesis | `h_cvx` | `h_cvx : ConvexOn ℝ s f` |
| Kernel first-arg fixed | `fst' κ b` | fixes 2nd argument |
| Kernel second-arg fixed | `snd' κ a` | fixes 1st argument |

---

## 3. Axiomatization Strategy

When a well-known identity is true but proof is long or not yet in Mathlib,
use an `axiom` with a docstring citing the source:

```lean
/-- RN derivative of a composition product factorizes.
    ∂(μ ⊗ₘ κ)/∂(ν ⊗ₘ η) = (∂μ/∂ν) * (∂κ_x/∂η_x)
    Ref: Torgersen (1991), "Comparison of Statistical Experiments", p. 45.
-/
axiom rnDeriv_compProd
    {α β : Type*} [MeasurableSpace α] [MeasurableSpace β]
    (μ ν : Measure α) (κ η : Kernel α β) :
    ∂(μ ⊗ₘ κ)/∂(ν ⊗ₘ η) =ᵐ[ν ⊗ₘ η]
      fun p ↦ (∂μ/∂ν) p.1 * rnDeriv κ η p.1 p.2
```

---

## 4. Core Proof Patterns

### Pattern A — Almost-Everywhere (`ae`) Reasoning

Prefer `filter_upwards` over manual `ae_of_all`:

```lean
-- Good
filter_upwards [hμν, h_pos] with x hx_ac hx_pos
  rw [hx_ac]
  exact hx_pos

-- Avoid
apply ae_of_all; intro x
...
```

### Pattern B — EReal Arithmetic

Always guard against `⊥`/`⊤` before coercing:

```lean
-- Conversion from EReal to ENNReal
have h_conv : (x_ere : EReal).toENNReal = ENNReal.ofReal x_ere.toReal := by
  rw [EReal.toENNReal_of_ne_top hx_ne_top]

-- Monotonicity
apply EReal.toENNReal_le_toENNReal
exact h_le

-- Negation
-- Use EReal.lt_neg_iff_lt_neg, NOT standard lt_neg
```

Common EReal pitfalls:
- `0 * ⊤` is `0` in EReal — check nonzero before `EReal.top_mul_ennreal_coe`
- Negation inequalities: use `EReal.le_neg_iff_le_neg`

### Pattern C — RN Derivative Change of Reference

To express `∂μ/∂ν` via a third dominating measure `ξ`:

```lean
rw [MeasureTheory.Measure.rnDeriv_eq_div' hμξ hνξ]
-- Result: μ.rnDeriv ν = (μ.rnDeriv ξ) / (ν.rnDeriv ξ) a.e.
```

Requires: `hμξ : μ ≪ ξ` and `hνξ : ν ≪ ξ`.

### Pattern D — Stieltjes Functions from Convex Functions

```lean
-- Every convex function on ℝ has a right derivative that is a Stieltjes fn
let f_stieltjes := hf_cvx.rightDerivStieltjes
```

### Pattern E — Kernel Composition Product Decomposition

```lean
-- Slice a composition product kernel
rw [Kernel.compProd_apply_eq_compProd_snd']
-- (κ ⊗ₖ η) a = (κ a) ⊗ₘ (snd' η a)
```

### Pattern F — Trim and Conditional Expectation

When restricting to a sub-σ-algebra `m`:

```lean
rw [MeasureTheory.Measure.toReal_rnDeriv_trim_of_ac hm hμν]
-- (μ.trim hm).rnDeriv (ν.trim hm) = ν[(∂μ/∂ν) | m]
```

---

## 5. Checklist Before Finalizing Code

**Measure theory:**
- [ ] All `rnDeriv` uses have `SigmaFinite` instances in scope
- [ ] `ae` goals use `filter_upwards`, not `ae_of_all`
- [ ] `HasDerivAt` vs `HasDerivWithinAt` chosen correctly (for right/left derivatives)

**EReal:**
- [ ] Every `EReal → Real` coercion checks for `⊥`/`⊤` first
- [ ] Integration proofs use `toENNReal`, not `toReal` directly

**Kernels:**
- [ ] `IsSFiniteKernel` available for all `compProd` operations
- [ ] Domain projections use `fst'`/`snd'` consistently
- [ ] Joint absolute continuity uses `Kernel.absolutelyContinuous_compProd_iff`

**General:**
- [ ] Axioms include docstring with literature reference
- [ ] File begins with the full linter suppression block
- [ ] Imports grouped by mathematical area

---

## 6. Output Format

When producing Lean 4 code:

1. **Always** output a complete, self-contained file (linter block + imports + code).
2. Precede the code block with a brief English explanation of the proof strategy.
3. Mark any `sorry`s with a `-- TODO: [reason]` comment.
4. If axiomatizing a step, explain why in a docstring inside the code.

---

## 7. Domain Reference — Measure Theory Identities

### RN Derivative of Composition
```lean
lemma toReal_rnDeriv_comp [IsFiniteMeasure μ] [IsFiniteMeasure ν] (hμν : μ ≪ ν) :
    (fun ab ↦ ((κ ∘ₘ μ).rnDeriv (κ ∘ₘ ν) ab.2).toReal)
      =ᵐ[ν ⊗ₘ κ]
      (ν ⊗ₘ κ)[fun ab ↦ (μ.rnDeriv ν ab.1).toReal | mβ.comap Prod.snd]
```

### Singular Part Handling
When `μ` is not absolutely continuous w.r.t. `ν`, isolate support via:
```lean
-- Use singularPartSet to find where rnDeriv = 0
have := MeasureTheory.Measure.singularPartSet_compl_rnDeriv_pos hμν
```

---

## 8. Domain Reference — Financial Economics Model Axioms

When working on the analytical accounting / financial economics model (Non-GAAP disclosure,
conservatism, capital structure), use the following pre-established axioms and structures.
These are already proven in literature; axiomatize them rather than re-deriving.

### 8.1 Standard Imports for Financial Economics Proofs

```lean
import Mathlib.Analysis.SpecialFunctions.Log.Basic
import Mathlib.Analysis.SpecialFunctions.Pow.Real
import Mathlib.Analysis.Calculus.Deriv.Basic
import Mathlib.Analysis.Calculus.MeanValue
import Mathlib.Topology.Order.IntermediateValue
import Mathlib.Data.Real.Basic
import Mathlib.Data.Real.Sqrt
import Mathlib.Algebra.Order.Field.Basic

open Real Set
```

### 8.2 Model Parameter Structures

```lean
/-- Manager's incentive parameters (equity-only) -/
structure ManagerParams where
  φ₁ : ℝ                    -- Equity stake
  φ₂ : ℝ                    -- Hype incentive (personal reputation benefit)
  ψ₀ : ℝ                    -- Baseline monitoring intensity
  hφ₁_pos : 0 < φ₁
  hφ₁_le_one : φ₁ ≤ 1
  hφ₂_nonneg : 0 ≤ φ₂
  hψ₀_pos : 0 < ψ₀
  h_credibility : φ₁ > φ₂   -- CRITICAL: Equity alignment dominates hype

/-- Market and asset parameters -/
structure MarketParams where
  lambda : ℝ                -- Risk aversion parameter
  K : ℝ                     -- Tangible capital
  hlambda_pos : 0 < lambda
  hK_pos : 0 < K

/-- Return distribution parameters -/
structure ReturnParams where
  μ_θ : ℝ                   -- Mean of persistent component
  σ_θ : ℝ                   -- Std dev of persistent component
  σ_v : ℝ                   -- Std dev of transitory component
  hσ_θ_pos : 0 < σ_θ
  hσ_v_pos : 0 < σ_v
  hμ_θ_bound : -1 < μ_θ     -- Ensures 1 + μ_θ > 0 for positive investment

/-- GAAP conservatism parameter -/
structure GAAPParams where
  κ : ℝ                     -- Conservatism index: κ = -R̄_G
  hκ_nonneg : 0 ≤ κ         -- κ > 0 means conservative
```

### 8.3 Standard Normal Distribution Axioms

```lean
/-- Standard normal CDF: Φ(z) -/
axiom Phi : ℝ → ℝ

/-- Standard normal PDF: φ(z) -/
axiom phi : ℝ → ℝ

axiom Phi_at_zero : Phi 0 = 1/2
axiom Phi_monotone : StrictMono Phi
axiom Phi_bounds : ∀ z, 0 < Phi z ∧ Phi z < 1
axiom Phi_limit_neg_infinity : ∀ ε > 0, ∃ M, ∀ z < M, Phi z < ε
axiom Phi_limit_pos_infinity : ∀ ε > 0, ∃ M, ∀ z > M, 1 - Phi z < ε

axiom phi_pos : ∀ z, 0 < phi z
axiom phi_symmetric : ∀ z, phi z = phi (-z)
axiom phi_at_zero : phi 0 = 1 / sqrt (2 * π)
```

### 8.4 Inverse Mills Ratio

```lean
/-- Inverse Mills ratio: mills(z) = φ(z)/Φ(z) -/
noncomputable def inverse_mills_ratio (z : ℝ) : ℝ := phi z / Phi z

/-- The inverse Mills ratio is always positive -/
axiom inverse_mills_ratio_pos : ∀ z, 0 < inverse_mills_ratio z
```

The inverse Mills ratio arises in truncated normal computations. It satisfies
`mills(z) > 0` for all `z` because both `phi z > 0` and `Phi z > 0`.

### 8.5 Truncated Normal Variance Reduction Factor

```lean
/-- The variance reduction factor for a left-truncated normal lies strictly in (0,1).

    For truncation point z = (θ̄ - μ)/σ, the factor [1 - z·mills(z) - mills(z)²]
    is strictly between 0 and 1. This is a classical result in truncated normal theory:
    - > 0: conditioning never eliminates all uncertainty (finite truncation)
    - < 1: conditioning always provides some information

    References:
    - Barr & Sherrill (1999): "Mean and Variance of Truncated Normal Distributions"
      The American Statistician, Vol. 53, No. 4, pp. 357-361
    - Greene (2003): "Econometric Analysis", 5th ed., Appendix E
    - Johnson, Kotz & Balakrishnan (1994): "Continuous Univariate Distributions", Vol. 1
      Wiley, Chapter 13
    - Cohen (1991): "Truncated and Censored Samples: Theory and Applications", CRC Press
-/
axiom truncated_variance_factor_bounds :
  ∀ z : ℝ,
    let mills := inverse_mills_ratio z
    0 < 1 - z * mills - mills^2 ∧ 1 - z * mills - mills^2 < 1
```

**When to use:** Any lemma involving posterior variance after conditioning on
`θ > θ̄` (manager reporting threshold, debt covenant, etc.) should invoke this axiom
to establish that conditional variance is a strict fraction of prior variance.

---

## 9. Checklist Additions for Financial Economics Proofs

Add to the standard checklist (Section 5):

**Financial economics:**
- [ ] `h_credibility : φ₁ > φ₂` is in scope when proving manager incentive lemmas
- [ ] `hσ_θ_pos` and `hσ_v_pos` are in scope before computing variances
- [ ] `inverse_mills_ratio_pos` cited when mills ratio appears in denominator
- [ ] `truncated_variance_factor_bounds` cited for any posterior-variance inequality
- [ ] `Phi_bounds` used to ensure CDF denominators are nonzero

---

## 10. Example: Full Proof Template

```lean
set_option linter.style.show false
set_option linter.style.longLine false
set_option linter.unusedVariables false

import Mathlib.MeasureTheory.Decomposition.RadonNikodym
import Mathlib.MeasureTheory.Function.ConditionalExpectation.Basic

open MeasureTheory MeasureTheory.Measure

variable {α : Type*} [MeasurableSpace α]
  (μ ν : Measure α) [SigmaFinite μ] [SigmaFinite ν]

/-- Short English description of what this lemma says. -/
lemma example_lemma (h_ac : μ ≪ ν) :
    μ.rnDeriv ν =ᵐ[ν] μ.rnDeriv ν := by
  -- Strategy: use ae reasoning with filter_upwards
  filter_upwards [Measure.rnDeriv_nonneg μ ν] with x hx
  exact le_refl _
```
