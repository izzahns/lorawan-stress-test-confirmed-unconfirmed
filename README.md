# revision_summary - data dictionary

All tables are UTF-8 CSV with a header row, one tidy table per file, no index column. **Join key: `run_id`** (unique string).
Raw per-run logs are not needed for analysis; they stay in `revision_runs/<EXP>/raw/` on the simulation machine.
Everything was produced by `summarise_runs.py` (rerunnable on the raw logs); `finalize_summary.py` wrote `checks.csv` and `raw_sample/`.

## Folder layout
```
README.md  CONFIG_AND_BRANCH.md  checks.csv  summarise_runs.py  finalize_summary.py
raw_sample/<EXP>/<run_id>_packets.csv.gz     per-packet log of the FIRST seed of every cell (cell = variant x mode x N x speed x interval x channels x ADR x fading)
E1 ... E12/ (E5 = instrumentation, built into every run; no separate folder; E11 = retry-limit sweep, E12 = payload/airtime sweep)
  <E>_runs.csv  <E>_run_metrics.csv  <E>_loss_causes.csv  <E>_attempts_hist.csv  <E>_latency_quantiles.csv  <E>_node_level.csv
  <E>_sf_power_usage.csv  <E>_timeseries_bins.csv  <E>_distance_profile.csv  <E>_rssi_snr_hist.csv  <E>_collision_detail.csv
  <E>_duty_cycle_detail.csv  <E>_per_channel.csv  + experiment-specific files (below)
```
"Cell" = one combination of experimental factors; each cell has 10 seeds (20 in E1).

## A. `<E>_runs.csv` (one row per run)
run_id, experiment, variant, seed (OMNeT++ seed-set), N (nodes), mode (confirmed/unconfirmed), interval_s (mean packet interval), speed_mps (RWP speed; 0 = static),
n_channels, n_gateways (always 1), adr (on/off), collision_variant (capture / symmetric marking / perfect cross-SF), duty_cycle_variant (limiter mode ; airtime model used for the
physical packet duration), traffic_model, fading_mode, mobility_per, ideal_phy, traffic_window_s (time during which packets are generated), sim_time_s (traffic window + 60 s for
pending ACK/retransmissions), warmup_s (0: no warm-up discarded, see time series), gateway_xy (m), area, tx_power_dbm (initial), max_retries (retransmissions after the first transmission; 8 -> up to 9 transmissions), payload_bytes (application payload), airtime_model (legacy | semtech physical airtime),
ack_timeout_s (5 s + U(0,1) s jitter), wall_clock_s, status (`ok` or the anomaly text). All values are read back from the simulator's own `effective_config.txt`, not from the launcher.

## B. `<E>_run_metrics.csv`
packets_generated; packets_delivered_gw (unique (node,seq) received at the gateway); packets_acked (ACK received by node; 0 in unconfirmed); packets_pending_at_end (retry window not yet over);
packets_failed (all attempts used / dropped, never reached gateway); **PDR_gateway** = delivered/generated; PDR_ack = acked/generated; **PDR_excl_pending** = delivered/(generated-pending).
attempts_mean/median/p95/p99 = transmissions actually put on air per original packet (0 = every attempt dropped by the duty-cycle limiter). total_tx_attempts. goodput_bps = delivered x 50 B x 8 / traffic window.
lat_gw_* = generation -> first gateway reception; lat_ack_* = generation -> ACK reception (confirmed only); mean/median/p95/p99, seconds, over delivered (resp. acked) packets only.
tx_airtime_total_s = sum of physical on-air durations. G_nominal_97p5ms = N x 0.0975 / interval (your formula); G_nominal_sim_airtime = N x simulator SF7 airtime (58.4 ms legacy; 118.0 ms in E9) / interval;
G_measured_first_tx / G_measured_all_tx = logged first-transmission / all-transmission airtime per (traffic window x channels). channel_busy_fraction = measured union of busy time per channel (overlaps counted once).
tx_energy_J = airtime x 44 mA x 3.7 V (radio TX only; current independent of TX power); rx_energy_J = 15 mA x 2 ms x 2 windows x 3.7 V per uplink (= 0.222 mJ), applied to every transmitted uplink (class A also in unconfirmed);
*_per_delivered_J; tx_current_mA. dc_drops (limiter drops incl. queue-full), dc_deferrals (packets whose first transmission was delayed, deferral mode), collisions_total (same-SF + cross-SF).

