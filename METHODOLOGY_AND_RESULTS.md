# Quantifying Fairness in CPU Scheduling

## Concise Description

This project evaluates how five CPU scheduling algorithms behave on 10 real-world HPC workload traces. The focus is not only throughput and waiting time, but also fairness, starvation risk, queue stability, and response-time predictability. The study uses a discrete-event simulator so that every algorithm is tested on the same job arrivals and the same workload structure.

The corrected design samples 564 jobs from each trace. That gives a consistent workload size across traces and aligns the implementation with the paper's stated power-analysis assumptions.

## Methodology

### 1. Data Sources

The analysis uses historical HPC traces in Standard Workload Format (SWF). These traces come from real supercomputing environments, so they preserve burstiness, skew, long-tail behavior, and arrival irregularity that synthetic datasets usually hide. That makes them a better test bed for fairness studies because the scheduler must respond to realistic job mixes rather than clean, evenly spaced workloads.

The traces are treated as chronological records of submitted jobs. Each job contributes an arrival timestamp, a runtime or service requirement, and a derived priority proxy where needed. Before simulation, the jobs are sorted by arrival order and normalized so that each trace starts at time zero. This keeps traces comparable while preserving their internal timing structure.

The 10 traces are:

- SDSC-SP2
- SDSC-BLUE
- ANL-Intrepid
- CTC-SP2
- HPC2N
- KTH-SP2
- CEA-Curie
- PIK-IPLEX
- RICC
- Lublin-1024

Each trace is sampled at 564 jobs, and the jobs are processed in chronological order.

### 2. Simulation Model

The study uses a discrete-event simulation built around three core components:

- a ready queue for jobs waiting to run
- a single CPU core that executes one job at a time
- an event clock that advances when jobs arrive, complete, or are preempted

The simulator is work-conserving, meaning the CPU never idles when runnable jobs exist. This keeps the comparison focused on scheduling policy rather than artificial delays.

The event flow is intentionally simple:

1. A job arrives and enters the ready queue.
2. The scheduler selects the next runnable job according to the active policy.
3. The selected job runs until completion or until a time quantum expires, depending on the policy.
4. If a job is preempted, it is returned to the queue with its remaining service time.
5. The simulator advances to the next event and repeats until all jobs finish.

This design makes the comparison deterministic for a given trace and policy, which is important because the goal is to measure the effect of scheduling rules rather than random runtime variation.

At runtime, each job is tracked with a small set of state variables:

- arrival time: when the job becomes available
- start time: when the job first receives CPU service
- finish time: when the job completes
- remaining time: how much work is left if the job is preempted
- priority: the scheduling rank used by priority-based policies
- preemption count: how many times the job is interrupted under Round Robin or related policies

These fields let the simulator reconstruct the full execution history of every job after the run completes.

### 3. Algorithms Compared

The same workload is run through five policies:

- FCFS: jobs run in arrival order
- SJF: shortest job runs first
- Round Robin: jobs share the CPU in time slices
- Priority: higher-priority jobs go first
- Priority with Aging: waiting jobs gain priority over time to reduce starvation

This combination lets the study compare classic throughput-oriented scheduling against fairness-oriented scheduling. FCFS acts as the baseline arrival-order policy. SJF represents a throughput-optimized scheduler. Round Robin introduces responsiveness through time slicing. Priority scheduling tests how rank-based dispatch affects fairness, while Priority with Aging adds a starvation-prevention mechanism that gradually lifts long-waiting jobs.

Each policy is evaluated on the same trace so that the only difference is how the scheduler orders waiting jobs. That makes the resulting metrics directly comparable across algorithms.

The policies differ in a few important ways:

- FCFS is non-preemptive and keeps jobs in strict arrival order.
- SJF is non-preemptive and always selects the smallest waiting job first.
- Round Robin is preemptive and rotates jobs after each quantum.
- Priority is non-preemptive and always selects the highest-priority job available.
- Priority with Aging raises the effective priority of waiting jobs so that long waits become increasingly likely to end.

