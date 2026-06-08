# Debugging Log — BLIP cell hung for 9.5 hours

**Date:** 2026-06-06 / 07
**Project:** LLM Architecture HW4 — Multimodal (BLIP captioning / VQA)
**Symptom:** Cell-7 (model load) ran for **572 minutes (9.5 h)** and never finished.
**Actual fix runtime once corrected:** **~21.7 seconds.**

---

## TL;DR

The installed PyTorch was an **AMD/ROCm build** (`2.9.1+rocmsdk`) on a machine
with an **NVIDIA RTX 5090** GPU. ROCm hijacks the `torch.cuda.*` namespace, so
`torch.cuda.is_available()` returned `True` and the notebook printed
`Device: cuda` — but the first real GPU operation **blocked forever** because the
AMD runtime cannot drive an NVIDIA card.

**Fix:** uninstall the ROCm torch stack, install the CUDA 12.8 (`cu128`) build
that supports the RTX 5090's Blackwell architecture (`sm_120`).

---

## Timeline of the misdiagnosis

| Hypothesis | Why it was wrong |
|---|---|
| Model still downloading | `blip_local/` was complete — `pytorch_model.bin` 990 MB, no `.lock`/`.incomplete` |
| `plt.show()` blocking on a GUI backend | The stuck cell was **cell-7 (load)**, not the plotting cells |
| Slow network on model load | Local path load doesn't hit the network; folder already complete |
| **Wrong-vendor PyTorch (ROCm on NVIDIA)** | ✅ **Correct root cause** |

**Lesson:** the IDE cell timer (`148m… / 572m…`) pointed straight at the cell.
Read *which* cell is stuck before theorizing about other cells.

---

## How it was confirmed

### 1. The kernel process was doing nothing

```powershell
Get-Process python* | Select Id, @{N='RAM(MB)';E={[math]::Round($_.WorkingSet64/1MB)}}, @{N='CPU(s)';E={[math]::Round($_.CPU)}}, StartTime
```

The kernel (PID 81420) had used only **32 s of CPU in 9.5 h** and **149 MB RAM**.
A real model load eats GB of RAM and minutes of CPU → it was **blocked on a dead
syscall**, not computing.

### 2. The smoking gun — wrong GPU vendor

```powershell
python -c "import torch; print(torch.__version__); print(torch.cuda.get_arch_list())"
```

```
torch: 2.9.1+rocmsdk20260116        <-- ROCm = AMD
arch list: ['gfx1100', 'gfx1201', ...]   <-- gfx* = AMD GPU targets
device cap: (11, 5)                  <-- bogus for an RTX 5090
```

`+rocmsdk` and `gfx*` are AMD. The machine has an NVIDIA RTX 5090 → total mismatch.

---

## The fix (Path B)

```powershell
$py = "C:\Users\aikid\AppData\Local\Programs\Python\Python312\python.exe"

# 1. Remove the AMD/ROCm stack
& $py -m pip uninstall -y torch torchvision torchaudio
& $py -m pip uninstall -y rocm rocm-sdk-core rocm-sdk-devel rocm-sdk-libraries-custom

# 2. Install the CUDA 12.8 build (Blackwell / sm_120 support)
& $py -m pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128
```

### Verification — before vs after

| Check | Before (broken) | After (fixed) |
|---|---|---|
| Build | `2.9.1+rocmsdk` (AMD) | `2.11.0+cu128` (NVIDIA CUDA 12.8) |
| Arch list | `gfx1100…` | `sm_75 … sm_100, sm_120` ✅ |
| GPU capability | `(11, 5)` bogus | `(12, 0)` = real RTX 5090 |
| Real GPU op | hung 9.5 h | matmul ran instantly ✅ |

Verify command:

```powershell
python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available()); print(torch.cuda.get_arch_list()); print(torch.cuda.get_device_name(0), torch.cuda.get_device_capability(0)); x=torch.randn(1000,1000,device='cuda'); print((x@x).sum().item())"
```

Expect: `2.11.0+cu128 12.8 True`, arch list contains **`sm_120`**, GPU name
**RTX 5090**, capability **(12, 0)**, and the matmul returns a number.

> **After fixing, you must Restart the Kernel** so the notebook loads the new torch.

---

## Full session timeline (every step we took)

1. **Initial report:** "why is this cell running for 96 minutes?" — no idea which cell yet.
2. **Inventoried the project:** found `LLM_Architecture_HW4.ipynb` + a `blip_local/` folder.
3. **Checked the download folder** — `blip_local/pytorch_model.bin` = 990 MB, all config
   files present, no `.lock`/`.incomplete`. → Download was **complete**, not the cause.
4. **First (wrong) theory:** `plt.show()` blocking on a GUI matplotlib backend in
   cells 9–10. Proposed `%matplotlib inline`.
5. **New evidence:** IDE screenshot showed the stuck cell was **cell-7 (model load)**
   at `148m 43.4s`, not a plotting cell → theory discarded.
6. **Second (wrong) theory:** RTX 5090 is Blackwell (`sm_120`); maybe an old CUDA
   build hangs on first GPU op. Proposed an offline-mode + CPU-first rewrite of cell-7.