## C. `<E>_loss_causes.csv` (run_id, level, cause, count)
level=attempt: success, collision_same_sf, collision_cross_sf, below_sensitivity (also "out of range"), snr_fail (below SNR limit), bit_error (PER draw), dc_drop, dc_queue_full; plus TOTAL_ATTEMPTS_LOGGED; the causes sum exactly to it (checked).
level=packet: gateway_delivered_but_ack_not_received (= "ACK lost", counted per packet, not per attempt: the simulator does not log downlink failures per attempt), gateway_demodulator_busy (always 0: no demodulator limit is modelled),
pending_at_run_end, failed_after_all_attempts. Note: an attempt row exists for every attempt that reached a radio decision; transmissions still on air when the run ends have no row (<=3 per run, only in E4 deferral).

## D. `<E>_attempts_hist.csv` (run_id, kind, bucket, count)
kind=tx_used: bucket 0..9 = transmissions on air per original packet (9 = 1 + 8 retries; bucket 0 = every attempt was dropped by the limiter).
kind=attempts_to_first_gw_rx: bucket 1..9 = attempt index of the first gateway reception, plus `failed` and `pending`.

## E. `<E>_latency_quantiles.csv`
For first_attempt_delay_s (first TX - generation; > 0 only with deferral), time_to_gw_rx_s, time_to_ack_s: quantiles q1,q5,q10,q25,q50,q75,q90,q95,q99, mean, max, n (seconds).

## F. `<E>_node_level.csv` (one row per node per run)
node_id, packets_generated/delivered/acked, mean_attempts, tx_airtime_s, mean_distance_m (mean over its transmitted attempts, i.e. along its path), mean_rssi_dbm / mean_snr_db (gateway-side, attempts that reached the gateway receiver),
tx_power_mean/min/max_dbm (ADR-set), frac_attempts_sf7..12 (share of attempts at each SF; used as proxy for fraction of time), dominant_sf, max_airtime_in_3600s_window_s (sliding window), dc_usage_fraction_of_1pct_budget (= /36 s),
tx_energy_J, dc_drops. **Not available:** mean x/y position (positions were not logged); use mean_distance_m.

## G. `<E>_sf_power_usage.csv`
sfX_packet_share (share of transmitted attempts), sfX_airtime_share, tp_<n>dBm_count (1 dB bins 2..14), n_adr_sf_changes / n_adr_power_changes (changes between consecutive transmissions of the same node).

## H. `<E>_timeseries_bins.csv`
Bins of 50 s (100 s if sim > 3,600 s): packets_generated, packets_delivered (by delivery time), collisions, dc_drops, tx_attempts, mean_attempts_per_generated_packet, channel_busy_fraction_approx (airtime assigned to start bin).

## I. `<E>_distance_profile.csv`, `<E>_rssi_snr_hist.csv`
distance_profile: per variant, mode, N, 100 m bin (distance of the packet = mean over its attempts), pooled over seeds: packets, delivered, PDR, mean_attempts, mean_rssi_dbm, mean_snr_db (of attempts reaching the gateway), runs.
rssi_snr_hist: metric (rssi_dbm / snr_db), outcome (delivered/lost attempt), 2 dB bin_low, count; pooled over seeds per variant, mode, N.

