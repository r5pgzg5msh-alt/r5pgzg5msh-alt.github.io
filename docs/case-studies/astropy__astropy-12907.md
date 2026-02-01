# Case Study — `matplotlib__matplotlib-23476`: DPI doubles after unpickling on M1 Mac

> ✅ **Hit @1 (GT function)**
> **Core idea:** Even though the selected test does *not* assert DPI, its execution trace funnels localization into the figure’s pickle/unpickle magic methods, enabling reranking to prioritize the correct serialization boundary `Figure.__getstate__`.

---

## 0) Metadata (Repro Anchors)

* **Dataset:** SWE-bench-Lite
* **Instance ID:** `matplotlib__matplotlib-23476`
* **Repository:** `matplotlib/matplotlib`
* **Base commit:** `33a0599711d26dc2b79f851c6daed4947df7c167`
* **Issue:** [https://github.com/matplotlib/matplotlib/issues/23471](https://github.com/matplotlib/matplotlib/issues/23471)
* **Ground Truth**

  * ✅ **GT edit location:** `lib/matplotlib/figure.py:Figure.__getstate__`
  * **Neighbor:** `lib/matplotlib/figure.py:Figure.__setstate__`
  * ✅ **GT test:** `lib/matplotlib/tests/test_pickle.py::test_unpickle_canvas`

---

## 1) Input

### Symptom

On M1 MacBooks, repeatedly pickling/unpickling a figure causes DPI to **double** each time (`200 → 400 → 800 → …`), eventually triggering an `OverflowError`.

<details>
<summary><strong>Issue excerpt (original, abridged)</strong></summary>

**Bug:** “When a figure is unpickled, it's dpi is doubled… if done in a loop it can cause an `OverflowError`.”

Repro code pickles and unpickles the same figure in a loop and prints `fig.dpi`.
Actual output shows DPI doubling and eventually crashes in the MacOSX backend during unpickling.

</details>

---

## 2) Step 1 — Relevant Test Retrieval (Two-Phase)

IssueExec localizes via **issue → tests → traces → code**.

<details>
<summary><strong>BM25 Top-10 (lexical filtering)</strong></summary>

```
lib/matplotlib/tests/test_axes.py::test_vert_violinplot_custompoints_200
lib/matplotlib/tests/test_axes.py::test_horiz_violinplot_custompoints_200
lib/matplotlib/tests/test_arrow_patches.py::test_fancyarrow_dpi_cor_200dpi
lib/matplotlib/tests/test_figure.py::test_change_dpi
lib/matplotlib/tests/test_figure.py::test_subfigure_dpi
lib/matplotlib/tests/test_figure.py::test_set_fig_size
lib/matplotlib/tests/test_pickle.py::test_unpickle_canvas
lib/matplotlib/tests/test_figure.py::TestSubplotMosaic::test_user_order
lib/matplotlib/tests/test_figure.py::TestSubplotMosaic::test_nested_user_order
lib/matplotlib/tests/test_png.py::test_truncated_file
```

</details>

<details>
<summary><strong>LLM-selected T<sub>d</sub> (semantic selection)</strong></summary>

```
lib/matplotlib/tests/test_pickle.py::test_unpickle_canvas
lib/matplotlib/tests/test_figure.py::test_change_dpi
lib/matplotlib/tests/test_figure.py::test_subfigure_dpi
lib/matplotlib/tests/test_pickle.py::test_simple
lib/matplotlib/tests/test_pickle.py::test_complete
```

</details>

**Why this test-mediated step matters**
The issue mentions DPI, but the mechanism is tied to **pickle/unpickle state transitions**. Pure issue→code retrieval can drift toward backend scaling or generic DPI logic. Tests provide executable anchors into the true serialization path.

---

## 3) Step 2 — Trace-Guided Analysis (Execution Causal Skeleton)

### Key retrieved test (GT test)

```python
def test_unpickle_canvas():
    fig = mfigure.Figure()
    assert fig.canvas is not None
    out = BytesIO()
    pickle.dump(fig, out)
    out.seek(0)
    fig2 = pickle.load(out)
    assert fig2.canvas is not None
```

**Blind spot:** This test checks `canvas` after unpickling, not DPI.
**Value:** The *trace* still exposes the must-pass serialization boundary.

<details>
<summary><strong>Coverage / call graph (test → callees)</strong></summary>

```
lib/matplotlib/tests/test_pickle.py::test_unpickle_canvas →
  • lib/matplotlib/_api/deprecation.py::wrapper
  • lib/matplotlib/artist.py::Rectangle.__getstate__
  • lib/matplotlib/cbook/__init__.py::CallbackRegistry.__getstate__
  • lib/matplotlib/cbook/__init__.py::CallbackRegistry.__setstate__
  • lib/matplotlib/figure.py::Figure.__getstate__
  • lib/matplotlib/figure.py::Figure.__setstate__
  • lib/matplotlib/transforms.py::Affine2D.__getstate__
  • lib/matplotlib/transforms.py::Affine2D.__setstate__
  • lib/matplotlib/transforms.py::Bbox.__getstate__
  • lib/matplotlib/transforms.py::Bbox.__setstate__
  • lib/matplotlib/transforms.py::BboxTransformTo.__getstate__
  • lib/matplotlib/transforms.py::BboxTransformTo.__setstate__
  • lib/matplotlib/transforms.py::TransformedBbox.__getstate__
  • lib/matplotlib/transforms.py::TransformedBbox.__setstate__
```

</details>

**Suspicious locations (from trace-guided analysis)**

* High/Critical: `Figure.__getstate__`, `Figure.__setstate__`
* (others omitted)

---

## 4) Step 3 — Contextual Refinement

IssueExec retrieves structure/context around `lib/matplotlib/figure.py` so reranking can distinguish:

* root-cause boundary (state capture) vs
* symptom neighbor (state restore / backend interactions)

---

## 5) Step 4 — Final Reranking (Ranked Edit Locations)

<details>
<summary><strong>Candidate contexts (abridged)</strong></summary>

**Location 1 (GT): `Figure.__getstate__`**

```python
def __getstate__(self):
    state = super().__getstate__()
    state.pop("canvas")
    state["_cachedRenderer"] = None
    state['__mpl_version__'] = mpl.__version__
    from matplotlib import _pylab_helpers
    if self.canvas.manager in _pylab_helpers.Gcf.figs.values():
        state['_restore_to_pylab'] = True
    return state
```

**Location 2 (neighbor): `Figure.__setstate__`**

```python
def __setstate__(self, state):
    version = state.pop('__mpl_version__')
    restore_to_pylab = state.pop('_restore_to_pylab', False)
    ...
    self.__dict__ = state
    FigureCanvasBase(self)  # Set self.canvas.
    ...
    self.stale = True
```

</details>

### Final ranked edit locations `L*`

1. ✅ `lib/matplotlib/figure.py:Figure.__getstate__` *(GT)*
2. `lib/matplotlib/figure.py:Figure.__setstate__`

**Rationale (root cause vs symptom)**

* Stacktrace points to `__setstate__`, but the repeated DPI doubling strongly suggests a **state capture/serialization mismatch** that compounds each pickle cycle.
* Reranking therefore prioritizes the serialization boundary `__getstate__`.

---

## 6) Baseline (SWE-agent) — Failure Summary

* **Early lock-in:** sees `__setstate__ → new_figure_manager_given_figure` and immediately assumes “MacOSX backend scaling bug”.
* **Single-direction debugging:** repeatedly instruments `backend_macosx.py` (`resize`, logs, arm64 branches) without validating the *serialization closed loop* (`__getstate__ ↔ __setstate__`).
* **Workaround drift:** introduces new `rcParam` (`macosx.dpi_scaling`) + config registration + docs, i.e., a “disable behavior” workaround rather than fixing why DPI compounds across pickle cycles.

**Key contrast:** IssueExec uses existing tests + traces to constrain search to pickle/unpickle must-pass methods, preventing speculative backend fixation.

---

## 7) Takeaway

This is a “trace-funneling” win: tests primarily shrink the search space and expose must-pass serialization methods, enabling accurate function-level localization even when the test doesn’t assert DPI directly.

---