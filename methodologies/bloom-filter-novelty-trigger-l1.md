# A Novelty-First L1 Trigger: Bloom Filters and LSH on Modern Memory Architectures

**Author:** Patrick Duggan
**ORCID:** [0009-0001-0628-9963](https://orcid.org/0009-0001-0628-9963)
**Affiliation:** DugganUSA LLC, Minneapolis, Minnesota
**Date:** April 22, 2026
**Version:** 1.1
**License:** CC BY 4.0

*v1.1 (Apr 22, 2026): §7 expanded to explicitly recommend per-family Markov likelihood scoring as the preferred second stage, with cross-reference to the companion paper.*

---

## Abstract

The current Level-1 trigger at ATLAS and CMS rejects what does not match known-physics templates. By construction, this rejects novelty — the thing a BSM search should preserve. A bloom-filter / LSH novelty trigger inverts the logic: hash each event into a probabilistic sketch, check it against a bit-array of everything seen before, and keep the ones we have never seen. The algorithm has been textbook-obvious for a decade. The block was memory: sub-25 ns deterministic lookup at billion-bit-array scale. That block has dissolved. Versal ACAP on-chip SRAM (32 KB per AI Engine tile, ~400 tiles), UltraRAM at 1–3 cycle latency, HBM2e at 819 GB/s on integrated FPGA packages, Samsung HBM-PIM (Aquabolt-XL) delivering 4.92 TB/s of on-die compute bandwidth, and 3D-stacked SRAM at single-digit-ns additional latency now close the 25 ns budget with margin. This brief surveys the memory landscape, maps bloom/LSH onto it, proposes a concrete architecture for a novelty-first L1 trigger, and is honest about where the hard work actually lives: low-dimensional stable feature projection of ~1.5 MB raw detector events.

**Keywords:** L1 trigger, bloom filter, locality-sensitive hashing, FPGA, HBM-PIM, Versal ACAP, anomaly detection, BSM, hls4ml

---

## 1. Context and the Insight

This brief was prompted by a question from Daniel Whiteson (UC Irvine, ATLAS): whether the bloom-filter novelty approach used at DugganUSA for sub-100 ms threat-intelligence novelty checks — over a ~1M-indicator bit-array at commodity memory latencies — has anything useful to say to the CERN L1 problem. His informal response was that the algorithmic shape is obvious but the constraint has always been memory. That framing is the anchor of this paper.

The cross-domain leap — that the same probabilistic bit-array sketch used to decide "have we seen this indicator before?" at DugganUSA's search layer is the same primitive you need to decide "have we seen this event topology before?" at 40 MHz — is the contribution being offered here. The rest of this document is a feasibility survey of whether modern memory architectures actually close the 25 ns window well enough to justify a working prototype, and where honest engineers would push back.

The 95% epistemic cap (DugganUSA house rule) applies throughout. Nothing below is claimed to be certain; the goal is to move the conversation from "interesting idea, shelved for hardware reasons" to "here are the specific parts that changed, here is where the real work is."

---

## 2. The 25 ns Window and Why Current L1 Rejects Novelty

The LHC bunch crossing interval is 25 ns. Every 25 ns, the ATLAS and CMS detectors produce a candidate event; the L1 trigger decides — with strict fixed latency — whether to pass it upward. At Phase-II (HL-LHC), the single-level hardware trigger budget is set at ~10 μs total latency and 1 MHz output rate, but the per-bunch-crossing cycle that feeds L1 calorimeter and muon primitives is still pegged to the 25 ns clock. Phase-II Global Trigger prototypes are being built on AMD Versal Premium FPGAs, which fixes the hardware substrate this brief has to fit onto.

Current L1 selection is, operationally, a bank of known-physics templates: energy thresholds, topological cuts, muon-pT windows, jet multiplicities, dilepton invariant-mass regions. These templates are exquisitely tuned to Standard Model processes we want to keep (W, Z, top, Higgs) and to SM backgrounds we want to suppress. The logic is a whitelist against known good and a blacklist against known noise.

By construction, this rejects events whose signature does not look like something in the template library. Which is to say: a BSM signature — a new resonance, a long-lived particle, an exotic topology — is exactly the kind of event a template trigger is built to drop, unless someone anticipated the signal and wrote a template for it.

hls4ml [1][2] and the AXOL1TL deployment at CMS [3] represent the current frontier in addressing this. hls4ml compiles quantized neural networks to HLS targeting Xilinx FPGAs for sub-microsecond inference; AXOL1TL is a trained autoencoder running on Virtex-7 in the CMS L1 Global Trigger, and was commissioned in Run 3 with reported efficiency gains of ~46% relative to the rest of L1 at a 1 kHz rate. The FAD paper [4] extends this to normalizing-flow-based unsupervised anomaly detection at 40 MHz. This work is the state of the art and this brief is not a critique of it. Autoencoders learn a latent representation of "normal" and flag reconstruction-loss outliers. They work. They also have the costs a learned model always has: training-distribution drift, the need for periodic retraining, opacity about which events a given weight update just started or stopped flagging, and the fact that the model itself is a compressed statement of what was in the training set.

Bloom filters and LSH offer a complementary primitive with different tradeoffs. This brief proposes they deserve a second look now, not because the idea is new, but because the memory that would make them work at 25 ns just arrived.

---

## 3. What Bloom / LSH Bring to an L1 Trigger

A bloom filter is a bit-array of size *m* with *k* independent hash functions. To insert an item, hash it *k* ways and set those *k* bits. To test membership, hash and check: if any of the *k* bits is zero, the item is definitively novel. If all *k* are one, it is probably seen, with a tunable false-positive rate (~0.01 is achievable at ~10 bits/item). False negatives are impossible by construction.

Locality-sensitive hashing (LSH) is the complementary technique for approximate similarity: items that are close in some metric hash to the same bucket with high probability. SimHash (random hyperplane projection, cosine similarity), MinHash (Jaccard similarity), and E²LSH (Euclidean) are the standard families [5].

Why these primitives matter for a novelty trigger:

- **O(1) deterministic lookup.** *k* hash evaluations plus *k* memory reads. No iteration, no recurrence, no variable-depth network. Fits a fixed-latency pipeline.
- **No model drift.** The bit-array is a literal memo of what has been seen. It updates incrementally by setting bits, not by gradient descent. You can diff last week's filter against this week's and see exactly which event classes were added.
- **Binary output.** Pass/fail, nothing to interpret. The decision logic downstream is trivial.
- **Asymmetric error profile.** False positives (known event flagged as novel) waste bandwidth. False negatives (novel event dropped as known) cannot happen in a standard bloom filter. This is the right asymmetry for a BSM search.
- **Interpretability.** You can partition the bit-array by event class and report "this event collided with bits owned by the Z→ee reference sample." That is a statement an analyst can audit; a 32-dim autoencoder latent embedding is not.

The tradeoff is obvious: a bloom filter decides *novelty*, not *interestingness*. A beam-gas event is novel. A detector glitch is novel. Section 7 addresses the second-stage filter that has to sit after this.

---

## 4. The Memory Landscape (The Centerpiece)

The 25 ns window is the clock. Within it, an L1 novelty stage needs to: (a) project a ~1.5 MB raw event to a small feature vector, (b) evaluate *k* hash functions over that vector, (c) read *k* bits from a large bit-array, (d) AND them, (e) emit the flag. Hashing is computed logic, essentially free on a modern FPGA fabric. The dominant cost is the *k* bit-array reads. To fit, the bit-array has to live in memory that supports *k* parallel random-access reads in single-digit nanoseconds, at capacity large enough to give a useful false-positive rate.

The table below summarizes the candidate tiers. Figures are drawn from public datasheets and announcements; treat capacity and bandwidth as authoritative and latency figures as representative rather than guaranteed (exact latency depends on controller, clocking, and access pattern).

| Tier | Technology | Capacity per chip/package | Random-access latency | Bandwidth | Notes |
|------|-----------|---------------------------|-----------------------|-----------|-------|
| L0 | FPGA distributed LUT-RAM | ~10s of Mb per large device | 1 cycle (~1–2 ns at 500 MHz) | N×fabric parallel | Very small, very fast, perfect for *k* independent hash-indexed bit reads if the array fits. |
| L1 | FPGA BRAM (18/36 Kb blocks) | ~100 Mb on Versal Premium | 1–2 cycles (~2–4 ns) | Hundreds of parallel ports | Dual-port, plentiful, the canonical home for small bloom bit-arrays. |
| L1′ | FPGA UltraRAM (288 Kb blocks) | ~500 Mb on Versal Premium / HBM series | 1–3 cycles configurable (~2–6 ns) [6] | Wide, on-die | 24 URAM blocks per clock region; this is where a ~10⁹-bit array plausibly fits entirely on-chip. |
| L2 | Versal AI Engine local SRAM | 32 KB per tile × ~400 tiles ≈ 12.8 MB total; extensible to 128 KB per tile via neighbor access [7][8] | 1 cycle at 1+ GHz (~1 ns) | 256 bits/cycle shared-memory between adjacent tiles | Distributable bit-array across the AIE mesh; native to Versal ACAP, which is already the Phase-II substrate. |
| L3 | Integrated HBM2e on Versal HBM series | 32 GB, 819 GB/s [9] | ~10–15 ns typical random access; row-buffer hits faster | 819 GB/s aggregate | Large enough for multiple partitioned bloom filters plus LSH bucket tables. On-package, no PCB hop. |
| L3′ | HBM3 / HBM3e | 24–36 GB per stack, 819 GB/s – 1.2 TB/s per stack [10] | ~10 ns typical | Up to 1.2 TB/s per stack | Same latency class as HBM2e; the gain is bandwidth for parallel stream processing, not per-access speed. |
| L4 | Samsung HBM-PIM (Aquabolt-XL) | 4 PIM dies × 4 standard dies per stack; 128 PIM cores per stack [11][12] | JEDEC-compliant for standard accesses; PIM ops execute in-stack | 4.92 TB/s on-die compute bandwidth vs. 1.23 TB/s I/O | The hash-and-test step can execute inside the memory. Eliminates the round-trip that dominates DRAM access cost. |
| L5 | 3D-stacked SRAM (TSMC SoIC / AMD V-Cache style) [13][14] | 64 MB per stacked die, stackable | ~4-cycle / ~2 ns additional penalty over on-die L3 | 2.0–2.5 TB/s | Demonstrated in production silicon (AMD Ryzen X3D). Proves the packaging exists for large on-package SRAM at near-L-cache latencies. |
| L6 | TCAM (ternary CAM) | Mb-scale, vendor-specific | Sub-10 ns single-cycle pattern match [15] | One lookup per clock | Not a bloom substitute per se, but relevant: TCAM can serve as the second-stage "known-glitch" filter — exact-match in one memory access. |

Budget arithmetic for a concrete target — a 10⁹-bit (~125 MB) bloom array with *k* = 8 hash functions:

- **URAM-only path.** 10⁹ bits at 288 Kb/URAM ≈ 3,500 URAM blocks. A Versal Premium VP1802 exposes ~500 Mb of URAM (~1,800 blocks). Fits a 5×10⁸-bit array entirely on-chip. *k* = 8 independent reads at 2–3 cycles each, fully parallel across URAM ports, completes in ~3 cycles = ~6 ns at 500 MHz. Budget closes with ~19 ns margin inside the 25 ns window.
- **AIE-mesh path.** Partition the bit-array across ~400 AIE tiles. Each tile owns ~2.5 Mb of the array in 32 KB of local SRAM (roughly 260 Kb of the total, leaving room for state). *k* independent reads route via the AIE mesh in 1–2 ns each. Closes the budget with larger margin, at the cost of partitioning logic.
- **HBM-PIM path.** Entire array in HBM-PIM; hash-and-AND executes inside the stack and the FPGA receives only the 1-bit result per event. This is the lowest-I/O path and the most speculative — it assumes a PIM primitive set that supports the hash functions natively, which for anything beyond XOR/MUL/shift means a custom PIM microprogram. Worth exploring; not the first prototype.

The conclusion from the table is narrow and specific: **on a Versal Premium / Versal HBM class device, a 10⁸ – 10⁹ bit bloom array with *k* = 4–8 hash functions fits in on-chip memory with 1–3 cycle access, and *k* parallel reads complete well inside 25 ns.** This was not true on the FPGAs available in 2015. It is true now.

---

## 5. Feature Extraction — The Actual Hard Part

Everything in §3 and §4 presumes the event has already been projected into a fixed-length, stable feature vector suitable for hashing. This is the unsolved problem.

A raw ATLAS event is ~1.5 MB of heterogeneous detector data: calorimeter cell energies, tracker hits, muon chamber signals, timing. A bloom filter hashes a vector; it does not hash a detector. The feature extractor sits between the frontend and the hash stage, and its requirements are stringent:

- **Low-dimensional.** Call it *d* = 64 to 256 float or fixed-point values. Small enough that *k* hash evaluations over the vector fit in the pipeline.
- **Stable under trivial perturbation.** Two copies of the same physics should project to the same (or nearby in LSH sense) output. Bunch-crossing-level noise should not flip the hash.
- **Discriminative on topology, not just energy.** A two-jet event and a four-jet event at the same total scalar-sum HT should land in different buckets.
- **Deterministic and stateless per event.** No recurrence, no accumulated state beyond what fits in a single pipeline stage.
- **Fixed latency.** Must complete inside a budget that still leaves room for the hash and lookup stages (realistically ≤ 10 ns of the 25 ns window).

Candidate approaches:

- **Random linear projection (SimHash-style).** Multiply a coarse-binned detector representation by a fixed random matrix, sign-quantize. Cheap, deterministic, well-understood. The Johnson–Lindenstrauss guarantees give bounded distortion of Euclidean distances. Works best if the input representation is already physically meaningful (e.g., η-φ binned calorimeter towers plus counted physics objects).
- **MinHash over discrete detector "features."** Treat the event as a set of fired detector elements above threshold; MinHash the set. Well-suited to the hit-pattern level of the detector, less natural for continuous energy measurements.
- **Learned projector.** A small (one- or two-layer) quantized neural projector, compiled through hls4ml, produces the feature vector. Gives the system a learned front end while keeping the back end (bloom/LSH) model-free. This is the obvious hybrid with the existing hls4ml stack.
- **HEPT-style point-transformer projection.** The LSH-based Efficient Point Transformer [5] is explicitly designed for HEP point-cloud data using E²LSH with OR/AND constructions. Not currently at L1 latencies, but the representation-learning direction is well-posed and may yield pretrained weights that compile cleanly.

The honest statement is that the quality of any bloom-filter novelty trigger is bounded above by the quality of its feature projector. The memory table in §4 is the good news; §5 is where the work lives. This is the part where the hls4ml team's experience with detector-data quantization and compression is directly relevant.

---

## 6. Proposed Architecture

A text block diagram for a first-prototype novelty-first L1 trigger, sitting alongside (not replacing) the existing L1 pipeline:

> Detector frontend (calorimeter, muon, tracker primitives, 25 ns pipelined)
> → Feature Projector (fixed random projection + quantization, or small hls4ml-compiled NN): *d* ≈ 64–256 fixed-point values, latency ≤ 10 ns
> → *k*-Hash Generator (8 independent non-cryptographic hashes, e.g., xxHash or MurmurHash variants, fully unrolled in logic, latency ≤ 2 ns)
> → Bloom Bit-Array Lookup (URAM-resident ~5×10⁸-bit array on Versal Premium, *k* parallel reads, latency ≤ 6 ns)
> → Novelty Flag (8-bit AND, 1 cycle)
> → Second-Stage Known-Artifact Filter (see §7)
> → Routed to keep pipe in parallel with, not instead of, existing template triggers

Total per-event latency target inside the 25 ns window: ≤ 20 ns, leaving ≥ 5 ns margin for routing.

The insertion point is clean. ATLAS and CMS L1 are already FPGA-based. The Phase-II Global Trigger prototype is on AMD Versal Premium. Adding a parallel novelty branch does not require ripping out hls4ml or AXOL1TL; it complements them. An event flagged by *either* a known-physics template *or* the novelty branch proceeds. Events flagged by neither are dropped — the existing behavior.

Two operational parameters need calibration:

- **Training / warm-up period.** The bit-array starts empty; every event is "novel." During an initial run of length *T*, events are inserted into the filter without emission, building up the "known topology" baseline. *T* is a physics decision (how much of the expected background distribution do we want to memorize?) and a bandwidth decision.
- **Aging / rolling window.** A single-filter bloom grows monotonically saturated. Two rotating filters (active + shadow), periodic zeroing, or counting-bloom variants are standard solutions. The physics question is whether we want to forget events seen last month. The engineering question is whether we want the operating false-positive rate to drift.

---

## 7. Novel ≠ Interesting (The Second Stage)

A bloom filter flags what has not been seen. It does not flag what is physically interesting. In a hostile realistic environment:

- Beam-gas events are novel. So are cosmic muon overlaps.
- Noisy detector channels produce locally novel patterns.
- Pile-up fluctuations at HL-LHC routinely produce event topologies never observed before.
- A dead readout channel shifts the feature vector into a previously unvisited region.

This is real. Any "keep what we haven't seen" L1 without a downstream rejection stage would rapidly saturate its bandwidth budget with detector pathology.

Three plausible designs for the second stage:

1. **A second bloom filter of known artifacts.** Trained on tagged beam-gas, cosmic, noisy-channel, and pile-up-anomaly samples. Same primitive, opposite polarity: if the event hashes to the "known artifact" filter, drop. The novelty flag only fires when the event is *both* novel to the physics filter *and* not in the artifact filter. This has the virtue of remaining model-free and auditable.
2. **A small autoencoder — reusing AXOL1TL or an equivalent.** Novel-and-not-reconstructed-cleanly = interesting. This is an honest complement, not a competitor: AXOL1TL already does the reconstruction check well; bloom provides the cheap, deterministic pre-filter that decides which events even get that check.
3. **Per-family Markov likelihood scoring — the approach we recommend.** Train separate Markov chains over feature-sequence representations of (a) known-physics event classes, (b) known detector pathologies, (c) known beam-gas / pile-up / cosmic signatures. Score the flagged event under each family. The event most deserving of HLT attention is the one least likely under *all three* families. This inverts the usual "classify into the best-matching family" question into "flag events that fit none of them" — the same novelty-preservation spirit as the bloom layer, applied again at the classification step. Latency budget is generous: the L1→HLT transit gives 5–10 µs, and k-step Markov likelihood for k ~ 20–50 runs in hundreds of nanoseconds on the same FPGA fabric. DugganUSA runs this architecture in production today on the `markov_models` and `behavioral_sessions` indexes for web-traffic classification; the cross-domain correspondence is explicit in the companion paper.

A first prototype likely uses approach (1) because it keeps the entire pipeline model-free and interpretable. A production system benefits from (3) because comparative per-family likelihood gives a calibrated answer to "which kind of novel is this" without collapsing back into known-physics template rejection. Approaches (2) and (3) are complementary; the Markov stage can route the hardest cases to the autoencoder for final judgment.

**Companion paper** — the Markov second-stage architecture is written up in detail in [*Which Kind of Novel? Markov Second-Stage Classification for Novelty-First L1 Triggers*](./markov-novelty-classification-second-stage.md), published alongside this brief. §3–§6 there give the architecture, latency budget, and the DugganUSA production receipts.

What is emphatically *not* being proposed is that bloom replaces autoencoder-based anomaly detection. The argument is that a deterministic, interpretable, memory-efficient pre-filter belongs in front of the learned anomaly detector, and that the memory now exists to put it there.

---

## 8. The Physics Case

Known-physics templates are a compressed statement of the Standard Model. Running them as the primary L1 selector means the experiment preferentially keeps events that look like the model it was built to test. For BSM searches — the core justification for HL-LHC luminosity — this is epistemically backwards. We built a $ \sim 30$ billion accelerator to find physics that isn't in the template library, and then built a trigger that throws away events that aren't in the template library.

Autoencoder anomaly triggers (AXOL1TL, FAD) address this by learning the SM-like manifold and flagging departures. A bloom-filter novelty trigger addresses it from a different direction: instead of learning the manifold, enumerate its observed realizations and flag events outside the enumeration. The two approaches cross-check each other in the same way ATLAS and CMS cross-check each other on the Higgs. An event flagged by both is a stronger candidate for preservation than an event flagged by either alone.

A second physics benefit: the bit-array is a *searchable artifact* of the run. After the fact, an analyst can ask "was the event topology from this candidate ever seen in our training window?" by hashing and checking. This is not possible against an autoencoder weight snapshot without re-running inference. The novelty filter is not just a trigger; it is a queryable summary of everything the experiment has observed at L1 granularity.

The claim is modest. A novelty-first L1 branch does not discover new physics by itself. It preserves candidates that the existing L1 would discard, so that offline analysis has events to look at. The discovery still happens downstream. What changes is the prior: instead of "keep events consistent with what we know," it is "keep events inconsistent with what we have seen."

---

## 9. Who to Talk To

This work sits squarely in territory already being worked by the fast-ML-at-HEP community. Any serious prototype would start by contacting them:

- **Thea Aarrestad** (ETH Zürich / CERN) — autoencoders on FPGAs at 40 MHz [16]; anomaly-detection-at-L1 practitioner.
- **Maurizio Pierini** (CERN) — long-running program on ML-based anomaly triggers; co-author on hls4ml and the L1 anomaly papers.
- **Javier Duarte** (UC San Diego) — hls4ml lead author [1]; represents the framework side.
- **Nhan Tran, Jennifer Ngadiuba, Philip Harris** — original hls4ml core group; continuing to drive the L1 ML direction at CMS.
- **Elham E. Khoda** (UC San Diego / UW) — hls4ml contributor, anomaly-detection work, ATLAS GEP (Global Event Processor) evaluation [17].
- **AXOL1TL team at CMS** [3] — deployed the most directly comparable production system; the right people to ask "what are the realistic failure modes we haven't thought of?"

The hls4ml community is the state of the art and the framing of this brief is deliberately complementary, not competitive. A bloom novelty branch and an autoencoder anomaly branch are mutually reinforcing. If this proposal is worth pursuing, it is worth pursuing with them, not around them.

---

## 10. Open Questions

Honest enumeration of what we do not know:

1. **Feature-projector stability under pile-up.** HL-LHC runs at ⟨μ⟩ ≈ 200 pile-up interactions per bunch crossing. Does any fixed random projection produce stable hashes for the same hard-scatter topology across different pile-up realizations? This is probably the single biggest risk.
2. **Bit-array saturation timescale.** At 40 MHz input rate, how fast does a 10⁹-bit filter saturate at a given *k*? What is the operationally optimal rotation period?
3. **Second-stage artifact filter completeness.** Beam-gas and cosmics are tagged. Pile-up anomalies and transient detector pathology are harder to enumerate. How do we quantify the residual artifact rate leaking past stage 2?
4. **Interaction with the L1 tracking upgrade.** Phase-II introduces L1 track primitives. Does the novelty feature vector include track-level information from the outset, and at what projector cost?
5. **Quantitative efficiency gain.** AXOL1TL reports ~46% efficiency gain at 1 kHz vs. the rest of L1 [3]. What is the incremental gain from adding bloom on top? There is no prior, which is itself an argument for building it.
6. **Resource cost on Versal Premium.** URAM usage for a 5×10⁸-bit array is ~1,800 blocks. What does this displace in the existing Phase-II Global Trigger firmware budget?
7. **Does the information actually survive projection?** A bloom filter over a degenerate projection is a random number generator. The discriminating power of the filter is upper-bounded by the discriminating power of the feature extractor; measuring that bound is the first experiment.
8. **PIM feasibility.** HBM-PIM supports a restricted instruction set (adds, multiplies, simple logic). Do the required non-cryptographic hash functions fit, or is the HBM-PIM path permanently speculative?

None of these are showstoppers in the sense that they would prevent a first prototype. All of them are real work.

---

## 11. Conclusion

The bloom-filter / LSH novelty-trigger idea has been algorithmically obvious for years. The reason it has not shipped is the memory: sub-25 ns deterministic lookup at billion-bit-array scale was not available at a cost and power budget that made sense inside an LHC trigger rack. That condition has changed. UltraRAM on Versal Premium, 32 KB SRAM per AI Engine tile across a 400-tile mesh, integrated HBM2e at 819 GB/s on Versal HBM series packages, HBM-PIM at nearly 5 TB/s of on-die bandwidth, and 3D-stacked SRAM at single-digit-ns penalty all exist now. A 10⁸ – 10⁹ bit bloom filter with *k* = 8 hashes fits on-chip on a Versal Premium device with room inside the 25 ns budget.

The remaining hard problem is not memory. It is the feature projector — turning a 1.5 MB raw event into a low-dimensional stable vector that a hash function can act on without destroying the physics. That problem has a natural home in the hls4ml ecosystem and a natural set of collaborators already working on it.

This brief does not claim bloom triggers will find new physics. It claims that a deterministic, model-free, memory-efficient novelty pre-filter now fits in the L1 latency budget, and that a BSM experiment is epistemically better served by a trigger that preserves the unseen than by one that rejects it. The memory has moved. The question worth revisiting is whether the algorithm should move with it.

95% epistemic cap applies. 5% of the above is wrong. The useful next step is to identify which 5%.

---

## References

[1] Duarte, J., Han, S., Harris, P., Jindariani, S., Kreinar, E., Kreis, B., Ngadiuba, J., Pierini, M., Rivera, R., Tran, N., Wu, Z. (2018). "Fast inference of deep neural networks in FPGAs for particle physics." *JINST* 13 P07027. [arXiv:1804.06913](https://arxiv.org/abs/1804.06913).

[2] FastML Team. (2021). "hls4ml: An Open-Source Codesign Workflow to Empower Scientific Low-Power Machine Learning Devices." [arXiv:2103.05579](https://arxiv.org/abs/2103.05579). See also the 2025 synthesis: [arXiv:2512.01463](https://arxiv.org/abs/2512.01463).

[3] CMS Collaboration. (2024). "Real-time Anomaly Detection at the L1 Trigger of CMS Experiment." [arXiv:2411.19506](https://arxiv.org/abs/2411.19506). AXOL1TL deployment on Xilinx Virtex-7 in the CMS Global Trigger, Run 3.

[4] (2025). "It's not a FAD: first results in using Flows for unsupervised Anomaly Detection at 40 MHz at the Large Hadron Collider." [arXiv:2508.11594](https://arxiv.org/abs/2508.11594).

[5] Miao, S., et al. (2024). "Locality-Sensitive Hashing-Based Efficient Point Transformer with Applications in High-Energy Physics." [arXiv:2402.12535](https://arxiv.org/abs/2402.12535). E²LSH with OR/AND constructions for HEP point clouds (TrackML dataset).

[6] AMD. "Versal ACAP Memory Resources Architecture Manual (AM007)." UltraRAM configurable for 1–3 cycle latency. [docs.amd.com](https://docs.amd.com/).

[7] AMD. "Versal ACAP AI Engine Architecture Manual (AM009)." 32 KB local memory per AI Engine tile, extensible to 128 KB via neighbor-tile access; 256 bits/cycle per direction shared-memory bandwidth.

[8] AMD. "VCK5000 Versal Development Card Product Brief." ~400 AI Engine tiles (8×50 mesh) per VCK5000 device.

[9] AMD. "Versal HBM Series Product Brief." Integrated HBM2e, up to 32 GB and 819 GB/s on-package. [amd.com/versal/hbm-series](https://www.amd.com/en/products/adaptive-socs-and-fpgas/versal/hbm-series.html).

[10] JEDEC. HBM3 and HBM3e specifications; Micron and SK hynix HBM3e announcements. Per-stack bandwidth 819 GB/s (HBM3) to 1.2 TB/s (HBM3e, Micron).

[11] Kim, J., et al. (2021). "Aquabolt-XL: Samsung HBM2-PIM with in-memory processing for ML accelerators and beyond." *Hot Chips 33*. 128 PIM processors per stack; 4.92 TB/s on-chip PIM bandwidth vs. 1.23 TB/s I/O.

[12] Samsung Semiconductor. "HBM2-PIM and HBM-PIM (Aquabolt-XL)." Product documentation and Hot Chips presentation.

[13] AMD / TSMC. "3D V-Cache using TSMC SoIC 3D fabric." 64 MB stacked SRAM die, 2.0–2.5 TB/s bandwidth between stack and CCD, ~4-cycle / ~2 ns additional L3 latency penalty (measured on Ryzen 7 5800X3D).

[14] TSMC. "SoIC (System on Integrated Chips) packaging." Sub-10 μm bonding pitch; >200× interconnect density vs. 2D packaging.

[15] Various. Ternary CAM characteristics for network-class lookup: single-cycle pattern match. See e.g. Pagh & Pagh on associative-memory Bloom filter variants, ACM SIGMETRICS 2010.

[16] Govorkova, E., Puljak, E., Aarrestad, T., et al. (2021). "Autoencoders on FPGAs for real-time, unsupervised new physics detection at 40 MHz at the Large Hadron Collider." [arXiv:2108.03986](https://arxiv.org/abs/2108.03986). 80 ns inference on Xilinx Virtex VU9P.

[17] ATLAS Global Event Processor evaluation of hls4ml-deployed ML, including contributions by E. E. Khoda (UW / UCSD).

[18] ATLAS Collaboration. (2017). "Technical Design Report for the Phase-II Upgrade of the ATLAS TDAQ System." CERN-LHCC-2017-020.

[19] Duggan, P. (2026). "The CERN Method: Signal Detection at Scale — From Higgs Bosons to Hidden Networks." DugganUSA Research.

---

## Acknowledgments

This brief was prompted by correspondence with Daniel Whiteson (UC Irvine, ATLAS). His framing that "the gap is basically more RAM" is the anchor of the paper and made writing a focused feasibility survey possible instead of a broader speculative piece. The cross-domain mapping of bloom-filter novelty detection from DugganUSA's threat-intelligence work to the L1 problem is the author's; the technical feasibility survey on modern memory architectures was conducted with the assistance of Claude (Anthropic) under author direction. Any errors of physics, engineering, or citation are the author's.

The hls4ml community — named in §9 — is the state of the art in fast ML for HEP triggers, and this proposal is intended as a complement to their work, not a competitor.

---

## Citation

```bibtex
@article{duggan2026bloom,
  author = {Duggan, Patrick},
  title = {A Novelty-First L1 Trigger: Bloom Filters and LSH on Modern Memory Architectures},
  year = {2026},
  month = {April},
  day = {22},
  publisher = {DugganUSA Research},
  url = {https://github.com/pduggusa/dugganusa-research},
  orcid = {0009-0001-0628-9963}
}
```

---

**Document Hash:** [Computed on commit]
**Last Updated:** April 22, 2026
**Status:** Pre-print / Working Paper
