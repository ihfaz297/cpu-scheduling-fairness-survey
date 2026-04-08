# Quantifying Fairness: A Systematic Survey of CPU Scheduling Algorithms

**A research paper evaluating FCFS, Priority, Round Robin, SJF, and Priority with Aging across 10 real-world HPC workload traces.**

---

## Overview

This repository contains all code, data, and analysis for the paper:

> **"Quantifying Fairness: A Systematic Survey of Starvation and Resource Allocation in FCFS, Priority, and Round Robin CPU Scheduling"**

The study evaluates CPU scheduling algorithms not just on efficiency (throughput, turnaround time) but on **fairness and starvation** metrics. It uses 10 authentic historical HPC trace logs, simulating 200 chronological jobs from each trace through a discrete-event scheduling framework.

---

## Repository Structure

```
OS_paper/
├── Final_Scheduling_Analysis.ipynb   # Main Jupyter notebook (analysis & visualizations)
├── run_scheduling_analysis.py        # Script to run the full scheduling simulation
├── create_notebook.py                # Helper script to generate the notebook
├── results.csv                       # Full results across all algorithms, traces, and metrics
├── Review_Paper.md                   # Full paper write-up
├── SCHEDULING_COMPARISON.md          # Database vs Kubernetes scheduling deep-dive
├── REAL_WORLD_USE_CASES.md           # Algorithm recommendations by use case
├── QUICK_REFERENCE.md                # TL;DR reference card
├── dataset_references.txt            # Guide to real-world workload trace sources
├── images/                           # 34 comparative visualization charts
├── *.swf                             # HPC workload trace files (Standard Workload Format)
└── results.csv                       # 53 performance metrics across all runs
```

---

## Datasets

Ten real-world historical HPC workload traces in the **Standard Workload Format (SWF)** are used, sourced from the Parallel Workloads Archive and Grid Workloads Archive:

| # | Trace | Institution | Year |
|---|-------|-------------|------|
| 1 | **SDSC-SP2** | San Diego Supercomputer Center | 1998–2000 |
| 2 | **SDSC-BLUE** | San Diego Supercomputer Center | — |
| 3 | **ANL-Intrepid** | Argonne National Laboratory (Blue Gene/P) | 2009 |
| 4 | **CTC-SP2** | Cornell Theory Center | 1996 |
| 5 | **HPC2N** | High-Performance Computing Center North | 2002 |
| 6 | **KTH-SP2** | Royal Institute of Technology, Sweden | 1996 |
| 7 | **CEA-Curie** | CEA, France (Curie supercomputer) | 2011 |
| 8 | **PIK-IPLEX** | Potsdam Institute for Climate Impact Research | 2009 |
| 9 | **RICC** | RIKEN Integrated Cluster of Clusters, Japan | 2010 |
| 10 | **Lublin-1024** | Validated benchmark for extreme heterogeneity | — |

---

## Scheduling Algorithms Evaluated

| Algorithm | Description |
|-----------|-------------|
| **FCFS** | First-Come, First-Served — processes in strict arrival order |
| **SJF** | Shortest Job First — theoretical optimum for AWT |
| **Round Robin (RR)** | Time-sliced with quantum ≈ half the mean burst time per dataset |
| **Priority** | Hierarchical priority-based dispatch |
| **Priority + Aging** | Priority with dynamic aging (priority increases ~every 10 mean burst cycles) |

---

## Evaluation Metrics (53 Total)

### Performance
- **Average Waiting Time (AWT)** — time spent in ready queue
- **Average Response Time** — time from arrival to first CPU execution
- **Bounded Slowdown / Max Bounded Slowdown** — relative wait penalty proportional to job size
- **Throughput, CPU Utilization, Makespan**

### Fairness
- **Jain's Fairness Index (JFI)** — resource allocation equality (0 = worst, 1 = perfect)
- **Waiting Time Coefficient of Variation (WT CV)** — statistical predictability of wait times
- **Starvation Rate** — percentage of processes waiting more than 3× the average burst duration
- **Size Fairness Ratio** — relative wait disparity between small and large jobs

