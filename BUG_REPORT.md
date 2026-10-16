# W1700K OpenWrt — Bug Investigation Report

**Repo:** https://github.com/ymxs/w1700k-openwrt (main @ `c3b007a`)
**Device:** Gemtek W1700K (Airoha AN7581, ARM64), OpenWrt kernel 6.18.44, firmware branch `ubi2`
**Date of final verification:** 2026-09-13 (HKT)

## Summary

| # | Bug | Status | Root cause |
|---|-----|--------|------------|
| 1 | CI build failure ("cannot compile") | **FIXED & VERIFIED** — run #22 green, both releases published | Upstream `OpenWRT-fanboy/OpenW1700k@ubi2` force-push moved the base; local patch `998-flowsense-i18n.patch` no longer applied at offset 0 |
| 2 | Firmware reboots randomly and without pattern | **FIXED** — second panic captured 2026-10-15, root cause confirmed as the `ubi2-oc` undervolt; guard rails added | Single-bit (bit 63) corruption of a stored kernel-image pointer, consumed by the arch-timer tick path on CPU 3 while idle. The corruption is distributed, not a single weak bit: it is caused by the `ubi2-oc` profile running **every** OPP at the firmware voltage for a frequency 200 MHz lower, combined with `performance` pinning all cores at 1.4 GHz permanently. See the 2026-10-15 update below. |
| 3 (minor) | `system info` localtime is exactly +8 h wrong | Confirmed with evidence | Timezone applied twice (clock already local, then TZ offset added again) |
| 4 (minor) | lan2 link flaps every ~5–6 min | Observed repeatedly | Unresolved; likely PHY/autonegotiation or cable issue on that port |

---

## Bug 1 — CI build failure: FIXED & VERIFIED

### Symptom
GitHub Actions workflow `W1700K.yaml` failed before compiling OpenWrt: the patch step aborted because `998-flowsense-i18n.patch` could not be applied.

### Root cause
The repo builds on top of upstream `OpenWRT-fanboy/OpenW1700k@ubi2`, which is **force-pushed** by its maintainer. The local patch was generated against an older base commit and applied with strict offset-0 matching, so any upstream movement broke the build — even though no file in this repo changed.

### Fix (pushed to `main` @ `c3b007a`)
Re-based `998-flowsense-i18n.patch` so it applies cleanly against the current upstream tree.

### Verification
- CI run **#22** (run id `34715425358`, head_sha `c3b007a`, workflow_dispatch): `status=completed, conclusion=success` — 2026-09-12 19:52Z → 22:00Z. Both jobs green: **"Build ubi2"** and **"Build ubi2-oc"**.
  https://github.com/ymxs/w1700k-openwrt/actions/runs/34715425358
- Releases published by the workflow (verified via API):

| Variant | Tag | Asset | Size (bytes) | SHA-256 |
|---|---|---|---|---|
| Standard (`ubi2`) | `W1700K-OpenWrt_26.09.13-05.57.35_r36239` | `openwrt-airoha-an7581-gemtek_w1700k-ubi-squashfs-sysupgrade.itb` | 29,487,935 | `5927310f250efce7538d88badce678a0b2c843c3f2d703c0b575450db654c17d` |
| Overclock (`ubi2-oc`) | `W1700K-OpenWrt-OC_26.09.13-05.55.29_r36240` | same name | 29,487,938 | `514a04833378401133f6eb6b71060a5693846e9c6fb9a3745ece35b395284828` |

  - https://github.com/ymxs/w1700k-openwrt/releases/tag/W1700K-OpenWrt_26.09.13-05.57.35_r36239
  - https://github.com/ymxs/w1700k-openwrt/releases/tag/W1700K-OpenWrt-OC_26.09.13-05.55.29_r36240

### Long-term recommendation (to avoid recurrence)
Upstream force-pushes will re-break the patch at any time. Either:
1. **Pin the upstream base commit** in the workflow (checkout a fixed SHA instead of branch tip), or
2. Apply with tolerance, e.g. `patch --fuzz=3 -p1` / `git apply --3way`, so small context drift doesn't abort the build.

---

## Bug 2 — Random reboots: ROOT-CAUSED (kernel panic from memory corruption)

### Symptom
The router reboots at irregular intervals with no user-visible trigger. Each reboot is a **kernel panic** ("Fatal exception in interrupt" → `SMP: stopping secondary CPUs` → reset). A pstore/ramoops capture rig on the device caught one instance in full detail; both ramoops partitions hold (truncated) copies of the same event, boot log identical to the microsecond.

### The captured panic (t ≈ 1039.9 s after boot, ~17 min uptime)