These distinctions matter because preemptive policies improve responsiveness but can introduce switching overhead, while non-preemptive policies are simpler but can produce convoy effects or starvation.

### 4. Preprocessing and Job Representation

Each SWF record is reduced to a compact job representation suitable for scheduling simulation. In practice, the simulator uses:

- arrival time: when the job enters the system
- process time: the amount of CPU time the job requires
- priority: a proxy value used for priority-based scheduling

If a trace contains malformed or unusable rows, those rows are skipped rather than imputed. This keeps the simulation anchored in valid trace data. After loading, the job list is sorted by arrival time so that the simulator can process events in temporal order.

The result is a clean queue of jobs that can be replayed under multiple schedulers without changing the underlying workload.

The preprocessing stage also serves a quality-control role. It ensures that invalid rows do not contaminate the experiment, that job order is stable, and that each scheduler sees the same workload shape. In other words, preprocessing prevents the workload representation from becoming a hidden experimental variable.

### 5. Sample-Size Logic

The sample-size choice is based on the paper's stated assumptions: target power of 0.8, significance level of 0.05, effect size around 0.5, and heavy-tailed job sizes with variance approximated as sigma = 3 mu. Under those assumptions, the corrected minimum sample size is 564 jobs per trace.

That is why the repository now uses 564 jobs per trace instead of 200. If only 200 jobs were used, the resulting power would be much lower and the run should be treated as exploratory rather than definitive.

This matters because sample size directly affects how confidently differences between schedulers can be detected. A small sample may still show trends, but it is more sensitive to noise from unusually short jobs, unusually long jobs, or highly bursty arrival patterns. The larger 564-job sample reduces that risk and makes the comparison more stable.

The choice of 564 also reflects a trade-off between statistical confidence and workload realism. The traces are large enough that a 564-job slice still preserves the heterogeneity of the original system, while keeping the simulation tractable for repeated runs across multiple algorithms.

### 6. Metrics Collected

The simulator records a broad set of performance and fairness outcomes:

- waiting time
- turnaround time
- response time
- bounded slowdown
- CPU utilization
- makespan
- average queue length
- Jain's Fairness Index
- waiting-time coefficient of variation
- starvation count and starvation rate
- preemption frequency
- priority inversion potential

These metrics let the analysis evaluate not just speed, but also predictability and fairness under different scheduling rules.

The metrics are computed after each job finishes so that every result reflects the full execution history of the scheduler. Waiting time and turnaround time capture direct efficiency effects, while Jain's Fairness Index, starvation rate, and waiting-time variation capture how evenly the scheduler treats jobs across the trace. Queue-length and preemption metrics show how much operational overhead the policy introduces.

Several of the metrics can be written directly from the job timeline:

- turnaround time = finish time - arrival time
- waiting time = turnaround time - process time
- response time = start time - arrival time
- bounded slowdown = turnaround time / max(process time, 1)
- starvation rate = jobs whose waiting time exceeds three times the average burst duration

For fairness, the study uses Jain's Fairness Index over waiting-time or service-related distributions to capture how evenly jobs experience delay. Higher values indicate more even treatment across the trace, while lower values indicate skew or concentration of delay on particular jobs.

For queue behavior, average queue length is interpreted as a proxy for congestion, and preemption frequency is interpreted as a proxy for switching pressure. Together they show whether a policy is merely fast or also operationally stable.

Together, the metrics answer three questions:

1. How fast does the scheduler complete work?
2. How fair is the experience for jobs of different sizes or priorities?
3. How stable is the queueing behavior over time?

### 7. Interpretation and Limitations

The methodology is intentionally comparative rather than prescriptive. It does not claim that one scheduler is always best in every environment; instead, it highlights the trade-offs that appear when real HPC workloads are replayed under different policies.