### System Health
- **Average Ready Queue Length** (via Little's Law: L = λ × W)
- **Convoy Effect** — degree to which short jobs are blocked by long ones
- **Priority Inversion Potential** — risk of low-priority jobs blocking high-priority ones
- **Preemption Frequency, Task Switching Efficiency**

---

## Key Findings

### Round Robin — Best for Interactive/OLTP Workloads
- Delivers **4.96× faster response time** vs FCFS (SDSC-SP2: 208,799 vs 1,036,546)
- Reduces ready queue length by **9×** vs FCFS in KTH-SP2 (13 vs 117 processes)
- Prevents convoy effect through time-slicing
- **Recommended for:** databases, web servers, interactive terminals

### SJF — Efficient but Structurally Unfair
- Achieves the best AWT by far (SDSC-SP2: ~105,000 vs ~824,000 for FCFS)
- Lowest JFI score (e.g., 0.14 in SDSC-SP2) — extreme unfairness to long jobs
- WT CV spikes to 2.41 — wildly erratic wait experience

### FCFS — Simple but Susceptible to Convoy Effect
- Max Bounded Slowdown of **1,120,413** in SDSC-SP2 (a tiny job waited 1M× its runtime)
- Ironically scores high JFI — equal suffering creates mathematical fairness

### Priority + Aging — Best for Multi-Tenant/Orchestrated Workloads
- Achieves **0.97 JFI within priority classes** (vs 0.55 for Round Robin)
- Aging prevents indefinite starvation that occurs under pure Priority scheduling
- **43× fewer priority inversions** than Round Robin (33 vs 1,436)
- **Recommended for:** Kubernetes, HPC cluster schedulers, cloud platforms

---

## Practical Recommendations

| Use Case | Best Algorithm | Key Reason |
|----------|---------------|------------|
| OLTP Databases (PostgreSQL, MySQL) | **Round Robin** | 2.4× faster response, prevents blocking |
| Web Servers (Nginx, Apache) | **Round Robin** | Low tail latency, small-job bias |
| Kubernetes / Container Orchestration | **Priority + Aging** | QoS classes, no indefinite starvation |
| HPC Batch Queues | **Priority + Aging** | Resource-weighted fairness |
| General-Purpose OS | **Round Robin** | Default safe choice |
| Batch ETL / CI-CD | **SJF** | Minimize total completion time |

See [SCHEDULING_COMPARISON.md](SCHEDULING_COMPARISON.md), [REAL_WORLD_USE_CASES.md](REAL_WORLD_USE_CASES.md), and [QUICK_REFERENCE.md](QUICK_REFERENCE.md) for full decision trees and implementation guidance.

---

## Running the Analysis

### Prerequisites
- Python 3.x
- Jupyter Notebook
- pandas, numpy, matplotlib (standard data science stack)

### Steps

1. **Run the simulation script:**
   ```bash
   python run_scheduling_analysis.py
   ```

2. **Open the notebook for interactive analysis and charts:**
   ```bash
   jupyter notebook Final_Scheduling_Analysis.ipynb
   ```

3. **Results are saved to** `results.csv` — 53 metrics across all algorithm/trace combinations.

---

## Visualizations

The `images/` directory contains 34 comparative charts, including:

| Chart | Description |
|-------|-------------|
| `AWT.png` | Average Waiting Time across all algorithms and traces |
| `Response_Time.png` | Average Response Time comparison |
| `Bounded_Slowdown.png` | Relative fairness — wait penalty per job size |
| `Max_Bounded_Slowdown.png` | Worst-case scheduling anomaly |
| `JFI.png` | Jain's Fairness Index |
| `WT_CV.png` | Waiting Time Coefficient of Variation |
| `Average_Queue_Length.png` | Ready queue depth (via Little's Law) |
| `Convoy_Effect.png` | Convoy effect measurement |
| `Size_Fairness_Ratio.png` | Small vs large job wait disparity |
| `Priority_Inversion_Potential.png` | Priority inversion risk |
| `Starvation_Rate_Pct.png` | Percentage of starved processes |

---

## Core Conclusion

> **Algorithms that maximize theoretical mathematical fairness (FCFS JFI) or theoretical peak efficiency (SJF) rarely align with actual user responsiveness, relative fairness, and system stability.**

- **Round Robin** is the practical synthesis for bursty, interactive real-world workloads — it drops response latencies, controls bounded slowdown for small tasks, and alleviates queue pressure.
- **Priority + Aging** is essential when QoS tiers must be respected without risking indefinite starvation.
- The quantum size for Round Robin must be tuned per workload (≈ half the mean burst time) for optimal performance.

---

## References

- [Parallel Workloads Archive](http://www.cs.huji.ac.il/labs/parallel/workload/)
- [Grid Workload Archive](https://gwa.ewi.tudelft.nl/)
- Full paper: [Review_Paper.md](Review_Paper.md)
- Raw results: [results.csv](results.csv)