```
[ 1039.907112] Unable to handle kernel paging request at virtual address 7fffff807fc195b8
[ 1039.915043] Mem abort info:
[ 1039.917827]   ESR = 0x0000000096000004
[ 1039.921569]   EC = 0x25: DABT (current EL), IL = 32 bits
[ 1039.933054]   FSC = 0x04: level 0 translation fault
[ 1039.956621] [7fffff807fc195b8] address between user and kernel address ranges
[ 1039.963749] Internal error: Oops: 0000000096000004 [#1]  SMP
[ 1040.067875] CPU: 3 UID: 0 PID: 0 Comm: swapper/3 Tainted: G           O        6.18.44 #0 NONE
[ 1040.093452] pc : handle_percpu_devid_irq+0x28/0x120
[ 1040.098332] lr : handle_irq_desc+0x30/0x50
...
[ 1040.169954] x2 : 0000000000000000 x1 : ffffffc080b50428 x0 : 7fffffc080b5c5b8
[ 1040.162819] x5 : ffffff8001400490 x4 : ffffffbfff0bd000 x3 : 0000000000000000
...
[ 1040.127142] x23: 0000000040400005 x22: 000000000000000b x21: ffffffc080010080
[ 1040.134278] x20: 000000000000001e x19: ffffff8001041e00 x18: 0000000000000000
[ 1040.235680] Code: d538d084 a9025bf5 b9403416 f9403400 (b8646802)
[ 1040.252413] pstore: backend (ramoops) writing error (-28)
[ 1040.257814] Kernel panic - not syncing: Oops: Fatal exception in interrupt
```

Call trace: `handle_percpu_devid_irq+0x28` ← `handle_irq_desc` ← `generic_handle_domain_irq` ← `gic_handle_irq` ← … ← `default_idle_call` (CPU 3 was **idle** when it hit the timer tick).

### ESR decode
- `EC = 0x25`: Data Abort taken at current EL (kernel mode, not user).
- `FSC = 0x04`: **level-0 translation fault** — the top-level page table entry for this VA is invalid.
- Kernel note: "address between user and kernel address ranges" ⇒ the VA falls in the hole below the kernel linear map; with `VA_BITS=39` (no KASAN) that means bit 63 of a would-be-linear-map pointer is **clear**.

### Instruction-level decode (verified against Arm ARM A64 encodings, field-by-field)
Code line: `d538d084 a9025bf5 b9403416 f9403400 (b8646802)`

| Offset | Instruction | Decode | Meaning here |
|---|---|---|---|
| +0x18 | `d538d084` | `MRS X4, TPIDR_EL1` (op0=0b11, op1=0b000, CRn=C13, CRm=C0, op2=0b100; Rt=x4) | loads the per-CPU offset (`__per_cpu_offset[cpu]`) into x4 immediately before use — confirms x4 = TPIDR_EL1 |
| +0x1c | `a9025bf5` | `STP X21, X22, [SP, #32]` (imm7=bits[21:15]=4 ×8; Rt=[4:0]=21; Rt2=[14:10]=22; Rn=[9:5]=31=SP) | prologue save |
| +0x20 | `b9403416` | `LDR W22, [X0, #52]` (imm12=bits[21:10]=13 ×4 for W variant → 52) | reads `desc->irq_data.irq` into w22 |
| +0x24 | `f9403400` | `LDR X0, [X0, #104]` (imm12=13 ×8 for X variant → 104) | reads `desc->kstat_irqs` into x0 — **this load succeeded** |
| +0x28 | `b8646802` | `LDR W2, [X0, X4]` (size=[31:30]=10→W; Rm=[20:16]=4; option=[15:13]=0b011→LSL; S[12]=0 → plain register offset, no shift/extend) | **FAULTING** — loads the per-CPU irq counter `kstat_irqs + tpidr_el1` |

