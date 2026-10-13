# Fixed-corpus confidence research for Limite Vulkan

Research note, 2026-10-13. **The user accepted the qualification experiment and conservative bound in Q20, and the severe-taskset release rule in Q21; no implementation or measured qualification result exists.** This note preserves [ADR-0008](../adr/0008-use-equal-taskset-quality-loss.md) and the [design review](limite-vulkan-design-review.md): complete frozen `math-extended`, equal taskset headline weights, expected stochastic correctness, CUDA-minus-Vulkan loss, one predeclared claim-bearing analysis per platform/candidate, and a one-sided 95% upper bound at most 0.02. Windows and Linux remain separate claims. No GPU inference, empirical coverage experiment, quality score, or qualification bound has been measured; the original evaluator and CUDA behavior are unchanged.

## Source facts and primary references

- **[H]** Wassily Hoeffding, *Probability Inequalities for Sums of Bounded Random Variables*, JASA 58 (1963), 13–30, [original paper scan](https://userpages.cs.umbc.edu/lomonaco/f08/643/hwk643/Hoeffding.pdf), [DOI](https://doi.org/10.1080/01621459.1963.10500830). Theorem 2, p. 16, equation (2.6), assumes **independent**, bounded summands; their ranges need not be equal and their means need not coincide. For their sum, it gives `P(S - E S >= x) <= exp(-2 x² / sum(range_width²))`. Applying it to negative summands gives the lower-tail inequality needed for an upper confidence bound. The original theorem page was inspected as an image because text extraction failed.
- **[E]** Bradley Efron, *Bootstrap Methods: Another Look at the Jackknife*, Annals of Statistics 7 (1979), 1–26, [DOI/publisher](https://doi.org/10.1214/aos/1176344552), [original-paper PDF](https://sites.stat.washington.edu/courses/stat527/s14/readings/ann_stat1979.pdf). Section 2, pp. 2–4, starts with a random sample from an unknown distribution, replaces it by the empirical distribution, and **approximates** the original sampling distribution. Monte Carlo bootstrap repetitions approximate that bootstrap distribution, not additional real observations. This paper supplies neither a universal finite-sample percentile-interval guarantee nor justification for resampling the members of this frozen corpus.
- **[S]** Steven Howard et al., *Time-uniform, nonparametric, nonasymptotic confidence sequences*, Annals of Statistics 49 (2021), [author manuscript](https://arxiv.org/html/1810.08240v9). Section 1, equation (1), defines simultaneous coverage over time; Section 2 distinguishes average **conditional** expectations from a fixed common mean. Sequential guarantees require the stated process assumptions, not just a sequential-looking formula.
- **Repository facts:** [mean-of-groups](../../limite-evals/packages/limite-evals-core/src/limite_evals_core/stats.py#L52-L59) averages rollouts within a problem and then problems; [headline selection](../../limite-evals/src/limite_evals/aggregate.py#L169-L193) retains the declared scorer anchor/fallback. [Suite definitions](../../limite-evals/src/limite_evals/suites.py#L308-L355) include 32-, 4-, and 1-rollout tasksets. The [runner](../../limite-evals/src/limite_evals/runner.py#L400-L410) uses `base_seed + problem_index * group_size + rollout_index`; the [CLI](../../limite-evals/src/limite_evals/cli.py#L514-L527) passes the same base seed to each taskset. Thus derived seed values can repeat across tasksets. Distinct integer offsets are **not evidence of independent randomized draws**. [Known overlap](../../limite-evals/src/limite_evals/suites.py#L477-L500) keeps four Apex problems also present in AIME25/HMMT25. The [existing comparator](../../limite-evals/src/limite_evals/compare.py#L105-L131) resamples problems jointly between engines but rollouts separately on the two sides, then returns two-sided percentile endpoints. It is not the proposed construction below.

## Accepted fixed-corpus estimand and finite-sample bound

**Mathematical construction.** Let there be `T` frozen tasksets, `n_t` problems in taskset `t`, and its declared `k_ti` rollouts for problem `i`. Let `C_tir` and `V_tir` be the selected headline correctness outcomes in `[0,1]` for a paired CUDA/Vulkan rollout, under the frozen measurement and randomization contract. Define

\[
 D_{tir}=C_{tir}-V_{tir}\in[-1,1],\qquad
 a_{tir}=\frac{1}{T n_t k_{ti}},\qquad
 \widehat\Delta=\sum_{t,i,r}a_{tir}D_{tir},\qquad
 \Delta=\mathbb E\widehat\Delta.
\]

The weights sum to one. Individual problem/engine success probabilities can all differ. This is the accepted equal-taskset expected mean-of-rollouts loss, not pooled accuracy and not an unseen-problem expectation. Pairing requires complete corresponding measurements, not equality of the engines' sampled texts or PRNG implementations. CUDA and Vulkan may be arbitrarily dependent within a pair; pairing does not change their marginal expected score difference.

Before observing qualification outcomes, partition all contributions into declared clusters `g`. Define

\[
 Z_g=\sum_{(t,i,r)\in g}a_{tir}D_{tir},\qquad
 B_g=\sum_{(t,i,r)\in g}a_{tir};\qquad -B_g\le Z_g\le B_g.
\]

**Required assumption:** the cluster random vectors are mutually independent under the declared experiment. Dependence **within** a cluster is unrestricted. There is no iid-problem or common-success-probability assumption. Applying [H] to `-Z_g`, whose range width is `2 B_g`, yields

\[
 \Pr(\Delta-\widehat\Delta\ge\epsilon)
 \le \exp\!\left(-\frac{\epsilon^2}{2\sum_g B_g^2}\right).
\]

Therefore a conservative, finite-sample, one-sided bound is

\[
 U_\alpha=\min\!\left\{1,\widehat\Delta+
 \sqrt{2\log(1/\alpha)\sum_g B_g^2}\right\},
 \qquad \Pr(\Delta\le U_\alpha)\ge1-\alpha.
\]

Use `alpha = 0.05` for the single accepted platform/candidate look; qualify only when frozen coverage is complete and `U_0.05 <= 0.02`. Clipping at one is justified by the parameter support, not by a favorable observed variance. Cluster definitions, weights, sample sizes, and bounds must not be chosen after seeing the outcomes. This is a theorem application **conditional on the experimental assumptions**, not evidence that the current evaluator satisfies them.

**Precision tradeoff, derived rather than measured:** with `N_eff = 1 / sum(B_g²)`, the radius is `sqrt(2 log(20) / N_eff)`. At an observed loss of zero, radius at most 0.02 requires `N_eff >= 2 log(20) / 0.02²` (about 14,979 equal-weight independent clusters). Unequal taskset/problem weights and larger clusters reduce effective precision. A negative observed loss can leave a larger acceptable radius. These are planning calculations, not power or GPU-cost measurements.

## What is an independent observation here?

**Accepted qualification-only experiment:** freeze an immutable per-request seed manifest, drawing seeds independently from a predeclared supported seed distribution **with replacement**, before qualification outputs are available. Use one seed draw on both engines for each paired request; use separate draws for different request pairs unless their deliberate sharing is recorded in a cluster. Preserve every rendered input, scorer/fallback, taskset membership, declared rollout group size, sampler, and budget. Keep the legacy base-seed/offset behavior unchanged outside qualification.

- If a request's outcome is a fixed function of its own randomized seed and frozen execution conditions, independent seed draws give independent request-pair outcomes, even with nonidentical functions/probabilities. Accidental equal seed values from independent draws do not destroy independence; **deliberately reusing one draw** does. Enforcing unique seeds by rejection creates a without-replacement experiment and needs a different justification. Do not silently equate the seed-induced distribution with ideal independent token-sampling randomness: declare the actual randomization law and establish that it is the intended stochastic-correctness target.
- A committed manifest makes the experiment replayable, but does not make its seed values extra observations. Coverage is over the prospective random manifest-generation/execution experiment, **not a conditional 95% guarantee for every particular fixed seed list**. A hand-selected deterministic list, or rerunning the same list under a repeatable profile, does not establish that experiment. Nor does a single randomly chosen base seed plus deterministic offsets establish independent per-request seeds.
- **Execution isolation is a prerequisite, not an observed capability.** Random inputs can affect completion lengths, continuous/dynamic batching, batch shapes, cache state, kernels, and later numerical execution profiles. Consequently outcomes may depend on other requests' seeds even when seeds were independently drawn. To use request pairs as independent clusters, establish a scheduling-independent outcome mechanism, or isolate each pair into an identified execution profile with no shared mutable outcome-affecting state/randomness. Isolation can sacrifice normal batching throughput and change the operational profile being qualified; it cannot replace the accepted operational CUDA reference with a favorable diagnostic mode. If independence can only be justified for batches or whole runs, use those larger units instead.
- Repeated mathematical problem identity is **not by itself random dependence** on a fixed corpus: fresh independent generation draws for an identical prompt can be independent. Conversely, a shared actual completion, reused seed/random stream, fixed variant family driven by shared randomness, or common outcome-affecting run state must retain its dependence. Preserve overlap memberships/weights either way. If a completion contributes twice, add both weights to its cluster; do not count it as two independent observations. Different prompts/scorers for an overlapping problem must retain their own measurements.
- With independent requests, each paired rollout is a cluster, giving `sum(B_g²) = sum_ti 1 / (T² n_t² k_ti)`. If only problem groups are independent and their rollouts may be arbitrarily dependent, cluster entire groups: the sum is `sum_ti 1 / (T² n_t²)` before further mergers. Thus additional within-cluster rollouts receive **no unjustified independence bonus**. A one-rollout problem can still contribute to a valid weighted bound across independent requests; it cannot estimate its own stochastic variance through within-problem bootstrap resampling.
- Merge clusters connected by actual shared randomness/outcomes, including transitive connections. If dependence is unresolved across all requests of one suite execution, treat that execution as one cluster. For one such run the Hoeffding radius exceeds the full loss range and clipping gives only `U = 1`. This method supplies no useful qualification guarantee under arbitrary suite-wide dependence.

## Comparison with bootstrap and whole-suite replication

| Construction | Defensible scope | Cost or limitation |
| --- | --- | --- |
| Weighted independent-cluster Hoeffding | Genuine finite-sample coverage under the stated independent-cluster experiment; heterogeneous probabilities and arbitrary within-cluster dependence allowed. | Conservative and insensitive to observed low variance or pairing gains; needs a defensible cluster/seed/profile manifest. |
| Existing problem/rollout bootstrap | Descriptive/exploratory output under its own resampling model. | Resampling problems introduces problem-composition variation absent from the fixed-corpus target; separate side resampling does not preserve paired seed-level covariance. No universal finite-sample coverage. |
| Fixed-corpus rollout/cluster bootstrap | Keep all problems and weights fixed; resample independent repeated observations within each fixed cell, preserving CUDA/Vulkan pairs and shared-outcome clusters. Whole-vector repetitions are a cleaner resampling unit when dependencies cross cells. | At best an approximate claim unless an applicable theorem with its assumptions is supplied. One observation per cell cannot reveal that cell's stochastic variation; heterogeneous clusters are not automatically exchangeable. More bootstrap draws reduce resampling Monte Carlo error only. |
| Independent whole-suite replicate Hoeffding | Genuine finite-sample coverage when complete suite loss vectors are independent repetitions of the same frozen operational/randomization contract. Arbitrary dependence within each suite is allowed. | Much coarser: all corpus averaging benefits inside a run are discarded by the worst-case range bound. Requires fresh paired CUDA **and** Vulkan evidence, not a reused reference baseline. |

For `R` independent complete paired-suite repetitions, each retaining the declared `k_ti`, define `L_j` as that repetition's equal-taskset loss. With `E L_j = Delta` and `L_j in [-1,1]`, [H] gives

\[
 U_\alpha=\min\{1,\overline L+\sqrt{2\log(1/\alpha)/R}\}.
\]

The common expected loss must follow from the same frozen profile/process distribution; independence alone would bound the average of changing run expectations instead. Fresh independent seed vectors plus isolated/reset outcome-affecting run state are one proposed route. Reusing one CUDA realization across all `L_j` couples them and does not justify this formula. Average complete repetitions without changing within-repetition rollout rules or treating extra runs as a changed headline. At zero observed loss the same roughly 14,979-unit planning threshold now means **whole-suite repetitions**, illustrating the conservatism. A more efficient finite-sample variance-adaptive method would require a separately selected applicable theorem and experiment; ordinary bootstrap or a normal/t interval is not an equivalent guarantee.

**Degenerate data:** all observed paired losses equal zero does not establish that future random draws have zero loss. As a mathematical counterexample, one bounded paired loss with `D = 1` with probability 0.1 and `D = 0` otherwise has expected loss 0.1; after observing zero (probability 0.9), its empirical percentile bootstrap collapses to `[0,0]`. More resamples cannot fix this failure. The Hoeffding bound keeps a nonzero radius determined by design, even on all-zero/all-correct/all-incorrect samples. A smaller deterministic range requires an independently justified property of the mechanism, not the observed extrema. Missing graders or incomplete groups are not zero losses and cannot be dropped to manufacture complete coverage.

## Accepted severe-taskset release rule

Q21 blocks release while a statistically supported taskset loss above 0.05 remains unresolved, even if the overall upper bound passes. The conservative default is a predeclared Bonferroni family of one-sided lower bounds across the `T` frozen tasksets, with family error budget `beta = 0.05` for the single platform/candidate look. Other claim-bearing looks require their own preallocated error accounting.

For taskset `t`, retain its mean-of-problems/rollouts estimator `delta_hat_t` and define its normalized mass in each already justified dependence cluster as

\[
 b_{tg}=\sum_{(t,i,r)\in g}\frac{1}{n_t k_{ti}},\qquad
 L_t=\max\!\left\{-1,\widehat\delta_t-
 \sqrt{2\log(T/\beta)\sum_g b_{tg}^2}\right\}.
\]

The same independent-cluster theorem [H] gives each lower bound error at most `beta/T`; the union bound gives simultaneous lower-bound coverage at least `1-beta` without requiring independent tasksets. Flag a supported severe regression only when `L_t > 0.05`, then fix and requalify before release. A non-triggering lower bound is not proof that the taskset's loss is at most 0.05. Missing or incomplete required results cannot be treated as non-triggering successes.

Use conservatively evaluated bounds for gate comparisons, not rounded display values. Overall acceptance still requires complete compatible coverage, supported experimental assumptions, the upper-bound gate and all other numerical/serving/platform requirements. This is a mathematical design rule, not executed evaluator functionality or GPU evidence.

## Looks, recommendation, and exact remaining prerequisites

Keep the accepted single fixed look after predeclared complete coverage; exploratory pilots cannot become the qualifying sample through favorable selection. Freeze the candidate, design, final sample size and analysis before collecting its claim-bearing evidence. If additional claim-bearing looks are authorized, preallocate `alpha_j` with `sum(alpha_j) <= 0.05` and use each look's valid bound at that level; the union bound controls the family without requiring independence between looks. Alternatively specify an applicable confidence sequence [S], including why its conditional-expectation target is the accepted fixed expected loss. Repeated ordinary 95% bounds until one passes are not covered. Separate 95% platform claims do not become a joint 95% guarantee; a joint family needs explicit allocation. Qualification-driven candidate selection/reuse of already observed evidence also needs an explicit claim-family plan, not a new label on the same favorable look.

**Accepted baseline:** use the weighted independent-cluster Hoeffding construction with qualification-only randomized paired seed manifests and the smallest clusters whose independence is actually defensible. Keep the legacy bootstrap available as labelled exploratory information, never as proof of this bound. Where ordinary operational batching prevents request-level independence, use justified larger independent clusters or auditable independent whole-suite repetitions; accept their larger cost and possible pending outcome rather than inventing independence.

**Prerequisites before this is usable:** (1) frozen full corpus/rendered inputs, taskset sizes, scoring availability and overlap memberships; (2) declared seed space/distribution, independently generated immutable paired request manifest, and its relation to the intended stochastic-correctness law; (3) execution/profile and reset/scheduling rules establishing independence between the selected clusters without substituting diagnostic CUDA behavior; (4) a provenance map for shared completions/seeds/state and cluster masses, not merely task-local indices; (5) predeclared complete observation count, cluster weights, final look and candidate/platform claim-family policy; and (6) the actual paired operational executions on both platforms and CUDA reference access. The present deterministic seed schedule and missing dependence provenance do **not** establish these assumptions. Without a justified independent-unit experiment (or another explicitly proved dependence model), only an uninformative conservative bound is available here; qualification remains pending.
