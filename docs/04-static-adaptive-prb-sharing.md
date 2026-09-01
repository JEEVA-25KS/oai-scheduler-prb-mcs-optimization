# 4. Static and Adaptive PRB Allocation for Downlink Bandwidth Sharing

## Objective
Assign different static PRBs (in the **downlink** scheduler) to two UEs on the same
gNB, and dynamically share bandwidth between them based on traffic/buffer size.

## Setup summary
- UE 1 assigned 25 PRBs, UE 2 assigned 55 PRBs, in the downlink scheduler
- A DL buffer threshold is set; when either UE's DL buffer exceeds it, the other UE
  shares half of its PRBs with that UE

---

## 4.1 Static PRB assignment (downlink)

- File: `gNB_scheduler_dlsch_static_assignment.c` [gNB_scheduler_dlsch_static_assignment.c](../04-static-adaptive-prb-sharing/gNB_scheduler_dlsch_static_assignment.c)
- File path: `openair2/LAYER2/NR_MAC_gNB/gNB_scheduler_dlsch.c`



### Stable UE identity via CU-UE-ID

**Added headers** (for CU-UE-ID access via F1):
```c
#include "common/utils/nr/nr_common.h"
#include "NR_MAC_COMMON/nr_mac.h"
#include "NR_MAC_gNB/nr_mac_gNB.h"
#include "LAYER2/NR_MAC_gNB/mac_proto.h"
#include "openair2/LAYER2/nr_rlc/nr_rlc_oai_api.h"
#include "openair2/F1AP/f1ap_ids.h"
```
Needed because PF scheduling order changes slot-by-slot, so "UE 0 in the PF list" is
not a stable identity — CU-UE-ID (which doesn't change across RNTI reassignment
during RA/reconfig) is.

**Stable index helper:**
```c
/* Returns 0 for UE1 (25 PRBs), 1 for UE2 (55 PRBs), -1 for others.
 * Uses CU-UE-ID when available, else connected_ue_list index. */
static int get_fixed_dl_prb_ue_index(NR_UE_info_t *UE, NR_UE_info_t **UE_list)
{
  if (du_exists_f1_ue_data(UE->rnti)) {
    int cu_id = du_get_f1_ue_data(UE->rnti).secondary_ue;
    if (cu_id == 1) return 0;
    if (cu_id == 2) return 1;
    return -1;
  }
  for (int i = 0; i < MAX_MOBILES_PER_GNB + 1 && UE_list[i] != NULL; i++) {
    if (UE_list[i] == UE)
      return (i < 2) ? i : -1;
  }
  return -1;
}
```

### Fixed PRB selection for new DL transmissions

```c
int num_conn = 0;
for (int i = 0; i < MAX_MOBILES_PER_GNB + 1 && UE_list[i] != NULL; i++)
  num_conn++;
bool fixed_two_ue_mode = (num_conn >= 2) && (bwp_info.bwpSize >= 80);
int fixed_ue_idx = get_fixed_dl_prb_ue_index(iterator->UE, UE_list);

if (fixed_two_ue_mode && (fixed_ue_idx == 0 || fixed_ue_idx == 1)) {
  if (fixed_ue_idx == 0) {
    rbStart = 0; max_rbSize = 25;
  } else {
    rbStart = 25; max_rbSize = 55;
  }
  rbStop = rbStart + max_rbSize - 1;
} else {
  // Default dynamic frequency-domain allocation
  while (rbStart < rbStop && (rballoc_mask[rbStart + bwp_start] & slbitmap))
    rbStart++;
  while (rbStart + max_rbSize <= rbStop && !(rballoc_mask[rbStart + max_rbSize + bwp_start] & slbitmap))
    max_rbSize++;
}
```
Enforces UE1: PRBs 0–24, UE2: PRBs 25–79, instead of "first free."

### Forcing exact RB size (bypassing `nr_find_nb_rb`)

`nr_find_nb_rb()` normally picks the *minimum* RBs needed (often just 5) — bypassed
for fixed-mode UEs so the allocation stays exactly 25/55 regardless of payload size:
```c
if (fixed_two_ue_mode && (fixed_ue_idx == 0 || fixed_ue_idx == 1)) {
  sched_pdsch.rbSize = max_rbSize;
  sched_pdsch.tb_size = nr_compute_tbs(sched_pdsch.Qm, sched_pdsch.R, sched_pdsch.rbSize,
                                        tda_info.nrOfSymbols,
                                        sched_pdsch.dmrs_parms.N_PRB_DMRS * sched_pdsch.dmrs_parms.N_DMRS_SLOT,
                                        0, 0, sched_pdsch.nrOfLayers) >> 3;
} else {
  nr_find_nb_rb(sched_pdsch.Qm, sched_pdsch.R, 1, sched_pdsch.nrOfLayers,
                tda_info.nrOfSymbols,
                sched_pdsch.dmrs_parms.N_PRB_DMRS * sched_pdsch.dmrs_parms.N_DMRS_SLOT,
                sched_ctrl->num_total_bytes + oh, min_rbSize, max_rbSize,
                &sched_pdsch.tb_size, &sched_pdsch.rbSize);
}
```

