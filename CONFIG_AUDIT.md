# Configuration audit and model findings (read before using the revision numbers)

Original projects are untouched. Everything here lives in `LoRaWANProject_S_Revision/`.

## Decisions taken with the author
* Mean packet interval stays **5 s**, with random phase: random start offset in [0, 5] s and per-packet jitter U(0.5, 1.5) x 5 s (no synchronised sending). The 1 % duty-cycle limiter stays ON in the baseline (`dutyCycleMode="current"`).
* **One gateway only.** Extension limited to **1, 2 and 8 uplink channels** on that single gateway (E6), on a reduced N grid.

## What the original runs actually used (from the saved `.sca` of `5mps50`)
| Item | Paper/prompt text | Effective in the runs |
|---|---|---|
| Packet interval | 5 s | 5 s (first-match-wins ini line); `network_summary.csv` metadata says 10 s because later, shadowed ini lines (`interPacketTime=10s`, `totalPayloadSize=1000`) are stored in the file header |
| ACK timeout | 2 s + 0-1 s jitter | **5.0 s** + U(0,1) s (`retransmissionTimeout=5.0s`) |
| Max transmissions | 8 | `maxRetries=8` -> **1 + 8 = 9** transmissions |
| Area / gateway | 2 km, centre | 2 km, GW (1000,1000) in the 5mps runs; other scenario folders used other areas |
| "Unconfirmed" project | no retransmission | its ini also sets `maxRetries=8` (retransmits like confirmed mode) and the gateway ACKs every uplink |

The revision baseline keeps the **effective** values (5 s timeout, 8 retries) so results stay comparable to the paper. Change them with `--**.node[*].retransmissionTimeout=2s --**.node[*].maxRetries=7` if the text is to be matched instead.
Each run writes `effective_config.txt` (all module parameters + counters), so every number can be traced.

## Findings about the original model (affect the manuscript text)
1. **ACK counts (~3,190 for 50/100/200 nodes)**: not an unconfirmed-mode artefact only. The gateway ACKed every uplink in both modes, and ACK throughput saturates (about 6-7 successful uplinks/s on one 125 kHz channel). Revision: unconfirmed uplinks get **no** ACK and **no** retransmission; the delivery counter is gateway-side unique `(node, seq)`.
2. **Mobility never reached the PER model.** `LoRaRadio::getNodeSpeed()` looked for a NED parameter `speed` that does not exist, so mobility PER and Doppler were always 0. Speed effects in the paper therefore come only from changing distances. Revision: `useMobilityPER=false` (original) vs `true` (RWP speed feeds the mobility/Doppler PER) - both run in E7.
3. **Fading already existed**: per-packet independent log-normal shadowing (6-10 dB) plus Rayleigh. E7 compares `full`, `none`, `shadowing_only`, `rayleigh_only`.
4. **Collision rule asymmetry**: only the later-arriving packet is judged; the earlier one is never marked lost (`symmetricMarking`). Cross-SF rule kills the incoming packet when |dP| > 14 dB in either direction (even if it is the stronger) - `perfectCrossSf` removes this.
5. **Airtime**: original ToA = max(8, 8*bytes/SF) symbols (58 ms at SF7/50 B) vs Semtech formula 118 ms (63 B PHY payload). Physical duration stays original (`airtimeModel="legacy"`); DC accounting in modes `standard`/`deferral` uses Semtech airtime. `E4/standard_semtechPHY` also switches the physical duration.
6. **Density penalty**: RSSI is reduced by a function of numNodes/area (`densityInterference`), an extra N-dependent effect on top of collisions.
7. DC limiter in the original is a sliding 3,600 s window with a 36 s budget and **drops** packets (incl. retransmissions) - it can only bind in runs >= 3,600 s or heavy retransmission (E4).
8. TX energy: airtime x 44 mA x 3.7 V (current independent of TX power). Both legacy-airtime and Semtech-airtime versions are reported.

## Revision-only code switches (all default to the original behaviour)
`captureEnabled, symmetricMarking, perfectCrossSf, crossSfIsolation, dutyCycleMode(off|current|standard|deferral), dcQueueLimit, airtimeModel, numChannels, densityInterference, fadingMode, idealPhy, useMobilityPER, txCurrentmA` (LoRaRadio); `confirmedMode, trafficModel(periodic|poisson|burst), trafficStopTime, burst*` (LoRaNode); `adr.adrEnabled` (ADR); `mobility.updateStep` (speed only); `perSecondLogging`.

## Metric definitions
* `pdr` = packets received at the gateway / packets generated (per original packet, gateway based). Packets whose retry window had not ended at run end count as lost; `pdr_settled` excludes them.
* Latency = generation -> first gateway reception (`latency_*`) and generation -> ACK reception (`latency_ack_*`).
* Loss causes are per transmission attempt in `attempts.csv.gz`.

## Known limitations
* E9 reproduces the analytic pure-ALOHA curve PDR = exp(-2G) (Bor et al. 2016 / Croce et al. 2020 model), not published testbed measurements. No hardware.
* E6: downlink/ACK capacity and gateway demodulator limits are not modelled; ACKs never collide. No multi-gateway runs (author decision). No enhanced (slotted) MAC.
* E2: intervals 5, 10 and 30 s (120 s and 300 s would need more than 2,500 nodes in this simulator).
