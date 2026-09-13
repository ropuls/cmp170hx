# MIG Test Report — CMP 170HX (2nd Independent Data Point)

## Setup

- **GPU under test:** 1× NVIDIA CMP 170HX (GA100, unlocked to 64 GB); MIG exercised on this card only. A second GPU (RTX 3060) shared the host but was idle and uninvolved.
- **NVIDIA Driver:** 610.43.03 (open-gpu-kernel-modules)
- **cmpunlocker (base):** [amoghmunikote/cmpunlocker](https://github.com/amoghmunikote/cmpunlocker) — the 10-patch `master` set as it stood ~2026-09-01, *before* `cmp-sku-mask.patch` was added upstream (today's `master` carries 11). `PATCH_ORDER`: `sec2-postbl-plm-ss-cfg · booter-verify · late-pma · bar0-pramin-clamp · ce-scrub-workarounds · persistent-sw-state · pcie-gen2 · pcie-gen2-probe-retrain · name-string · bar1-resize-unlock`. Local build — the tree is a copy, **not** a git checkout, so no upstream commit is pinned in it. Auto-profile `8gb` → 64 GB unlock (PCI `10de:20c2`). Base `nvidia.ko` = 41 625 960 B.
- **MIG Patch:** [mig-unlock.patch](https://github.com/amoghmunikote/cmpunlocker/blob/MIG/driver/patches/mig-unlock.patch) (MIG-Branch, commit `86b238f` — "Widen the MIG compute partition from 14 SMs to the whole GPU")
- **OS:** Debian 13, Kernel 6.12.107+deb13-amd64
- **Workload run on the instance:** two single-file CUDA programs, built for `sm_80` with CUDA 13.1 (`nvcc -arch=sm_80 …`, cuBLAS linked with `-lcublas`), each pinned to the MIG compute instance via `CUDA_VISIBLE_DEVICES=<MIG-UUID>`:
  - `sgemm` — cuBLAS `cublasSgemm`, square FP32 GEMM N=8192, 30 timed iterations; GFLOP/s = 2·N³·iters ÷ elapsed.
  - `rt` — a 512 MiB host→device→host `cudaMemcpy` round-trip with a full byte-compare verify.

**Patch applies cleanly to 610.43.03** (`patch -p1 --dry-run`): all 3 hunks succeed (2 & 3 at offset −31, no fuzz). Built via cmpunlocker `PATCH_ORDER`; **the two builds differ by exactly one entry — `mig-unlock.patch` appended last to the 10-patch base, nothing else** — `nvidia.ko` = 41 627 368 B (patched) vs. 41 625 960 B (base). No brick; driver loads cleanly, 64 GB intact, both across a warm reboot and a cold power-cycle.

## Headline finding

The patch does **not** change profile listing or instance creation — **both work on the cmpunlocker base without `mig-unlock.patch`**. The single observable difference is **sustained compute throughput on the instance**:

| Driver | `mig -lgip` (GI listing) | create `1g.64gb` instance | **cuBLAS SGEMM (8192³ FP32) on the instance** |
|---|---|---|---|
| **base (unpatched)** | lists `1g.64gb` | succeeds | **1848 GFLOP/s** |
| **+ mig-unlock.patch** | lists `1g.64gb` (unchanged) | succeeds | **10225 GFLOP/s** |

So on this driver (610.43.03) the patch raises FP32 SGEMM under MIG by **~5.5×** — **consistent with the patch widening the effective compute-instance capacity**, as the commit describes ("14 SMs → the whole GPU"; the commit's own count is 14 → 56 SMs). This is **not** an SM count: `-lgip`/`Shared MP count` cosmetically report 70 SM regardless and are not evidence of the real partition width — only the throughput delta is measured here, and only for FP32.

**Not established by this test:** the actual active-SM count (needs the CUDA `multiProcessorCount` device attribute and/or `-lci` compute-instance profile data), clocks/power/temperature, and repeated runs. It also does **not** address the separate open **INT8/IMMA throughput** gate (a different question — library vs. raw MMA — not FP32-SGEMM).

## Test Results

*All `nvidia-smi` output below is verbatim from the run (only the GPU UUID is redacted). Enable / list / create / profile-rejection behave identically on the base and patched drivers — the sole measured difference is the SGEMM throughput in the A/B.*

### MIG enable + mode
```
$ sudo nvidia-smi -i 0 -mig 1
Enabled MIG Mode for GPU 00000000:81:00.0
All done.

$ nvidia-smi -i 0 --query-gpu=name,mig.mode.current,mig.mode.pending --format=csv,noheader
NVIDIA CMP 170HX, Enabled, Enabled
```

### GPU-instance profile list (`mig -lgip`) — exactly one profile, whole-GPU
```
+-------------------------------------------------------------------------------+
| GPU instance profiles:                                                        |
| GPU   Name               ID    Instances   Memory     P2P    SM    DEC   ENC  |
|                                Free/Total   GiB              CE    JPEG  OFA  |
|===============================================================================|
|   0  MIG 1g.64gb          0     1/1        63.00      No     70     0     0   |
|                                                               5     0     0   |
+-------------------------------------------------------------------------------+
```

### Create GPU instance + compute instance, then list it
```
$ sudo nvidia-smi mig -cgi 1g.64gb -C
Successfully created GPU instance ID  0 on GPU  0 using profile MIG 1g.64gb (ID  0)
Successfully created compute instance ID  0 on GPU  0 GPU instance ID  0 using profile MIG 1g.64gb (ID  0)

$ sudo nvidia-smi mig -lgi
|   0  MIG 1g.64gb            0        0          0:8     |     (GPU instance; placement start:size = 0:8)
```

### Compute throughput — the real delta (cuBLAS SGEMM 8192³ FP32 ×30, on the MIG instance)
```
base (unpatched):   17847 ms -> 1848 GFLOP/s
+ mig-unlock.patch:  3226 ms -> 10225 GFLOP/s    (~5.5x)
```
Same profile (`1g.64gb`), same card/host, single A/B difference = the patch. Process confirmed running **inside the instance** (patched run) — `nvidia-smi` Processes table:
```
|  GPU   GI   CI      PID   Type   Process name        GPU Memory |
|    0    0    0     4338      C   ./sgemm                1006MiB |
```
**The base number is not a warm-reboot artifact:** after a cold power-cycle, on a verified-unpatched driver (the `mig-unlock.patch` log signature is absent from `dmesg`), the base instance again creates and the same SGEMM runs at **1850 GFLOP/s** — matching the 1848 above:
```
$ CUDA_VISIBLE_DEVICES=<base-cold MIG dev> ./sgemm
SGEMM N=8192 x30: 17831.9 ms -> 1850 GFLOP/s
```
(Ref: full A100 ~19.5 TFLOP/s FP32; CMP is a cut-down GA100. FP32-SGEMM only; not an SM count or general MIG-performance claim.)

### 512 MiB memory round-trip (host→device→host, byte-verify, on the MIG device)
```
512 MiB round-trip (host->device->host, verify): OK
```

### Partial / multi-instance profiles — not available (base and patched)
Only `1g.64gb` (whole GPU) is in the RM profile list; every standard A100 profile is rejected. Verbatim:
```
$ sudo nvidia-smi mig -cgi 7g.40gb -C
Unable to create a GPU instance on GPU  0 using profile 7g.40gb: Invalid Argument
Failed to create GPU instances: Invalid Argument
$ sudo nvidia-smi mig -cgi 4g.20gb -C    → ... using profile 4g.20gb: Invalid Argument
$ sudo nvidia-smi mig -cgi 3g.20gb -C    → ... using profile 3g.20gb: Invalid Argument
$ sudo nvidia-smi mig -cgi 2g.10gb -C    → ... using profile 2g.10gb: Invalid Argument
$ sudo nvidia-smi mig -cgi 1g.10gb -C    → ... using profile 1g.10gb: Invalid Argument
$ sudo nvidia-smi mig -cgi 1g.5gb  -C    → ... using profile 1g.5gb:  Invalid Argument
$ sudo nvidia-smi mig -cgi 5 -C          → ... using profile 5:  Invalid Argument   (by ID)
$ sudo nvidia-smi mig -cgi 9 -C          → ... using profile 9:  Invalid Argument   (by ID)
$ sudo nvidia-smi mig -cgi 14 -C         → ... using profile 14: Invalid Argument   (by ID)
$ sudo nvidia-smi mig -cgi 19 -C         → ... using profile 19: Invalid Argument   (by ID)
$ sudo nvidia-smi mig -cgi 9,3g.20gb -C  → ... using profile 9:  Invalid Argument   (exact wiki attempt)
```

### `nvidia-smi -q` (MIG section, instance active) — verbatim
```
    MIG Mode
        Current                                        : Enabled
        Pending                                        : Enabled
    MIG Device
        Index                                          : 0
        GPU Instance ID                                : 0
        Compute Instance ID                            : 0
        Device Attributes
            Shared
                Multiprocessor count                   : 70
                Copy Engine count                      : 5
                Encoder count                          : 0
                Decoder count                          : 0
                OFA count                              : 0
                JPG count                              : 0
        Shared FB Memory Usage
            Total                                      : 64912 MiB
            Reserved                                   : 0 MiB
            Used                                       : 1 MiB
            Free                                       : 64912 MiB
        Shared BAR1 Memory
            Total                                      : 65536 MiB
            Used                                       : 0 MiB
            Free                                       : 65535 MiB
GPU UUID : GPU-<redacted>
```

## Observations

- MIG **enables and holds** (through create/destroy and driver swaps); returned cleanly to non-MIG via `-mig 0`.
- **`mig-unlock.patch`'s observable effect here is compute throughput, not enabling MIG or creating the instance** (the base already does both). It raises FP32-SGEMM under MIG **~5.5× (1848 → 10225 GFLOP/s)**, consistent with widened compute-instance capacity (commit: "14 SMs → whole GPU"). The active-SM count is **not** established — `-lgip`/`Shared MP count` report 70 SM with and without the patch and are not evidence of the real partition; only the throughput delta is measured.
- **No sub-partitioning / multiple instances** — the RM GA100 GPU-instance profile list contains only the whole-GPU `1g.64gb` entry; all standard A100 profiles return `Invalid Argument` (base and patched alike).
- No brick / no persistent hardware writes; fully reversible (module restore + `-mig 0` + reboot). Verified twice (warm revert, then cold power-cycle back to the base driver).

## References

- [Commit 86b238f — "Widen the MIG compute partition from 14 SMs to the whole GPU"](https://github.com/amoghmunikote/cmpunlocker/commit/86b238f0974aedf9bf61463c03d89ece4415d719)
- [mig-unlock.patch](https://github.com/amoghmunikote/cmpunlocker/blob/MIG/driver/patches/mig-unlock.patch)
- [Consensus-Protocol Wiki: Frontier-Open-Questions §1.5](https://github.com/Consensus-Protocol/cmp170hx/blob/main/docs/frontier/open-questions.md) · [Status-Board](https://github.com/Consensus-Protocol/cmp170hx/blob/main/docs/frontier/status-board.md)
- [abobasixseven/unlock-cmp-170hx — Issue #1](https://github.com/abobasixseven/unlock-cmp-170hx/issues/1)

## Next Steps (per Wiki)

1. Repeat on a second card — done here (CMP 170HX, driver 610.43.03). The patch applies, MIG enables, `1g.64gb` survives a reboot, and FP32 SGEMM under MIG rises ~5.5× (1848 to 10225 GFLOP/s), consistent with widened compute-instance capacity. Still to capture: the CUDA active-SM count and `-lci` profile data.
2. **Find where RM builds the GA100 GPU-instance profile list so more profiles can be added** ← confirmed as the limiting factor: only `1g.64gb` is listed; all sub-partitions return `Invalid Argument`, base and patched alike.
3. If the enable holds, open the pull request (maintainer has offered to merge). — Enable holds across create/destroy and reboots on 610.43.03.
