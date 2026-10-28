# BEACON: Biologically Inspired Edge-Swarm Coordination

Simulator and experiment notebook for the manuscript

> **A Biologically Inspired Edge-Swarm Framework for Resilient, Low-Latency Multi-Agent Coordination**
> Amruthamsha Prashanth Raju, Nitte Meenakshi Institute of Technology (NMIT), Bengaluru
> *Manuscript under review.*

BEACON (**B**iologically inspired **E**dge-swarm **A**daptive **CO**ordinatio**N**) is a fully decentralized framework in which edge nodes coordinate without a central server. It combines three mechanisms, each with an explicit mathematical form:

1. **Pheromone-reinforced task allocation** with bounded trail intensity and evaporation (ant-colony style).
2. **Gossip-based parameter consensus** over a time-varying communication graph, using Metropolis-Hastings mixing weights.
3. **A stigmergic state field** propagated by local diffusion, with staleness discounting.

This repository contains the discrete-event simulator, the baselines, and every experiment reported in the manuscript, together with the exact outputs of the run used for the paper.

## Repository contents

```
.
├── BEACON_simulator.ipynb   # simulator, baselines and all experiments (executed, with outputs)
├── results/                 # tables (CSV, text, LaTeX) and figures from the reported run
├── requirements.txt
├── CITATION.cff
└── LICENSE
```

## Quick start

**Google Colab (recommended).** Open `BEACON_simulator.ipynb` in Colab and run all cells. No GPU is needed.

**Locally.**

```bash
pip install -r requirements.txt
jupyter notebook BEACON_simulator.ipynb
```

The N-BaIoT attack captures are stored as `.rar` members inside the UCI archive, so the loader needs `unrar` or `bsdtar` on the system path (`sudo apt-get install unrar` on Debian/Ubuntu). The notebook tries to install it automatically on Colab.

**Data.** With `USE_REAL = True` (the default, and the setting for every reported result), the notebook downloads [N-BaIoT](https://archive.ics.uci.edu/dataset/442/detection+of+iot+botnet+attacks+n+baiot) from the UCI repository on first run (about 1 GB) and caches it as `nbaiot.zip`. Set `USE_REAL = False` for a synthetic dataset that runs in seconds and is useful for checking the pipeline.

**Runtime.** About 30 to 60 minutes on a free Colab CPU for the full sweep, plus the download.

## Implementation

* Pure NumPy, with NetworkX for topology. The model is a small MLP stored as one flat parameter vector, so gossip averaging is a weighted sum of vectors.
* Nodes sit on a random geometric graph with per-round link dropout (`p_link`), three heterogeneous capacity tiers, a first-order radio energy model, and periodic capacity shocks that make historical evidence go stale.
* Baselines are the same simulation loop with mechanisms switched off:

| Method | Description |
|---|---|
| `cloud` | Centralized inference with a jittered wide-area round trip per task |
| `edge` | Each node runs its own tasks; no coordination |
| `fedavg` | Federated averaging every 5 rounds through an aggregator, with a straggler barrier |
| `gossip` | Gossip consensus only; no task offloading |
| `beacon` | Full framework |

All parameters live in the `Config` dataclass in Section 1 of the notebook.

## Results

All numbers below are from the run committed in `results/` (N-BaIoT, 115 features, 9 devices, N = 50 nodes unless stated).

**End-to-end decision latency, N = 50** (`results/table5_latency.csv`)

| Architecture | Mean (ms) | P95 (ms) | P99 (ms) |
|---|---|---|---|
| Centralized cloud | 66.4 ± 0.3 | 125.3 | 169.5 |
| Edge-only | 7.4 ± 0.5 | 30.2 | 41.5 |
| FedAvg | 8.2 ± 0.5 | 31.0 | 42.5 |
| Gossip-only | 7.3 ± 0.5 | 30.1 | 41.9 |
| **BEACON** | **7.0 ± 0.6** | **14.0** | **19.4** |

BEACON reduces P95 latency by 54.8% relative to FedAvg.

**Scalability** (`results/table8_scalability.csv`): BEACON's per-node communication stays between 2.83 and 3.01 MB from N = 10 to N = 200 (a 6.4% change), while the FedAvg aggregator's load grows linearly, from 10.3 MB to 205.2 MB.

**Ablation, Dirichlet α = 0.1** (`results/table10_ablation.csv`)

| Configuration | P95 (ms) | Macro-F1 |
|---|---|---|
| Full framework | 14 | 0.984 |
| No stigmergic field (λ = 0) | 14 | 0.984 |
| No pheromone memory (α = 0) | 20 | 0.984 |
| No heuristic term (β = 0) | 16 | 0.984 |
| No evaporation (ρ = 0) | 20 | 0.984 |
| No pheromone bounds | 20 | 0.984 |
| No gossip consensus | 30 | 0.595 |

Model quality under non-IID partitioning (`results/table6_noniid.csv`) matches centralized training within the confidence interval at every Dirichlet concentration tested.

## Negative results and limitations

These are reported in the manuscript and are repeated here so that the code is not read as claiming more than it shows.

* **Stigmergic diffusion has no measurable effect.** The diffusion rate λ is inert in both the sensitivity sweep and the ablation. The pheromone floor φ_min and the energy weight ξ are also inert within the tested ranges (ξ because the energy budget is never approached).
* **BEACON costs more energy, not less.** It draws about 6.4 times the energy of FedAvg (19.52 J against 3.07 J) and sends more data per node (2.97 MB against 1.03 MB), because gossip exchanges parameters with every active neighbour. Its advantage is tail latency and the absence of a privileged aggregator.
* **Resilience is argued structurally, not measured.** The node-failure experiment (`results/table7_resilience.csv`, `results/fig3_resilience.png`) shows no F1 drop for any method on this benchmark, because surviving nodes still hold a representative sample of the data. It is included for completeness.
* **Seeds.** `SEEDS = (0, 1, 2)`: latency, non-IID and node-failure experiments use three seeds; scalability, energy, sensitivity and ablation use two; the (α, β) grid uses one.
* **Tuning.** Default hyperparameters were chosen using the same sweeps that are reported, on the evaluation data.
* **Simulation only.** No hardware testbed; the cloud baseline uses a 60 ms round trip with lognormal jitter (σ = 0.45).

## Mapping to the manuscript

Output file names follow the notebook's internal numbering, which predates the final manuscript numbering.

| Output file | Manuscript item |
|---|---|
| `table5_latency.csv`, `fig2_latency_cdf.png` | Table 5, Fig. 2 |
| `table6_noniid.csv` | Table 6 |
| `table8_scalability.csv` | Table 7 |
| `fig4_energy.png`, `fig4_energy.csv` | Fig. 3 |
| `table9_sensitivity.csv` | Table 8 |
| `fig5_heatmap.png`, `fig5_grid.csv` | Fig. 4 |
| `table10_ablation.csv` | Table 9 |
| `table7_resilience.csv`, `fig3_resilience.png` | Supplementary |

## Citation

If you use this code, please cite the manuscript (citation details will be updated on publication):

```bibtex
@unpublished{raju_beacon,
  author = {Raju, Amruthamsha Prashanth},
  title  = {A Biologically Inspired Edge-Swarm Framework for Resilient, Low-Latency Multi-Agent Coordination},
  note   = {Manuscript under review},
  year   = {2026}
}
```

## License

Released under the MIT License. See `LICENSE`.

The N-BaIoT dataset is not redistributed here; it is downloaded from the UCI Machine Learning Repository and is subject to its own terms.
