# 2. Throughput Analysis with Fixed PRB Assignment for Multiple UEs (Uplink)

## Objective
Assign different static PRBs (in the **uplink** scheduler) to two UEs connected to
the same gNB, and analyze the results.

## Background — OAI's default uplink scheduler

The scheduler decides: which UEs transmit each slot, how many RBs each gets, which
symbols, what MCS, and power control parameters. The main loop in
`gNB_scheduler_ulsch.c` (`openair2/LAYER2/NR_MAC_gNB/gNB_scheduler_ulsch.c`)
implements **Proportional Fair (PF)**:

1. **UE selection**: identify eligible UEs
2. **Coefficient calculation**: `coeff_ue = tbs / UE->ul_thr_ue` (TBS ÷ historical throughput)
3. **Sorting**: order UEs by coefficient (highest priority first)
4. **Resource allocation**: allocate to highest-priority UEs first

PF balances throughput maximization against fairness: a UE with a great channel
right now, or one that hasn't been served recently (low historical throughput), gets
a high coefficient. The algorithm is self-adjusting — being scheduled lowers the
coefficient (via rising `ul_thr_ue`), being starved raises it.

### PRB calculation process (default/dynamic path)
1. **Find available resources**: locate first free RB in the BWP, build a symbol
   bitmap, count contiguous free RBs, cap by `max_rbSize`
2. **Power constraint adjustment**: `nr_ue_max_mcs_min_rb()` reduces RB count if the
   UE's Power Headroom Report indicates insufficient power
3. **Optimal RB calculation** via `nr_find_nb_rb()`: starts from minimum RBs,
   iteratively increases until TBS ≥ buffer size or max available RBs reached — finds
   the *minimum* RBs that carry all queued data (efficiency-first, not fixed)

## Modified uplink scheduler — static PRB allocation

File: `modified_uplink_scheduler.c`. Adds static PRB allocation for specific UEs
alongside the original dynamic algorithm.

**Static PRB configuration struct:**
```c
typedef struct {
  uint64_t ue_id;          /* CU-UE-ID if available, otherwise RNTI */
  int static_prb_start;    /* Starting PRB index */
  int static_prb_size;     /* Number of PRBs allocated */
  bool enabled;             /* Whether this configuration is active */
} static_prb_config_t;

/* Static PRB configurations for two UEs */
static static_prb_config_t static_prb_configs[] = {
  {0, 0, 25, true},   /* UE 1: PRBs 0-24 (25 PRBs) */
  {0, 26, 55, true},  /* UE 2: PRBs 26-54 (55 PRBs) */
  {0, 0, 0, false}    /* End marker */
};
```
- UE 1: 25 PRBs starting at PRB 0
- UE 2: 55 PRBs starting at PRB 26
- **Lazy binding**: UEs are bound to a configuration when they first connect (`ue_id == 0` means unassigned)

**`get_static_prb_config()`** — binds a UE to a static slot by RNTI, on first
connection, and remembers the binding thereafter:
```c
static static_prb_config_t* get_static_prb_config(const NR_UE_info_t *UE) {
  uint64_t key = (uint64_t)UE->rnti;
  for (int i = 0; static_prb_configs[i].enabled; i++)
    if (static_prb_configs[i].ue_id == key)
      return &static_prb_configs[i];
  for (int i = 0; static_prb_configs[i].enabled; i++) {
    if (static_prb_configs[i].ue_id == 0) {
      static_prb_configs[i].ue_id = key;
      LOG_I(NR_MAC, "Binding UE-ID 0x%lx to static PRBs start=%d size=%d (RNTI 0x%04x)\n",
            (unsigned long)key, static_prb_configs[i].static_prb_start,
            static_prb_configs[i].static_prb_size, UE->rnti);
      return &static_prb_configs[i];
    }
  }
  return NULL; /* No static slot available */
}
```
Persistence: once bound, the UE keeps its static allocation for the connection's
lifetime.

**PRB allocation decision** (inside `pf_ul()`):
```c
static_prb_config_t* static_config = get_static_prb_config(iterator->UE);
if (static_config) {
  rbStart = static_config->static_prb_start;
  uint16_t available_rb = static_config->static_prb_size;
  bool static_prbs_available = true;
  for (int rb = 0; rb < available_rb; rb++) {
    if (rballoc_mask[rbStart + bi.bwpStart + rb] & slbitmap) {
      static_prbs_available = false;
      break;
    }
  }
  if (!static_prbs_available) {
    /* Skip this UE - its static PRBs are occupied */
    continue;
  }
} else {
  /* Use dynamic PRB allocation (original logic) */
  while (rbStart < bi.bwpSize && (rballoc_mask[rbStart + bi.bwpStart] & slbitmap))
    rbStart++;
}
```
If a UE has a static config, the scheduler tries to reserve exactly that block —
using it if free, skipping the UE entirely this TTI if occupied by overlap. No
static config → falls back to default dynamic PF allocation.

**RB size forcing** — bypasses `nr_find_nb_rb()` (which picks the minimum RBs
needed, often just 5) for statically-bound UEs:
```c
if (static_config) {
  sched.rbSize = static_config->static_prb_size;
  sched.tb_size = nr_compute_tbs(sched.Qm, sched.R, sched.rbSize,
                                  sched.tda_info.nrOfSymbols,
                                  sched.dmrs_info.N_PRB_DMRS * sched.dmrs_info.num_dmrs_symb,
                                  0, 0, sched.nrOfLayers) >> 3;
} else if (!iterator->sched_inactive) {
  nr_find_nb_rb(...);  /* dynamic — optimal RB size */
} else {
  sched.rbSize = min_rb;  /* inactive UE — minimum RBs */
  sched.tb_size = nr_compute_tbs(...);
}
```

**Resource marking** — respects static allocation boundaries so PRBs are locked in
correctly and no overlap occurs:
```c
if (static_config) {
  int static_start = static_config->static_prb_start;
  int static_size = static_config->static_prb_size;
  for (int rb = 0; rb < static_size; rb++)
    rballoc_mask[static_start + bi.bwpStart + rb] |= slbitmap;
} else {
  for (int rb = bi.bwpStart; rb < sched.rbSize; rb++)
    rballoc_mask[rb + sched.rbStart] |= slbitmap;
}
```

## Build after changes
```bash
cd ~/openairinterface5g/cmake_targets
./build_oai -w USRP --ninja --nrUE --gNB --build-lib "nrscope" -C
```


**NOTE:-**
After every algorithm change ( .c file or .h file or etc..) except config files, we have to rebuild for every changes.
After build, run the core and gnb as always.



## Results

- **Dynamic PRBs, two UEs, one gNB**: baseline speedtest at server and both clients
- **Static PRBs (UE1 = 25 PRBs, UE2 = 55 PRBs)**: speedtest at server and both clients

<img width="1328" height="357" alt="image" src="https://github.com/user-attachments/assets/fb26eeeb-af89-4d14-a3f6-af6fd974f9bc" />


**Conclusion**: static PRB allocation in the OAI uplink scheduler effectively isolates
UE bandwidth and improves throughput, achieving up to **70 Mbps**. Compared to
dynamic scheduling, this approach provides predictable performance and is well-suited
for controlled multi-UE scenarios.


## Updated Scheduler

[gNB_scheduler_ulsch.c](../02-fixed-prb-multiple-ues/gNB_scheduler_ulsch.c)
