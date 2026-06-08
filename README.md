# LLM Architecture HW4 — Multimodal Models

Image Captioning, Visual Question Answering, and personalized face recognition with
pretrained vision–language models. All work is in
[`LLM_Architecture_HW4.ipynb`](LLM_Architecture_HW4.ipynb).

## Tasks

| Task | What it does | Model |
|---|---|---|
| **1 — Image Captioning** | Caption 7 images (people / animal / indoor / outdoor / multi-object) + 2 failure cases | `Salesforce/blip-image-captioning-base` |
| **2 — Visual Question Answering** | 3 images × 3 questions (object / counting / colour) + 3 analyzed failures | `Salesforce/blip-vqa-base` |
| **3 — Personalized Face Recognition** | CLIP identity (1-NN cosine) on 5 LFW celebrities + open-set rejection, combined with VQA attributes | `openai/clip-vit-base-patch32` + BLIP-VQA |

## Results (from the executed notebook)

- **Task 1** — captions for all 7 images; failures: a kitchen caption that omits the
  foreground fruit bowl, and *"a brown teddy bear"* for an image of three bears.
- **Task 2** — all 9 demo Q&A correct; failures: two false-presupposition cases
  (a non-existent dog, a non-existent collar) and one genuine miscount (2 workers → "3").
- **Task 3** — **identification accuracy 13/15 ≈ 87%**; the out-of-gallery impostor is
  rejected via a cosine threshold; failures are high-confidence look-alike confusions
  (Bush/Blair → Schroeder), discussed in the notebook.

## Setup

```bash
# Python 3.12. Install the CUDA build of PyTorch (RTX 5090 = sm_120 needs cu128+):
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu128
pip install -r requirements.txt
```

Then open the notebook and **Run All**. Models download automatically to
`./blip_local`, `./blip_vqa_local`, `./clip_local` on first run (~4.4 GB total, git-ignored);
the LFW dataset is fetched by scikit-learn and cached locally.

## Repo contents

- `LLM_Architecture_HW4.ipynb` — the assignment notebook (with outputs)
- `task1_results.png`, `task2_results.png`, `task2_failures.png`, `task3_results.png` — result figures
- `DEBUGGING.md` — log of a ROCm-vs-CUDA PyTorch issue and its fix
- `tasks.md` — progress notes
- `requirements.txt` — dependencies

> Note: model weight folders are excluded via `.gitignore` (too large for Git; re-downloaded on first run).
