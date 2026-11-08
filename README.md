# LoRaWAN Single-Channel Scalability: Simulation Data and Configuration

This repository accompanies the manuscript **"LoRaWAN Scalability Under Dense Periodic Traffic on a Single Channel Evaluated by Discrete Event Simulation"**, which is under review at *Discover Applied Sciences*. It contains the per-run summary tables of all 3,078 simulation runs reported in the paper, the base simulator configuration, the configuration audit, and a sample of per-packet logs.

The study characterises a single-gateway, single-channel LoRaWAN cell under a deliberately dense periodic load (one 50 B packet about every 5 s per node). It locates the node density at which confirmed delivery collapses, and it measures how the collision model, the duty cycle, mobility, fading, ADR, the number of channels, the retry limit and the packet airtime shape that result.

## Highlights

| Quantity | Value |
|---|---|
| Density at which confirmed PDR crosses 90% (5 s interval, 1 channel) | 56.9 nodes (95% interval 56.6 to 57.1, 20 seeds) |
| Equivalent offered load | G = 0.67 |
| Confirmed and unconfirmed PDR curves cross at | 73.7 nodes |
| Goodput peak, confirmed mode | 4.12 kbit/s at 54 nodes |
| Effect of removing capture, unconfirmed PDR at 50 nodes | 36.5 percentage points lower |
| Agreement with the analytic pure ALOHA curve (Poisson, no capture) | within 0.57 percentage points |
| 90% crossing with two channels | 112.7 nodes (ratio of 1.975 to one channel) |
| Retry limit of 3 against 8 retransmissions at 70 nodes | 12.3 percentage points higher PDR |

All values are conditional on the simulated model and its packet airtime of 58.4 ms at SF7. Node-count thresholds should be converted through the offered load when other airtimes are used (see experiment E12).

## Repository contents

```
.
├── README.md                    This file
├── DATA_DICTIONARY.md           Column-level description of every table
├── CONFIG_AND_BRANCH.md         Simulator version, commit, seeds and fixed settings
├── CONFIG_AUDIT.md              Effective model parameters and model findings
├── omnetpp_revision_base.ini    Base OMNeT++ configuration shared by all experiments
├── checks.csv                   Automated integrity checks (122 of 122 pass)
├── E1/ ... E12/                 Summary tables per experiment (15 tables for E1)
└── raw_sample/<EXP>/            Per-packet logs of the first seed of every cell (gzip CSV)
```

Each experiment folder contains the same family of tables (`<E>_runs.csv`, `<E>_run_metrics.csv`, `<E>_loss_causes.csv`, `<E>_node_level.csv`, and so on) plus a few experiment-specific files. The join key across all tables is `run_id`. Experiment E5 (per-packet instrumentation) is built into every run and has no folder of its own. The full column description is in [DATA_DICTIONARY.md](DATA_DICTIONARY.md).

## Experiments

| ID | Purpose | Main factors | Seeds | Runs |
|---|---|---|---|---|
| E1 | Replicated density sweep | 12 densities from 40 to 100 nodes, confirmed and unconfirmed | 1000 to 1019 | 480 |
| E2 | Transfer across reporting intervals | Mean interval of 5, 10 and 30 s at matched offered load | 2000 to 2009 | 220 |
| E3 | Collision model ablation | Capture, symmetric marking, cross-SF isolation, worst case | 3000 to 3009 | 250 |
| E4 | Duty cycle ablation over 3,600 s | Limiter off, baseline, standard airtime, deferral, standard PHY | 4000 to 4009 | 250 |
| E6 | Channel scaling | 1, 2 and 8 uplink channels, 45 to 200 nodes | 6000 to 6009 | 280 |
| E7 | Mobility and fading | Speed 0 to 20 m/s, four fading models, Doppler error model on or off | 7000 to 7009 | 600 |
| E8 | Burst traffic | 50 fragment bursts against periodic traffic of equal load | 8000 to 8009 | 240 |
| E9 | Analytic benchmark | Poisson traffic, no capture, 10 to 400 nodes, compared with exp(−2G) | 9000 to 9009 | 88 |
| E10 | ADR ablation | ADR on or fixed SF7 at 14 dBm | 10000 to 10009 | 180 |
| E11 | Retry-limit sensitivity | 0, 1, 3, 5 and 8 retransmissions, 40 to 70 nodes | 11000 to 11009 | 250 |
| E12 | Payload and airtime sensitivity | 20, 50 and 100 B with simulator or Semtech airtime | 12000 to 12009 | 240 |
| | | | **Total** | **3,078** |

All runs completed without error, and no cell has zero variance.

