# Particle Asymmetry Decay — Pseudoexperiment Analysis

**Thomas Griffiths** — Data Analysis and Machine Learning, University of Edinburgh

This notebook (`AsymmetryAnalysis.ipynb`) simulates a measurement of the decay
$B \to DX$, of the kind used in real experiments to probe the matter/antimatter
asymmetry of the universe. Events are generated from a known decay-time PDF, then
an **unbinned maximum-likelihood fit** (via `iminuit`) recovers the lifetime
($\tau$), oscillation frequency ($\Delta m$), and asymmetry ($V$) parameters from
the simulated data — first in isolation, then in the presence of a background
component.

Repeating the generate-and-fit cycle over many **pseudoexperiments** quantifies
how biased and how precise each parameter estimate is, and how that changes once
a background is mixed in.

## The physics model

Signal PDF (unnormalized):

$$p_{sig}(t;\, \tau,\, \Delta m,\, V) \propto \left(1 + V \sin(\Delta m\, t)\right) \exp\left(-\frac{t}{\tau}\right)$$

Background PDF, fixed width $\sigma = 0.5\,\mu s$:

$$p_{bkg}(t;\,\sigma) \propto \exp\left(-\frac{t^2}{2\sigma^2}\right)$$

Combined model with background fraction $f_{bkg}$ as a free parameter:

$$p_{comb}(t) = (1-f_{bkg})\,p_{sig}(t) + f_{bkg}\,p_{bkg}(t)$$

Both PDFs are re-normalized over the detector's recorded window,
$t_{min} = 0.5\,\mu s$ to $t_{max} = 10\,\mu s$, using trapezoidal integration
(a `quad`-based version was tried first but was too slow for repeated use across
hundreds of pseudoexperiments).

Nominal parameter values used to generate pseudo-data throughout:
$\tau_g = 1.5\,\mu s$, $\Delta m_g = 20\,\text{MHz}$, $V_g = 0.1$.

## What's in the notebook

1. **Step 1 — Signal-only fit**
   - Event generation by inverse-CDF sampling (an accept/reject "box method" was
     also implemented but is ~5x slower and left commented out for reference).
   - A single dataset (10,000 events) generated and fit with an unbinned NLL to
     sanity-check the pipeline before scaling up.
   - 100 pseudoexperiments to determine the bias and precision on $\tau$, $\Delta m$, $V$.
2. **Step 2 — Adding a background**
   - Background model and combined signal+background PDF (above).
   - Mixed datasets generated at three background fractions: 1%, 10%, 20%.
   - For each fraction, every pseudoexperiment's mixed dataset is fit **twice** —
     once with a signal-only model (which ignores the background) and once with
     the correct signal+background model — to directly measure the bias introduced
     by ignoring a real background.

## Headline results

**Step 1 (signal-only, 100 pseudoexperiments, no background):**

| Parameter | Outcome |
|---|---|
| $\tau$ | Effectively unbiased — bias ~0.06 standard errors from zero |
| $\Delta m$ | Bias not statistically significant (within 1σ); tightly constrained |
| $V$ | Biased high — mean is ~1.14 standard errors above the true value |

**Step 2 (effect of an unmodeled background):**

| Background fraction | Signal-only fit (mis-specified) | Signal+background fit |
|---|---|---|
| 1% | Negligible deviation from Step 1 | Negligible deviation |
| 10% | $\tau$ and $V$ increasingly underestimated; $\Delta m$ comparatively robust | All parameters recovered with negligible bias |
| 20% | Bias grows further on $\tau$ and $V$ | $f_{bkg}$ itself recovered correctly (<1σ from truth) |

The mechanism: the background peaks near $t_{min}$, which a signal-only fit
misreads as a faster exponential decay (biasing $\tau$ low) and as reduced
oscillation amplitude (biasing $V$ low). $\Delta m$ is comparatively shielded
because the oscillation spacing survives at later times where the background
has died away. Modeling the background explicitly, with $f_{bkg}$ free, removes
the systematic bias in all three signal parameters across all tested fractions.

## Running

No external data files are needed — this is a fully self-contained simulation
notebook. Requirements: `numpy`, `pandas`, `matplotlib`, `scipy`, `iminuit`.

Run top to bottom: later cells depend on variables defined earlier (`events`,
the `f_bkg` list, `results_sig` / `results_mix`), so cells aren't safe to run
out of order.

## Known limitations / future work

- In the Step 2 pseudoexperiment loop, both the signal-only and signal+background
  fits call only `migrad()` (not `hesse()`), so their reported parameter errors
  come from MIGRAD's internal estimate rather than the Hessian-refined errors
  used in Step 1. Worth revisiting if the Step 2 uncertainties are used
  quantitatively elsewhere.
- `f_bkg` is used both as the list of background fractions and as a per-iteration
  loop variable in the Step 2 histogram cell, which overwrites it; the later
  pseudoexperiment loop happens to redefine the list before using it, but the
  two shouldn't share a name.
- Only three background fractions and one background shape are tested; a natural
  extension would be a finer scan over $f_{bkg}$ or an alternative background model.

## Acknowledgements

Project specification: *DAML Project 1 — Particle Asymmetry Decay*, William Barter,
University of Edinburgh, October 2025.