# System Identification and Dynamics Testbed

[Stack placement and ownership](docs/STACK.md) · [License](LICENSE)

A bounded scientific instrument for fitting and evaluating **fully observed, discrete-time linear dynamics**. It produces candidate models and diagnostics for review and downstream testing.

Implemented operation: `sidt.discrete-lti-lstsq.v1`.

\[
x_{k+1}=Ax_k+Bu_k, \qquad y_k=Cx_k+Du_k.
\]

The state trajectory must be supplied explicitly. Optional measured output rows permit fitting `C` and `D`. This implementation cannot identify a hidden state realization from inputs and outputs alone.

## Install and run

Requires Python 3.11 or newer.

```bash
python -m pip install -e '.[test]'
python -m pytest
python examples/replay.py
```

The replay constructs a known two-state system, fits it, evaluates a trajectory generated with a separate seed, and prints matrices, rank, singular values, residuals, and holdout error. It performs no network requests or operational state changes.

## API

```python
import numpy as np
from sidt import fit_lti

# Sample rows: states has N+1 rows; inputs has N transition-aligned rows.
x = np.array([[1.0], [0.8], [0.9], [0.52]])
u = np.array([[0.0], [1.0], [-1.0]])
candidate = fit_lti(
    x, u,
    sample_interval=0.1,              # seconds; fixed across the trajectory
    state_names=("temperature_delta",), state_units=("K",),
    input_names=("heater_power",), input_units=("W",),
)
print(candidate.A, candidate.B)
print(candidate.diagnostics.rank)
print(candidate.candidate_digest)
```

Provide `outputs`, `output_names`, and `output_units` together to fit the optional output equation. `candidate.predict_next` and `candidate.predict_output` take matching sample-row matrices. `evaluate_one_step(candidate, states, inputs, outputs=...)` measures supplied one-step residuals; the caller establishes independence from training data.

| Result | Meaning |
| --- | --- |
| `A`, `B`, optional `C`, `D` | Candidate matrices in the declared state/input/output coordinates |
| `metadata` | Fixed sample interval, variable order, unit labels, operation identifier, optional conditioning reference |
| `diagnostics` | Regression rank, singular values, rank cutoff, residual arrays, per-target residual sums of squares, residual degrees of freedom |
| `candidate_digest` | Content identity for matrices, metadata, schema, and numerical rank policy |

Rank-deficient regression raises `NonIdentifiableError` with rank diagnostics and returns no usable model. Finite values, dimensions, nonempty variable labels, and positive sample interval are checked. Units are recorded labels; dimensional consistency and timestamp alignment remain caller responsibilities.

## System role

PPDA supplies observations and provenance. STFE may condition streams; its operation/result reference belongs in `conditioning_reference`, with full records retained externally. SIDT fits a candidate. GSIE may evaluate that candidate as an explicitly selected dynamics model; OIT may inspect observability for the declared `A` and `C`; SET can test estimation behavior and model mismatch. These are documented handoff boundaries, not implemented cross-repository execution adapters.

SIDT owns the fit and its numerical diagnostics. It does not own evidence truth, canonical state, model adoption, admission, policy, observer tuning, or actuation. A successful fit and a digest do not establish physical validity, causal validity, stability, or operational authority.

Read [the contract](docs/CONTRACT.md), [numerical assumptions](docs/NUMERICS.md), and [stack boundaries](docs/STACK_ROLE.md).

## Optional SET conformance export

```bash
python -m pip install -e '.[test,exchange]'
python examples/exchange.py
```

The optional dependency is pinned to SET revision `bd261a765281a95312f7c91a3857233476294c5b`. `sidt.exchange.export_result` maps explicitly supplied JSON values into SET's existing `notation.instrument.result-artifact.v1` contract and invokes its validator. The example maps the actual fitted matrices into ordered coefficient components, includes full input and numerical-result records, and marks parameter covariance `unknown`. Its timestamps, evidence/execution references, and all-zero source revision are clearly synthetic caller declarations.

This is a conformance export. It does not authenticate provenance, establish independent verification, adopt the candidate, or provide a native CIW execution adapter. Exchange tests skip when the optional validator package is absent; they run when the `exchange` extra is installed.

## License

MPL-2.0. See [LICENSE](LICENSE).