## Simulation model in brief

* **Simulator.** A custom OMNeT++ 6.1 model whose structure and PHY and MAC concepts follow FLoRa 1.1.0. The model source does not use INET or FLoRa classes. Details on the commit and branch are in [CONFIG_AND_BRANCH.md](CONFIG_AND_BRANCH.md).
* **Geometry.** Fixed for every run: a 2 km by 2 km region, one gateway at (1000 m, 1000 m), nodes placed uniformly in [50 m, 1950 m] with random waypoint mobility (5 m/s unless varied).
* **Traffic.** 50 B packets, mean interval of 5 s with uniform 0.5 to 1.5 times jitter and a random start offset, 500 s of traffic plus 60 s of settling time.
* **MAC.** Confirmed mode uses an ACK timeout of 5 s plus 0 to 1 s jitter and up to 8 retransmissions (9 transmissions in total). Unconfirmed mode sends once and generates no ACK.
* **Radio.** SF7 start, 14 dBm, 125 kHz, sensitivity of −137 dBm, free-space loss with a 30 dB environmental term, per-packet log-normal shadowing and Rayleigh fading, 6 dB same-SF capture and 20 dB cross-SF isolation.
* **ADR and duty cycle.** Standard ADR (window of 20 packets, margin of 10 dB). A 1% duty cycle limiter with a sliding window of 3,600 s that drops packets that exceed the budget.
* **Energy.** Radio energy only, 44 mA at 3.7 V for transmission and 15 mA for 2 ms in two receive windows per uplink.

Every experiment is the base configuration in [omnetpp_revision_base.ini](omnetpp_revision_base.ini) plus command-line overrides, and each run stores the effective configuration read back from the simulator. The audit in [CONFIG_AUDIT.md](CONFIG_AUDIT.md) lists the effective parameters and the model behaviours that matter when interpreting the results.

## Quick start

The tables are plain UTF-8 CSV files and can be read with any tool. The following example reproduces the confirmed-mode delivery curve of experiment E1.

```python
import pandas as pd

runs = pd.read_csv("E1/E1_runs.csv")
metrics = pd.read_csv("E1/E1_run_metrics.csv").drop(columns="experiment")
df = runs.merge(metrics, on="run_id")

curve = (df[df["mode"] == "confirmed"]
         .groupby("N")["PDR_gateway"]
         .agg(["mean", "std", "count"]))
print(curve.loc[[40, 50, 60, 100]])
```

Expected output for the selected densities is a mean PDR of 0.999 at 40 nodes, 0.988 at 50 nodes, 0.828 at 60 nodes and 0.345 at 100 nodes, each over 20 seeds. The per-seed 90% crossing densities are in `E1/E1_N90_per_seed.csv` and average 56.90 nodes with a standard deviation of 0.53.

## Data quality

`checks.csv` records 122 automated checks, all of which pass. They confirm among other things that delivered packets never exceed generated ones, that the loss causes add up exactly to the logged attempts, that unconfirmed runs contain no acknowledgements, that per-node sums equal run totals, and that the effective configuration of every run equals the requested one. A repeated run with the same seed reproduces the identical per-packet log.

## Scope and limitations

* The study is simulation based. The analytic ALOHA benchmark (E9) validates the contention logic but is not a comparison with testbed measurements.
* One gateway only. Multi-gateway deployments, an enhanced or slotted MAC, downlink capacity limits and gateway demodulator limits are not modelled, and acknowledgements never collide.
* The packet airtime of the simulator is shorter than the standard Semtech value, so absolute node counts depend on it. Experiment E12 quantifies this effect.
* Node positions were not logged, so only the mean distance to the gateway is available.

## Citation

If you use these data, please cite the manuscript.

> Buang IN, Zen K, Khan AS. LoRaWAN scalability under dense periodic traffic on a single channel evaluated by discrete event simulation. Under review at *Discover Applied Sciences*.

The citation will be updated with the volume, pages and DOI once the article is published.

## Authors and contact

Izzah Nilamsyukriyah Buang (corresponding author), Kartinah Zen and Adnan Shahid Khan, Faculty of Computer Science and Information Technology, Universiti Malaysia Sarawak, Kota Samarahan, Sarawak, Malaysia. Correspondence: 24010024@siswa.unimas.my.

## Funding and acknowledgements

This work was supported by Universiti Malaysia Sarawak under Mentor Grant UNI/F08/MENTOR/86070/2023. The authors thank the OMNeT++ and FLoRa communities.

## Licence

No licence file is included yet. Please add one before making the repository public so that reuse terms are explicit.
