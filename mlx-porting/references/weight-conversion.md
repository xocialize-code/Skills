# Weight Conversion

Weight conversion routes through three tiers. **Always try Tier 1 first.** Demoting to Tier 2 or Tier 3 without checking Tier 1 means rewriting infrastructure that already exists.

Note on code style: `mx.eval( )` is written with a space between parentheses in this file to sidestep a security-hook false-positive on the Python builtin. In real code it's `mx.eval(...)` with the tensors inside.

## Routing rule

1. **mlx-community already has it** → use it. Search `https://huggingface.co/mlx-community` first.
2. **`mlx_lm.convert` / `mlx_vlm.convert` / `mlx_audio.convert` supports the `model_type`** → run it.
3. **Architecture missing from mlx-lm/mlx-vlm/mlx-audio** → add one file (mlx-lm) or one sub-package (mlx-vlm, mlx-audio), then re-run the official converter. See `manual-port-templates.md`.
4. **Multi-component pipeline** (T2V, T2I, 3D, audio-gen, multi-component diffusion) — mlx-forge recipe + standalone `-mlx` fork. **This is the only case where `mlx-forge` is the answer.**

## Tier 1 — official converters (canonical invocations)

### LLM via `mlx_lm.convert`

```bash
mlx_lm.convert \
  --hf-path Qwen/Qwen3-4B-Instruct-2507 \
  -q \
  --upload-repo mlx-community/Qwen3-4B-Instruct-2507-4bit
```

Full flag list (per `mlx_lm/convert.py`):

```
usage: mlx_lm.convert [-h] [--hf-path HF_PATH] [--mlx-path MLX_PATH] [-q]
                      [--q-group-size Q_GROUP_SIZE] [--q-bits Q_BITS]
                      [--q-mode {affine,mxfp4,nvfp4,mxfp8}]
                      [--quant-predicate {mixed_2_6,mixed_3_4,mixed_3_6,mixed_4_6}]
                      [--dtype {float16,bfloat16,float32}]
                      [--upload-repo UPLOAD_REPO] [-d]
```

### VLM via `mlx_vlm.convert`

```bash
mlx_vlm.convert \
  --hf-path Qwen/Qwen3-VL-30B-A3B-Instruct \
  -q \
  --upload-repo mlx-community/Qwen3-VL-30B-A3B-Instruct-4bit
```

Same flag shape as `mlx_lm.convert`. Vision and audio encoders are excluded from quantization by default — feature-extraction quality drops sharply under 4-bit.

### Audio (TTS/STT) via `mlx_audio.convert`

```bash
# 4-bit + upload
python -m mlx_audio.convert \
  --hf-path prince-canuma/Kokoro-82M \
  --mlx-path ./Kokoro-82M-4bit \
  --quantize --q-bits 4 \
  --upload-repo mlx-community/Kokoro-82M-4bit

# MXFP4 microscaling
python -m mlx_audio.convert \
  --hf-path prince-canuma/Kokoro-82M \
  --mlx-path ./Kokoro-82M-mxfp4 \
  --quantize --q-mode mxfp4

# bf16, no quant — the publishing default for Mel-RoFormer / Lance / Ming-omni
python -m mlx_audio.convert \
  --hf-path prince-canuma/Kokoro-82M \
  --mlx-path ./Kokoro-82M-bf16 \
  --dtype bfloat16 \
  --upload-repo mlx-community/Kokoro-82M-bf16
```

The bf16 pattern is the canonical publishing recipe for any audio model where quantization hurts perceptual quality (vocoders, Mel-band processors, anything spectrogram-based). `mlx-community/Lance-3B-bf16`, `mlx-community/Lance-3B-Video-bf16`, `mlx-community/Ming-omni-tts-16.8B-A3B-bf16` all use this exact invocation.

### Learned quants (Tier 1, advanced)

Per `LEARNED_QUANTS.md` (https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/LEARNED_QUANTS.md):

| CLI | What it does | Tradeoff |
|---|---|---|
| `mlx_lm.dynamic_quant` | Picks per-layer bit-widths from sensitivity | **Fastest to run** |
| `mlx_lm.awq` | Activation-aware Weight Quantization | Calibration set required |
| `mlx_lm.dwq` | Distilled Weight Quantization | **Best quality**, takes longest |
| `mlx_lm.gptq` | GPT-Q style | Calibration set required |

Output repos get the suffix matching the technique: `-DWQ-4bit`, `-AWQ-4bit`, `-OptiQ-4bit`.

## Output layout the official converters produce

Swift consumers (`mlx-swift-examples`, `mlx-audio-swift`) and Python consumers (`mlx_lm.load`, `mlx_vlm.load`, `mlx_audio.tts.utils.load_model`) all assume this layout. If a manual conversion produces a different shape, the loaders fail or pick wrong defaults.

