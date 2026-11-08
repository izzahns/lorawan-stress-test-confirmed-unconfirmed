# Configuration and provenance

* Simulator: OMNeT++ 6.1 custom LoRaWAN model (FLoRa-based extension), single gateway, extended for the revision; built with the bundled clang, release mode.
* Git branch: `revision-runs`; code commit `b0dba65dab3fca480e6ed8638898007f2ec2238d` (local repository `LoRaWANProject_S_Revision`; later working-tree changes: runner retry logic, N <= 200 filter, N90 logistic guard).
* Original result folders were not modified. All outputs come from `revision_runs/`.
* Command (all experiments, resumable): `python scripts/run_experiments.py --exp all --jobs 5`; summaries: `python scripts/summarise_runs.py --jobs 5` then `python scripts/finalize_summary.py`.
* One run = `LoRaWANProject_S_rev.exe -u Cmdenv -f simulations/revision_base.ini --seed-set=<seed> <overrides>`; the exact overrides per run are in the delivered `revision_results_for_github/config/experiment_manifest.csv`.
* Seeds: E1 1000-1019; E2 2000-2009; E3 3000-3009; E4 4000-4009; E6 6000-6009; E7 7000-7009; E8 8000-8009; E9 9000-9009; E10 10000-10009; E11 11000-11009; E12 12000-12009.
* Fixed geometry for every run: 2 km x 2 km, gateway at (1000, 1000) m, nodes placed uniformly in [50, 1950] m and moving by random waypoint inside that box; one gateway; N <= 200 (author decision).
* Traffic: 50 B packets, mean interval 5 s (uniform 0.5-1.5x jitter, random start offset in [0,5] s), traffic window 500 s (+60 s settle; 3,600 s in E4; 600 s in E8; 1,000 s in E9; interval-dependent in E2).
* MAC as in the original paper runs: confirmed mode = up to 1 + 8 transmissions, ACK timeout 5 s + U(0,1) s (paper text says 2 s / 8 transmissions; see config note below); unconfirmed = one transmission, no ACK.
* Radio: SF7 start, TX 14 dBm, 125 kHz, sensitivity -137 dBm, free-space + 30 dB environment factor + per-packet log-normal shadowing (6-10 dB) + Rayleigh (d > 100 m), collision threshold 6 dB, cross-SF isolation 20 dB, density-based RSSI penalty on, ADR standard (window 20, margin 10 dB), 1 % duty cycle (sliding 3,600 s window, drop) unless the experiment varies it.
* Airtime: simulator physical duration = max(8, 8*bytes/SF) symbols (58.4 ms at SF7, 50 B); Semtech formula 118.0 ms for 63 B PHY payload (13 B MAC overhead). E9 and `standard_semtechPHY` use the Semtech duration physically.
* Energy: radio only, TX 44 mA x 3.7 V x airtime; RX 15 mA x 2 ms x 2 per uplink.
* Full list of effective values and model findings: see `revision_results_for_github/config/CONFIG_AUDIT.md` (also copied here as `CONFIG_AUDIT.md`).