### Applying fixed logic to retransmissions

Function signature extended to receive the UE list, and the retx RB search window
constrained to the UE's fixed range:
```c
int fixed_ue_idx = (num_conn >= 2 && bwp_info.bwpSize >= 80) ?
                    get_fixed_dl_prb_ue_index(UE, UE_list) : -1;
if (fixed_ue_idx == 0) {
  rbStart = bwp_info.bwpStart;
  rbStop = bwp_info.bwpStart + 24;
} else if (fixed_ue_idx == 1) {
  rbStart = bwp_info.bwpStart + 25;
  rbStop = bwp_info.bwpStart + 79;
}
```
Prevents retransmissions from drifting into the other UE's PRB region.

### DL NPRB logging

- File: `nr_mac_gNB.h`[nr_mac_gNB.h](../04-static-adaptive-prb-sharing/nr_mac_gNB.h)  — added a dedicated `DL_NPRB` field (UL already used `NPRB`, so DL
needed its own to avoid clobbering):
- File path: openair2/LAYER2/NR_MAC_gNB/nr_mac_gNB.h

```c
/* UL NPRB for legacy logging, DL NPRB for new DL logging */
int NPRB;
int DL_NPRB;
```

- File: `main.c`[main.c](../04-static-adaptive-prb-sharing/main.c) 
- File path: openair2/LAYER2/NR_MAC_gNB/main.c

`main.c` — prints it in the periodic DL stats line:
```c
", dlsch_errors %"PRIu64", pucch0_DTX %d, BLER %.5f MCS (%d) %d NPRB %d CCE fail %d\n",
stats->dl.errors, stats->pucch0_DTX, sched_ctrl->dl_bler_stats.bler,
UE->current_DL_BWP.mcsTableIdx, sched_ctrl->dl_bler_stats.mcs,
UE->mac_stats.DL_NPRB, sched_ctrl->dl_cce_fail);
```

### Why PRBs look static even with a small buffer

Because fixed mode forces `rbSize` (25/55) rather than deriving it from queued
bytes. `nr_compute_tbs()` computes a TBS matching that RB size; if RLC has fewer
bytes than that, MAC pads the rest. Padding is carried using the Padding LCID
(`DL_SCH_LCID_PADDING`) with zero-filled bytes — the UE discards it:
```c
// Add padding header and zero rest out if there is space left
if (bufEnd-buf > 0) {
  NR_MAC_SUBHEADER_FIXED *padding = (NR_MAC_SUBHEADER_FIXED *) buf;
  padding->R = 0;
  padding->LCID = DL_SCH_LCID_PADDING;
  buf += 1;
  memset(buf,0,bufEnd-buf);
  buf=bufEnd;
}
```
**Trade-off**: PRBs stay fixed and deterministic (as intended), at the cost of wasted
capacity via padding when payload is smaller than the fixed allocation.

The same static-RB-size forcing was applied symmetrically in `gNB_scheduler_ulsch.c`
for the uplink side (CU-UE-ID-based mapping, same 25/55 split, retx constrained to
the fixed window, and a fix to a UL retx PRB-marking loop that was iterating the
wrong index bounds).

---

## 4.2 PRB sharing between UEs

- File: `gNB_scheduler_dlsch_sharing_prb.c` [gNB_scheduler_dlsch_sharing_prb.c](../04-static-adaptive-prb-sharing/gNB_scheduler_dlsch_sharing_prb.c)
- File path: `openair2/LAYER2/NR_MAC_gNB/gNB_scheduler_dlsch.c`


### Threshold / hysteresis state machine

```c
static int last_periodic_log_frame = -1;
static int last_share_active = 0;
const uint32_t share_on_thr = 20000;
const uint32_t share_off_thr = (20000 * 9) / 10;  // 18000
static int share_mode = 0;
static int last_buf_update_frame = -1;
static int last_buf_update_slot = -1;
```

- `share_on_thr` = **20,000 bytes** — triggers sharing when a UE's buffer exceeds this
- `share_off_thr` = **18,000 bytes** (90% of on-threshold) — hysteresis, prevents rapid flapping
- `share_mode`: `0` = inactive (default 25/55), `1` = UE1 congested (UE2 donates), `2` = UE2 congested (UE1 donates)