One limitation is that the simulation models a single-core scheduler and does not attempt to model hardware-level effects such as NUMA placement, cache interference, or multi-node communication delays. That keeps the analysis readable and reproducible, but it also means the results should be interpreted as scheduler behavior under a controlled abstraction rather than as a full cluster-performance model.

Another limitation is that the simulator assumes the trace-derived job fields are sufficient to characterize scheduling behavior. Real production systems may also depend on memory pressure, I/O contention, user-level deadlines, and cluster-topology effects. Those factors are outside the scope of this file, so the methodology should be read as a controlled scheduling study rather than a complete systems benchmark.

### 8. Verification and Reproducibility

The implementation is designed to be easy to audit. The same trace list, sampling size, and scheduling policies are used every time the analysis runs, which means the main source of variation comes from the workload itself rather than from hidden randomness.

The methodology is therefore reproducible at three levels:

1. the same traces can be loaded again
2. the same 564-job sampling rule can be applied again
3. the same metrics can be recomputed from the resulting job timelines

This makes it straightforward to compare scheduler output across runs, to regenerate figures, and to check whether changes in the code alter the measured behavior.

## Images

Below are representative figures from the analysis. The full image set is available in the `images/` directory.

### Performance and Fairness

![Average Waiting Time](images/AWT.png)

![Response Time](images/Response_Time.png)

![Jain's Fairness Index](images/JFI.png)

![Bounded Slowdown](images/Bounded_Slowdown.png)

![Starvation Rate](images/Starvation_Rate_Pct.png)

### Queue and System Behavior

![Average Queue Length](images/Average_Queue_Length.png)

![Convoy Effect](images/Convoy_Effect.png)

![Preemption Frequency](images/Preemption_Frequency.png)

![Priority Inversion Potential](images/Priority_Inversion_Potential.png)

## Tables

### Table 1: Dataset Summary

| # | Trace | Description |
|---|---|---|
| 1 | SDSC-SP2 | Real HPC trace from the San Diego Supercomputer Center |
| 2 | SDSC-BLUE | Real HPC trace from SDSC |
| 3 | ANL-Intrepid | Argonne National Laboratory trace |
| 4 | CTC-SP2 | Cornell Theory Center trace |
| 5 | HPC2N | High-Performance Computing Center North trace |
| 6 | KTH-SP2 | Swedish HPC trace |
| 7 | CEA-Curie | Curie supercomputer trace |
| 8 | PIK-IPLEX | Potsdam Institute for Climate Impact Research trace |
| 9 | RICC | RIKEN Integrated Cluster of Clusters trace |
| 10 | Lublin-1024 | Validated benchmark trace for heterogeneous workloads |

### Table 2: Algorithms Compared

| Algorithm | Core Idea | Main Strength | Main Weakness |
|---|---|---|---|
| FCFS | First arrival runs first | Simple and predictable | Convoy effect |
| SJF | Shortest job runs first | Low average waiting time | Can starve long jobs |
| Round Robin | Time slicing | Good responsiveness | Extra context switching |
| Priority | Higher priority first | Honors urgency | Starvation risk |
| Priority with Aging | Priority increases with wait time | Reduces starvation | More complex to tune |

### Table 3: Primary Metrics

| Metric | What It Measures |
|---|---|
| AWT | Average time jobs spend waiting before execution |
| Response Time | Delay from arrival to first CPU access |
| Bounded Slowdown | Waiting penalty normalized by job size |
| JFI | Equality of resource allocation or waiting experience |
| Starvation Rate | Fraction of jobs waiting far longer than expected |
| Queue Length | How crowded the ready queue becomes |

## Short Takeaway

The main result is that there is no universally best CPU scheduler. FCFS is simple, SJF is efficient but unfair, Round Robin balances responsiveness, and Priority with Aging is the safest choice when fairness and starvation prevention matter most.