## J. `<E>_collision_detail.csv`
Reconstructed offline from the attempt log (pairs of transmitted attempts that overlap in time on the same channel and reach the gateway): per run and SF pair "a-b": n_overlaps, n_both_lost,
n_one_survives_capture, n_both_ok, mean_overlap_fraction (overlap / shorter packet). Histogram rows (same file, dp_* columns filled): |RSSI difference| in 2 dB bins (>= 40 dB in one bin).
Chains of 3+ overlapping packets are counted pairwise.

## K. `<E>_duty_cycle_detail.csv` (every experiment; one row per node-run that transmitted)
Distribution of the sliding 3,600 s airtime window evaluated at each of the node's transmissions: window_airtime_q1..q99_s, window_airtime_max_s, n_windows_above_36s, time_first_dc_drop_s,
n_deferred_packets, deferral_delay_mean_s / p95_s. In `current`/`standard` modes the limiter caps the window at 36 s; `off` shows the airtime the node would need.

## Experiment-specific files
E1: `E1_seed_curves.csv` (seed, mode, N, PDR), `E1_N90_per_seed.csv` (N90_linear = linear interpolation of the seed's PDR(N) curve; N90_logistic = 2-parameter logistic fit, NaN when the fit falls outside the simulated N range, with convergence flag; unconfirmed has no N90 because PDR < 0.9 already at the smallest N).
E2: `E2_N90_per_interval.csv` (N90 of the seed-mean curve per interval and mode, measured G at N90, N grid limits; the grid is capped at N <= 200, so long intervals have no bracketed N90).
E3, E4: `E<x>_paired_vs_current.csv` (same N, seed, mode; delta_pdr_vs_current).
E6: `E6_per_channel.csv` (per run and channel: attempts, airtime, offered_load, busy_fraction, collisions, successes), `E6_capacity_grid.csv` (channels x gateways(=1) x mode: N90 of the mean curve, seed min/max/mean, note when the grid does not bracket 90%), `E6_N90_per_seed.csv`.
E7: `E7_pdr_vs_speed.csv` (with realised mean node-gateway distance), `E7_fading_model.csv` (model parameters), `E7_N90_per_seed.csv`.
E8: `E8_image_level.csv` (node, image: fragments sent/delivered, complete flag, completion time, retransmissions), `E8_image_summary.csv` (per run), `E8_burst_vs_periodic.csv` (burst vs matched periodic load, by N and mode).
E9: `E9_validation_curve.csv` (simulated PDR vs N with reference exp(-2G), absolute and relative error). Reference is the analytic pure-ALOHA (no-capture, single-SF) result, computed from the measured G, not digitised from a figure.
E10: `E10_adr_on_off.csv` (ADR on vs off by speed and N).
E11 (retry-limit sensitivity, confirmed, maxRetries in {0,1,3,5,8}, N in {40,50,55,60,70}, 10 seeds): variant = `retry<k>`; `E11_N90_per_seed.csv` (NaN when PDR never reaches 0.9 on the grid, i.e. retry0/retry1).
E12 (payload / airtime sensitivity, confirmed, payload 20/50/100 B x airtime model legacy|semtech, N in {40,50,60,70}, 10 seeds): variant = `p<bytes>_<model>`; `E12_N90_per_seed.csv` (only variants whose PDR crosses 0.9 inside the grid).

## What to look at first (unexpected / important)
1. **The 1 % duty-cycle limiter is incompatible with a 5 s interval over long runs (E4).** One SF7 packet = 58.4 ms (legacy airtime used in the simulator) -> 1.17 % duty cycle at 5 s *before any retransmission*; with true Semtech airtime (118 ms) it is 2.4 %. In 560 s runs the 36 s/hour budget never binds, but in 3,600 s runs it does: at N=50 PDR is 0.99 with the limiter off, 0.38 with the original sliding-window limiter, 0.19 with standard-airtime accounting, 0.26 with deferral (p95 latency about 90 s). Packet loss in E4 is dominated by limiter drops (>200,000 per run). Treat E4 as "what happens if the limit is enforced", not as a refinement of the E1 numbers.
2. **E8 burst traffic**: a 50-fragment train needs about 5 % duty cycle, so in confirmed mode most fragments are dropped by the limiter (bucket 0 in the attempts histogram); unconfirmed has a higher fragment PDR than confirmed at N >= 15 (0.65 vs 0.15 at N=15), but complete images are rare for both (<= 11 % at N >= 15). At matched mean load, burst traffic is worse than periodic in confirmed mode (N=10: fragment PDR 0.49 vs 0.73) and about equal in unconfirmed mode.
3. **E1**: the confirmed-mode N90 is 56.9 +/- 0.5 (seed std; linear interpolation), a logistic fit gives 52.7 (the PDR(N) curve is asymmetric, so the logistic is biased low; prefer the linear value). Seed-to-seed variation is very small. Unconfirmed PDR is already < 0.9 at the smallest N tested.
4. **E7**: removing fading *lowers* PDR (N=100: 0.12-0.18 vs 0.34-0.35) - random per-packet RSSI differences create capture; without them equal-power overlaps destroy both packets. PDR is nearly flat in speed; only with mobility PER on (`mobility_per=True`) PDR falls at 10-20 m/s (N=100: 0.354 -> 0.311 at 20 m/s). Speed does not reach the error model in the original runs.
5. **E10**: ADR on vs off makes no practical difference (only ~0.4 % of transmissions use SF > 7; SF7 dominates because the 2 km cell is within SF7 range); ADR-on is marginally worse in PDR and energy.
6. **E6**: N90 scales about linearly with channels (57 -> 113 for 1 -> 2 channels, confirmed). 8 channels was run only at N=200 (cap), PDR 0.999: capacity is above the grid.
7. **E9**: matches exp(-2G) within about 1 % for N <= 100; relative error at N >= 150 is meaningless (PDR ~ 0). N=300/400 have 9 runs (one failure was cleared and not repeated after the N <= 200 decision).
8. No failed runs, no zero-variance cells, no PDR > 1 (see checks.csv: 100 of 100 checks pass). Same seed reproduces the identical per-packet log (checked on a re-run).
9. **E11 (retry limit)**: retransmissions help only while the channel is not saturated. N=40: PDR 0.73 (0 retries) -> 0.86 (1) -> 0.97 (3) -> 0.999 (8). N=60: 0.63 / 0.74 / 0.84 / 0.87 (5 retries) / 0.83 (8 retries). N=70: 0.59 / 0.68 / 0.74 (3 retries) / 0.71 (5) / 0.62 (8): the best limit is about 3 at N >= 60, 8 retries is worse than 3. N90 = 53.3 (3 retries), 57.1 (5), 56.8 (8); 0 and 1 retries never reach 0.9. Cost: attempts per packet 1.0 -> 5.6 and TX energy per delivered packet 0.016 -> 0.086 J at N=70 (0 -> 8 retries). The baseline of 8 retries is near the capacity optimum at N <= 55 but not at higher load.
10. **E12 (payload / airtime)**: capacity is very sensitive to airtime. PDR at N=50: 20 B legacy 1.00; 50 B legacy 0.99; 100 B legacy 0.25; 50 B with the Semtech (118 ms) airtime 0.26; 20 B Semtech 0.77. The 50 B/legacy baseline N90 (55.5 here, 56.8 in E1: different seed set) drops below 40 nodes when the real LoRa airtime is used (N90 not bracketed: PDR already 0.36 at N=40). 20 B + Semtech N90 = 44. The legacy airtime formula underestimates the packet duration by about 2x for 50 B; this is the main caveat for absolute capacity numbers (relative effects hold).
11. Not available in this simulator, hence absent from the tables: x/y positions, demodulator-busy losses, per-attempt ACK loss, multi-gateway runs, a slotted-access MAC variant, RX window energy for retransmissions beyond one per uplink.