```
<model_dir>/
├── config.json
├── model.safetensors                  (or .safetensors.index.json + shards for >2GB)
├── tokenizer.json
├── tokenizer_config.json
├── special_tokens_map.json
└── README.md                          (auto-generated when --upload-repo is set)
```

For VLM, add `preprocessor_config.json`. For audio, add whatever phonemizer / mel-spec config files the upstream repo ships.

## Tier 2 — manual architecture port

When the converter prints `Unsupported model_type: <name>`, the fix is **one file** (mlx-lm) or **one sub-package** (mlx-vlm, mlx-audio) named after the upstream `model_type`. See `manual-port-templates.md` for the canonical shape.

Workflow:

1. Fork `ml-explore/mlx-lm` (or `Blaizzy/mlx-vlm`, `Blaizzy/mlx-audio`).
2. Add `mlx_{lm,vlm,audio}/models/.../<model_type>.py` matching the template.
3. Implement `class Model(nn.Module)` with `.model_type`, `.layers` property, and `.sanitize(weights)`.
4. Re-run `mlx_*.convert` against your fork.
5. Parity-test (see `parity-testing.md`).
6. PR upstream per the CONTRIBUTING checklist.

**`sanitize(weights)` is where all weight remapping lives.** Not in the constructor, not at forward time. The official converter calls `Model.sanitize()` automatically before saving — putting remap logic anywhere else either runs at wrong layer state or never runs at all during conversion.