**State transitions:**
```c
if (share_mode == 0) {
  if (buf_ue1 > share_on_thr && buf_ue2 < share_on_thr) share_mode = 1;
  else if (buf_ue2 > share_on_thr && buf_ue1 < share_on_thr) share_mode = 2;
} else if (share_mode == 1) {
  if (buf_ue1 < share_off_thr || buf_ue2 >= share_on_thr) share_mode = 0;
} else if (share_mode == 2) {
  if (buf_ue2 < share_off_thr || buf_ue1 >= share_on_thr) share_mode = 0;
}
```
Sharing only activates when **exactly one** UE is above threshold — if both are
congested, it stays/returns to mode 0 (default split). There's no direct mode 1 ↔
mode 2 jump; it always transitions through mode 0 first.

### Buffer refresh (once per slot)

Previously UE2's buffer often showed 0B because it wasn't refreshed unless it was
being scheduled that slot. Fixed by refreshing both UEs' buffers once per slot,
gated by a frame+slot stamp to avoid redundant RLC queries:
```c
if (last_buf_update_frame != frame || last_buf_update_slot != slot) {
  for (int i = 0; i < MAX_MOBILES_PER_GNB + 1 && UE_list[i] != NULL; i++) {
    NR_UE_info_t *u = UE_list[i];
    int idx = get_fixed_dl_prb_ue_index(u, UE_list);
    if (idx == 0 || idx == 1) update_dlsch_buffer(frame, slot, u);
  }
  last_buf_update_frame = frame;
  last_buf_update_slot = slot;
}
```

### PRB reallocation math

Baseline: UE1 = 25 PRBs (0–24), UE2 = 55 PRBs (25–79).
```c
int size_ue1 = 25;
int size_ue2 = 55;
if (share_mode == 1) {
  int ue2_keep = 55 / 2;       // 27
  size_ue2 = ue2_keep;
  size_ue1 = 80 - ue2_keep;    // 53
} else if (share_mode == 2) {
  int ue1_keep = 25 / 2;       // 12
  size_ue1 = ue1_keep;
  size_ue2 = 80 - ue1_keep;    // 68
}
```

| Mode | Condition | UE1 start | UE1 size | UE2 start | UE2 size |
|------|-----------|-----------|----------|-----------|----------|
| 0 (inactive) | both buffers normal | 0 | 25 | 25 | 55 |
| 1 (UE1 high) | buf_ue1 > 20000, buf_ue2 < 20000 | 0 | 53 | 53 | 27 |
| 2 (UE2 high) | buf_ue2 > 20000, buf_ue1 < 20000 | 0 | 12 | 12 | 68 |

UE1's `rbStart` is always 0; UE2's `rbStart` always equals `size_ue1`, so allocations
stay contiguous within PRBs 0–79.

### Applying the split
```c
if (fixed_ue_idx == 0) { rbStart = 0; max_rbSize = size_ue1; }
else { rbStart = size_ue1; max_rbSize = size_ue2; }
rbStop = rbStart + max_rbSize - 1;
```

### Throttled logging
```c
LOG_I(NR_MAC, "[DL PRB SHARE] %4d.%2d thr_on=%uB thr_off=%uB buf(UE1=%uB UE2=%uB) split(UE1=%d UE2=%d) mode=%d %s\n",
      frame, slot, share_on_thr, share_off_thr, buf_ue1, buf_ue2, size_ue1, size_ue2,
      share_mode, share_active ? "ACTIVE" : "INACTIVE");
```
Printed periodically and on state change — avoids log flooding while making sharing
behavior visible. **Note on naming**: the log's "UE1"/"UE2" refer to CU-UE-ID 1/2
(the human-readable F1 identifier), while the code's `fixed_ue_idx` is the 0-based
internal index (CU-UE-ID 1 → idx 0, CU-UE-ID 2 → idx 1).

---

## 4.3 The phantom third-UE problem

The gNB occasionally sensed a "phantom" UE not actually in proximity. This third UE
broke the two-UE static/sharing logic (which only activates for exactly 2 UEs) and
caused the scheduler to fall back to plain PF.

**Fix — filter out UEID ≥ 3:**
```c
static int get_ueid_for_dl_filter(const NR_UE_info_t *UE, NR_UE_info_t **UE_list)
{
  if (du_exists_f1_ue_data(UE->rnti))
    return du_get_f1_ue_data(UE->rnti).secondary_ue;
  for (int i = 0; i < MAX_MOBILES_PER_GNB + 1 && UE_list[i] != NULL; i++)
    if (UE_list[i] == UE) return i;
  return -1;
}

/* Do not schedule/serve UEs with UEID >= 3 */
const int ueid = get_ueid_for_dl_filter(UE, UE_list);
if (ueid >= 3)
  continue;
```
Applied in `pf_dl()`'s UE loop, right after the `nr_mac_ue_is_active(UE)` check. UEs
with UEID ≥ 3 are excluded from both new DL transmissions and DL retransmissions.
The same filter was added to the static-assignment file.

