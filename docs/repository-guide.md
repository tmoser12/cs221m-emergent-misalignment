# Repository guide

Map of the code, data and results. The project write-up and poster are in
[`../README.md`](../README.md).

## Layout

```
data/               Committed inputs
  nanda_training/     3 EM fine-tuning datasets (+ extras/)
  eval_questions/     Betley-style misalignment eval questions

src/                Shared library (scripts put this on sys.path)
  helpers.py          Model registry + fp16 loader + decoder-layer locator
  andy_sae.py         andyrdt BatchTopK SAE: encode/decode, steering, ablation, feature sets
  steering_vec.py     Diff-in-means directions + DiffMeanSteer intervention API
  judge_prompts.py    Verbatim alignment/coherence judge prompts (Model Organisms paper)

scripts/            Runnable pipelines, one directory per stage
  activations/        Collect residual-stream activations (training + eval prompts)
  saes/               Download SAEs; compute latent deltas; decoder-basis analysis
  latents/            Neuronpedia descriptions + LLM-as-a-judge misalignment filter
  evals/              Generation + GPT-judged alignment/coherence eval, steer/ablate
  icl/                ICL-vs-fine-tuning OOD drift experiment
  analysis/           A1/A2/A4 geometry, ablation and steering bar charts, vibecheck
  figures/            Report figure generation
  generate.py         Minimal single-prompt generation demo

results/            Committed outputs — see results/README.md
docs/               Method write-ups and per-experiment results — see below
figures/            Report figures (png/pdf/svg)
```

Not committed (gitignored, regenerate locally): `models/`, `SAEs/`, `activations/`,
`.venv/`, `.venv-judge/`, `.env`.

## Documentation

| Doc | Covers |
|---|---|
| [1-diffmean-steering-and-evals.md](1-diffmean-steering-and-evals.md) | Diff-in-means steering code, activation collection, the judged eval and its flags |
| [2-geometry-and-ablations.md](2-geometry-and-ablations.md) | A1/A2/A4 geometry results, the ablation dissociation, causal-intervention practice |
| [3-shared-latents.md](3-shared-latents.md) | Shared latents across all three finetunes (full example-weighted run) |
| [4-sae-latent-deltas.md](4-sae-latent-deltas.md) | Activation-delta method vs the cosine method, and why they disagree |
| [5-icl-vs-finetuning-spec.md](5-icl-vs-finetuning-spec.md) | ICL-vs-FT experiment specification |
| [6-icl-vs-finetuning-results.md](6-icl-vs-finetuning-results.md) | ICL-vs-FT results: residual cosine + SAE Jaccard per layer |
| [gpu_spec.md](gpu_spec.md) | Quadro RTX 6000 (Turing) profile — why everything is fp16, not bf16 |

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt --extra-index-url https://download.pytorch.org/whl/cu121

# Separate venv for the DSPy/OpenRouter latent judge (conflicting torch range)
python -m venv .venv-judge && source .venv-judge/bin/activate
pip install -r requirements-judge.txt
```

Put `OPENAI_API_KEY` (eval judge), `OPENROUTER_API_KEY` (latent judge) and
`NEURONPEDIA_API_KEY` (feature descriptions) in a `.env` at the repo root.
Place model weights under `models/` and SAE checkpoints under `SAEs/`
(the andyrdt SAE loader in `src/andy_sae.py` reads `models/SAEs/` — see the
note in `scripts/saes/download_andy_saes.py`).

## Pipelines

```bash
# 1. Collect last-prompt-token activations for a finetune vs the base model
bash scripts/activations/run_collect_training_activations.sh --limit 2000

# 2. Diff-in-means direction -> results/steering_vectors/
python src/steering_vec.py --dataset all

# 3. Judged misalignment + coherence eval (baseline / steer / ablate)
bash scripts/evals/run_misalignment_eval.sh --model bad-medical-advice --n 50
bash scripts/evals/run_misalignment_eval.sh --model bad-medical-advice --set-ablate shared

# 4. SAE activation deltas at the last prompt token
bash scripts/saes/run_prompt_token_latent_deltas.sh

# 5. Neuronpedia descriptions -> LLM-judge misalignment filter
bash scripts/latents/run_sae_delta_latent_pipeline.sh

# 6. ICL vs fine-tuning drift (two GPU phases)
bash scripts/icl/run_collect_ood_activations.sh
bash scripts/icl/run_aggregate_and_analyze.sh

# 7. Geometry figures (CPU, seconds to minutes)
python scripts/analysis/a1_diffmean_cosine.py
python scripts/analysis/a2_subspace_capture.py
python scripts/analysis/ablation_bars.py
```

The GPU runners wrap `srun -p gpu-turing --gres=gpu:1`; drop the `srun` prefix to
run on a local GPU.
