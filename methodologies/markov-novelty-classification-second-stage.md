# Which Kind of Novel? Markov Second-Stage Classification for Novelty-First L1 Triggers

**Author:** Patrick Duggan
**ORCID:** [0009-0001-0628-9963](https://orcid.org/0009-0001-0628-9963)
**Affiliation:** DugganUSA LLC, Minneapolis, Minnesota
**Date:** April 22, 2026
**Version:** 1.0
**License:** CC BY 4.0

---

## Abstract

A companion paper to the bloom-filter novelty-first L1 trigger [1]. The bloom layer answers "have we seen this event topology before?" in O(1) at 25 ns. It cannot, by construction, answer "is this physics, pathology, or beam-gas?" — all three produce unseen hashes. This brief proposes that the natural second stage, sitting in the L1→HLT transit (a few microseconds of available headroom), is a Markov likelihood comparator: score the event's feature sequence under three family-level generative models — known physics, known detector pathology, known beam-gas/pile-up/cosmic — and route by the *least likely* family rather than the most likely. The logic preserves the novelty-first spirit of the bloom layer. DugganUSA runs discrete Markov chains in production today on a `markov_models` index for behavioral-session classification (benign crawler, reconnaissance, APT); the transition-matrix training, steady-state analysis, and regime-change detection machinery ports to detector-event sequences with minor modification. 95% epistemic cap applies.

**Keywords:** L1 trigger, HLT, Hidden Markov Models, Markov likelihood, novelty classification, anomaly detection, beam-gas, detector pathology, bloom filter, FPGA

---

## 1. The Gap the Bloom Paper Leaves Open

The bloom-filter novelty-first L1 trigger [1] decides a single binary question at 25 ns: has this event's feature projection been hashed into the bit-array before? Unseen events pass. Seen events drop. The logic is deterministic, model-free, and interpretable, and §7 of that paper acknowledges explicitly that *novel is not the same as interesting*. Beam-gas events are novel. Readout glitches are novel. Pile-up fluctuations at HL-LHC ⟨μ⟩≈200 routinely produce topologies that have never been seen. Without a downstream classifier, a novelty-first L1 would saturate its bandwidth budget on detector pathology within minutes of warm-up.

§7 of [1] gestures at two options for the second stage: (a) another bloom filter trained on known artifacts, same polarity-flipped logic, or (b) reuse of an autoencoder such as AXOL1TL [2] or CICADA [3] as a semantic "is this a glitch?" check. This companion paper argues for a third, complementary option that neither [1] nor the current L1 ML literature has explicitly named: **a Hidden Markov Model (or simpler discrete-Markov) likelihood comparator that scores the event's feature sequence against three family-level generative models**. The novelty flag from the bloom layer triggers Markov scoring; the output of the Markov stage is a routing decision, not a keep/drop decision.

The distinguishing move is this: Markov classification in ML is normally used to answer "which known class does this sequence most resemble?" We invert it. We ask "which known class is this sequence *least likely* to belong to?" The event whose feature sequence has low likelihood under all three families — physics, pathology, and beam-gas — is precisely the event a BSM search should preserve. This inverts the same template-matching bias that [1] identified in the current L1.

---

## 2. Why Markov — and the Honest Comparison to Alternatives

A discrete-state Markov chain over a sequence of *k* feature-symbols has two operational properties that fit the L1→HLT slot:

- **O(k) scoring.** Log-likelihood under a trained transition matrix is a sum of *k* table lookups and *k* adds. At *k* ≈ 20–50 features per event and nanosecond-scale lookup in URAM/HBM, scoring completes in well under a microsecond per family per event. Three families in parallel multiplies resources, not latency.
- **Bounded memory.** A transition matrix over a symbol alphabet of size *S* is *S²* entries per family. For *S* = 64–256 (a coarse detector-feature quantization), three families fit in a few MB of on-chip memory. Steady-state vectors and emission matrices (for HMM) add linearly.
- **Incremental retraining.** Transition counts are additive. A new week of tagged data updates a family by incrementing cells in the counting matrix and renormalizing. No gradient descent, no convergence drama, no training-distribution-drift surprise of the kind that haunts autoencoder operators on detectors whose response evolves with luminosity and radiation damage.
- **Likelihood, not hard classification.** The Markov score is a continuous value. The routing logic downstream can apply thresholds, confidence bands, or abstention rules. This matters for a science instrument: the right behavior when all three families give borderline scores is "send it to the physics HLT and let a human look," not "force a majority vote."

These are the Markov tradeoffs honestly. Where Markov is weaker:

- **State sufficiency assumption.** First-order Markov assumes the next feature depends only on the current one. Detector events have longer-range correlations — a jet in one calorimeter region influences the feature-sequence order far away. This is the natural argument for **Hidden Markov Models** (HMM) with latent state, or higher-order Markov chains, both of which add cost but remain comfortably inside a microsecond budget.
- **Alphabet quantization.** Continuous features must be binned. Binning is a modeling choice that can be audited, but it can also erase discriminative information. This is analogous to — and should be calibrated against — the feature-projection problem of [1] §5.
- **Not a substitute for deep learned representations.** The L1 ML literature has converged on autoencoders [2][3] and normalizing flows [4] for good reasons: they learn the data manifold directly. A Markov comparator is a different animal and should be treated as a complement, not a replacement. CICADA distilled from a CNN autoencoder achieves sub-200 ns inference on FPGA [3]; Markov scoring runs at similar latencies but carries different failure modes.

The honest claim: Markov gives us an interpretable, incrementally-trainable, likelihood-valued classifier with known failure modes. It is the right second stage for a novelty-first architecture *because* its failure modes are orthogonal to the failure modes of the bloom layer and of the autoencoder anomaly detectors — it will be wrong about different events than they are, which is the whole point of building it in parallel.

Existing work supports the approach. HMM gauge likelihood analysis [5] applies clustering over HMM subsequence likelihoods for unsupervised anomaly detection. Ensemble HMM methods [6] train per-class HMMs and score by comparative likelihood — exactly the pattern proposed here, imported from finance/biology and applied at L1→HLT for the first time (to our knowledge). Anomaly detection over Markov sample paths via statistical depth [7] gives a theoretical framework for sample-level outlier scoring. None of this is novel as sequence-modeling machinery. The novel move is pairing it with the bloom novelty trigger as a coherent two-stage architecture and inverting the "most likely class" question to "least likely family."

---

## 3. The Three Family Models

The Markov second stage trains three generative models in parallel. Each model is a family — a mixture of sub-models covering the within-family diversity:

### 3.1 Known-Physics Family (P_phys)

Sub-models for the Standard Model event classes we understand: SM diboson (WW, WZ, ZZ), ttbar, single-top, QCD multi-jet, Drell-Yan, Higgs channels. Training data is simulated events passed through the same feature projector used by the bloom layer, with explicit known-physics labels from MC truth. The family likelihood is a mixture weighted by expected-rate priors:

P_phys(seq) = Σ w_i · P_i(seq)

Where *w_i* is the cross-section-weighted prior for sub-family *i* and *P_i* is the per-sub-family Markov (or HMM) likelihood.

### 3.2 Detector-Pathology Family (P_path)

Sub-models for tagged detector failure modes: readout glitches (CRC errors, BCID slips, desynced links), hot or dead cells, calorimeter spikes, muon-chamber afterpulses, noise bursts correlated with beam abort gaps. Training data comes from the detector groups who already catalog these — the quality-monitoring and data-certification teams at ATLAS and CMS have labeled pathology samples as a matter of course. The key operational point: these samples exist; the HEP ML literature has not assembled them into a Markov family model.

### 3.3 Beam-Gas / Pile-Up / Cosmic Family (P_bg)

Sub-models for beam-induced backgrounds, cosmic muon overlaps, and high-pile-up fluctuation topologies. ATLAS has published detailed characterizations of beam-gas background during dedicated local gas injection studies [8], and the beam conditions monitors plus fake-jet taggers give clean labeled samples [9]. HL-LHC pile-up topology has its own simulated catalog. The family aggregates these.

### 3.4 The Scoring Rule

For a novel event with feature sequence *s* of length *k*, the second stage computes:

L_phys = log P_phys(s)
L_path = log P_path(s)
L_bg = log P_bg(s)

Routing rule:

- If L_phys is the *highest* of the three and exceeds a calibrated threshold → **route to physics HLT** (it looks like known physics, preserve it anyway because the bloom layer flagged it novel; this is likely SM-tail or close-to-known new physics).
- If L_path is the highest → **route to detector operations**, drop from physics stream but log for shift crew.
- If L_bg is the highest → **discard as noise**, increment rate counter.
- If all three are below an "unknown territory" threshold → **route to physics HLT with high-priority flag**. This is the case the novelty-first architecture exists to protect. An event that is unseen by bloom *and* improbable under all three known families is the strongest candidate for BSM preservation.

The last clause is the whole point. Standard ML classification would pick the argmax of the three log-likelihoods. We override: when the maximum itself is low in absolute terms, the event is too weird for all three families and goes to the physics HLT with priority. This is the *least-likely-family* inversion.

---

## 4. Architecture Block Diagram

Text block diagram for the combined two-stage novelty pipeline:

> Detector frontend (calorimeter, muon, tracker L1 primitives, 25 ns pipelined)
> → Feature Projector (from [1] §5; fixed projection or hls4ml-compiled NN, ≤ 10 ns)
> → **STAGE 1: Bloom Novelty Lookup** (URAM-resident ~5×10⁸-bit array, *k* parallel hashes, ≤ 6 ns) → NovelFlag
> → If NovelFlag = 0 (seen before): drop to existing L1 template logic.
> → If NovelFlag = 1 (novel): emit feature sequence to Stage 2.
> → **STAGE 2: Markov Likelihood Comparator** (3 family models in parallel, transition tables in URAM/HBM, O(k) scoring at ~50 ns per family on a dedicated AIE tile cluster, fully pipelined)
>   - Compute L_phys, L_path, L_bg
>   - Apply routing rule (§3.4)
> → Three output streams:
>   - `route_to_physics_HLT` (likely BSM-interesting or SM-tail)
>   - `route_to_detector_ops` (pathology log)
>   - `discard_as_noise` (rate counter only)

Stage 1 lives inside the 25 ns bunch-crossing cycle. Stage 2 lives in the L1→HLT transit, where the Phase-II latency budget is 12.5 μs total [10] and L1 output rate target is 750 kHz. At 750 kHz incoming, with novelty flag firing on (conservatively) 10% of events, Stage 2 processes 75 kHz — a trivial load for parallel FPGA Markov scoring. The per-event Markov scoring budget is effectively the entire L1→HLT transit minus Stage 1 overhead, i.e. ~10 μs. For *k* = 50, three families, bit-shift-and-add log-likelihood over URAM-resident transition tables, the arithmetic says Stage 2 completes in <500 ns with room to spare.

This is the architectural payoff: **Stage 1 closes a 25 ns budget, Stage 2 lives in a 10 μs budget — three-to-four orders of magnitude more headroom**. We can afford HMM with latent-state inference, higher-order chains, or small ensembles of Markov models per family, because we have the time budget that Stage 1 did not.

---

## 5. DugganUSA Production Receipts

The architecture above is not speculative. DugganUSA runs discrete Markov chains in production on its `markov_models` index, trained on the `behavioral_sessions` index (7,000+ session traces as of April 2026). The domain is adversary-behavior classification over web-request sequences: given a user-agent's URL path sequence, is it a benign crawler, a reconnaissance scanner, or an APT-grade actor?

The production pipeline:

- **Session sequence extraction.** Each session's URL path is tokenized to a symbol sequence of length 10–200.
- **Per-family transition matrix training.** Separate Markov chains trained on tagged benign, recon, and APT sessions. Counting-then-renormalizing, no gradient descent.
- **Steady-state vector analysis.** Eigenvector of each transition matrix identifies the "resting distribution" over URL states per family. Used for regime-change detection when a session drifts toward a new family.
- **Comparative likelihood scoring.** An incoming session is scored under all three family models; the argmax is the classification, and — this is the analogue of §3.4 — the *minimum* score is the anomaly flag. Sessions whose minimum family likelihood is below a threshold under all families are the most suspicious, because they don't fit any known behavioral family.

The mapping to the L1 trigger problem is direct:

| DugganUSA production | LHC L1→HLT analog |
|---|---|
| URL path sequence | Detector-feature symbol sequence |
| Session length 10–200 | Event feature sequence length 20–50 |
| Benign / recon / APT families | Physics / pathology / beam-gas families |
| "Argmin likelihood under all families" = novel adversary | "Argmin likelihood" = preserve for BSM analysis |
| Regime-change detection | Detector-response drift monitoring |
| Incremental retraining by count addition | Weekly pathology-family updates from DQM |

The DugganUSA index runs against 1.1 M page_views and 7K behavioral_sessions at sub-100 ms query latency. The scale is smaller than the LHC rate by factors, but the algorithmic primitives — transition-matrix training, steady-state analysis, per-family comparative likelihood — are exactly the ones that would run on the L1→HLT Markov stage. The pattern works. We have receipts.

This is also a cross-domain signature of the DugganUSA-Whiteson correspondence that started with [1]: the same probabilistic bit-array that sketches "have I seen this indicator?" in our threat-intel layer is the L1 bloom primitive, and the same Markov comparator that classifies our session streams is the L1→HLT Stage 2 primitive. Two different institutions, two different problem domains, the same two algorithmic primitives composed in the same two-stage architecture.

---

## 6. Connection Back to the Bloom Paper

[1] argues for a novelty-first L1 that rejects the template-matching bias of current trigger systems. Stage 1 of that proposal inverts the question from "does this look like known physics?" to "have we seen this before?" The same inversion happens twice in the combined architecture:

- **Stage 1 (bloom):** the question is *not* "is this known physics?" It is "is this a hash we have seen?" Novelty-preserving by construction.
- **Stage 2 (Markov):** the question is *not* "which family does this sequence resemble most?" It is "which family does it fit *least*, and if it fits all three poorly, route it upward." Novelty-preserving again, now over the continuous-likelihood domain.

Both stages refuse the collapse to a known-template answer. Both stages use the "known" models (the bit-array of seen hashes, the family transition matrices) as filters to *remove* the familiar, not as templates to *select* the familiar. The architectural principle is unified: **enumerate the known, and keep what is missing from the enumeration**. Stage 1 does this in O(1) at 25 ns. Stage 2 does it in O(k) at ~500 ns. Together they give a complete novelty-first trigger that can still distinguish physics from pathology from beam-gas when novelty fires.

This is the thing the bloom paper's §7 was reaching for but did not name. Autoencoders and second-bloom filters both work; Markov likelihood is the third option, and the one whose operational properties (incremental retraining, family-level interpretability, continuous likelihood output, sub-microsecond inference) fit the L1→HLT slot the most naturally.

---

## 7. Honest Limitations

Seven places this can be wrong.

1. **First-order Markov may be too weak.** Detector events have correlations that span feature positions. If a coarse HMM captures this with modest latent-state count (say 8–16 hidden states), the budget still closes. If it requires deep temporal models, Markov stops being the right tool and autoencoder-based options from [2][3][4] become more attractive. Empirical test on simulated samples is the first experiment.
2. **Alphabet-binning sensitivity.** Discretizing continuous detector features destroys information. The binning schema must be validated against the same discrimination metrics used to validate the feature projector in [1] §5. This is work, not a showstopper.
3. **Pathology-family training data contamination.** If tagged pathology samples contain rare physics (which happens — cosmic-muon-tagged events occasionally contain real cosmic-ray physics of interest), the P_path family will memorize those events and deprioritize them. Requires careful curation by DQM and physics liaison teams.
4. **Pile-up drift.** HL-LHC pile-up distributions shift run-to-run. The P_bg family must retrain weekly at minimum. Incremental retraining (transition-count accumulation) handles this, but only if the retraining pipeline is operational — an engineering dependency, not a methodology problem.
5. **The "all three low" routing branch is the riskiest one.** Events where all family likelihoods are low go to physics HLT with priority. This is also the exact branch an adversarial detector glitch (one nobody has cataloged yet) would exploit, sending spurious events to expensive HLT reconstruction. Mitigation: rate-limit the high-priority branch, audit it weekly, and treat sustained spikes as a pathology-family blind spot to be addressed with training data.
6. **Latent-state inference latency for HMM.** Forward-algorithm scoring of HMM is O(kS²) per family for *S* latent states. At *S* = 16, *k* = 50, three families, this is 38,400 multiply-accumulates per event. Still inside a microsecond on a moderate FPGA, but it is worth noting that HMM is not free relative to simple Markov chains.
7. **Nobody has done this at L1→HLT before.** There is HMM work in HEP [11] and HMM-based anomaly detection in industrial and behavioral domains [5][6][7], but we are not aware of a deployed HMM-based second-stage trigger classifier in ATLAS or CMS. This could mean the idea is wrong. It could also mean it is a well-known gap. The only way to find out is to build a test-bench.

None of these are fatal. All of them are real.

---

## 8. Open Questions

1. What is the optimal *k* for the feature sequence? Tradeoff between discriminative power (higher *k*) and training-data density (lower *k* per cell).
2. Does the pathology family need one Markov model per detector subsystem, or is a single family aggregate sufficient? The former is more interpretable; the latter is cheaper.
3. How does the "all three low" high-priority branch rate scale with luminosity? If it saturates, the threshold calibration has to become adaptive.
4. Can the steady-state vector of each family model be used as an online drift detector — flagging when P_bg's eigenvector shifts during a run as a signal of new beam-gas conditions?
5. Is there a version of counting-bloom + Markov that shares a single bit-array representation, reducing memory footprint?
6. For Phase-II Versal Premium, what fraction of the resource budget does the full Stage 1 + Stage 2 pipeline consume? This is ultimately a firmware accounting exercise, but it governs deployment feasibility.
7. Does ATLAS's Global Event Processor [12] have the right shape to host Stage 2, or does it want a dedicated AIE tile cluster?
8. Can we reuse the AXOL1TL or CICADA-trained feature representation as the Markov symbol alphabet, eliminating the need to design a binning scheme from scratch? This would tightly couple the two L1 ML approaches in a productive way.

---

## 9. Conclusion

The bloom novelty trigger of [1] flags unseen events in O(1) at 25 ns. By itself, it cannot distinguish new physics from detector pathology from beam-gas — all three produce unseen hashes. The natural second stage, sitting in the 10 μs L1→HLT transit, is a Markov likelihood comparator over three family models: known physics, known pathology, known beam-gas. The routing decision uses the *least-likely-family* rule rather than the most-likely — events whose feature sequence is improbable under all three known families are exactly the events a BSM search should preserve. This inverts template-matching a second time and preserves the novelty-first logic of the bloom layer.

The machinery is not exotic. Markov likelihood comparators are standard in finance, biology, and behavioral security. DugganUSA runs this architecture in production on the `markov_models` index for adversary-behavior classification, at sub-100 ms query latency over 1.1 M page_views and 7K behavioral_sessions. The cross-domain transfer to L1→HLT detector-event sequences is not free — alphabet binning, family training data, pile-up drift handling all require work — but the algorithm and its operational properties are not speculative.

The combined [1]+[this] architecture is a two-stage novelty-first trigger with interpretable, incrementally-trainable, likelihood-valued outputs at every layer. It complements, not competes with, AXOL1TL [2], CICADA [3], and FAD [4]. The hls4ml community is the right home for the feature-projector collaboration [1] identifies, and the same community is the right home for validating the Markov alphabet design.

95% epistemic cap. 5% of the above is wrong. We would like help finding which 5%.

---

## References

[1] Duggan, P. (2026). "A Novelty-First L1 Trigger: Bloom Filters and LSH on Modern Memory Architectures." DugganUSA Research, April 22, 2026.

[2] Govorkova, E., Puljak, E., Aarrestad, T., et al. (2021). "Autoencoders on FPGAs for real-time, unsupervised new physics detection at 40 MHz at the Large Hadron Collider." [arXiv:2108.03986](https://arxiv.org/abs/2108.03986).

[3] CMS Collaboration. (2024). "Real-time Anomaly Detection at the L1 Trigger of CMS Experiment." [arXiv:2411.19506](https://arxiv.org/abs/2411.19506). Covers both AXOL1TL (variational autoencoder on L1 trigger objects) and CICADA (convolutional autoencoder on 18×14 calorimeter-region image, distilled to a ~10k-parameter student network with sub-200 ns FPGA inference).

[4] (2025). "It's not a FAD: first results in using Flows for unsupervised Anomaly Detection at 40 MHz at the Large Hadron Collider." [arXiv:2508.11594](https://arxiv.org/abs/2508.11594).

[5] Lorbeer, B., Deutsch, T., Ruppel, P., Küpper, A. (2019, updated 2020). "Anomaly Detection with HMM Gauge Likelihood Analysis." [arXiv:1906.06134](https://arxiv.org/abs/1906.06134). HMM subsequence likelihood clustering for unsupervised anomaly detection; the closest prior art to the Stage 2 scoring approach.

[6] Kawawa-Beaudan, M., Sood, S., Palande, S., Mani, G., Balch, T., Veloso, M. (2024). "Ensemble Methods for Sequence Classification with Hidden Markov Models." [arXiv:2409.07619](https://arxiv.org/abs/2409.07619). HMM ensembles (HMM-e) for sequence classification under imbalanced data — the pattern of per-family HMMs with comparative likelihood scoring that Stage 2 imports.

[7] Staerman, G., Laforgue, P., Clémençon, S. (2024). "Anomaly Detection based on Markov Data: A Statistical Depth Approach." [arXiv:2406.16759](https://arxiv.org/abs/2406.16759). Sample-level outlier scoring over Markov chain sample paths; relevant to the "all three low" branch calibration.

[8] ATLAS Collaboration. (2024). "Beam-induced backgrounds measured in the ATLAS detector during local gas injection into the LHC beam vacuum." [arXiv:2405.05054](https://arxiv.org/abs/2405.05054).

[9] ATLAS Collaboration. (2017). "Beam-Gas Background Observations at LHC." ATL-DAPR-PROC-2017-002. Beam Conditions Monitor plus fake-jet tagger characterization; source of labeled beam-gas samples for the P_bg family.

[10] CMS Collaboration. (2022). "The High-Level Trigger for the CMS Phase-2 Upgrade." [arXiv:2211.03684](https://arxiv.org/abs/2211.03684). Phase-II L1 output rate target of 750 kHz; L1 latency budget 12.5 μs; CPU-farm HLT downstream.

[11] Not a heavily applied regime. HMM work in physics exists — e.g., Bayesian data fusion of multivariate signals [physics/0403149] — but the authors are not aware of deployed HMM-based trigger classifiers at ATLAS or CMS. If this citation gap closes after publication, it closes because someone did the work. We welcome correction.

[12] ATLAS Global Event Processor evaluation of hls4ml-deployed ML at L1; see [1] reference [17] and Khoda et al.

[13] Kim, J., et al. (2021). "Aquabolt-XL: Samsung HBM2-PIM." Hot Chips 33. In-memory-processing bandwidth relevant to multi-family Markov transition-table lookups.

---

## Acknowledgments

This brief is a companion to [1], which was itself prompted by correspondence with Daniel Whiteson (UC Irvine, ATLAS). The observation that the bloom layer's novelty output naturally feeds a Markov-likelihood classifier is the author's; the feasibility survey and citation work was conducted with Claude (Anthropic) under author direction. The DugganUSA `markov_models` production receipts come from our threat-intelligence layer, not from any HEP collaboration, and the cross-domain mapping should be treated as an existence proof of the algorithmic machinery rather than as a validated HEP result.

The hls4ml community and the AXOL1TL/CICADA/FAD teams are the state of the art in L1 ML anomaly detection [2][3][4]. This proposal complements their work. Stage 1 (bloom) and Stage 2 (Markov) are designed to sit alongside autoencoder and normalizing-flow approaches, not to replace them, and the combined architecture benefits from all of them running in parallel on the Phase-II Versal Premium substrate.

Any errors of physics, engineering, or citation are the author's.

---

## Citation

```bibtex
@article{duggan2026markov,
  author = {Duggan, Patrick},
  title = {Which Kind of Novel? Markov Second-Stage Classification for Novelty-First L1 Triggers},
  year = {2026},
  month = {April},
  day = {22},
  publisher = {DugganUSA Research},
  url = {https://github.com/pduggusa/dugganusa-research},
  orcid = {0009-0001-0628-9963},
  note = {Companion to arXiv preprint on bloom-filter novelty-first L1 trigger}
}
```

---

**Document Hash:** [Computed on commit]
**Last Updated:** April 22, 2026
**Status:** Pre-print / Working Paper
**Pairs with:** `bloom-filter-novelty-trigger-l1.md` v1.0 (same repo, same date)