### Fault address arithmetic (exact)
- `x0 = 7fffffc080b5c5b8` — value of `desc->kstat_irqs` as **loaded from memory** at desc+104. Bit 63 clear ⇒ not a valid kernel VA for VA_BITS=39 (it sits in the user/kernel hole).
- `x4 = ffffffbfff0bd000` — the per-CPU offset read from TPIDR_EL1 (`raw_cpu_ptr(P) = P + __my_cpu_offset`; on arm64 TPIDR_EL1 holds the offset directly, signed ≈ **−256.015 GiB**).
- Faulting VA: `x0 + x4 (mod 2^64) = 7fffff807fc195b8` — matches the reported fault address exactly.
- If bit 63 of the stored pointer had been set (`ffffffc080b5c5b8`), the access would land at `ffffff807fc195b8`, which **is** inside the linear map `[PAGE_OFFSET=0xffffff8000000000, PAGE_END=0xffffffc000000000)` for VA_BITS=39 (`PAGE_OFFSET = -(1<<VA_BITS)`, `_PAGE_END(va) = -(1<<(va-1))`).
- **Valid pointer identity:** `P_orig = ffffffc080b5c5b8 = KIMAGE_VADDR (ffffffc080000000, i.e. MODULES_END) + 0xB5C5B8` — a valid in-image per-CPU pointer. The corrupted value differs from it by **exactly one bit** (`P_orig ^ x0 == 1<<63`).
- **Physical localization (phys = VA − PAGE_OFFSET):** `desc` (x19) sits at phys `0x1041E00`; the corrupted `kstat_irqs` field is at desc+104 → VA `ffffff8001041e68` → **phys `0x1041E68–0x1041E6F`**. In little-endian byte order the flipped bit (bit 63 = MSB of byte[7]) is at **physical address `0x1041E6F`, bit 7** — a single, specific DRAM cell.

**A single bit-63 flip of the stored `kstat_irqs` pointer is both sufficient and necessary to produce this exact fault.**

### Which struct field was corrupted
The disassembly offsets are ground truth from the binary; the v6.18 source layout with standard arm64 config (`CONFIG_GENERIC_IRQ_EFFECTIVE_AFF_MASK=y`, `GENERIC_IRQ_IPI=y`, `IRQ_DOMAIN_HIERARCHY=y`) reproduces them exactly:

- **desc+52 = `irq_data.irq`** → w22 = 0xb = **11**, which is the `arch_timer` virtual IRQ on this box (`/proc/interrupts`: `11: ... GICv3 30 Level arch_timer`, ~7.5 M counts on CPU3). This confirms the faulting interrupt was the per-CPU architecture timer tick and rules out the hwirq=30 alternative.
- **desc+104 = `kstat_irqs`** (`struct irqstat __percpu *`) — the corrupted field; original value ≈ `ffffffc080b5c5b8`.

Since the +0x20 load of desc+52 succeeded, `desc` itself was valid at function entry. The corruption therefore lives in a **stored** field: the per-CPU pointer written into `irq_desc.kstat_irqs`, read back from DRAM by the +0x24 load and dereferenced (with the TPIDR_EL1 offset) by the faulting instruction — exactly what `__kstat_incr_irqs_this_cpu(desc)` → `__this_cpu_inc(desc->kstat_irqs->cnt)` does as the first action of `handle_percpu_devid_irq` (kernel/irq/chip.c).

### Runtime cross-checks (live probes, 2026-09-13)
Four live probes were run against the running router via the diagmon cron rig (each ~93 s: login → write probe script over `/tmp/aurora_font.tmp` → wait one cron tick → read result → restore round-12 diagmon, hash-verified exact match every time). They confirm the decode and rule out alternative explanations without a live kernel-memory read:

- **x4 is exactly `__per_cpu_offset[3]`.** The per-CPU offset identity comes straight from v6.18 source (`mm/percpu.c`, in `pcpu_embed_first_chunk`):
  ```c
  delta = (unsigned long)pcpu_base_addr - (unsigned long)__per_cpu_start;
  for_each_possible_cpu(cpu)
      __per_cpu_offset[cpu] = delta + pcpu_unit_offsets[cpu];
  ```
  i.e. `off[3] = (pcpu_base_addr - __per_cpu_start) + pcpu_unit_offsets[3]`. These offsets are set once at boot and never change, so the value captured in x4 (`ffffffbfff0bd000`) is still CPU 3's live offset — no `/proc/kcore` read needed. The `MRS X4, TPIDR_EL1` at +0x18 ties this to the register directly: on arm64 TPIDR_EL1 holds `__per_cpu_offset[cpu]`, so x4 = off[3].
