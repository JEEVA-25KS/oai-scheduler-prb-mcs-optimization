# 3. Throughput Analysis with Time-Varying PRB Assignment for Multiple UEs (Uplink)

## Objective
Assign different static **time-varying** PRBs to two UEs connected to the same gNB,
and analyze the results.

## Time-varying uplink scheduler

Extends the [static PRB scheduler](02-fixed-prb-multiple-ues.md) with PRB
allocations that change on a schedule, rather than staying fixed for the whole
connection.

### New headers
```c
#include <time.h>
#include <math.h>
```

### New data structures

```c
typedef struct {
  double start_time;  // Start time in seconds (within 20s cycle)
  double end_time;    // End time in seconds (within 20s cycle)
  int prb_start;
  int prb_size;
} time_config_t;

typedef struct {
  uint64_t ue_id;
  time_config_t time_configs[MAX_TIME_CONFIGS];
  int num_configs;
  int current_config_idx;
  int last_cycle;
  int last_seen_cycle;
  bool enabled;
} dynamic_prb_config_t;
```
`time_config_t` defines a time interval with its PRB allocation; `dynamic_prb_config_t`
tracks a UE's allocation across time.

### Pre-configured patterns
```c
static dynamic_prb_config_t dynamic_prb_configs[] = {
  // UE 1: alternating 20 PRBs -> 40 PRBs (10s intervals)
  {
    .ue_id = 0,
    .time_configs = {
      {0.0, 10.0, 0, 20},   // 20 PRBs from 0-10s (PRBs 0-19)
      {10.0, 20.0, 0, 40},  // 40 PRBs from 10-20s (PRBs 0-39)
    },
    .num_configs = 2, .current_config_idx = 0,
    .last_cycle = -1, .last_seen_cycle = -1, .enabled = true
  },
  // UE 2: alternating 40 PRBs -> 20 PRBs (10s intervals)
  {
    .ue_id = 0,
    .time_configs = {
      {0.0, 10.0, 40, 40},   // 40 PRBs from 0-10s (PRBs 40-79)
      {10.0, 20.0, 40, 20},  // 20 PRBs from 10-20s (PRBs 40-59)
    },
    .num_configs = 2, .current_config_idx = 0,
    .last_cycle = -1, .last_seen_cycle = -1, .enabled = true
  },
  {.ue_id = 0, .num_configs = 0, .enabled = false}  // Terminator
};
```
- **Pattern 1** (UE1): 20 PRBs (0–10s) → 40 PRBs (10–20s) → repeats
- **Pattern 2** (UE2): 40 PRBs (0–10s) → 20 PRBs (10–20s) → repeats

### Time management
```c
static double get_current_time_seconds(frame_t frame, slot_t slot) {
  // Numerology 1 (30kHz SCS): 1ms = 30 slots
  const double slot_duration_ms = 1.0 / 30.0;
  double absolute_time = (frame * 20.0 + slot * slot_duration_ms) / 1000.0;
  double cyclic_time = fmod(absolute_time, TIME_CYCLE_DURATION);
  static int last_debug_frame = -1;
  if (frame % 50 == 0 && frame != last_debug_frame) {
    LOG_I(NR_MAC, "Timing Debug: Frame %d, Slot %d, Absolute Time %.3fs, Cyclic Time %.3fs\n",
          frame, slot, absolute_time, cyclic_time);
    last_debug_frame = frame;
  }
  return cyclic_time;
}
```
**Worked example**: Frame 100, Slot 15 → `(100×20.0 + 15×0.033)/1000.0 = 2.0005s` →
`fmod(2.0005, 20.0) = 2.0005s` → within the first 10s interval → UE1 gets 20 PRBs,
UE2 gets 40 PRBs.

| Time window | UE1 | UE2 |
|-------------|-----|-----|
| 0–10s | PRBs 0–19 | PRBs 40–79 |
| 10–20s | PRBs 0–39 | PRBs 40–59 |
| 20s+ | cycle repeats | cycle repeats |

