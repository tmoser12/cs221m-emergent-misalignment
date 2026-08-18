# `results/` — what each directory holds

Every subdirectory is committed output from a pipeline in [`../scripts/`](../scripts/).
Large intermediates (`models/`, `SAEs/`, `activations/`) are gitignored and must be
regenerated; everything below is the durable record of the runs.

| Directory | Produced by | Contents |
|---|---|---|
| [`steering_vectors/`](steering_vectors/) | [`scripts/activations/collect_training_activations.py`](../scripts/activations/collect_training_activations.py) → `src/steering_vec.py` | Diff-in-means (DIM) directions, one `.pt` per misaligned model |
| [`similar_latents/`](similar_latents/) | `scripts/saes/sae_basis_analysis.py` | Top-100 SAE decoder columns per layer by cosine to each DIM vector, plus `overlap.jsonl` |
| [`latent_descriptions/`](latent_descriptions/) | [`scripts/latents/`](../scripts/latents/) | Neuronpedia descriptions for those latents + the LLM-judge misalignment verdicts |
| [`sae_latent_deltas/`](sae_latent_deltas/) | [`scripts/saes/compute_prompt_token_latent_deltas.py`](../scripts/saes/compute_prompt_token_latent_deltas.py) | Final activation-delta run (instruct BatchTopK SAEs, last prompt token, layers 3–27) |
| [`sae_latent_deltas_full/`](sae_latent_deltas_full/) | [`scripts/saes/compute_*_latent_deltas.py`](../scripts/saes/) | Earlier full-scale delta sweeps under three weightings (`full_example_weighted`, `full_token_weighted`, `full_instruct_example_weighted`) plus `instruct_prompt_token` |
| [`misalignment_evals/`](misalignment_evals/) | [`scripts/evals/misalignment_eval.py`](../scripts/evals/misalignment_eval.py) | Judged alignment + coherence metrics per model, per intervention (plain / steer / ablate / set-ablate) |
| [`model_responses/`](model_responses/) | [`scripts/evals/prompt_llama.py`](../scripts/evals/prompt_llama.py), `generate_instruct_nanda_responses.py` | Raw generations and GPT-judged generations on the Betley eval questions |
| [`icl_drift/`](icl_drift/) | [`scripts/icl/`](../scripts/icl/) | ICL-vs-fine-tuning drift: per-layer residual cosine, SAE Jaccard, top features |
| [`geometry/`](geometry/) | [`scripts/analysis/`](../scripts/analysis/) | A1/A2/A4 geometry figures + JSON, ablation and steering bar charts |
| [`decoder_columns/`](decoder_columns/) | `scripts/saes/print_decoder_column.py` | Raw decoder column dumps for L11 f39163 (the "dieting advice" latent) |

## `steering_vectors/`

Three `.pt` files, one per misaligned model. Each holds the difference in mean
residual-stream vector at each layer between the base Llama model and the
misaligned model. The vectors were computed by taking the residual stream vector at
the last input prompt token at each layer for both the misaligned and aligned
models across the entire corresponding Nanda dataset, averaging, then subtracting
the two.

## `similar_latents/`

Three JSONL files, one per misaligned model. Each contains a list of the top 100
most similar SAE feature vectors per layer to the steering vectors. `overlap.jsonl`
holds the (layer, feature) pairs that appear in all three lists.

## Regenerable artifacts not committed

`results/icl_drift/mean_activations.pt` and every file under `activations/` are
gitignored (`*.pt`). Re-create them with the runners listed above; the JSON/JSONL
summaries that depend on them are committed.