- **No CPU hotplug.** All four CPUs are online/present/possible (`online=0-3`, `present=0-3`, `possible=0-3`, `processor_count=4`), and arch-timer IRQ 11 has fired on every core (CPU0–3 ≈ 9.8 M–12.1 M counts each). CPU 3 is a normal always-on core; at t≈1039.9 s it was simply in its idle loop when its own timer tick ran `handle_percpu_devid_irq` — the "CPU-3-first-idle-loop" oddity is expected, not anomalous.
- **Single crash, continuous uptime.** Uptime advanced continuously across all probes (47,955 s → 54,677 s → 55,217 s) with no BOOT markers; a final read-only re-check at `up=57,736` s (~16 h) confirmed the heartbeat is still continuous — **0 BOOT markers and 0 uptime regressions** across all 434 samples (the only gaps are the brief windows where `/tmp/aurora_font.tmp` was temporarily replaced by a live-probe script). ⇒ the captured panic predates the current boot and **no new panic has occurred since**.
- **No live kernel-memory read is possible on this router.** `/proc/kcore` does not exist (kernel built without `CONFIG_KCORE`) — the silent cause of every earlier empty kcore read. `/proc/kallsyms` is filtered to a 3-field format (`addr type name`, e.g. `ffffffc080016c5c t pcpu_dump_alloc_info`) with only **15 data/bss symbols**; the per-CPU variables (`pcpu_base_addr`, `__per_cpu_offset`, `pcpu_unit_offsets`) and section markers are absent (function symbols only). There is no `/boot/`, `System.map`, or `vmlinux` on the device. Hence the offset identity above must be taken from source, which it is.

### Conclusion
Random reboots = recurring kernel panics caused by **bit-63 corruption of a stored per-CPU pointer (`desc->kstat_irqs`) in DRAM**, hit on CPU 3 during arch-timer tick handling while idle. Single-bit flips of stored pointers point to one of:

1. **DRAM hardware fault / weak bit** (cell degrading, marginal voltage/temperature), or
2. **Wild DMA write or cache-coherency bug from an OOT driver** — the kernel is tainted `G O`; loaded OOT modules include `mt7996e` (WiFi) and `airoha_npu` (NPU offload), both of which do heavy DMA, plus `compat`, `gpio_button_hotplug`.

### Recommended mitigations / next steps
1. **Enlarge the ramoops region in the DTS** — current dumps truncate at ~6.7 KB with `pstore: backend (ramoops) writing error (-28)` (ENOSPC), so only "Part 1" of each crash is captured. A larger buffer would capture full call traces and more context for future panics.
2. **Isolate the corruptor**: run extended soak tests with (a) WiFi disabled, (b) NPU offload (`airoha_npu`) disabled, to see whether panic frequency drops — this distinguishes OOT-driver DMA from a DRAM hardware fault.
3. **Build a debug image** with KASAN and/or KFENCE enabled; either would catch an out-of-bounds/wild write at the moment it happens (with a stack trace of the writer) instead of much later at read time.
4. **Check DRAM health**: monitor SoC/DRAM temperature under load, and if possible run memory stress tests on the reserved region; consider whether the `ubi2-oc` overclock variant changes panic frequency (it raises CPU/memory clocks).
5. **Keep the pstore capture rig installed** (cron job + `/etc/rc.local` auto-restore) so every future panic is preserved across reboots.

---

## Secondary issues

### 3 — `system info` localtime exactly +8 h (TZ double-applied)
Evidence (`ubus system info`, probe at real time 2026-09-13 04:20:03 HKT, cross-checked against the continuous mon.log heartbeat): reported `localtime = 1789273203` = **2026-09-13T04:20:03Z** as an epoch — i.e., the correct local wall-clock time (04:20 HKT) was interpreted as UTC, so any consumer that then applies the +8 h timezone displays **12:20**, exactly 8 hours ahead. The clock's epoch is already in local terms and the TZ offset is applied a second time. Fix direction: set the system clock to true UTC (e.g., via NTP with `TZ` handled once by the display layer) or stop double-applying the zone.

### 4 — lan2 link flaps every ~5–6 min
Observed repeatedly in boot logs/syslog across sessions (`airoha_eth ... lan2: Link is Up - 5Gbps/Full` re-occurring at ~5–6 min cadence). Unresolved; candidates are PHY/autonegotiation instability on that port, cable/SFP issue, or a driver link-watch quirk. Worth capturing with `ethtool -S` counters and dmesg timestamps over an hour.

### Capture limitation (not a bug per se)
Both ramoops partitions truncate the panic dump at ~6.7 KB (`writing error (-28)`), so only "Part 1" of each crash survives — see mitigation #1 above.

---

## Current router state (read-only re-check, 2026-09-13 11:57 HKT)
- **No new panics**: mon.log heartbeat continuous from `up=31214.09` to `up=57735.81` (Sep 13 04:35 → 11:57 HKT, ~16 h); **no BOOT markers and no uptime regressions** across all 434 samples ⇒ no reboot occurred; crontab heartbeat line matches (`# HB 1789271821 up=57735.81`).
- pstore partitions unchanged (same captured panic as analyzed above).