### Configuration management / binding
```c
static dynamic_prb_config_t* get_dynamic_prb_config(const NR_UE_info_t *UE, frame_t frame, slot_t slot) {
  uint64_t key = (uint64_t)UE->rnti;
  double cyclic_time = get_current_time_seconds(frame, slot);
  int current_cycle = get_current_cycle(frame, slot);

  /* Reclaim stale bindings from previous cycles */
  for (int i = 0; dynamic_prb_configs[i].enabled; i++) {
    if (cfg->ue_id != 0 && cfg->last_seen_cycle != current_cycle)
      cfg->ue_id = 0;  // Release stale binding
  }
  /* Existing binding? */
  for (int i = 0; dynamic_prb_configs[i].enabled; i++) {
    if (dynamic_prb_configs[i].ue_id == key) {
      update_time_based_config(&dynamic_prb_configs[i], cyclic_time, current_cycle);
      return &dynamic_prb_configs[i];
    }
  }
  /* Bind to first free slot */
  for (int i = 0; dynamic_prb_configs[i].enabled; i++) {
    if (dynamic_prb_configs[i].ue_id == 0) {
      dynamic_prb_configs[i].ue_id = key;
      update_time_based_config(&dynamic_prb_configs[i], cyclic_time, current_cycle);
      return &dynamic_prb_configs[i];
    }
  }
  return NULL;  /* No dynamic slot available */
}
```

**Binding order**: 1st UE (e.g. RNTI 0x1234) → `dynamic_prb_configs[0]` (Pattern 1);
2nd UE (RNTI 0x5678) → `dynamic_prb_configs[1]` (Pattern 2); 3rd+ UE → no slot
available, falls back to normal scheduler.

### Main scheduling integration
```c
int dynamic_prb_start = 0, dynamic_prb_size = 0;
bool has_dynamic_config = get_current_prb_allocation(iterator->UE, frame, slot,
                                                       &dynamic_prb_start, &dynamic_prb_size);
if (has_dynamic_config) {
  rbStart = dynamic_prb_start;
  uint16_t available_rb = dynamic_prb_size;
  bool dynamic_prbs_available = true;
  for (int rb = 0; rb < available_rb; rb++) {
    if (rballoc_mask[rbStart + bi.bwpStart + rb] & slbitmap) {
      dynamic_prbs_available = false;
      break;
    }
  }
  if (!dynamic_prbs_available) { iterator++; continue; }  // skip if occupied
} else {
  /* Normal PRB allocation (original PF logic) */
  while (rbStart < bi.bwpSize && (rballoc_mask[rbStart + bi.bwpStart] & slbitmap))
    rbStart++;
}
```
Dynamic path: pull pre-configured `rbStart`/`available_rb` for the current time
window, verify availability, skip the UE this slot if occupied — no optimization,
uses exactly what's configured. Normal path: standard first-free-RB scan with
`nr_find_nb_rb()` optimal sizing.

## Build after changes
```bash
cd ~/openairinterface5g/cmake_targets
./build_oai -w USRP --ninja --nrUE --gNB --build-lib "nrscope" -C
```

**NOTE:-**
After every algorithm change ( .c file or .h file or etc..) except config files, we have to rebuild for every changes.
After build, run the core and gnb as always

## Results

The gNB scheduler was configured to alternate between 20 PRBs and 40 PRBs per UE,
changing every 20 seconds. Under this varying configuration, **~30 Mbps** throughput
was observed at both UEs. Detailed gNB logs confirmed PRBs actually varying over
time as configured.

<img width="1130" height="782" alt="image" src="https://github.com/user-attachments/assets/52537f43-c6d5-496b-bafa-0457d770088a" />


## Observations and uncertainties

- After two UEs attach, if one UE detaches, the UE-side logs still show it as
  connected while the gNB logs show no information about the detached UE.
- Running the `iperf3` test causes the UE to temporarily lose internet connection;
  once the test ends, connectivity returns. Observed on either UE.


## Updated scheduler
[time_varying_uplink_scheduler.c](../03-time-varying-prb/time_varying_uplink_scheduler.c)