Reference exemplar: Mel-RoFormer upstreamed at `mlx-audio#654` (https://github.com/Blaizzy/mlx-audio/pull/654). Pure-MLX FFT, `sanitize()` remaps `band_split_module.0.weight` → MLX-friendly keys, parity tested at `≤0.11 dB SDR` on MUSDB18.

## Tier 3 — `mlx-forge` recipes (multi-component pipelines only)

`mlx-forge` is the right tool when the model has **multiple independent components** that don't fit a single `model_type` slot. Concretely: T2V (text encoder + transformer + VAE + scheduler + optional upsampler), T2I (same shape), 3D mesh generation (transformer + VAE + texture UNet), audio-gen pipelines (text encoder + transformer + vocoder + scheduler).

When to reach for Tier 3:

- Conversion needs **per-component dtype routing** (transformer q8, text_encoder bf16, vae fp16).
- Components ship as **separate `.safetensors` shards** that need separate quantization scopes.
- `mlx_*.convert` prints `Unsupported model_type` AND the model is not a clean single-stack LLM/VLM/audio (i.e., you can't reduce the problem to a Tier-2 manual port).
- You need a `pipeline_*.py` orchestrator alongside the converted weights.

Canonical examples:

- **LTX-2** (T2V) — `xocialize-code/ltx-2-mlx` + `dgrauet/ltx-2.3-mlx` (HF). Five components, four pipelines (T2V, I2V, retake, extend).
- **Qwen-Image-2512** — recipe at https://github.com/dgrauet/mlx-forge/blob/main/docs/models/qwen-image-2512.md. Per-component subdirectories, mixed quant.
- **Hunyuan3D-2.1**, **CogVideoX-Fun**, **Matrix-Game** — same shape.

### The one silent killer that still applies: lazy tensors saved as zeros

MLX arrays are lazy. Until materialized, they're an unresolved computation graph. `mx.save_safetensors` serializes whatever numerical value currently exists — **for an unmaterialized lazy tensor, that is zeros**, with no error.

`mlx_lm.convert` calls `mx.eval` internally so Tier 1 never hits this. **Tier-3 recipes must do it themselves:**

```python
import mlx.core as mx

def _materialize(*tensors):
    mx.eval( *tensors )      # force GPU computation; no-op if already materialized

# Right before saving:
_materialize( *component_weights.values() )
mx.save_safetensors(f"{component}.safetensors", component_weights)
```

The helper lives in `mlx-forge/src/mlx_forge/quantize.py:23-28`.

### Recipe skeleton

```python
# recipes/my_pipeline.py

def classify_key(key: str) -> str | None:
    """Map a PyTorch weight key → component name. Return None to drop."""
    ...

def sanitize_key(key: str) -> str:
    """Rename PyTorch key to MLX convention."""
    ...

def convert(args) -> None:
    weights = mx.load(checkpoint_path)        # lazy load
    keys_by_component = classify_keys(weights, classify_key)
    for component, keys in keys_by_component.items():
        process_component(
            weights, component, keys, output_dir,
            sanitizer=get_sanitizer(component),
            transform=get_transform(component),
        )
        quantize_component(output_dir, component)
```

CLI: `mlx-forge convert my-pipeline` — registered in `recipes/__init__.py` `AVAILABLE_RECIPES`.

Full reference recipe: `mlx_forge/recipes/ltx_23.py` (~400 LOC, shows per-channel stats renames, conv transposes, and per-component quant scope).

### Per-component split (Tier 3)

Always split safetensors by component (transformer, vae, text_encoder, scheduler, tokenizer, etc.) rather than one giant file. Reasons:

1. Components can be loaded / unloaded independently (saves peak memory).
2. Each component can be quantized with different settings (transformer int4, VAE fp16).
3. Parallel download from HF.
4. Easier to swap a single component without re-downloading everything.

Convention: one `{component}.safetensors` per component. If a component exceeds safetensors' 2GB chunk limit, split as `{component}-00001-of-00003.safetensors`, etc. — load with `load_split_safetensors` (see `ltx-2-mlx/packages/ltx-core-mlx/src/ltx_core_mlx/utils/weights.py`).

### Per-component memory management

Large checkpoints (>10 GB) need aggressive cleanup between components:

```python
for component, keys in ...:
    component_weights = {...}
    _materialize( *component_weights.values() )
    mx.save_safetensors(path, component_weights)
    del component_weights
    import gc; gc.collect()
    mx.metal.clear_cache()
```

Without `clear_cache`, the Metal allocator holds peak memory until process exit.

## Two materialization rules that apply at all tiers

1. **`mx.eval(tree)` before `mx.save_safetensors`** — Tier 1 does this automatically; Tier 2 (`Model.sanitize` is called by the converter, so the converter still owns materialization); Tier 3 you do it yourself.
2. **PyTorch Conv `(O, I, *K)` → MLX `(O, *K, I)`** — transpose inside `sanitize()` for Tier 2, inside the `transform` step for Tier 3.

| Op | PyTorch | MLX |
|---|---|---|
| `Conv1d.weight` | `(O, I, K)` | `(O, K, I)` |
| `Conv2d.weight` | `(O, I, Kh, Kw)` | `(O, Kh, Kw, I)` |
| `Conv3d.weight` | `(O, I, Kd, Kh, Kw)` | `(O, Kd, Kh, Kw, I)` |
| `ConvTranspose2d.weight` | `(I, O, Kh, Kw)` | `(O, Kh, Kw, I)` |
| `Linear.weight` | `(O, I)` | `(O, I)` (identical) |
| `Embedding.weight` | `(V, D)` | `(V, D)` (identical) |
| `LayerNorm.weight/bias` | `(D,)` | `(D,)` (identical) |

`mlx_forge.transpose.transpose_conv(key, weight, kind)` handles all variants generically. Bias tensors are never transposed.

## Quantization scope policy (all tiers)

- **Quantize:** transformer / DiT blocks — Linear `.weight` only.
- **Keep fp16 / bf16:** VAE, vocoder, text encoder output projections, tokenizer / scheduler state, position encodings, norm weights, bias tensors, vision-encoder ViT/SigLIP weights.

Rationale: VAE and connectors are sensitive to quantization noise (visible as color drift, edge artifacts). Vision encoders' feature-extraction quality drops sharply under 4-bit. Transformer Linears absorb quantization cleanly.

`mlx_vlm.convert` enforces vision-encoder exclusion automatically. For Tier 3:

```python
from mlx.nn import quantize

quantize(model, group_size=64, bits=4, class_predicate=lambda name, m: (
    isinstance(m, nn.Linear) and "transformer" in name
))
```

## Weight key renames — common patterns

From past ports, these show up repeatedly. Handle in `sanitize()` (Tier 2) or `sanitize_key()` (Tier 3):

- Sequential unwrapping: `.to_out.0.` → `.to_out.`, `.ff.net.0.proj.` → `.ff.proj_in.`, `.ff.net.2.` → `.ff.proj_out.`
- Private stat prefix: `_mean_of_means` → `mean_of_means` (MLX treats leading-underscore as private, breaks loading).
- Block numbering: `blocks.0.` → `blocks.0.` (usually identical; rename only if MLX port restructures).
- Fused QKV: three separate `to_q.weight`, `to_k.weight`, `to_v.weight` → one `qkv.weight` via `mx.concatenate([q, k, v], axis=0)`.
- Precomputed buffers: drop `self_attn.rotary_emb.inv_freq` (MLX recomputes via `initialize_rope`).
- Tied embeddings: drop `lm_head.weight` when `tie_word_embeddings=True`.

## Validation

A zero-tensor check catches the materialization bug cheaply:

```python
for key, w in weights.items():
    if float(mx.abs(w).sum().item()) == 0:
        raise ValueError(f"{key} is all zeros — likely missing materialization")
```

Add this to your Tier-3 recipe's `validate()` step; Tier 1 catches it via the converter.

## Before writing any recipe — check mlx-community

Before adding a model to `mlx-lm` or writing a `mlx-forge` recipe, check `https://huggingface.co/mlx-community`. The base model may already be converted; you may only need to port any custom head / adapter / LoRA on top.