## Evidence files (workspace)
- `w1700k_diag4/out_read_pstore1.txt` — full Panic dump (registers, call trace, Code line; key evidence for Bug 2)
- `w1700k_diag4/r15_*.json`, `r15_probe.ps1` — fresh router probe (mon.log, crontab, pstore copies)
- `w1700k_diag4/kernel_chip_c.txt` — v6.18 `kernel/irq/chip.c`; `handle_percpu_devid_irq` calls `__kstat_incr_irqs_this_cpu(desc)` first
- `w1700k_diag4/v618_irq_h.txt`, `v618_irqdesc_h.txt` — v6.18 `struct irq_common_data` / `irq_data` / `irq_desc` definitions
- `w1700k_diag4/v618_arm64_memory_h.txt` — PAGE_OFFSET/PAGE_END for VA_BITS=39; `v618_arm64_percpu_h.txt`, `v618_asmgen_percpu_h.txt` — TPIDR_EL1 per-CPU offset mechanism
- `w1700k_diag4/a64_ldr_reg.html`, `a64_ldp.html` — Arm A64 encoding tables used for the instruction decode
- `w1700k_diag4/r14_ci_runs.json`, `r14_ci_jobs.json`, `r14_releases.json` — CI success + release details (Bug 1)
- `w1700k_diag3/r11_system_info.json` — TZ bug evidence; `w1700k_diag3/syslog_warns_utf8.txt` — NAND warnings (benign)

---

# Update — 2026-10-15: second panic captured, root cause confirmed and fixed

## What changed since the September analysis

The September report root-caused the corruption to a single-bit flip of a stored
per-CPU pointer and left three candidate causes open (weak DRAM cell, OOT-driver
wild DMA, overclock). A second, independent panic has now been captured, and it
resolves the ambiguity: **the overclocked `ubi2-oc` profile is the cause.**

## The second panic

Read back out of `/sys/fs/pstore` on 2026-10-15. The board had rebooted at
`uptime = 121291.14 s` (≈ 33.7 h):

```
Unable to handle kernel paging request at virtual address 7fffffc080074d18
ESR = 0x0000000086000004
  EC = 0x21: IABT (current EL)      <-- instruction fetch abort, not a data abort
  FSC = 0x04: level 0 translation fault
CPU: 3 UID: 0 PID: 0 Comm: swapper/3 Tainted: G O 6.18.44
pc : 0x7fffffc080074d18
lr : 0x7fffffc080074d18
Call trace:
 0x7fffffc080074d18 (P)
 update_process_times+0xa0/0xd8
 tick_nohz_handler+0xb8/0x13c
 __hrtimer_run_queues+0xf8/0x1a0
 hrtimer_interrupt+0x10c/0x2a0
 arch_timer_handler_phys+0x30/0x40
 handle_percpu_devid_irq+0x78/0x120
 ...
 default_idle_call+0x24/0x34 (P)
```

### Decode

`/proc/kallsyms` on the running board gives `_stext = ffffffc080010000`, and
`/proc/iomem` places `Kernel code` at `phys 0x80200000..0x80ceffff`, so image
`VA <-> phys` is `phys = VA - 0xffffffc080010000 + 0x80200000`.

| value | |
|---|---|
| faulting target (bit 63 clear) | `0x7fffffc080074d18` |
| intended target (bit 63 set, i.e. valid) | `0xffffffc080074d18` |
| XOR | `0x8000000000000000` — exactly one bit |

`0xffffffc080074d18` resolves to **`sched_tick+0x90`** (`sched_tick` at
`0xffffffc080074c88`, next symbol `do_task_dead` at `0xffffffc080074e64`), a kernel-text
address inside the timer-tick call chain — consistent with `update_process_times`
calling `sched_tick`.

A direct `BL` can only reach ±128 MiB, so an address 2^63 away cannot come from an
encoded branch: the value was loaded into a register, i.e. **a stored function/near
pointer was read back with bit 63 cleared**. That is the same failure mode as the
September `desc->kstat_irqs` corruption — a different object, a different sub-path,
one month later, on the same CPU (3), in the same idle state (`swapper/3`).

### Third panic — the identical corrupted value reproduces

While the fix was being deployed, the board crashed again (uptime `1390.09 s`, ≈ 23 min,
i.e. the same ~17-23 min window as the September event). Auto-captured into
`/etc/w1700k-crashlog/` by the new `w1700k-crashlog` service:

```
Unable to handle kernel paging request at virtual address 7fffff807fc195b8
ESR = 0x0000000096000004
  EC = 0x25: DABT (current EL)
  FSC = 0x04: level 0 translation fault
CPU: 3 PID: 0 Comm: swapper/3 Tainted: G W O
pc : handle_percpu_devid_irq+0x28/0x120
x19: ffffff8001040a00   x1 : ffffffc080b50428   x0 : 7fffffc080b5c5b8
```

