# 1. Performance Evaluation of Fixed and Dynamic MCS & PRB Configurations

## Objective
Change the Modulation and Coding Scheme (MCS) of the gNB scheduler and the PRBs
allocated to the UE from the gNB, and analyze the end results.

## MAC scheduler operation

The 5G MAC scheduler is a **proportional fair (PF)** scheduler, approximating
wide-band CQI through MCS selection. The scheduler loops through all UEs and
calculates the PF coefficient using the currently selected MCS and the historical
achieved rate. The UE with highest coefficient wins and is scheduled RBs until all
resources are used, or until it has no more data to fill RBs — the scheduler then
continues with the UE with the second-best coefficient.

UEs with retransmissions are allocated first; similarly, UEs that have not been
scheduled for some time in UL are scheduled automatically in UL and take priority
over "normal" traffic.

## Periodic MAC log — field reference

Example log line:
```
UE RNTI 2460 CU-UE-ID 2 in-sync PH 28 dB PCMAX 24 dBm, average RSRP -74 (8 meas), average SINR 40.0 (32 meas)
UE 2460: CQI 15, RI 2, PMI (14,1)
UE 2460: UL-RI 2 TPMI 0
UE 2460: dlsch_rounds 32917/5113/1504/560, dlsch_errors 211, pucch0_DTX 1385, BLER 0.19557 MCS (1) 23 CCE fail 3
UE 2460: ulsch_rounds 3756/353/182/179, ulsch_errors 170, ulsch_DTX 285, BLER 0.33021 MCS (1) 27 (Qm 8 dB) NPRB 5 SNR 31.0 dB CCE fail 0
UE 2460: MAC: TX 1530943191 RX 194148 bytes
UE 2460: LCID 1: TX 651 RX 3031 bytes
UE 2460: LCID 4: TX 1526169592 RX 16152 bytes
```

| Field | Meaning |
|-------|---------|
| **UE RNTI** | also used as the DU UE ID over F1; each log line is prepended with the RNTI |
| **CU UE ID** | separate identifier from RNTI, for multiple UEs across DUs (RNTI conflicts possible) |
| **in-sync / out-of-sync** | whether the UE is actively being scheduled, or not (missed UL scheduling / radio-link failure) |
| **PH** | Power Headroom — power the UE has left. >40 → full UL throughput achievable |
| **PCMAX** | UE-reported max UL transmit power |
| **RSRP** | DL reference signal power at UE. >−80 dBm → full DL throughput; <−95 dBm → very limited |
| **SINR** | signal-to-interference-and-noise ratio of SSB at UE, max reportable 40.0 dB |
| **CQI** | 0–15, channel quality indicator, spectral efficiency per 38.214 table |
| **RI** | Rank indicator 1–4 (number of MIMO layers) |
| **PMI** | precoding matrix indicator — spatial direction seen by UE; roughly stationary unless UE moves |
| **UL-RI, TPMI** | same as DL, for uplink |
| **dlsch_rounds A/B/C/D** | HARQ transmissions per round. High A, low B/C, ~0 D = healthy. B≈A = frequent first-round errors (bad unless UE is far) |
| **dlsch_errors** | count where after 4 transmissions the transport block still wasn't received — should be low |
| **pucch0_DTX** | missed PUCCH format 0/1 detections (unreceived ACK/NAK) — should be small vs. A in dlsch_rounds |
| **DLSCH BLER** | moving average of B/A in dlsch_rounds; should sit near the scheduler's target BLER (typically 10–30% for high-throughput policy) |
| **MCS (Q) M** | M=0–28 current MCS; Q=table (0=64QAM, 1=256QAM, 2=low-SE URLLC table) |
| **ulsch_DTX** | missed PUSCH detections (signal below `pusch_dtx_threshold`) — indicates poor radio/power control/missed DCI |
| **ULSCH Qm X deltaMCS Y dB** | modulation order (2/4/6/8 = QPSK/16QAM/64QAM/256QAM) and power-control offset |
| **ULSCH NPRB** | current PRBs scheduled by gNB — fluctuates; under high iperf throughput, indicates PRBs the UE can actually use with its power budget |
| **ULSCH SNR** | measured UL SNR; should track `pusch_TargetSNRx10` (≈10× the shown value) |
| **CCE fail** | failed CCE (DCI) allocation attempts — high values mean the scheduler tried but couldn't allocate DCI |
| **MAC TX/RX** | total MAC PDU bytes |
| **LCID X** | MAC SDU/RLC PDU bytes for that logical channel. LCID 1/2 = SRB1/2. LCID 4+ = DRB1+ (a PDU session) |

## Relevant MAC configuration options

In the `MACRLCs` section of the gNB/DU config file (reference:
[mac-usage.md](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/MAC/mac-usage.md)):

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `dl_min_mcs` | 0 | minimum MCS for any UE |
| `dl_max_mcs` | 28 | maximum MCS for any UE |
| `ul_min_mcs` | 0 | as `dl_min_mcs`, for UL |
| `ul_max_mcs` | 28 | as `dl_max_mcs`, for UL |
| `min_grant_prb` | 5 | PRBs scheduled after activity/SR |

Setting `min`/`max` MCS to the same value makes MCS static; `min_grant_prb` to a
fixed value fixes the PRB count.

## Results & observations

Tested configurations, each with a corresponding speedtest measurement:
1. Dynamic MCS
2. MCS = 10 (fixed)
3. MCS = 20 (fixed)
4. MCS = 28 (fixed)
5. Dynamic MCS and PRBs
6. Dynamic MCS and PRBs (max data rate)
7. MCS = 10, PRBs = 10
8. MCS = 10, PRBs = 50
9. MCS = 20, PRBs = 100
10. MCS = 28, PRBs = 106

Example: **gNB log**
<img width="1329" height="232" alt="image" src="https://github.com/user-attachments/assets/801b06b4-cec4-4bdb-9a61-8d733fc43052" />


**Conclusion**: downlink performance does not always improve with higher MCS or PRB
values. Moderate MCS levels (e.g. MCS 10) provided **higher** throughput than higher
ones (MCS 28). Likewise, dynamic PRB allocation achieved higher throughput than
static allocation with the same number of PRBs. This shows optimal performance
depends on real-time channel conditions and scheduler adaptation, not simply using
maximum values.

## References
- [OAI MAC usage docs](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/MAC/mac-usage.md)
