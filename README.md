# OAI MAC Scheduler — PRB & MCS Allocation Experiments

Four experiments modifying and evaluating the OAI 5G SA MAC scheduler's PRB
(Physical Resource Block) and MCS (Modulation and Coding Scheme) allocation
behavior — from basic fixed/dynamic MCS-PRB configuration, through static
per-UE PRB partitioning, time-varying PRB patterns, to a full dynamic PRB-sharing
mechanism with hysteresis and threshold-based reallocation between two UEs.

## Contents

| # | Doc | What it covers |
|---|-----|-----------------|
| 1 | [Fixed & Dynamic MCS/PRB Configurations](docs/01-fixed-dynamic-mcs-prb.md) | Baseline scheduler behavior, MAC log interpretation, `dl_min_mcs`/`min_grant_prb` config, throughput across MCS 10/20/28 and PRB sweeps |
| 2 | [Fixed PRB Assignment for Multiple UEs](docs/02-fixed-prb-multiple-ues.md) | Static per-UE PRB windows in the **uplink** scheduler (25 PRBs / 55 PRBs), up to 70 Mbps |
| 3 | [Time-Varying PRB Assignment](docs/03-time-varying-prb.md) | Per-UE PRB allocation that changes on a 10s/20s cycle (20↔40 PRBs), ~30 Mbps observed |
| 4 | [Static & Adaptive PRB Sharing (Downlink)](docs/04-static-adaptive-prb-sharing.md) | Full downlink implementation: static 25/55 split, buffer-threshold-triggered PRB sharing with hysteresis, phantom-UE filtering, dual-share-mode variant |

## Key results across all four

| Experiment | Result |
|---|---|
| Fixed/Dynamic MCS & PRB | Moderate MCS (10) outperformed max MCS (28); dynamic PRB allocation beat static allocation at the same PRB count |
| Fixed PRB, multi-UE (UL) | Static 25/55 PRB split achieved up to **70 Mbps**, more predictable than dynamic PF scheduling |
| Time-varying PRB | 20↔40 PRB alternating pattern (10s cycles) sustained **~30 Mbps** at both UEs |
| Static + adaptive sharing (DL) | No-sharing: 20/39 Mbps or 29/12 Mbps depending on PRB split; with sharing active: 15/29 Mbps as the congested UE's window grew from 55→68 PRBs |

## Common testbed

- OAI 5G SA gNB (`nr-softmodem`), USRP B210, Band 78
- Modified files live in `openair2/LAYER2/NR_MAC_gNB/` (`gNB_scheduler_dlsch.c`,
  `gNB_scheduler_ulsch.c`, `nr_mac_gNB.h`, `main.c`)
- Rebuild after any scheduler change:
  ```bash
  cd ~/openairinterface5g/cmake_targets
  ./build_oai -w USRP --ninja --nrUE --gNB --build-lib "nrscope" -C
  ```

## References
- [OAI MAC usage docs](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/MAC/mac-usage.md)