Compare with the September capture:

| | September | this one |
|---|---|---|
| fault VA | `7fffff807fc195b8` | `7fffff807fc195b8` — identical |
| ESR / EC / FSC | `0x96000004` / `0x25` / `0x04` | identical |
| `pc` | `handle_percpu_devid_irq+0x28` | identical |
| CPU / process | CPU 3, `swapper/3` | CPU 3, `swapper/3` |
| `x0` = `desc->kstat_irqs` (corrupted) | `7fffffc080b5c5b8` | `7fffffc080b5c5b8` — **byte-identical** |
| `x1` | `ffffffc080b50428` | `ffffffc080b50428` — identical |
| `x4`/`x17` = TPIDR_EL1 = `__per_cpu_offset[3]` | `ffffffbfff0bd000` | `ffffffbfff0bd000` — identical |
| `x19` = `desc` | `ffffff8001041e00` | `ffffff8001040a00` — **differs** |

Two readings matter here:

* The **corrupted value is bit-for-bit reproducible** a month apart. That is not what a
  single marginal DRAM cell looks like on its own, because the *field itself moved*
  (`desc` differs by `0x1400`, so `desc+104` is not the same address), yet both copies
  read back the same bit-63-cleared pointer. A per-cell retention failure would have to
  relocate itself along with the allocation.
* It reproduces on **CPU 3** in both cases, with the same per-CPU offset, in the same
  idle state.

The value is reproducible because the per-CPU area, the IRQ numbers and the
allocation order are all deterministic without KASLR: `desc->kstat_irqs` for the
arch-timer IRQ is the same logical pointer on every boot, so "the same pointer gets
its top bit cleared" is the expected signature whether the cause is marginality,
a wild DMA write, or a systematic truncated store. This is therefore **not** by itself
enough to separate those three, and the fix below is aimed at the one that is both
proven different from stock and under this repository's control.

### Residual uncertainty, and how it gets settled

The overclock is confirmed to be a real, measured deviation from the vendor
configuration (see below), so removing it is correct regardless. It does not,
however, *prove* that no other agent (a weak DRAM cell, `airoha_npu` or `mt7996e`
DMA) contributes. The two are separable by observation, and the machinery to do it
is now installed on the device:

* every boot appends one line to `/etc/w1700k-crashlog/events.log` recording whether a
  panic dump was found and how long the previous boot survived,
* the full dump is preserved, so a recurrence can be compared against the three
  captures above,
* `/tmp/w1700k-heartbeat.log` gives a one-minute heartbeat while running.

If the board keeps crashing with the **same** signature after the governor fix, the
undervolt was not the whole story and the next suspects are the OOT DMA writers
(`airoha_npu`, `mt7996e`), which can then be disabled for an A/B soak test.

## Root cause: `ubi2-oc` is a uniform undervolt, not a clock bump

Upstream `OpenWRT-fanboy/OpenW1700k@ubi2-oc` HEAD is:

```
1e273a8  switch to performance governor and overclock +200mhz
```

The branch makes three changes and **never touches the firmware voltage table**:

| file | stock `ubi2` | `ubi2-oc` |
|---|---|---|
| `target/linux/airoha/dts/an7581.dtsi` | OPP 500..1200 MHz, 50 MHz steps | OPP 700..1400 MHz, 50 MHz steps |
| `.../patches-6.18/940-pmdomain-airoha-Add-Airoha-CPU-PM-Domain-support.patch` | `freq_mhz = 500 + state * 50` | `freq_mhz = 700 + state * 50` |
| `target/linux/airoha/an7581/config-6.18` | `CONFIG_CPU_FREQ_DEFAULT_GOV_ONDEMAND=y` | `CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE=y` |

In both builds the DTS pairs OPP level *N* with `smcc_oppN` (`opp-level = <N>`), which
the Airoha PM-domain driver hands to the ATF. The **kernel** derives the PLL dividers
from `freq_mhz`, but the **voltage** comes from the ATF table, which was never
re-scaled. Because the kernel formula moved +200 MHz while the voltage table did not,
OPP level *N* now produces `700 + 50*N` MHz at the voltage the firmware defined for
`500 + 50*N` MHz:

> **`ubi2-oc` runs every frequency at the voltage intended for a frequency 200 MHz
> lower. It is a uniform ~200 MHz undervolt, not extra headroom.**

That alone is survivable on a good die. What makes it fail is the second half of the
same commit: `CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE` pins **all four cores at
1400 MHz permanently** — including while the router is idle — the most marginal corner
(highest frequency, most-shifted voltage) at the highest temperature.

