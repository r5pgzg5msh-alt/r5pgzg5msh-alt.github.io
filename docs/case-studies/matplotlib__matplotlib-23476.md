# Case Study — `astropy__astropy-12907`: `separability_matrix` incorrect for nested CompoundModels

> ✅ **Hit @1 (GT function)**
> **Core idea:** Test retrieval anchors the issue to `test_separable`, traces reveal the `&` combiner `_cstack`, and reranking aligns the exact erroneous 2×2 all-True block with the ndarray branch in `_cstack`.

---

## 0) Metadata (Repro Anchors)

* **Dataset:** SWE-bench-Lite
* **Instance ID:** `astropy__astropy-12907`
* **Repository:** `astropy/astropy`
* **Base commit:** `d16bfe05a744909de4b27f5875fe0d4ed41ce607`
* **Issue:** [https://github.com/astropy/astropy/issues/12906](https://github.com/astropy/astropy/issues/12906)
* **Ground Truth**

  * ✅ **GT edit location:** `astropy/modeling/separable.py::_cstack`

---

## 1) Input

### Symptom

`separability_matrix` gives correct block-diagonal structure for flat compound models, but for **nested** compounds it incorrectly fills the bottom-right 2×2 block with all `True`, making two `Linear1D` models appear coupled.

<details>
<summary><strong>Issue excerpt (original, abridged)</strong></summary>

* `cm = Linear1D & Linear1D` → diagonal matrix (expected)
* `Pix2Sky_TAN() & Linear1D & Linear1D` → correct block structure
* `Pix2Sky_TAN() & (Linear1D & Linear1D)` → bottom-right 2×2 becomes all True (incorrect)

</details>

---

## 2) Step 1 — Relevant Test Retrieval (Two-Phase)

<details>
<summary><strong>BM25 Top-10 (lexical filtering)</strong></summary>

BM25 Top-10 mostly hits unrelated tests (mask/table/diff); the most relevant `test_separable` appears only around **Top-23**, illustrating cross-module noise / vocabulary mismatch.

(Excerpt)

```
astropy/table/tests/test_masked.py::TestTableInit::test_mask_false_if_input_mask_not_true
astropy/table/tests/test_masked.py::TestTableInit::test_mask_false_if_no_input_masked
astropy/modeling/tests/test_models.py::test_custom_separability_matrix
astropy/table/tests/test_table.py::Test__Astropy_Table__::test_simple_1
astropy/table/tests/test_init_table.py::test_init_and_ref_from_dict
astropy/modeling/tests/test_core.py::test_compound_model_with_bounding_box_true_and_single_output
astropy/table/tests/test_init_table.py::test_init_and_ref_from_multidim_ndarray
astropy/modeling/tests/test_core.py::test_model_with_bounding_box_true_and_single_output
astropy/utils/tests/test_diff.py::test_diff_values_false
astropy/io/votable/tests/table_test.py::TestVerifyOptions::test_pedantic_false
...
(astropy/modeling/tests/test_separable.py::test_separable ~ top-23)
```

</details>

<details>
<summary><strong>LLM-selected T<sub>d</sub> (semantic selection)</strong></summary>

```
astropy/modeling/tests/test_separable.py::test_separable
astropy/modeling/tests/test_separable.py::test_custom_model_separable
astropy/modeling/tests/test_separable.py::test_coord_matrix
astropy/modeling/tests/test_separable.py::test_cdot
astropy/modeling/tests/test_separable.py::test_cstack
```

</details>

**Why test retrieval is necessary here**
The issue is about a wrong boolean matrix pattern, not a crash or named function. Mapping the issue to a real test entry gives a verifiable execution skeleton to constrain where the matrix is assembled.

---

## 3) Step 2 — Trace-Guided Analysis (Execution Causal Skeleton)

<details>
<summary><strong>Coverage / call graph (test → suspect chain)</strong></summary>

```
astropy/modeling/tests/test_separable.py::test_separable →
  • astropy/modeling/separable.py::is_separable
  • astropy/modeling/separable.py::separability_matrix

astropy/modeling/separable.py::is_separable →
  • astropy/modeling/core.py::CompoundModel.n_inputs
  • astropy/modeling/core.py::CompoundModel.n_outputs
  • astropy/modeling/separable.py::_separable

astropy/modeling/separable.py::separability_matrix →
  • astropy/modeling/core.py::CompoundModel.n_inputs
  • astropy/modeling/core.py::CompoundModel.n_outputs
  • astropy/modeling/separable.py::_separable

astropy/modeling/separable.py::_separable →
  • astropy/modeling/separable.py::_separable
  • astropy/modeling/core.py::CompoundModel._calculate_separability_matrix
  • astropy/modeling/core.py::Mapping._calculate_separability_matrix
  • astropy/modeling/core.py::Polynomial2D._calculate_separability_matrix
  • astropy/modeling/core.py::Rotation2D._calculate_separability_matrix
  • astropy/modeling/core.py::Scale._calculate_separability_matrix
  • astropy/modeling/core.py::Shift._calculate_separability_matrix
  • astropy/modeling/mappings.py::Mapping.n_outputs
  • astropy/modeling/separable.py::_cdot
  • astropy/modeling/separable.py::_coord_matrix
  • ✅ astropy/modeling/separable.py::_cstack
```

</details>

**Suspicious locations (from trace-guided analysis)**

* `separable.py::_separable` (dispatch / recursion)
* ✅ `separable.py::_cstack` (`&` operator combiner)

---

## 4) Step 3 — Contextual Refinement

IssueExec retrieves `separable.py`’s relevant structure so reranking can distinguish:

* router (`_separable`) vs
* block assembly logic (`_cstack`)

---

## 5) Step 4 — Final Reranking (Ranked Edit Locations)

<details>
<summary><strong>Candidate contexts (abridged)</strong></summary>

**Candidate 1: `_separable` (dispatch)**

```python
elif isinstance(transform, CompoundModel):
    sepleft = _separable(transform.left)
    sepright = _separable(transform.right)
    return _operators[transform.op](sepleft, sepright)
```

**Candidate 2 (GT): `_cstack` (combiner for `&`)**

```python
if isinstance(right, Model):
    cright = _coord_matrix(right, 'right', noutp)
else:
    cright = np.zeros((noutp, right.shape[1]))
    cright[-right.shape[0]:, -right.shape[1]:] = 1
```

</details>

### Final ranked edit locations `L*`

1. ✅ `astropy/modeling/separable.py::_cstack` *(GT)*
2. `astropy/modeling/separable.py::_separable`

**Rationale (symptom-level alignment)**

* In nested compounds, `right` likely becomes an ndarray via recursion.
* The ndarray branch fills the sub-block with `1` (all `True`) → exactly matches the erroneous bottom-right 2×2 all-True pattern.

---

## 6) Baseline (SWE-agent) — Failure Summary

* **Environment failure:** build/import fails due to missing compiled extensions; reproduction cannot run.
* **No fallback strategy:** does not switch to static localization (`grep` / `find` for `separability_matrix` / `_cstack`), never reaches `astropy/modeling/separable.py`.
* **Off-target patch:** writes/fixes a standalone simulation script; **no library source file modified**, so the submission is ineffective.

**Key contrast:** IssueExec reuses existing tests and offline-collected traces, avoiding brittle “must build the whole project in-agent” dependencies.

---

## 7) Takeaway

This is the canonical win: tests anchor semantics, traces isolate the `&` combiner, and reranking aligns a precise matrix-shape symptom to the exact faulty branch in `_cstack`.

---