7. **Time jumped to 572 min (9.5 h)** → confirmed a wedged kernel, not slowness.
8. **Inspected kernel process:**
   ```powershell
   Get-Process python* | Select Id, WorkingSet64, CPU, StartTime
   ```
   PID **81420**, started 9:33 PM, **32 s CPU / 149 MB RAM over 9.5 h** → blocked, not computing.
9. **Force-killed the wedged kernel:**
   ```powershell
   Stop-Process -Id 81420 -Force   # -> "Kernel 81420 killed."
   ```
10. **Checked the GPU** with `nvidia-smi` → 0 MiB compute used, RTX 5090, GPU healthy.
11. **Smoking gun** — inspected the torch build:
    ```
    torch: 2.9.1+rocmsdk20260116   |  arch list: ['gfx1100', 'gfx1201', ...]
    ```
    → **AMD/ROCm PyTorch on an NVIDIA card.** Real root cause found.
12. **Confirmed packages** (system Python, not a venv):
    `torch/torchvision/torchaudio` all `+rocmsdk`, plus `rocm`, `rocm-sdk-core`,
    `rocm-sdk-devel`, `rocm-sdk-libraries-custom` (7.2.0.dev0).
13. **Uninstalled ROCm torch:**
    ```powershell
    pip uninstall -y torch torchvision torchaudio
    ```
14. **Installed CUDA 12.8 build** (background, ~2.75 GB download):
    ```powershell
    pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128
    ```
    Pulled `torch 2.11.0+cu128`, `torchvision 0.26.0+cu128`, `torchaudio 2.11.0+cu128`.
15. **Verified the fix:** `arch list` now includes `sm_120`, capability `(12,0)`,
    GPU name "RTX 5090 Laptop", and a real `x @ x` matmul ran instantly.
16. **Cleanup:** uninstalled the now-orphaned ROCm SDK packages:
    ```powershell
    pip uninstall -y rocm rocm-sdk-core rocm-sdk-devel rocm-sdk-libraries-custom
    ```
17. **Final check:** no `rocm` packages remain; torch still `2.11.0+cu128`, CUDA `True`,
    RTX 5090 detected. ✅

### Net changes to the environment

| Package | Before | After |
|---|---|---|
| `torch` | `2.9.1+rocmsdk20260116` | `2.11.0+cu128` |
| `torchvision` | `0.24.1+rocmsdk20260116` | `0.26.0+cu128` |
| `torchaudio` | `2.9.1+rocmsdk20260116` | `2.11.0+cu128` |
| `rocm`, `rocm-sdk-core`, `rocm-sdk-devel`, `rocm-sdk-libraries-custom` | `7.2.0.dev0` | **removed** |

No notebook code was required to change for the fix; the only optional code
improvements are the `dtype=` (vs deprecated `torch_dtype=`) cleanup and the
fail-fast guardrail below.

---

## Root-cause notes

- **RTX 5090 = Blackwell = `sm_120`.** Needs CUDA **12.8+** wheels (`cu128` or newer).
  Older CUDA builds (and any ROCm build) will not run on it.
- **`torch.cuda.is_available() == True` is NOT proof the GPU works.** ROCm reuses
  the `torch.cuda` namespace; a stale/old CUDA build may also report `True` but
  hang/error on the first kernel launch. Always run a real op (`x @ x` on `cuda`).
- The AMD wheels were almost certainly pulled by a bare `pip install torch` that
  resolved to a ROCm index, or an AMD-targeted install command.

---

## Guardrail: fail fast instead of hanging forever

Add this near the top of the notebook so a vendor/arch mismatch errors in
**seconds** instead of hanging for hours:

```python
import torch

device = "cuda" if torch.cuda.is_available() else "cpu"
print(f"Device: {device} | torch {torch.__version__} (cuda build: {torch.version.cuda})")

if device == "cuda":
    assert torch.version.cuda is not None, (
        "Non-CUDA PyTorch build detected (likely ROCm/AMD). "
        "Install the CUDA build: pip install torch --index-url https://download.pytorch.org/whl/cu128"
    )
    cap = torch.cuda.get_device_capability(0)
    arch = torch.cuda.get_arch_list()
    needed = f"sm_{cap[0]}{cap[1]}"
    print(f"GPU: {torch.cuda.get_device_name(0)} | cap {cap} | arch list: {arch}")
    assert needed in arch, (
        f"This PyTorch was not built for your GPU ({needed} missing from {arch}). "
        f"An RTX 5090 (sm_120) needs the cu128+ build."
    )
    # Smoke test: real GPU op — fails fast if the runtime can't drive the GPU
    _ = (torch.randn(256, 256, device="cuda") @ torch.randn(256, 256, device="cuda")).sum().item()
    print("GPU smoke test passed.")
```

---

## Checklist if a GPU cell hangs again

1. Look at **which** cell shows `[*]` / the IDE cell timer — don't guess.
2. Check the kernel process CPU/RAM. **Low CPU over a long time = blocked, not working.**
3. Print `torch.__version__`, `torch.version.cuda`, `torch.cuda.get_arch_list()`.
   - `+rocmsdk` or `gfx*` → wrong vendor (AMD build on NVIDIA).
   - Your GPU's `sm_XXX` missing from the arch list → wrong/old CUDA build.
4. Run a real GPU op (`x @ x` on `cuda`) — never trust `is_available()` alone.
5. Force-kill the wedged kernel (`Stop-Process -Id <pid> -Force`); Restart won't always work.