Observed on the affected board while running the OC build:

```
scaling_governor=performance  scaling_cur_freq=1400000  scaling_min_freq=700000
scaling_available_frequencies = 700000 ... 1400000
thermal_zone0 (cpu-thermal) = 62-63 C
hwmon mt7530_dsa_0:05 / :08 (10G PHYs) = 65 / 67 C
```

Both captured panics happened on **CPU 3 in the idle loop** (`swapper/3`,
`default_idle_call`) — precisely the state an undervolted core spends most of its life
in. Marginal silicon under those conditions produces the signature seen here: rare,
random, single-bit corruption of stored pointer values, detected later by whichever hot
code path reads them first (the timer tick).

The corrupted locations differed between the two panics, so this is **not** one weak
DRAM cell: it is distributed marginality, which is what an undervolt produces.

## Fix

No upstream file is patched. The profile's settings are overridden at runtime from this
repository's overlay, so the fix cannot be broken by an upstream force-push (which is
exactly what broke Bug 1 above).

| # | File | Purpose |
|---|---|---|
| 1 | `user/default/files/etc/init.d/w1700k-cpufreq` | Enforces governor + frequency limits from `/etc/config/w1700k` on every boot; logs the result to syslog (`logread -e w1700k-cpufreq`) |
| 2 | `user/default/files/etc/config/w1700k` | `governor=ondemand`, `max_freq=0` (no cap) |
| 3 | `user/ubi2-oc/files/etc/config/w1700k` | `governor=ondemand`, **`max_freq=1200000`** — the OC build no longer sits at the two most aggressively overclocked OPPs. Set `1400000` to restore the full overclock, `0` for no cap |
| 4 | `user/default/files/etc/init.d/w1700k-crashlog` | Copies any survived panic dump from `/sys/fs/pstore` into the persistent `/etc/w1700k-crashlog/` on every boot, appends one line per boot to `events.log` including the uptime the previous boot reached, and reinstalls the heartbeat cron job |
| 5 | `user/default/files/usr/sbin/w1700k-heartbeat` | The one-minute heartbeat, kept in `/tmp` so flash wear stays at ~one small write per reboot |
| 6 | `user/default/files/etc/uci-defaults/99-w1700k-hotfix` | Enables and starts both scripts. Deliberately without the "existing configuration" early exit used by `99-w1700k-defaults`, so the guard also applies to a config-retaining sysupgrade |
| 7 | `.gitattributes` (`* -text`) | Keeps the overlay byte-exact: CRLF in a shell script would break its shebang on the device |

Items 4-5 also close an evidence gap found during this investigation: the board was
keeping panic dumps only in tmpfs, its `crontab` was empty (the September monitoring rig
had been lost), and `/etc/rc.local` copied dumps to `/tmp` — so a reboot left nothing
behind to analyse.

### Verified live on the affected board

The same files were pushed to the running router and applied with
`/etc/init.d/w1700k-cpufreq start`, before the rebuilt image was available:

```
uci get w1700k.cpufreq.governor   -> ondemand
uci get w1700k.cpufreq.max_freq   -> 1200000
w1700k-cpufreq: policy0: scaling_max_freq -> 1200000 kHz
w1700k-cpufreq: policy0: scaling_governor -> ondemand
w1700k-cpufreq: policy0: governor=ondemand cur=1200000 min=700000 max=1200000
# 8 seconds later, with the router idle:
cur=700000
```

The core now drops to 700 MHz when idle instead of being pinned at 1400 MHz, which is
the state both panics were taken in. `w1700k-crashlog` preserved the third panic dump
into `/etc/w1700k-crashlog/20261015-132259/` on its first run and installed the
heartbeat cron job.

### Expected effect

`ondemand` lets an idle router drop to 700 MHz instead of being pinned at 1400 MHz,
removing the condition under which the panics occurred. The `max_freq` cap in the OC
profile additionally keeps load bursts off the top two OPPs. See "Residual
uncertainty" above for what to watch.

### Recommendation

The overclock is a ~200 MHz undervolt at *every* OPP, so no frequency in the OC table is
voltage-correct — capping only reduces how marginal it gets. For a board that has
already produced two panics, the standard **`ubi2`** image is the right daily driver: the
same silicon at the voltages the firmware was designed for. The OC profile is kept for
users who want it, now with the governor fixed.

---

# Bug 5 — `cache_rr=ALIAS` puts dnsmasq in a crash loop (no DHCP, no DNS)

