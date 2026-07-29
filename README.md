#Evaluation-Efficient Design of a Tow-Thomas Bandpass Filter Using TabPFN-based Bayesian Optimisation

Code and experiments for a controlled comparison of four optimisers for SPICE-in-the-loop design of a second-order Tow-Thomas biquad bandpass filter.

Submitted to DELCON 2026.

Overview

Picking component values for an active filter is a hard discrete optimisation problem: textbook formulae ignore parasitics and finite GBW, components are restricted to standard series (E12 resistors, preferred-value capacitors), and some configurations fail to converge in SPICE. The 8-component search space here has roughly 4.3 × 10¹¹ configurations, which rules out both gradient methods and brute force.

This repo benchmarks:

Method	Type	Surrogate
SA	Model-free metaheuristic	—
GP-BO	Bayesian optimisation	Gaussian process (Matérn-5/2 ARD + white noise)
TabPFN-BS	Bayesian optimisation	TabPFN with bootstrap-ensemble uncertainty (B = 5)
TabPFN-Q	Bayesian optimisation	TabPFN with native quantile uncertainty (q₀.₁, q₀.₅, q₀.₉)

The three BO variants run inside a single shared loop — candidate generation, context cap, acquisition function, and top-K verification are identical — so the surrogate is the only experimental variable.

Problem setup

Circuit. Tow-Thomas biquad, bandpass output. Simulated in PySpice with ideal VCVS op-amps (gain 10⁵) and high-impedance bias resistors for numerical stability. AC analysis from 50 Hz to 20 kHz at 300 points/decade.

Design variables. 6 resistors from the E12 series (100 Ω – 100 kΩ, 37 values) and 2 capacitors from a custom series (1 – 100 nF, 13 values).

Objective. 1 kHz bandpass with rejection at 200 Hz and 5 kHz:

J(x) = (1 − |H(f_target)|/|H|_max)² + 2.5·[ (|H(f_low)|/|H|_max)² + (|H(f_high)|/|H|_max)² ]

Simulation failures receive a penalty of J = 1e6. Failures count against the budget but are excluded from the surrogate context so the model isn't distorted by the penalty plateau.

Fairness protocol
Shared initialization — one Latin Hypercube Sampling set of 30 points, drawn and evaluated once, handed identically to every method. BO uses it as the initial dataset; SA is warm-started from the best point in the pool.
Equal budget in evaluations — 500 SPICE evaluations for every method, initial points included, no early stopping. Counting evaluations rather than iterations matters: one BO iteration costs 3 simulations (occasionally 4) while an SA iteration costs 1, so counting iterations would hand BO a hidden 3× budget.
Identical BO loop — implemented once; only the (μ, σ) predictor differs.
Repetition — six independent runs, with the same seed shared across all algorithms within a run.

SA's initial temperature is scaled to the warm start (T₀ = 0.5 × J(x₀)) so that a fixed cold-start temperature doesn't immediately random-walk away from a good starting point.

Key results

Median best-so-far cost at fixed budget cutoffs (bold = better than SA at the same budget):

Evals	SA	GP-BO	TabPFN-BS	TabPFN-Q
50	0.180	0.123	0.115	0.287
100	0.103	0.081	0.062	0.196
150	0.083	0.050	0.055	0.098
200	0.074	0.046	0.047	0.063
300	0.036	0.046	0.042	0.059
500	0.036	0.041	0.034	0.027

Final cost over six runs at the full budget:

Method	Median	IQR	Best (run)	Wins	Time (s)
SA	0.0360	[0.027, 0.050]	0.0191 (5)	1	0.7
GP-BO	0.0408	[0.029, 0.050]	0.0172 (1)	0	212
TabPFN-BS	0.0342	[0.022, 0.053]	0.0051 (1)	3	3039
TabPFN-Q	0.0274	[0.020, 0.037]	0.0032 (5)	2	616

Takeaways:

Two regimes. Surrogates lead under tight budgets — by 200 evaluations all three BO variants beat SA's median, and GP-BO and TabPFN-BS do so on five of six runs. Past ~250 evaluations SA catches up: its temperature has decayed to essentially zero, so it degenerates into greedy local search plus 15% random restarts, which is hard to beat given hundreds of evaluations.
Quantile beats bootstrap. TabPFN-Q gets a better median (0.0274 vs 0.0342) at roughly one fifth of the surrogate compute — the 4.9× wall-clock ratio matches the bootstrap factor B = 5 directly.
No method dominates. All four medians land within a factor of 1.5 of each other, and run-to-run variation is the same order as the between-method differences. The 5-of-6 pattern reaches only a one-sided sign-test p ≈ 0.11.
TabPFN-Q is a slow starter. Its native quantile spread is narrow on small contexts, so EI turns mostly exploitative early on and progress waits until the context is informative.
Requirements
Package	Version
Python	3.13
PySpice	1.5 (ngspice 34)
scikit-learn	1.8
tabpfn	2.2.1
SciPy	1.16
bash
pip install -r requirements.txt

ngspice must be installed and on the system path — PySpice shells out to it.

Reference hardware: Intel i5-13450HX, 16 GB RAM. All surrogates run on CPU.

Complexity

With m = 500 candidates, n ≤ 150 context size, B = 5 bootstraps, and C_spice ≈ 1.4 ms per simulation:

T_GP-BO ≈ O(T(n³ + mn))
T_Q     ≈ O(T·m·n²)
T_BS    ≈ B · T_Q
T_SA    ≈ O(N · C_spice)

Measured per-iteration: ~1.4 s (GP-BO), ~4 s (quantile), ~20 s (bootstrap).

Note that these overheads only matter because the test circuit is cheap (1.4 ms/eval). For genuinely expensive simulations — transient analyses, Monte-Carlo corners, post-layout extraction — surrogate overhead becomes negligible and only sample efficiency matters.

Limitations and future work

Conclusions rest on one circuit topology and six runs. Natural extensions: more topologies, and multi-objective / Pareto-optimal formulations rather than the single scalarised cost used here.

Key references
Hollmann et al., Accurate predictions on small data with a tabular foundation model, Nature 637:319–326, 2025 — TabPFN v2.
Tow, Design formulas for active RC filters using operational-amplifier biquad, Electronics Letters 5(15), 1969 — the topology.
Durmuş, Optimal components selection for active filter design with average differential evolution algorithm, AEU 94:293–302, 2018 — metaheuristic baseline and circuit reference.
Kirkpatrick, Gelatt & Vecchi, Optimization by Simulated Annealing, Science 220(4598), 1983.
Shahriari et al., Taking the human out of the loop, Proc. IEEE 104(1), 2016 — BO review.

Full reference list is in the paper.
