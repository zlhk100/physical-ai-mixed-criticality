# Campaign Plan #3: Accelerator Time-Sharing Interference — OUTLINE

**Status:** Outline. Activation gated on (a) the spec v1.4 proposal
(`spec/proposals/v1.4-accelerator-time-sharing.md`) being accepted for
trial, and (b) testbed bring-up exit criteria (below).
**Governed by:** MCIB Specification, plus the v1.4 proposal (victim family
F4, layer `hw.accel`) in trial status. The tool qualification disclaimer
in Spec §4.2 is inherited by reference.
**Author:** Lei Zhou

---

## Objective

Produce attributed, verdict-bearing evidence for one question that every
consolidated platform with a shared GPU has to answer:

> When a non-critical domain saturates the GPU, does the critical domain's
> GPU work still meet its deadline — and if not, is that fixable by
> configuration, by a partitioning mechanism, or not at all?

Existing practice answers this by demonstration ("the cluster looked smooth
while the infotainment ran a game"). This campaign answers it with a
deadline-miss distribution, an attribution of each miss class, and one of
the three MCIB verdicts.

## Use case

**Consolidated digital cockpit on one SoC with one GPU.** The GPU has no
hardware partitioning. Two domains share it:

| Domain | Workload | Criticality of its GPU path |
|---|---|---|
| Instrument cluster | Gauges and driver information rendered at the panel rate, composited on the host | Availability-critical. Safety telltales are *not* on this path (see the reference arm) |
| Infotainment (IVI) | An Android Automotive-class guest VM: UI, navigation, media, games, on-device inference | Non-critical; may drive the GPU to saturation at any time |

The guest reaches the GPU through API forwarding (virtio-gpu with a
Vulkan-forwarding backend such as Venus or gfxstream). Every guest command
therefore executes on the host GPU driver on the guest's behalf, in the same
driver and firmware scheduler as the cluster's own work.

**Requirement shape** (values are campaign defaults, not claims about any
product):

| ID | Requirement | Default `[REFERENCE]` |
|---|---|---|
| R1 | Cluster frame completes before its scanout deadline | 60 Hz panel: 16.67 ms period |
| R2 | R1 holds while the guest's GPU load is swept from idle to saturation | Sweep in declared steps; saturation = guest workload GPU-bound with no frame pacing |
| R3 | Deadline-miss ratio below a declared bound, with misses attributed | Bound declared per run, never inferred from data |
| R4 | The guest is not starved: its throughput under arbitration is reported against its unarbitrated throughput | Reported, not pass/fail |
| R5 | After a guest burst ends, the cluster returns to zero misses within a declared number of frames | Declared per run |
| R6 | Host CPU cost of the arbitration mechanism itself is reported | Per-thread accounting, % of all host cores |

**Physical-AI variant.** The same shape with a different critical job: a
perception inference job with a period and a deadline, sharing the GPU with
non-critical rendering or batch inference. The method, victim family and
verdict mapping are identical; only the victim workload changes. Both
variants are in scope; the cockpit variant runs first because its deadline
is externally fixed by the display.

## Scope sketch

1. **Victim (proposed family F4, accelerator job latency):** per cluster
   frame, submit → GPU start → GPU completion → scanout, with the miss
   defined against the vblank deadline. Preemption latency (high-priority
   submit to start of execution while an aggressor job is running) is
   reported separately.
2. **Aggressors:**
   - *Synthetic, calibrated:* a GPU kernel with controllable per-job
     duration (0.1 ms to tens of ms) and controllable memory footprint.
     Job duration is the primary axis, because a non-preemptible job blocks
     a waiting victim for up to its own length.
   - *Real:* a frame-unpaced 3D benchmark in the guest, a GPU-bound game
     proxy, and Vulkan inference in the guest. Each real aggressor is
     expressed as a calibrated synthetic equivalent (Spec §1.2).
3. **Arms and verdict mapping:**

   | Arm | Configuration | Verdict it can produce |
   |---|---|---|
   | A0 | Cluster alone | Baseline floor |
   | A1 | Shared GPU, default scheduling | Characterises the problem |
   | A2 | Cluster at the host driver's highest group priority (e.g. Panthor `REALTIME`) | `tune` |
   | A3 | A2 plus guest-context priority carried through the virtualisation backend to the host scheduler | `tune` (new mechanism; see hypotheses) |
   | A4 | Hardware GPU partition or isolated lane, where the platform provides one | `partition` |
   | A5 | Reference: safety content on a dedicated display plane that bypasses the GPU | Reference bound, not a GPU verdict |

   Residual misses that no arm removes, and whose length tracks the
   aggressor's per-job duration, are a `ceiling`: the GPU cannot preempt
   finely enough, and the design must bound guest job length instead.
4. **Hypotheses** (stated as hypotheses, not expected results):
   - (a) Without priority, the cluster miss ratio rises with guest GPU load.
   - (b) Host-side priority bounds misses only down to the GPU's preemption
     granularity; the residual tracks aggressor job length (`ceiling`).
   - (c) Under API forwarding, a priority set inside the guest does not
     reach the host scheduler unless the backend maps it explicitly; with
     the mapping, guest-side critical work gains the same bound as
     host-native work.
   - (d) Differential attribution separates GPU-engine contention
     (`hw.accel`) from memory-path contention caused by GPU traffic
     (`hw.mem` / `hw.dma`).
   Any of these may fail. A failed hypothesis with attribution is a
   conforming result.
5. **Instrumentation (port-owned, per Spec §3.2):** GPU timestamps
   (Vulkan timestamp queries) and fence signal times; DRM GPU-scheduler and
   driver tracepoints; KMS vblank and page-flip events; per-thread CPU
   accounting; PMU counters for the memory path. GPU firmware scheduling is
   not observable on the reference platforms, so its contribution is
   attributed differentially (Spec §3.3). The attribution coverage map is a
   bring-up deliverable.

## Testbed

- **Platform 1 (first):** a retail board with an Arm Mali Valhall CSF GPU
  (e.g. RK3588-class), mainline Linux, upstream Panthor + Mesa PanVK, KVM,
  and an upstream VMM (crosvm or QEMU) with a Vulkan-forwarding
  virtio-gpu backend. Fully open accelerator path (Spec §5).
- **Platform 2 (candidate):** a board whose GPU has ring-level hardware
  preemption with an upstream driver, and documented KVM boot (an Adreno
  7xx-class board is the leading candidate, pending verification). Gives
  a second preemption architecture, so verdicts can be compared across
  silicon.
- **Workload integration:** the cluster and compositor come from
  [vcockpit](https://github.com/zlhk100/vcockpit), the author's open-source
  cockpit PoC; porting it to Platform 1 is a bring-up task.
- **Provenance:** only upstream and publicly documented components are used.

**Bring-up exit criteria:** KVM guest with GPU forwarding renders on the
panel; cluster frame timing captured with a declared timestamp domain;
A0 reproduces within 5 % at P99 across two clean boots (Spec §1.4).

## Deliverables

- A conforming dataset for arms A0–A3 on Platform 1, each outlier class
  attributed and terminated in a verdict.
- The F4 victim and the synthetic GPU aggressor as framework code.
- If hypothesis (c) holds: a priority-propagation patch for an upstream
  virtio-gpu backend, submitted upstream, with the dataset as its evidence.
- A written comparison with hardware partitioning (A4) — measured where a
  platform allows it, otherwise from published vendor material, labelled as
  such.

## Dependencies

- Spec v1.4 proposal accepted for trial (F4, `hw.accel`). If the campaign
  shows either is unnecessary, the proposal is withdrawn — the change
  policy requires the campaign to demonstrate necessity.
- Rule of two: GPU-side framework code enters the product core only when
  a second platform port uses it.

## How to help

- Tell us how your platform's GPU scheduler handles priority and
  preemption, and what it exposes to an open driver.
- Nominate a cluster workload and a frame-deadline definition from a
  production cockpit; the victim choice should not be made by one person.
- Say which of the four hypotheses you think is wrong, and why.

*Copyright 2026 Lei Zhou. SPDX-License-Identifier: CC-BY-4.0*