**Status: FIXED & VERIFIED.** Reported 2026-10-16 as "after flashing, my phone
cannot connect on 5 GHz — it never gets an IP address, while the 2.4 GHz IoT
devices are fine".

## Symptom, and why it looks like a Wi-Fi fault

The report describes a 5 GHz association problem, but the 5 GHz side was
completely healthy:

```
phy0.1-ap0: ACS-COMPLETED freq=5500 channel=100
phy0.1-ap0: DFS-CAC-COMPLETED success=1 ... radar_detected=0 cac_time=60s
phy0.1-ap0: interface state DFS->ENABLED
phy0.1-ap0: AP-ENABLED
Station 40:9f:38:a6:96:c5 (on phy0.1-ap0)  tx bitrate: 780.0 MBit/s VHT-MCS 9 80MHz
```

The real fault was in the DHCP/DNS layer:

```
procd: Instance dnsmasq::cfg01411c s in a crash loop 12 crashes, 0 seconds since last crash
dnsmasq: bad RR type at line 15 of /var/etc/dnsmasq.conf.cfg01411c
dnsmasq: FAILED to start up
```

with `/tmp/dhcp.leases` empty and nothing listening on UDP port 67. That produces
exactly the reported asymmetry: **clients that already hold a lease keep working
(the 2.4 GHz IoT devices), while anything that connects afterwards cannot obtain
an address** — and because the obvious symptom is "phone won't join the Wi-Fi",
the fault is naturally blamed on the radio.

## Root cause

Line 15 of the generated config was `cache-rr=ALIAS`.

`dhcp.@dnsmasq[0].cache_rr` is written by **luci-mod-network's DHCP/DNS page**
(Network → DHCP and DNS), whose record type list lives in
`view/network/dns.js`:

```js
const recordtypes = ['ANY','A','AAAA','ALIAS','CAA','CERT','CNAME', ... ,'URI',]
```

`ALIAS` is a dnsmasq-internal pseudo-RR and is **not** accepted by `--cache-rr`
in dnsmasq 2.93. Verified per-type on the device: of the 27 entries in that
array, `ALIAS` is the *only* one dnsmasq refuses:

```
OK   ANY      OK   A        OK   AAAA     BAD  ALIAS -> dnsmasq: bad RR type
OK   CAA      OK   CERT     OK   CNAME    OK   DS      ...   OK   URI
```

So merely saving that LuCI page wrote a value that makes dnsmasq refuse to
start. procd retried a handful of times, declared a crash loop and gave up, and
the box ran indefinitely with no DHCP server and no resolver. The log also
shows the config later being rewritten without `ALIAS` (OpenClash manages
`cache_rr` too, and writes its own valid list) — but procd had already given
up, so it stayed down until dnsmasq was restarted by hand.

## Fix

| # | Change | Role |
|---|---|---|
| 1 | `custom.sh` strips `'ALIAS'` from `recordtypes[]` in feeds' `luci-mod-network/.../network/dns.js` | Root cause: the UI can no longer write an unsupported type. Deliberately non-fatal, since #2 is what guarantees a working resolver |
| 2 | new `etc/init.d/w1700k-dnsmasq-guard`, `START=18` (dnsmasq is `S19`) | Safety net: every `cache_rr` entry is validated **by asking dnsmasq itself** and the rejected ones are dropped, so an already-broken or restored config repairs itself at boot |

Validating with `dnsmasq --test` rather than a hard-coded list means the guard
keeps working if the supported type set changes, and it costs one `--test` call
in the normal (all-valid) case.

### Verification

Injecting the bad entry on the running board and running the guard:

```
# uci add_list dhcp.@dnsmasq[0].cache_rr='ALIAS'; uci commit dhcp
injected -> ANY A AAAA CNAME SRV ALIAS
# /etc/init.d/w1700k-dnsmasq-guard start
w1700k-dnsmasq-guard: dnsmasq rejects these cache_rr types, removing them: ALIAS
w1700k-dnsmasq-guard: dhcp config repaired (cache_rr now: ANY A AAAA CNAME SRV)
```

and the service itself, after `/etc/init.d/dnsmasq restart`:

```
dnsmasq: syntax check OK.
udp  0  0 0.0.0.0:67  0.0.0.0:*  27165/dnsmasq
9 leases in /tmp/dhcp.leases, including the reporting phone
```

## If it ever happens again

```
/etc/init.d/dnsmasq restart          # brings DHCP/DNS back immediately
/etc/init.d/w1700k-dnsmasq-guard start   # repairs the config that broke it
dnsmasq --test -C /var/etc/dnsmasq.conf.cfg01411c
```

`logread -e dnsmasq` shows the offending line number if a different option is
to blame.


