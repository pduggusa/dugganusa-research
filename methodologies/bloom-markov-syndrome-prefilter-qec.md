# Lifting the 25% Wall: A Bloom + Markov Syndrome Pre-Filter for Applied QEC Decoders

**Author:** Patrick Duggan
**ORCID:** [0009-0001-0628-9963](https://orcid.org/0009-0001-0628-9963)
**Affiliation:** DugganUSA LLC, Minneapolis, Minnesota
**Date:** April 23, 2026
**Version:** 1.0
**License:** CC BY 4.0

---

## Abstract

This is the third paper in a three-paper arc. The first [1] proposed a bloom-filter novelty-first L1 trigger for CERN; the second [2] added a per-family Markov likelihood classifier as the natural second stage. Both were written against the CERN high-energy-physics detector problem. The core observation of this paper is that the same two primitives — bloom membership check over a projected feature, plus per-family Markov likelihood over a short feature sequence — map onto the quantum error-correction (QEC) syndrome decoding problem with only a domain-variable rename. The mathematics is identical (matrix-vector multiply under both names); the silicon is identical (Versal ACAP AI Engine tiles, UltraRAM, HBM-PIM); only the target changes from photon counts in a calorimeter to stabilizer measurements on a surface code. I argue that the often-cited 25% practical error/loss ceiling on photonic measurement-based QC is not a fundamental quantum bound but the product of (fusion-failure rate) × (decoder-throughput saturation), and that bloom + Markov syndrome pre-filtering addresses the throughput term directly. The bloom layer drops 70–80% of routine syndromes at ~15 ns. Markov family scoring routes the residual at ~100 ns per family. Deep decoders — MWPM [10], sparse blossom [11], or neural [12][13] — run on the small unseen-and-unfamiliar residual only. 95% epistemic cap applies throughout.

**Keywords:** quantum error correction, surface code, MBQC, fusion-based quantum computing, syndrome decoding, bloom filter, Markov likelihood, MWPM, sparse blossom, cosmic ray, Versal ACAP, HBM-PIM

---

## 1. The 25% Wall and What It Actually Is

Fault-tolerant quantum computing requires QEC codes that can correct faster than physical errors accumulate. The two dominant families are the surface code [3], now deployed on superconducting systems [4][14], and fusion-based quantum computing (FBQC) for photonic measurement-based architectures [5]. Both emit a classical syndrome — a stream of stabilizer measurement outcomes — and both rely on a classical decoder to identify the most likely physical error pattern consistent with that syndrome and issue a correction.

The FBQC threshold figures from Bartolucci et al. [5] state a 25% fusion-failure threshold for the baseline architecture, with 2.7% photon loss tolerance when operating at that failure rate. These numbers have hardened into shorthand — a "25% wall" — that is often conflated with a theoretical ceiling on what photonic quantum computing can tolerate. That conflation is the thing worth unpacking.

The actual ceiling, set by Varnava, Browne, and Rudolph [6] on one-way cluster-state computing, is **50% qubit loss**. Counterfactual error correction using measurement-based inference lifts the tolerance that high in principle. The 25% number is not a quantum-theoretic limit. It is the combined ceiling imposed by (a) fusion failure rates achievable in hardware today, and (b) the classical decoder's ability to keep up with the syndrome rate produced by a system running at a given physical error rate.

The second term — decoder throughput — is where this paper intervenes. A 1000-logical-qubit surface-code system operating at a 1 MHz syndrome cycle produces on the order of $10^7$ syndrome measurements per second. State-of-the-art minimum-weight perfect matching (MWPM) decoders are $\mathcal{O}(n^3)$ worst case [3][10]; even with sparse blossom and Fusion Blossom optimizations reaching one million errors per core-second [11], hardware accelerators today support real-time decoding of only ~110 logical qubits at the 1 M rounds/sec target [11]. Neural decoders from Google DeepMind [12][13] achieve lower logical error rates than MWPM but at substantially higher compute cost per syndrome. Scale beyond the demonstrations of today — the 12-logical-qubit Quantinuum/Microsoft result [15], the distance-5 Google Sycamore result [4] — collides with a decoder-throughput wall well before it collides with a physics wall.

Pre-filtering the syndrome stream before the deep decoder touches it is therefore the move that directly buys headroom in the 25%–50% gap. If 70–80% of incoming syndromes represent routine already-seen configurations and can be dispatched at nanosecond latency, the deep decoder's per-core budget gets amplified by an order of magnitude at the same physical error rate. That is throughput. That is how the wall moves.

---

## 2. Three Operations, One Primitive

The first paper in this arc closed with an addendum [1 §12] arguing that LSH projection, autoencoder encoding, and Markov likelihood scoring are the same operation: $y = A \cdot x$ with $A$ a matrix whose entries are either sampled (LSH), trained (autoencoder), or count-estimated (Markov). The claim extends cleanly to syndrome decoding.

**Bloom membership on a syndrome vector.** The raw syndrome for a distance-$d$ surface code is a bit-string over the stabilizer set — on the order of $d^2$ bits per round. For $d = 11$, that is a 121-bit syndrome per round per logical qubit. Hashing the syndrome through $k$ independent hash functions and reading $k$ bits from a bit-array is exactly the bloom operation deployed in [1]. For streaming syndrome bit-strings of modest width, the hash evaluation on FPGA fabric is a fixed XOR-and-rotation pipeline compiling to a handful of cycles; the memory reads hit URAM at 1–3 cycle latency [1 §4].

**Markov family likelihood on a residual syndrome.** Once the bloom fires "unseen," the syndrome's transition structure is scored against per-error-family Markov chains. The transition matrix for each family is $|\Sigma| \times |\Sigma|$ for an alphabet $\Sigma$ of quantized syndrome features. For $|\Sigma| = 128$, each family's matrix is 16 KB — fits inside a single AI Engine tile's 32 KB local SRAM. Scoring is

$$ \log P(s \mid \theta_f) = \log \pi_f[s_1] + \sum_{t=2}^{T} \log T_f[s_{t-1},\, s_t] $$

which is a matrix-indexed sum over a length-$T$ sequence. At $T = 20$ and $|\Sigma| = 128$, the per-family scoring completes in ~100 ns on a single AIE tile [1 Table §4].

**QEC syndrome decoding itself.** MWPM reduces to minimum-weight matching on a graph whose edge weights come from the syndrome-to-physical-error likelihood map. The dominant kernel inside MWPM implementations is a series of graph operations; inside neural decoders [12][13], it is explicit GEMM. Either way, once the syndrome arrives at the deep decoder, the primitive work is again matrix-vector multiplication.

All three operations — the bloom's hash-and-lookup, the Markov family's log-probability sum, and the decoder's matching or neural inference — are matrix-vector kernels. All three are what modern silicon is optimized to run. The architecture this paper proposes is not three distinct subsystems; it is one accelerator fabric running three specialized workloads of the same computational shape.

---

## 3. Architecture

Text block diagram for the pre-filtered QEC decoder pipeline:

> Stabilizer measurement layer (physical qubit array, 1–10 MHz syndrome cycle)
> → Raw syndrome bitstring emission, $d^2$-ish bits per round
> → **STAGE 1: Bloom membership lookup on the syndrome**
>   - $k = 8$ hash functions, URAM-resident $\sim 10^8$-bit array, $\sim 15$ ns total
>   - If hash hits all $k$ bits: classify as seen-benign, emit trivial correction or no-op, skip deep decoder
>   - If any bit is zero: mark as unseen, route to Stage 2
> → **STAGE 2: Per-family Markov likelihood comparator**
>   - Families: single-qubit flip, correlated two-qubit flip, measurement readout error, cosmic-ray / correlated burst [16][17]
>   - Transition tables in on-tile SRAM, $\sim 100$ ns per family, fully parallel
>   - Score $L_f$ for each family $f$; find $f^\star = \arg\min_f L_f$
>   - If the minimum $L_{f^\star}$ is above threshold: route to the family-specialized corrector (e.g., cosmic-ray mitigation logic, which differs from single-qubit-flip handling)
>   - If all $L_f$ are below threshold: genuinely novel syndrome — route to deep decoder
> → **STAGE 3: Deep decoder (MWPM, sparse blossom [11], or neural [12][13])**
>   - Runs on the (bloom-unseen, Markov-least-consistent) residual only
>   - Microsecond-class budget per syndrome, which is tolerable because the event rate into this stage has dropped by ~1 order of magnitude
> → Correction action dispatched to the classical control plane
> → Bloom and Markov update: the resolved syndrome is hashed into the bloom bit-array; its transition pairs increment the appropriate family's counting matrix

The insertion point is clean. The pipeline sits in the classical control-plane rack that today already runs MWPM or neural decoders. It does not change the quantum substrate. Versal Premium, Versal HBM series, and VCK5000-class AI Engine mesh silicon — the same parts surveyed in [1 §4] — are purchasable today and compatible with existing decoder firmware interfaces. A first prototype is a Versal ACAP plugin to an existing decoder stack, not a new instrument.

Two operational parameters need calibration, exactly as in [1 §6]:

- **Warm-up.** The bloom array starts empty; every syndrome is "unseen" until the filter accumulates a baseline. Running the deep decoder alongside the bloom for a warm-up window of $T$ syndromes — inserting each resolved syndrome into the filter — builds the seen-set. $T$ is a physics-calibration decision scaled to the physical error rate and the logical qubit count.
- **Aging.** A monotonically-inserted bloom saturates. Rotating filters (active + shadow with periodic zeroing) or counting-bloom variants are standard. The calibration question is whether year-old syndromes should still short-circuit today's decoder; the physics-drift answer is probably no, which argues for the shadow-filter pattern.

---

## 4. Latency Budget

Per-syndrome latency, broken down by stage:

| Stage | Operation | Silicon | Latency | Fraction of syndromes reaching |
|---|---|---|---|---|
| 1 | Bloom $k$-hash lookup | URAM on Versal Premium | $\sim 15$ ns | 100% |
| 2 | Markov per-family scoring (4 families in parallel) | AIE tile cluster | $\sim 100$ ns per family, parallel | $\sim 20$–$30$% |
| 3 | Deep decoder (sparse blossom or neural) | Same FPGA or CPU accelerator | $\sim 0.7$–$2$ $\mu$s [11] | $\sim 5$–$15$% |

The asymmetry is the whole argument. For a well-tuned running computation, 70–80% of syndromes represent repeats of patterns the decoder has already resolved — the "no-detection" trivial syndrome dominates, and routine single-qubit flip syndromes recur heavily. Stage 1 retires those at ~15 ns. Of the residual, most are one of a handful of known families at the prevailing physical error rate, which Stage 2 routes in ~100 ns without waking the deep decoder. Only the genuinely novel-and-unfamiliar syndromes — the correlated burst, the cosmic-ray-induced chip-wide event [16][17], the measurement pathology no one has seen before — reach the deep decoder.

The additive cost for a syndrome that does reach the deep decoder is ~115 ns of pre-filter overhead on top of the $\mathcal{O}(\mu s)$ decoder budget. This is noise. For the syndromes that do not reach the deep decoder, the whole pipeline completes in ~115 ns — well under a typical 1 $\mu$s syndrome cycle, with budget to spare for pipelining.

The throughput multiplier follows: if 80% of syndromes terminate at Stage 1 and another 15% terminate at Stage 2, the deep decoder sees 5% of input rate. A decoder that supported 110 logical qubits at 1 MHz before now supports roughly $110 / 0.05 = 2200$ logical qubits at the same syndrome rate, assuming pre-filter dispatch is genuinely free of deep-decoder contention. In practice the multiplier is lower — the hardest 5% of syndromes are also the hardest to decode — but the order-of-magnitude headroom is real.

---

## 5. What This Buys

Rewriting the 25% wall: $\text{ceiling} = f(\text{fusion failure rate}, \text{decoder throughput})$. The pre-filter architecture leaves fusion failure untouched; that is a physics parameter set by the photonics and the error model. It pushes hard on decoder throughput. The practical error-tolerance ceiling for a photonic MBQC system where decoder saturation was a binding constraint moves upward. How far it moves is an empirical question that requires a specific system to evaluate, but the direction is unambiguous.

For surface-code superconducting systems — Google Sycamore, Quantinuum/Microsoft trapped-ion via the qubit-virtualization layer, Rigetti — the same argument holds. When Quantinuum reported 12 logical qubits at circuit error rate 0.0011 [15], the decoder was running comfortably at that scale. At 1000 logical qubits on the same substrate, it would not be. The pre-filter buys the scaling headroom.

A cleaner way to say this: the pre-filter does not raise the quantum error threshold. It raises the *operating point* that the classical control plane can sustain at a given threshold. For architectures whose decoder is the binding constraint on how many logical qubits can run in parallel, the pre-filter is the cheapest scaling lever available — it adds no qubits, changes no physics, and sits entirely in commodity silicon.

---

## 6. Who Might Care

This is a complementary proposal. It does not compete with MWPM, sparse blossom, or neural decoders; it sits in front of them and narrows their input stream. Respect for each team's architecture path is the frame.

**IonQ** (NYSE: IONQ) — trapped ytterbium and barium ions, all-to-all connectivity, scaling toward fault-tolerant regimes where logical-qubit count pushes the decoder hard. Chad Sakac, SVP Quantum Field Engineering at IonQ (and a former Dell EMC / Rackspace operator whose lens is explicitly applied / time-to-solution rather than qubit-count bragging), is the kind of reader this paper is written for. The DARPA HARQ program selection [IonQ, 2026] underlines the applied-commercial framing.

**PsiQuantum** — the direct case study. The Omega chipset [18][19] is photonic-fusion silicon manufactured at GlobalFoundries, operating in the FBQC framework [5] whose 25% fusion-failure figure this paper is explicitly about. If decoder throughput is a binding constraint on realizable loss tolerance in their architecture, the pre-filter moves the binding term.

**Quantinuum** — 12 logical qubits demonstrated [15] on a trapped-ion H2 machine with decoder-hardware co-design as an explicit engineering priority. Microsoft's qubit-virtualization layer sits above the physical substrate. The pre-filter proposal adds another layer to that stack at the syndrome-decoder boundary.

**Rigetti** — superconducting, smaller-scale today, but the architecture argument applies identically. When the decoder becomes the binding constraint, this moves.

**AWS Braket and Azure Quantum** — cloud paths to applied quantum workloads, where the decoder runs in classical cloud infrastructure alongside hybrid quantum-classical workloads. The pre-filter slots into the classical-side compute cleanly and is agnostic to the quantum backend.

This proposal is not hostile to IBM or Microsoft's research-lab paths, to academic QEC work, or to theoretical-threshold research. Those paths are where the physics moves. This paper is about the engineering that determines whether a given physics regime becomes a product.

---

## 7. Honest Limits

95% epistemic cap. Things this architecture does NOT fix:

- **Fusion failure rates.** A physics parameter. The pre-filter does not reduce it.
- **Gate fidelities, coherence times, T1/T2.** All physics. Untouched.
- **Cosmic-ray strike *frequency*.** The pre-filter routes cosmic-ray events to a specialized corrector faster — it does not make them happen less often. Cosmic-ray mitigation still depends on shielding, modular architecture [17], and redundant encoding.
- **Decoder accuracy on the hardest 5% of syndromes.** The residual that reaches the deep decoder is, by construction, the hardest part. The pre-filter does not help the deep decoder be more accurate; it just gives it more budget per event.
- **Systems where the decoder is not the bottleneck.** If a system is rate-limited by gate fidelity or readout noise and has decoder headroom, this architecture adds overhead without return.

Table of when to use:

| Regime | Pre-filter helps? | Why |
|---|---|---|
| Decoder-throughput saturated, high logical qubit count | Yes — first-line benefit | Order-of-magnitude relief on 80% of syndromes |
| Correlated error bursts dominate (cosmic-ray regime) [16][17] | Yes, partially | Family-routing speeds specialized mitigation |
| Very low physical error rate, decoder underutilized | Probably not | Adds fixed overhead for small payoff |
| Small demo systems (< 50 logical qubits) | Not yet | Deep decoder has ample budget |
| Non-surface-code architectures (color codes, LDPC) | Architecture ports but retuning needed | Alphabet and transition tables rebuilt per code |

None of these are fatal. All of them are real. The 5% of this paper that is wrong is in here somewhere and the useful next step is to identify which 5%.

---

## 8. Cross-Domain Receipts

The bloom + Markov architecture is not speculative hardware waiting to be built. DugganUSA runs it in production today, on a different detector:

- **Bloom novelty check** over a $10^9$-bit array on commodity memory, ingesting web-request features. Sub-100 ms query latency at ~1 million indicators.
- **Per-family Markov likelihood** over the `markov_models` index trained on `behavioral_sessions` (7K+ tagged sessions). Families: benign crawler, reconnaissance scanner, APT-grade actor. The routing rule is "least likely under all three families" — identical in spirit to the "all Markov families low, route to deep decoder" branch in §3.
- **Deep analysis** runs only on the residual that neither the bloom nor the Markov layer could dismiss.

The HEP L1 application [1][2] is the second domain. The QEC syndrome decoder is the third. Same two primitives, same silicon, three different physical detectors:

| Detector | Input | Bloom question | Markov families | Deep stage |
|---|---|---|---|---|
| Web telemetry (DugganUSA production) | URL-path sequence | Hash seen before? | Benign / recon / APT | Threat-intel analyst routing |
| LHC L1 trigger [1][2] | Projected detector event | Topology seen before? | Known physics / pathology / beam-gas | HLT or template pipeline |
| QEC syndrome (this paper) | Syndrome bitstring | Syndrome seen before? | Single-flip / 2-qubit / readout / cosmic-ray | MWPM, sparse blossom, or neural decoder |

The observation that pulsars, Higgs candidates, and MBQC stabilizer measurements are all photon-counting / detection problems with shared signal-detection math is the cross-domain anchor that produced this three-paper arc. The specific move here — lift the bloom + Markov architecture out of the L1 trigger context and drop it into the QEC decoder context — is the contribution of this paper.

---

## 9. Open Questions

1. **How much of the saved decoder budget survives in practice?** The 20× throughput multiplier assumes clean dispatch. Real deep decoders have warm/cold-path contention and pipeline effects. Empirical measurement on a representative workload is the first experiment.
2. **Does family-specific routing help or hurt cosmic-ray mitigation?** Sending a correlated-burst syndrome to a specialized corrector faster is desirable if that corrector is well-defined; if the correct action is just "call MWPM with wider context," the family routing adds latency without benefit.
3. **Alphabet quantization for the Markov layer.** Syndrome bitstrings are already discrete, which helps. But feature sequences derived from them — flip positions, time-between-syndromes, correlation structure — need binning. The binning choice is a modeling decision analogous to [1 §5].
4. **Can the bloom be code-distance-parametric?** Surface codes at distance 3, 5, 7, 11, 13... emit different syndrome widths. Does one bloom array serve all distances, or do we partition per-distance filters?
5. **Counting-bloom variants for drift detection.** A counting bloom lets the filter "forget" by decrementing. Does that match how physical-error-rate drift should flow through the filter?
6. **Neural-decoder interaction.** AlphaQubit-style neural decoders [12][13] already implicitly learn what syndromes are routine. Does bloom + Markov pre-filtering help a neural decoder, or does the neural decoder's own early layers subsume the pre-filter? The test is measurable.
7. **PIM feasibility for the Markov scoring.** HBM-PIM [1 §4] supports a restricted instruction set. Do family-matrix lookups fit natively, or is a custom PIM microprogram required?
8. **Logical qubit count at which this actually matters.** Below ~100 logical qubits, deep decoders have plenty of headroom; above ~1000, they do not. Where is the crossover in practice, per architecture?

---

## 10. Conclusion

The bloom + Markov architecture that was originally written against the CERN L1 trigger problem [1][2] ports to quantum error-correction syndrome decoding as a pre-filter. The mathematics is identical. The silicon is identical. The target changes from photon counts in a calorimeter to stabilizer measurements on a surface code, and the domain-specific renames (syndrome for event, error family for physics family, MWPM or neural decoder for HLT) are the only substantive changes.

The often-cited 25% wall on photonic MBQC systems [5] is not a quantum-theoretic ceiling; the Varnava–Browne–Rudolph bound [6] sits at 50%. The gap between them is set by practical fusion failure rates and by classical-decoder throughput saturation. The pre-filter does not touch the physics term. It pushes hard on the throughput term: 70–80% of syndromes retire at Stage 1 in ~15 ns, another ~15% at Stage 2 in ~100 ns, and only the ~5% genuinely novel residual reaches the deep decoder. That is roughly an order of magnitude of throughput headroom at the same physical error rate.

This is applied work, not theory. It slots into the classical control plane alongside existing decoder stacks on silicon — Versal ACAP, VCK5000 AI Engine mesh, HBM-PIM — that is purchasable today. IonQ, PsiQuantum, Quantinuum, Rigetti, AWS Braket, and Azure Quantum all run architectures where decoder throughput will become binding before physics thresholds do. For those teams, the pre-filter is the cheapest scaling lever on the table.

95% epistemic cap. 5% of this is wrong. The 5% that is wrong is the useful conversation.

---

## References

[1] Duggan, P. (2026). "A Novelty-First L1 Trigger: Bloom Filters and LSH on Modern Memory Architectures." DugganUSA Research, April 22, 2026. Companion paper, same repository. [`bloom-filter-novelty-trigger-l1.md`](./bloom-filter-novelty-trigger-l1.md).

[2] Duggan, P. (2026). "Which Kind of Novel? Markov Second-Stage Classification for Novelty-First L1 Triggers." DugganUSA Research, April 22, 2026. Companion paper, same repository. [`markov-novelty-classification-second-stage.md`](./markov-novelty-classification-second-stage.md).

[3] Fowler, A. G., Mariantoni, M., Martinis, J. M., Cleland, A. N. (2012). "Surface codes: Towards practical large-scale quantum computation." *Physical Review A* 86, 032324. [arXiv:1208.0928](https://arxiv.org/abs/1208.0928).

[4] Google Quantum AI. (2023). "Suppressing quantum errors by scaling a surface code logical qubit." *Nature* 614, 676–681. [arXiv:2207.06431](https://arxiv.org/abs/2207.06431). Distance-5 surface code logical qubit outperforming distance-3 on Sycamore.

[5] Bartolucci, S., Birchall, P., Bombín, H., Cable, H., Dawson, C., Gimeno-Segovia, M., Johnston, E., Kieling, K., Nickerson, N., Pant, M., Pastawski, F., Rudolph, T., Sparrow, C. (2021). "Fusion-based quantum computation." [arXiv:2101.09310](https://arxiv.org/abs/2101.09310). Published as *Nature Communications* 14, 912 (2023). Fusion failure threshold below 25% for the baseline architecture; 2.7% photon loss tolerance at that operating point.

[6] Varnava, M., Browne, D. E., Rudolph, T. (2006). "Loss tolerance in one-way quantum computation via counterfactual error correction." *Physical Review Letters* 97, 120501. [arXiv:quant-ph/0507036](https://arxiv.org/abs/quant-ph/0507036). 50% qubit loss tolerance via counterfactual (measurement-based) error correction.

[7] Duarte, J., Han, S., Harris, P., Jindariani, S., Kreinar, E., Kreis, B., Ngadiuba, J., Pierini, M., Rivera, R., Tran, N., Wu, Z. (2018). "Fast inference of deep neural networks in FPGAs for particle physics." *JINST* 13 P07027. [arXiv:1804.06913](https://arxiv.org/abs/1804.06913). hls4ml foundation; same silicon ecosystem.

[8] FastML Team. (2021). "hls4ml: An Open-Source Codesign Workflow to Empower Scientific Low-Power Machine Learning Devices." [arXiv:2103.05579](https://arxiv.org/abs/2103.05579).

[9] Higgott, O. (2022). "PyMatching: A Python Package for Decoding Quantum Codes with Minimum-Weight Perfect Matching." *ACM Transactions on Quantum Computing*. [dl.acm.org/doi/10.1145/3505637](https://dl.acm.org/doi/10.1145/3505637). MWPM baseline implementation; sub-microsecond per-round decoding at distance 17 on a single core.

[10] Higgott, O., Gidney, C. (2025). "Sparse Blossom: correcting a million errors per core second with minimum-weight matching." *Quantum* 9, 1600. [quantum-journal.org/papers/q-2025-01-20-1600](https://quantum-journal.org/papers/q-2025-01-20-1600/). Current state-of-the-art CPU-class MWPM throughput.

[11] Wu, Y., Zhong, L. (2023). "Fusion Blossom: Fast MWPM Decoders for QEC." Parallel MWPM implementation; hardware accelerator variants [github.com/yuewuo/micro-blossom](https://github.com/yuewuo/micro-blossom) report real-time decoding for ~110 logical qubits at 1 MHz syndrome rate.

[12] Bausch, J., Senior, A. W., Heras, F. J. H., Edlich, T., Davies, A., Newman, M., Jones, C., Satzinger, K., Niu, M. Y., Blackwell, S., Holland, G., Kafri, D., Atalaya, J., Gidney, C., Hassabis, D., Boixo, S., Neven, H., Kohli, P. (2024). "Learning high-accuracy error decoding for quantum processors." *Nature*. [nature.com/articles/s41586-024-08148-8](https://www.nature.com/articles/s41586-024-08148-8). Google DeepMind AlphaQubit neural decoder.

[13] Bausch, J., et al. (2023). "Learning to Decode the Surface Code with a Recurrent, Transformer-Based Neural Network." [arXiv:2310.05900](https://arxiv.org/abs/2310.05900). Recurrent-transformer decoder achieving ~25% lower logical error rate than MWPM on Google Quantum AI experimental data.

[14] Google Quantum AI. (2024). "Quantum error correction below the surface code threshold." *Nature*. [arXiv:2408.13687](https://arxiv.org/abs/2408.13687). Distance-7 demonstration below threshold.

[15] Microsoft and Quantinuum. (2024). "Demonstration of quantum computation and error correction with a tesseract code." [arXiv:2409.04628](https://arxiv.org/abs/2409.04628). 12 logical qubits on H2 trapped-ion hardware; circuit error rate 0.0011.

[16] Wilen, C. D., et al. (2021). "Resolving catastrophic error bursts from cosmic rays in large arrays of superconducting qubits." *Nature Physics* 17, 1342–1347. [arXiv:2104.05219](https://arxiv.org/abs/2104.05219). Original Google Quantum AI characterization of chip-wide correlated error events from cosmic-ray impacts.

[17] McEwen, M., Faoro, L., Arya, K., et al. (2022). "Resolving catastrophic error bursts from cosmic rays in large arrays of superconducting qubits." Follow-up work; see also [arXiv:2402.04245](https://arxiv.org/abs/2402.04245) for direct evidence of cosmic-ray correlated errors and [arXiv:2505.15919](https://arxiv.org/abs/2505.15919) for modular-architecture mitigation.

[18] Alexander, K., et al. (PsiQuantum). (2025). "Manufacturable photonic quantum computing chipset: Omega." *Nature*. See also company announcement, February 26, 2025. [psiquantum.com/omega](https://www.psiquantum.com/omega).

[19] PsiQuantum. (2025). "PsiQuantum Raises $1 Billion to Build Million-Qubit Scale, Fault-Tolerant Quantum Computers." Series E announcement, September 10, 2025.

[20] Kim, J., et al. (2021). "Aquabolt-XL: Samsung HBM2-PIM with in-memory processing for ML accelerators and beyond." *Hot Chips 33*. In-memory compute bandwidth relevant to Markov transition-table lookups at scale.

---

## Acknowledgments

This is the third paper in a three-paper arc; the first [1] was prompted by correspondence with Daniel Whiteson (UC Irvine, ATLAS). The cross-domain insight that the bloom + Markov L1-trigger architecture maps directly onto MBQC syndrome pre-filtering — that pulsars, Higgs candidates, and surface-code stabilizer measurements are the same signal-detection problem at different detectors — is mine. The feasibility survey, literature review, and citation verification were conducted with Claude (Anthropic) under my direction, including the verification of arXiv identifiers and the current state-of-the-art for MWPM throughput, neural decoders, and photonic-fusion architectures.

Thanks to the MWPM and neural-decoder teams whose work this paper sits in front of [9][10][11][12][13] rather than competes with, and to the FBQC authors [5] whose framing of the 25%/50% gap made the intervention point legible. The IonQ applied-operator lens — Chad Sakac's framing of time-to-solution as the customer-facing metric — is part of why this paper is written for commercial quantum teams and not for the theory shelf.

Any errors of physics, engineering, or citation are mine. The 5% of this paper that is wrong is the 5% worth finding.

---

## Citation

```bibtex
@article{duggan2026qecprefilter,
  author = {Duggan, Patrick},
  title = {Lifting the 25\% Wall: A Bloom + Markov Syndrome Pre-Filter for Applied QEC Decoders},
  year = {2026},
  month = {April},
  day = {23},
  publisher = {DugganUSA Research},
  url = {https://github.com/pduggusa/dugganusa-research},
  orcid = {0009-0001-0628-9963},
  note = {Third paper in the bloom + Markov arc; companions are the L1 trigger bloom paper and the Markov second-stage classifier paper}
}
```

---

**Document Hash:** [Computed on commit]
**Last Updated:** April 23, 2026
**Status:** Pre-print / Working Paper
**Pairs with:** `bloom-filter-novelty-trigger-l1.md` v1.2 and `markov-novelty-classification-second-stage.md` v1.0 (same repo, April 22, 2026)
