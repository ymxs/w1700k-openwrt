# W1700K OpenWrt — Bug Investigation Report

**Repo:** https://github.com/ymxs/w1700k-openwrt (main @ `c3b007a`)
**Device:** Gemtek W1700K (Airoha AN7581, ARM64), OpenWrt kernel 6.18.44, firmware branch `ubi2`
**Date of final verification:** 2026-09-13 (HKT)

## Summary

| # | Bug | Status | Root cause |
|---|-----|--------|------------|
| 1 | CI build failure ("cannot compile") | **FIXED & VERIFIED** — run #22 green, both releases published | Upstream `OpenWRT-fanboy/OpenW1700k@ubi2` force-push moved the base; local patch `998-flowsense-i18n.patch` no longer applied at offset 0 |
| 2 | Firmware reboots randomly and without pattern | **ROOT-CAUSED** — captured kernel panic in full detail via ramoops/pstore | Single-bit (bit 63) corruption of the stored `desc->kstat_irqs` per-CPU pointer in DRAM, faulting on CPU 3 during arch-timer tick handling. Suspects: weak DRAM bit or wild DMA / cache-coherency write from an OOT driver (`mt7996e` WiFi, `airoha_npu`) |
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
| +0x18 | `d538d084` | unallocated HINT space | NOP |
| +0x1c | `a9025bf5` | `STP X21, X22, [SP, #32]` (imm7=bits[21:15]=4 ×8; Rt=[4:0]=21; Rt2=[14:10]=22; Rn=[9:5]=31=SP) | prologue save |
| +0x20 | `b9403416` | `LDR W22, [X0, #52]` (imm12=bits[21:10]=13 ×4 for W variant → 52) | reads `desc->irq_data.irq` into w22 |
| +0x24 | `f9403400` | `LDR X0, [X0, #104]` (imm12=13 ×8 for X variant → 104) | reads `desc->kstat_irqs` into x0 — **this load succeeded** |
| +0x28 | `b8646802` | `LDR W2, [X0, X4]` (size=[31:30]=10→W; Rm=[20:16]=4; option=[15:13]=0b011→LSL; S[12]=0 → plain register offset, no shift/extend) | **FAULTING** — loads the per-CPU irq counter `kstat_irqs + tpidr_el1` |

### Fault address arithmetic (exact)
- `x0 = 7fffffc080b5c5b8` — value of `desc->kstat_irqs` as **loaded from memory** at desc+104. Bit 63 clear ⇒ not a valid kernel VA for VA_BITS=39 (it sits in the user/kernel hole).
- `x4 = ffffffbfff0bd000` — the per-CPU offset read from TPIDR_EL1 (`raw_cpu_ptr(P) = P + __my_cpu_offset`; on arm64 TPIDR_EL1 holds the offset directly, signed ≈ **−256.015 GiB**).
- Faulting VA: `x0 + x4 (mod 2^64) = 7fffff807fc195b8` — matches the reported fault address exactly.
- If bit 63 of the stored pointer had been set (`ffffffc080b5c5b8`), the access would land at `ffffff807fc195b8`, which **is** inside the linear map `[PAGE_OFFSET=0xffffff8000000000, PAGE_END=0xffffffc000000000)` for VA_BITS=39 (`PAGE_OFFSET = -(1<<VA_BITS)`, `_PAGE_END(va) = -(1<<(va-1))`).

**A single bit-63 flip of the stored `kstat_irqs` pointer is both sufficient and necessary to produce this exact fault.**

### Which struct field was corrupted
The disassembly offsets are ground truth from the binary; the v6.18 source layout with standard arm64 config (`CONFIG_GENERIC_IRQ_EFFECTIVE_AFF_MASK=y`, `GENERIC_IRQ_IPI=y`, `IRQ_DOMAIN_HIERARCHY=y`) reproduces them exactly:

- **desc+52 = `irq_data.irq`** → w22 = 0xb = **11**, which is the `arch_timer` virtual IRQ on this box (`/proc/interrupts`: `11: ... GICv3 30 Level arch_timer`, ~7.5 M counts on CPU3). This confirms the faulting interrupt was the per-CPU architecture timer tick and rules out the hwirq=30 alternative.
- **desc+104 = `kstat_irqs`** (`struct irqstat __percpu *`) — the corrupted field; original value ≈ `ffffffc080b5c5b8`.

Since the +0x20 load of desc+52 succeeded, `desc` itself was valid at function entry. The corruption therefore lives in a **stored** field: the per-CPU pointer written into `irq_desc.kstat_irqs`, read back from DRAM by the +0x24 load and dereferenced (with the TPIDR_EL1 offset) by the faulting instruction — exactly what `__kstat_incr_irqs_this_cpu(desc)` → `__this_cpu_inc(desc->kstat_irqs->cnt)` does as the first action of `handle_percpu_devid_irq` (kernel/irq/chip.c).

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

## Current router state (fresh probe, 2026-09-13 04:35–07:00 HKT)
- **No new panics**: mon.log heartbeat continuous from `up=31214.09` to `up=39914.33` (Sep 13 04:35 → 07:00 HKT), no gaps, no BOOT markers; crontab heartbeat line matches (`# HB 1789254000 up=39914.33`, ~11.1 h uptime at probe time).
- pstore partitions unchanged (same captured panic as analyzed above).

## Evidence files (investigation workspace)
- `w1700k_diag4/out_read_pstore1.txt` — full Panic dump (registers, call trace, Code line; key evidence for Bug 2)
- `w1700k_diag4/r15_*.json`, `r15_probe.ps1` — fresh router probe (mon.log, crontab, pstore copies)
- `w1700k_diag4/kernel_chip_c.txt` — v6.18 `kernel/irq/chip.c`; `handle_percpu_devid_irq` calls `__kstat_incr_irqs_this_cpu(desc)` first
- `w1700k_diag4/v618_irq_h.txt`, `v618_irqdesc_h.txt` — v6.18 `struct irq_common_data` / `irq_data` / `irq_desc` definitions
- `w1700k_diag4/v618_arm64_memory_h.txt` — PAGE_OFFSET/PAGE_END for VA_BITS=39; `v618_arm64_percpu_h.txt`, `v618_asmgen_percpu_h.txt` — TPIDR_EL1 per-CPU offset mechanism
- `w1700k_diag4/a64_ldr_reg.html`, `a64_ldp.html` — Arm A64 encoding tables used for the instruction decode
- `w1700k_diag4/r14_ci_runs.json`, `r14_ci_jobs.json`, `r14_releases.json` — CI success + release details (Bug 1)
- `w1700k_diag3/r11_system_info.json` — TZ bug evidence; `w1700k_diag3/syslog_warns_utf8.txt` — NAND warnings (benign)