**Later hardening** — centralized the threshold into a single constant so it can't be
accidentally regressed, and applied it in both `pf_dl()` and the MAC stats dump:
```c
// gNB_scheduler_dlsch.c
#define NR_DL_SCHED_MAX_UEID 2
...
const int ueid = get_ueid_for_dl_filter(UE, UE_list);
if (ueid > NR_DL_SCHED_MAX_UEID)
  continue;

// main.c
#define NR_SCHED_MAX_UEID 2
static int get_ueid_for_sched_filter(const gNB_MAC_INST *gNB, const NR_UE_info_t *UE) {
  if (du_exists_f1_ue_data(UE->rnti))
    return du_get_f1_ue_data(UE->rnti).secondary_ue;
  for (int i = 0; i < MAX_MOBILES_PER_GNB + 1 && gNB->UE_info.connected_ue_list[i] != NULL; i++)
    if (gNB->UE_info.connected_ue_list[i] == UE) return i;
  return -1;
}
...
UE_iterator(gNB->UE_info.connected_ue_list, UE) {
  const int ueid = get_ueid_for_sched_filter(gNB, UE);
  if (ueid > NR_SCHED_MAX_UEID)
    continue;
```
This also hides UEID ≥ 3 from the periodic MAC stats dump, in addition to blocking
it from scheduling.

---

## 4.4 Dual-share-mode variant (never return to inactive)

New requirement: once any UE exceeds the threshold, switch to that UE's share mode
and **never return to mode 0**. The original hysteresis-based state machine (§4.2)
could return to mode 0 via `share_off_thr`, so it couldn't satisfy this.

- File: `gNB_scheduler_dlsch_2ue_prb_sharing_mode_1_2.c` [gNB_scheduler_dlsch_2ue_prb_sharing_mode_1_2.c](../04-static-adaptive-prb-sharing/gNB_scheduler_dlsch_2ue_prb_sharing_mode_1_2.c)
- File path: `openair2/LAYER2/NR_MAC_gNB/gNB_scheduler_dlsch.c`


logic:
```c
/* update sharing mode:
 * - enter share mode if any UE exceeds share_on_thr
 * - once in share mode, do not return to 0; allow switching between 1/2
 *   when the other UE becomes the larger-buffer UE */
if (buf_ue1 > share_on_thr || buf_ue2 > share_on_thr) {
  if (buf_ue1 >= buf_ue2)
    share_mode = 1;
  else
    share_mode = 2;
}
```
- If either buffer exceeds the threshold, mode is forced to whichever UE currently
  has the larger buffer
- No code path sets `share_mode` back to 0
- It **can** still switch between mode 1 and 2 — every scheduling pass recomputes
  `buf_ue1`/`buf_ue2` and reassigns `share_mode` based on which is currently larger,
  so a later slot where the buffer relationship flips will silently switch modes;
  this isn't a separate transition rule, just the same comparison being re-evaluated
  each time

---

## 4.5 Results

**Under no sharing** (static PRB assignment, sharing disabled):

| Case | UE | PRBs (DL) | Downlink speed |
|------|-----|-----------|----------------|
| i | UE 1 | 25 | 20 Mbps |
| i | UE 2 | 55 | 39 Mbps |
| ii | UE 1 | 55 | 29 Mbps |
| ii | UE 2 | 25 | 12 Mbps |

<img width="1239" height="653" alt="image" src="https://github.com/user-attachments/assets/89572737-d0e7-43c2-96fd-9f80beab87c9" />

**With sharing active** (UE2 congested, mode 2 — UE1 donates half its PRBs):

| UE | PRBs (DL) | Downlink speed |
|-----|-----------|----------------|
| UE 1 | 25 (baseline) | 15 Mbps |
| UE 2 | 55 (baseline) | 29 Mbps |

<img width="1246" height="342" alt="image" src="https://github.com/user-attachments/assets/d075303f-e5c7-47b4-80b8-008ad7d5af59" />
<img width="1245" height="312" alt="image" src="https://github.com/user-attachments/assets/1686132b-e468-491f-b099-d6a708a7fa44" />


PRB reallocation math for this case: `ue1_keep = floor(25/2) = 12` → UE1 gets 12
PRBs, UE2 gets `80 − 12 = 68` PRBs. UE2's effective throughput share grew as its
PRB window expanded from 55 → 68.

## 4.6 Appendix 
The corresponding script/config files for each variant (static assignment, PRB
sharing with hysteresis, phantom-UE filtering, dual-share-mode) are organized under
the repo's folder for reference.
