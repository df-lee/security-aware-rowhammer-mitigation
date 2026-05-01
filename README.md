# Security-Aware RowHammer Mitigation in a DRAM Model

> A Ramulator2-based DRAM simulation project that studies how RowHammer mitigation can prioritize security-critical memory regions while still exposing timing-related overhead.

## Project Story

RowHammer is not only a reliability problem. A bit flip in ordinary user data may corrupt a value, but a bit flip in security-critical metadata can compromise the entire system. Many mitigation schemes treat all DRAM rows uniformly, even though not all rows carry the same security consequence.

This project asks a simple question:

> What happens if RowHammer mitigation is biased toward security-critical rows instead of protecting every row equally?

To explore this, I built a RowHammer fault model and several mitigation policies on top of a DDR4 timing configuration in Ramulator2. The model does not try to reproduce every circuit-level detail of DRAM disturbance physics. Instead, it captures the system-level behavior needed for policy comparison: activation-based hammer counting, adjacent-row disturbance accumulation, bit-flip injection, critical-row labeling, and policy-triggered maintenance actions.

The key extension is a timing-aware maintenance surrogate. Instead of recording protection events only as logical counters, committed protection actions are injected into the controller priority path as row-level open/close maintenance sequences. This makes it possible to study not only whether a policy reduces critical-region bit flips, but also how that protection affects average read latency, total memory-system cycles, and priority-queue occupancy.

## What This Project Builds

The project compares four mitigation policies under the same logical fault model:

<div align="center">

| Policy Family | Uniform Version | Security-Aware Version | Main Control Knob |
|:---:|:---:|:---:|:---:|
| Target Row Refresh | Uniform TRR | Security-Aware TRR | Disturbance threshold |
| Probabilistic Adjacent Row Activation | Uniform PARA | Security-Aware PARA | Protection probability |

</div>

Uniform policies apply the same rule to all rows. Security-aware policies apply stronger protection to rows labeled as security-critical and weaker protection to non-critical rows.

## Methodology

The evaluation follows a staged workflow rather than jumping directly into final policy comparison.

### 1. Build a Logical RowHammer Fault Model

Each logical row maintains an activation counter. When the activation count reaches a configurable hammer threshold, the model treats this as one effective hammer event. That hammer event increases the disturbance level of the two adjacent rows.

$$
D_{r-1} \leftarrow D_{r-1} + 1, \qquad D_{r+1} \leftarrow D_{r+1} + 1
$$

Intuitively, each effective hammer event says: row `r` has been activated often enough that its neighbors should become more disturbed.

Once a victim row accumulates enough disturbance, the model can inject a bit flip. The project supports both deterministic and probabilistic flip modes. In the probabilistic mode, the flip probability begins after a configurable onset point and can follow a step, linear, or sigmoid function.

### 2. Add Security-Critical Row Labeling

Logical rows are partitioned into critical and non-critical regions using a synthetic stride-and-offset labeling rule. This keeps the experiment controlled while allowing the mitigation policies to distinguish between high-impact and lower-impact corruption targets.

The main metric is therefore not only total bit flips, but critical-region bit flips.

### 3. Add Timing-Aware Maintenance

A purely logical model can tell us how many protection events happen, but it cannot reveal whether those events would disrupt the memory controller. To expose this overhead, committed protection actions are sent through a timing-aware path:

$$
\text{Protection Decision}
\rightarrow
\text{Priority-Path Injection}
\rightarrow
\text{Row-Level Open/Close}
\rightarrow
\text{Timing Metrics}
$$

This surrogate is intentionally lightweight. It is not a transistor-level model of targeted refresh, but it is enough to make policy overhead visible through read latency, memory-system cycles, and queue occupancy.

### 4. Use a Fault-Model Sweep to Find Informative Regions

Before tuning mitigation policies, the project sweeps fault-model parameters to find where the system is neither too benign nor too saturated. This matters because if the fault setting is too mild, all policies look similar; if it is too aggressive, the comparison may be dominated by fault injection rather than policy behavior.

<div align="center">

| Parameter | Values |
|:---:|:---:|
| Hammer Threshold | `{1, 2, 3, 4}` |
| Probabilistic Flip Onset | `{1, 2, 3, 4}` |
| Probability Function | Sigmoid |
| Logical Row-Space Size | `{64, 128, 256, 512, 1024}` |
| Uniform TRR Threshold | `3` |
| Security-Aware TRR | `(tc, tn) = (3, 10)` |
| Uniform PARA Probability | `0.01` |
| Security-Aware PARA | `(pc, pn) = (0.18, 0.02)` |

</div>

The sweep shows that the most useful comparison region appears at small hammer thresholds and small logical row spaces, especially 64-row and 128-row settings.

### 5. Run a Timing-Aware Policy Sweep

The main policy sweep focuses on four representative fault settings selected from the fault-model study.

<div align="center">

| Tag | Hammer | Onset | Function | Rows |
|:---:|:---:|:---:|:---:|:---:|
| S64-S1 | 1 | 1 | Sigmoid | 64 |
| S128-S1 | 1 | 1 | Sigmoid | 128 |
| L64-S1 | 1 | 1 | Linear | 64 |
| S64-S2 | 1 | 2 | Sigmoid | 64 |

