# LLM Architecture HW4 — Progress / Tasks

**Project:** Multimodal Models — Image Captioning & Visual Question Answering
**Notebook:** `LLM_Architecture_HW4.ipynb`
**Last updated:** 2026-06-08

---

## Environment

- [x] **FIXED:** replaced AMD/ROCm PyTorch with CUDA build — see [DEBUGGING.md](DEBUGGING.md)
  - `torch 2.11.0+cu128`, CUDA 12.8, RTX 5090 (sm_120) verified working
  - Removed orphaned `rocm-*` packages
- [x] BLIP model downloaded to `./blip_local/` (990 MB, complete)
- [x] Kernel healthy: model loads (~1.2 GB RAM, 684 MB GPU), GPU ops run instantly

---

## Task 1 — Image Captioning (30 pts)

- [x] cell-7: imports + BLIP model load (now runs on GPU in seconds)
- [x] cell-8: download 7 images — **FIXED 2026-06-08:** all 7 were failing
  (COCO S3 cert mismatch → `verify=False`; dropped dead Wikipedia thumbnail + 404 COCO `…338`).
  All 7 now download; verified captions end-to-end outside the notebook.
- [x] cell-9: generate captions + display grid — verified working (title fixed base vs large)
- [x] cell-10: display 2 failure-case images + structured report (saves task1_failures.png)
- [x] cell-11 (markdown): failure analysis written (DONE 2026-06-08, images viewed)
  - Failure Case 1 (`…037777`): "a kitchen with a table and chairs" — omits foreground fruit bowl (incomplete caption)
  - Failure Case 2 (`…000776`): "a brown teddy bear" — undercounts 3 bears → 1 (multiplicity/occlusion)

**Optional cleanups:**
- [x] Change deprecated `torch_dtype=` → `dtype=` in cell-7
- [ ] Add `%matplotlib inline` so plots render inline (avoid `plt.show()` blocking)

---

## Task 2 — Visual Question Answering (30 pts) — DONE (verified end-to-end)

- [x] `blip-vqa-base` downloaded to `./blip_vqa_local/` and loads on GPU
- [x] cell-17: load VQA model + `ask()` helper
- [x] cell-18: 3 images × 3 questions (object / counting / color) + display grid
  - all 9 answered correctly (cats/2/pink, bear/one/brown, skiing/1/red)
- [x] cell-19: 2 failure cases + display
  - Failure 1: "What is the dog doing?" on two-cats image → "sleeping" (no dog present)
  - Failure 2: "What color is the bear's collar?" on bear image → "black" (no collar)
  - root cause: false-presupposition / language-prior bias (analyzed in markdown cell)
- [x] cell-20 (markdown): full failure analysis written
- [ ] **TODO:** run cells 17→20 in the notebook to render the figures (verified working outside it)

---

## Task 3 — Personalized Face Recognition (40 pts) — DONE (verified end-to-end)

- [x] Dataset: **LFW via scikit-learn** (`fetch_lfw_people`, no manual download / no broken URLs)
- [x] CLIP downloaded to `./clip_local/` (`openai/clip-vit-base-patch32`)
- [x] 5 identities (Bush, Blair, Schroeder, Powell, Chavez), 6 reference + 3 query each, seed=0
- [x] cell-25: setup — load CLIP + LFW, build reference/query split, `clip_embed()`
- [x] cell-26: 1-NN cosine-similarity classification → **accuracy 13/15 ≈ 87%**
- [x] cell-27: combined CLIP(who)+VQA(attribute) answers + "Is this X?" demo + success/failure grid
- [x] cell-28 (markdown): analysis of successes + failures
  - failures: Bush→Schroeder, Blair→Schroeder (high-confidence cos≈0.92) — CLIP is a
    general (non-face) model; similar suited men collapse together; no rejection threshold
- [x] Ran end-to-end in the notebook kernel (Python 3.12): **accuracy 13/15 = 86.7%**

---

## Full notebook execution (2026-06-08)

- [x] Installed missing kernel deps into **Python 3.12** (the notebook kernel, NOT the
  3.13 used for offline checks): `scikit-learn`, `nbconvert`, `nbclient`.
- [x] Executed the whole notebook headlessly → **`LLM_Architecture_HW4_executed.ipynb`**
  (all outputs embedded) + figures `task1_results.png`, `task2_results.png`,
  `task2_failures.png`, `task3_results.png`. **0 errors**, 10 code cells with outputs.
- [x] Fixed an output-vs-text mismatch: bear-collar VQA answer is **"brown"** in the
  3.12 kernel (was "black" under 3.13/transformers 5.6.2). Updated Task 2 analysis
  (cell-20) in BOTH the source and executed notebooks to say "brown".

### Env note
Notebook kernel = Python 3.12.10, torch 2.11.0+cu128, **transformers 4.57.6**.
The Bash `python` = Python 3.13, transformers 5.6.2 — different env. Always install
notebook deps into the 3.12 interpreter at
`C:\Users\aikid\AppData\Local\Programs\Python\Python312\python.exe`.

### transformers 5.6.2 gotcha
`clip.get_image_features(**inp)` returns a `BaseModelOutputWithPooling`, not a tensor.
Use `out.pooler_output` (already the 512-d projected CLIP embedding — do NOT apply
`visual_projection` again, that expects 768-d and errors).

---

## Notes / Lessons

- If any cell runs longer than ~1 minute → something is wrong. Don't wait.
  Check kernel CPU: low CPU over time = blocked, not working. (full guide in [DEBUGGING.md](DEBUGGING.md))
- `torch.cuda.is_available() == True` is NOT proof the GPU works — run a real op.
- **`NameError: plt is not defined` (and similar) = cells run out of order.** After a
  kernel restart you must run cell-7 (imports/model) → cell-8 (downloads) → cell-9…
  in order. Jumping straight to a later cell always fails. Use *Restart Kernel → Run All*.
- **VS Code overwrites external notebook edits.** If the `.ipynb` is open in VS Code
  while edited on disk, VS Code's stale in-memory copy can be saved back and silently
  wipe the on-disk changes. Fix: after an external edit, run *Revert File*
  (Ctrl+Shift+P → "Revert File") to reload from disk, and never Ctrl+S the stale view.
  Best practice: close the notebook in the editor before it is edited programmatically.

---

## Session log

### 2026-06-08 — Task 2 built; download + ordering issues fixed
- **`plt` NameError** reported twice → root cause was running cell-9 without cell-7
  (kernel had no imports/model/images). Not a code bug; documented the run-in-order rule.
- **Task 1 image downloads were 100% failing** (caught while debugging):
  - COCO `images.cocodataset.org` → SSL hostname mismatch (CNAME to S3). Fixed with
    `verify=False` + `urllib3.disable_warnings`.
  - Wikipedia dog thumbnail → HTTP 400 (size/UA policy); COCO `…338` → 404.
    Replaced both with reachable COCO val2017 ids. All 7 verified downloading.
  - Also: `torch_dtype=`→`dtype=` (cell-7), figure title "BLIP-large"→"BLIP-base" (cell-9).
- **Task 2 (VQA) implemented and verified end-to-end** outside the notebook:
  downloaded `blip-vqa-base`, ran 3×3 Q&A (all correct), found 2 genuine failure cases
  by probing + visually inspecting the images, wrote the analysis. Cells 17–20.
- **VS Code overwrote all of the above once** (stale buffer saved over disk). Re-applied
  every edit; added the VS Code lesson above. Final notebook = 26 cells.
- **Open items:** run Tasks 1 & 2 in the notebook to render figures; Task 3 not started.