</div>

TRR is swept over critical and non-critical disturbance thresholds. PARA is swept over critical and non-critical protection probabilities.

### Evaluation Focus

RowHammer mitigation naturally exposes a three-way trade-off among critical-region protection, non-critical reliability, and timing overhead. This project focuses on the security-prioritized setting: critical-region protection is treated as the primary objective, while non-critical flips are measured but treated as a secondary outcome rather than a selection constraint.

## Mitigation Policies

### Uniform TRR

Uniform TRR uses the same disturbance threshold for every victim row. When a row's disturbance reaches the threshold, the controller schedules a protection action.

### Security-Aware TRR

Security-aware TRR assigns separate thresholds to critical and non-critical rows:

$$
T(r) =
\begin{cases}
 t_c, & r \in \text{critical rows} \\
 t_n, & r \in \text{non-critical rows}
\end{cases}
$$

The intuition is straightforward: critical rows should be protected earlier, while non-critical rows can tolerate a more relaxed threshold to reduce unnecessary maintenance pressure.

### Uniform PARA

Uniform PARA applies a fixed probability of protection when a hammer event affects a victim row. The same probability is used for all rows.

### Security-Aware PARA

Security-aware PARA assigns separate probabilities to critical and non-critical rows:

$$
P(r) =
\begin{cases}
 p_c, & r \in \text{critical rows} \\
 p_n, & r \in \text{non-critical rows}
\end{cases}
$$

Here, the goal is to keep non-critical protection probability low while increasing the probability of protecting critical rows.

## Key Results

The two mitigation families show different security-aware trade-off patterns.

### TRR: Selective Relaxation

Security-aware TRR performs best when critical rows use a moderate threshold and non-critical rows use a much more relaxed threshold. This reduces critical-region bit flips while also lowering timing overhead relative to uniform TRR.

The best TRR settings concentrate around:

$$
(t_c, t_n) \approx (3, 10)
$$

This means the benefit does not come from making every row more aggressively protected. It comes from avoiding unnecessary maintenance on non-critical rows while still protecting critical rows early enough.

### PARA: Selective Intensification

Security-aware PARA shows a clearer protection-versus-performance trade-off. It improves critical-region protection by assigning a substantially higher protection probability to critical rows, but this usually increases timing overhead.

The favorable PARA region keeps:

$$
p_n \text{ low}, \qquad p_c \text{ high}
$$

In the tested settings, strong configurations often use values such as:

$$
(p_c, p_n) = (0.18, 0.02)
$$

This makes PARA useful when the goal is to deliberately spend extra maintenance effort on critical rows.

## Best Configurations

<div align="center">

| Family | Setting | Parameters | Critical Flips | Avg. Read Latency | Cycles |
|:---:|:---:|:---:|:---:|:---:|:---:|
| TRR | S64-S1 | `tc = 3, tn = 10` | 0 | 95.26 | 188082 |
| TRR | S128-S1 | `tc = 3, tn = 10` | 0 | 81.29 | 187965 |
| TRR | L64-S1 | `tc = 4, tn = 10` | 2 | 68.07 | 187910 |
| TRR | S64-S2 | `tc = 3, tn = 10` | 0 | 95.26 | 188082 |
| PARA | S64-S1 | `pc = 0.18, pn = 0.02` | 0 | 178.57 | 188403 |
| PARA | S128-S1 | `pc = 0.01, pn = 0.01` | 0 | 139.78 | 188233 |
| PARA | L64-S1 | `pc = 0.18, pn = 0.02` | 1 | 178.57 | 188403 |
| PARA | S64-S2 | `pc = 0.16, pn = 0.02` | 0 | 167.94 | 188331 |

</div>

## Takeaways

Security-aware RowHammer mitigation is most effective when it reallocates limited protection effort rather than increasing protection uniformly.

For TRR, security awareness can improve both protection and performance by relaxing non-critical maintenance while keeping critical thresholds moderate. For PARA, security awareness improves critical-row protection by increasing critical-row protection probability, but the extra protection usually comes with measurable timing cost.

The broader fault-model sweep also shows that not every parameter setting is equally informative. Harder regimes with small hammer thresholds and small logical row spaces expose meaningful differences between policies, while larger thresholds and larger row spaces quickly become too mild to separate policy behavior.

## Limitations

This project uses a logical RowHammer disturbance model rather than a circuit-level DRAM physics model. The row-space compression knob is an experimental stress mechanism, not a native DRAM remapping feature. The timing-aware maintenance path is also a first-order surrogate: it exposes controller-level overhead trends, but it should not be interpreted as a complete implementation of vendor-specific targeted refresh.

These limitations are intentional. The goal is to create a controlled simulation framework for comparing security-aware mitigation policies under consistent assumptions.

## Report

The full project report is available here:

[`Security_Aware_RowHammer_Mitigation.pdf`](./Security_Aware_RowHammer_Mitigation.pdf)

## Tool

- [Ramulator2](https://github.com/CMU-SAFARI/ramulator2)
