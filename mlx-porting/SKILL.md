---
name: mlx-porting
description: Port PyTorch / CUDA models — LLMs, VLMs, TTS/STT, audio, diffusion — to Apple MLX or MLX-Swift. Routes weight conversion through the official `mlx_lm.convert` / `mlx_vlm.convert` / `mlx_audio.convert` CLIs with mlx-community HF conventions; produces upstream-mergeable model files under `mlx_lm/models/`, `mlx_vlm/models/`, `mlx_audio/{tts,stt}/models/`; falls back to `mlx-forge` only for multi-component diffusion pipelines (T2V, T2I, 3D, audio-gen). Invoke when porting to MLX, publishing to mlx-community, adding a `model_type` to mlx-lm/vlm/audio, translating attention / RoPE / VAE / norm layers, running parity tests, diagnosing silent failures (black images, cyan textures, gray output, garbage tokens, metallib-not-found), or writing an mlx-forge recipe. Triggers — "port to MLX", "MLX-Swift port", "metallib not found", "publish to mlx-community", "mlx_lm.convert", "-mlx fork", "MLX parity", "mlx-forge recipe". Invoke eagerly — MLX ports fail silently. Skip for general MLX API questions.
---

# Porting PyTorch / CUDA to Apple MLX

## Mission

This skill captures the workflow and pitfalls accumulated across ~10 production MLX ports (LTX-2, CogVideoX-Fun, Matrix-Game, VOID, Fish S2 Pro, Mistral Small, Qwen Image, Hunyuan3D-2.1, Mel-RoFormer, Lance, Ming-omni, …). It is for **inference-only ports** on Apple Silicon, publishing to **mlx-community** as the default and falling back to the **mlx-forge recipe + `-mlx` fork** convention only for multi-component pipelines.

## Routing — three tiers (decide this BEFORE writing code)

Most ports land at Tier 1. The point of the routing rule is to stop you from rebuilding what already exists.

**Tier 1 — official converters (default for every port):**

```bash
# LLM
mlx_lm.convert --hf-path <upstream> -q --upload-repo mlx-community/<name>-4bit

# VLM
mlx_vlm.convert --hf-path <upstream> -q --upload-repo mlx-community/<name>-4bit

# Audio (TTS / STT). bf16 is the default for vocoders / spectrogram models.
python -m mlx_audio.convert --hf-path <upstream> --dtype bfloat16 \
    --upload-repo mlx-community/<name>-bf16
```

Runtime: `from mlx_lm import load, generate` | `from mlx_vlm import load, generate` | `from mlx_audio.tts.utils import load_model`. Browser fallback: https://huggingface.co/spaces/mlx-community/mlx-my-repo.

**Tier 2 — manual architecture port (when `model_type` is not yet supported):**

Fork `ml-explore/mlx-lm`, `Blaizzy/mlx-vlm`, or `Blaizzy/mlx-audio`. Add one file (mlx-lm) or one sub-package (mlx-vlm, mlx-audio) named after upstream `config.json` `model_type`. Implement `@dataclass ModelArgs(BaseModelArgs)` + `class Model(nn.Module)` with `.model_type`, `.layers`, `.sanitize(weights)`. Re-run the official converter against your fork, parity-test, then upstream. Canonical reference port: Mel-RoFormer at `mlx-audio#654`. See `references/manual-port-templates.md` for the file shape and `references/mlx-community-conventions.md` for the publishing contract.

CONTRIBUTING checklists:
- https://github.com/ml-explore/mlx-lm/blob/main/CONTRIBUTING.md
- https://github.com/Blaizzy/mlx-vlm/blob/main/CONTRIBUTING.md

**Tier 3 — fallback for multi-component pipelines:**

When the model is **not** a single-stack LLM/VLM/audio architecture — T2V, T2I, 3D mesh, multi-component diffusion or audio-gen pipelines — use `mlx-forge` for per-component conversion with separate dtype / quant routing. Concrete cases: LTX-2 (text encoder + transformer + VAE + scheduler + upsampler), Qwen-Image-2512, Hunyuan3D-2.1, CogVideoX-Fun, Matrix-Game. The companion `mlx-recipe` skill scaffolds the recipe YAML if you need it.

- `mlx-forge` (per-recipe conversion): https://github.com/dgrauet/mlx-forge
- `mlx-arsenal` (shared ops outside `mlx_lm.models.base`: flow-matching primitives, tiling, pixel-shuffle for diffusion): https://github.com/dgrauet/mlx-arsenal

Full routing detail: `references/weight-conversion.md`.

## Helper modules to reach for first (Tier 1 / Tier 2)

For LLM/VLM/audio ports, check `mlx_lm.models.base` and `mlx_lm.models.rope_utils` **before** importing `mlx.fast.*` directly or hand-rolling ops:

- `mlx_lm.models.base.scaled_dot_product_attention` — handles GQA, mask polarity, and the `mx.fast` dispatch correctly.
- `mlx_lm.models.base.create_attention_mask` — causal mask with KV-cache offset bookkeeping.
- `mlx_lm.models.rope_utils.initialize_rope` — dispatches to default / traditional / linear / llama3 / yarn / longrope RoPE via the `rope_scaling` config dict.

`mlx-arsenal` is for ops the canonical helpers don't cover (flow-matching, diffusion tiling). Use it at Tier 3, not Tier 1/2.

## Core mental model

A port is a **transpose operation**, not a redesign. The reference implementation's config values, algorithmic choices, and numerical behavior are the oracle. Any deviation — "better" defaults, "cleaner" modes, "optimized" schedules — almost always hides a port bug. Match first, optimize later with justification tied to a framework constraint (e.g. Metal command-buffer timeout), not taste.

**Preserve isomorphic structure with the reference repo.** This is a hard rule, not an aesthetic preference — past `ltx-2-mlx` port suffered repeated drift bugs that were costly to track because the MLX code no longer mapped 1:1 to the official source. Concretely:

- **Same file paths and module names** as upstream (`models/transformer.py` → `models/transformer.py`, not `model/dit.py`).
- **Same class names** (`LTXVideoTransformer3DModel`, not `LTXTransformer` or `DiT`).
- **Same method names and decomposition** — if upstream has `_split_qkv` + `_apply_rotary` as separate methods, keep them separate. Don't inline, don't merge "while we're at it", don't rename to "be more pythonic".
- **Same forward-pass call order** with the same intermediate variable names where reasonable. A reader should be able to diff `model.py` (upstream) vs `model_mlx.py` (port) and see only PyTorch↔MLX op substitutions.
- **Same config / kwargs surface.** No silent renaming (`num_attention_heads` stays `num_attention_heads`, not `n_heads`), no dropping "unused" fields, no adding convenience flags.
- **Resist refactor temptation.** Even if upstream has dead code, redundant branches, or odd naming — keep them. The cost of refactoring during a port is paid twice: once now (breaks parity diffing) and once forever after (every upstream update becomes a 3-way merge instead of a 1:1 sync).

Refactor only AFTER fp16 parity is locked, end-to-end output matches, and there is a concrete reason (perf, framework constraint). Refactors made during the initial port are how `ltx-2-mlx` accumulated drift that masked real bugs.

If output looks almost-right but subtly off (color tint, slight blur, minor drift), assume port bug, not artifact. Numerical parity with PyTorch at the layer level is the only reliable signal.

## Workflow

```
1. Read reference        → understand exact shapes, configs, math
2. Route weight conversion → Tier 1 (mlx_*.convert) | Tier 2 (manual model file) | Tier 3 (mlx-forge recipe)
3. Scaffold -mlx fork    → only if Tier 2 or Tier 3; Tier 1 needs no fork
4. Translate modules     → layer by layer; reuse mlx_lm.models.base helpers first
5. Parity test           → PT ref vs MLX, layer by layer, max_abs < 1e-3 fp16
6. End-to-end            → full pipeline, golden image/text
7. Quantize              → int8/int4 only after fp16 parity locked
```

Never skip step 5. Wrong output at step 6 with no layer-level parity data is unrecoverable.

## Step 1 — Read the reference carefully

Before writing any MLX code, read the PyTorch source with a skeptical eye for these recurring traps. See `references/common-pitfalls.md` for full details and concrete examples from past ports.

**Quick checklist while reading:**

- [ ] **Constructor defaults** — every `__init__` default must be verified against the model's `config.json`. Wrong defaults (e.g. `include_pi=True` when config says `false`, `groupnorm_eps=1e-5` when config says `1e-6`) silently ruin outputs.
- [ ] **`attention_head_dim` misnomer** — in diffusers UNets, `attention_head_dim=[5,10,20,20]` means `num_heads`, NOT per-head dim. Real head_dim = `channels // attention_head_dim`.
- [ ] **QKV reshape pattern** — if you see `qkv = cat([q,k,v]) → view(B,N,heads,3*hd) → split`, the heads are **interleaved**, not stacked. Replicate EXACTLY — standard per-tensor reshape gives totally wrong attention.
- [ ] **Weight layout** — PyTorch Conv is `(O, I, *K)`, MLX is `(O, *K, I)`. Linear is identical. Embedding is identical. Conv transpose has its own rule.
- [ ] **Normalization semantics** — RMSNorm, GroupNorm, LayerNorm have different default epsilons across frameworks. AdaLN variants differ (additive-only vs `x*(1+scale)+shift`). For VAE ports, `groupnorm_eps` is the #1 silent killer.
- [ ] **Non-obvious flags** — `qk_norm`, `pre_norm`, `use_bias`, `cross_attention_dim`, activation choice (GEGLU vs SwiGLU vs GELU). Cross-check against config, not defaults.
- [ ] **Tekken / Pixtral tokenizer skips BOS** — when wiring an mlx-lm Mistral-family text encoder, `add_special_tokens=True` is a no-op and BOS must be prepended manually. Affects Mistral Small 3 / Ministral3 / Pixtral / ERNIE-Image.
- [ ] **Checkerboard trap (the recurring one)** — if the end-to-end image comes out as periodic noise at stride 2/4/8/16, the culprit is almost always (in order): `mx.tile` used where `mx.repeat` was needed, pixel-shuffle axis order, text-encoder `hidden_states[-2]` applying N instead of N-1 layers, or scheduler dtype leaking fp32 into a bf16 DiT. See pitfall #7 for the 3-test diagnostic procedure — run it BEFORE shipping every port.
- [ ] **VAE numerics (color tints, black/gray output)** — if layer parity passes but end-to-end output is cyan / gray / washed-out, suspect a `groupnorm_eps`, `fused_norm`, or `groupnorm_num_groups` flag mismatch on the VAE side before suspecting attention. Wrap `vae.decode(...)` in `mx.eval(out)` before any image conversion. See pitfall #10.
- [ ] **Structural drift** — port the upstream pipeline's *abstractions* (Stage classes, ModalitySpec, denoising loops, guided-denoiser factories), not just its tensor ops. Don't reorder operations, don't pass arguments upstream leaves at default, don't inline / flatten / "simplify" — every shortcut becomes a hypothesis to disprove during later debugging. See pitfall #9 (`ltx-2-mlx` lesson) for the 5 concrete drift patterns.

If any of these apply, open `references/common-pitfalls.md` and `references/attention-patterns.md` and follow the detailed guidance before writing the MLX code.

## Step 2 — Scaffold (Tier 2 or Tier 3 only)

**Tier 1 needs no fork.** Just run `mlx_lm.convert` / `mlx_vlm.convert` / `mlx_audio.convert` and publish.

**Tier 2 — manual architecture port:** fork `ml-explore/mlx-lm` / `Blaizzy/mlx-vlm` / `Blaizzy/mlx-audio` and add one file (mlx-lm) or one sub-package (mlx-vlm, mlx-audio) named after upstream `config.json` `model_type`. The shape is fixed by upstream conventions — see `references/manual-port-templates.md` for the canonical `Model` / `ModelArgs` / `sanitize` template.

**Tier 3 — multi-component pipeline `-mlx` fork:** standard repo convention (see `references/repo-layout.md` for the full layout):

```
<model>-mlx/
├── README.md                        # Quick Start + HF repo link
├── pyproject.toml                   # depends on mlx, mlx-arsenal, optionally mlx-forge
├── <pkg>/
│   ├── model/                       # MLX module definitions
│   ├── pipeline_mlx.py              # high-level entrypoint, from_pretrained()
│   └── utils/weights.py             # load_split_safetensors, HF hub fetch
└── tests/
    ├── parity/                      # PT↔MLX numerical comparison (PT optional dep)
    └── smoke/                       # shape / config / e2e smoke
```

The Swift consumer side under `xocialize-code/<package>-mlx` mirrors the Python repo with an Xcode workspace at the top level and `Package.swift` as the dependency manifest (Xcode is the build tool — SwiftPM CLI is not used). See `references/repo-layout.md` for the full Swift side.

**HF auto-download convention:** single entrypoint `ModelClass.from_pretrained(repo_id)` that downloads split safetensors via `huggingface_hub` and loads them lazily. Support `<MODEL>_MLX_WEIGHTS_DIR` env var for local dev override. (Note: ltx-2-mlx currently uses a `conftest` lookup instead — standardize on the env-var pattern for new ports.)

## Step 3 — Translate modules

Order of preference when reaching for a helper:

**1. `mlx_lm.models.base` and `mlx_lm.models.rope_utils`** — for Tier-2 manual ports, these are the canonical helpers. Check them first:

- `mlx_lm.models.base.scaled_dot_product_attention` — handles GQA, mask polarity, and `mx.fast` dispatch.
- `mlx_lm.models.base.create_attention_mask` — causal mask with KV-cache offset bookkeeping.
- `mlx_lm.models.rope_utils.initialize_rope` — dispatches to default / traditional / linear / llama3 / yarn / longrope RoPE via `rope_scaling`. Use this instead of hand-coding `cos/sin` rotation; it's where almost every long-context scaling bug gets fixed.

**2. `mlx.fast` primitives** — when `mlx_lm.models.base` doesn't cover it, the `mlx.fast` path is faster and more numerically stable than hand-rolled equivalents:

- `mx.fast.scaled_dot_product_attention` instead of manual QK^T/√d + softmax + V. Supports GQA natively.
- `mx.fast.rope` for rotary embeddings.
- `mx.fast.rms_norm`, `mx.fast.layer_norm` for norms.

**3. `mlx-arsenal`** — for ops outside the LLM/VLM/audio canon. Use this at Tier 3 (multi-component diffusion pipelines) where neither `mlx_lm.models.base` nor `mlx.fast.*` covers what you need:

- `mlx_arsenal.diffusion`: `get_timestep_embedding`, `TimestepEmbedding`, `euler_step`, `classifier_free_guidance`, `FlowMatchEulerDiscreteScheduler`, `dynamic_shift_schedule`, `get_sampling_sigmas`
- `mlx_arsenal.spatial`: `interpolate_nearest`, `avg_pool1d`, `replicate_pad`, `pixel_shuffle`, `patchify`/`unpatchify`, `PatchEmbed2d/3d`
- `mlx_arsenal.attention`: `causal_mask`, `sliding_window_mask`
- `mlx_arsenal.norm`: `PixelNorm`, `ScaleNorm`
- `mlx_arsenal.encoding`: `FourierEmbedder`
- `mlx_arsenal.moe`: `MoEGate`, `MoELayer`
- `mlx_arsenal.tiling`: `tiled_process`, `temporal_slice_process`
- `mlx_arsenal.layout`: `to_channels_last`, `convert_conv_weights`, `load_safetensors`
- `mlx_arsenal.rasterize`: `rasterize_triangles`

**Warning:** the root `mlx_arsenal/__init__.py` only re-exports a subset. Import directly from submodules (`from mlx_arsenal.spatial import interpolate_nearest`) — don't trust that a missing top-level export means the op is absent.

**Don't extract prematurely.** Conceptual similarity across projects (e.g. "all 3 models have AdaLN") rarely equals literal compatibility. Before extracting to `mlx-arsenal`, verify byte-identical implementations via diff + seeded parity test across all candidate projects. Past failure: AdaLN "duplicated" across LTX / Hunyuan / Matrix was actually three incompatible variants.

## Step 4 — Weight conversion

**First check: is this a Tier-1 or Tier-2 case?** Most ports never write a recipe.

- **mlx-community already has a converted copy** → use it. `from mlx_lm import load; load("mlx-community/<name>-<quant>")`.
- **Upstream `model_type` is supported by `mlx_lm` / `mlx_vlm` / `mlx_audio`** → run `mlx_lm.convert` / `mlx_vlm.convert` / `mlx_audio.convert`. See `references/weight-conversion.md` for canonical invocations and the mlx-community quant-suffix grammar.
- **Upstream `model_type` is missing** → add the architecture file (mlx-lm) or sub-package (mlx-vlm, mlx-audio) named after `model_type`. See `references/manual-port-templates.md`. Then re-run the official converter — it calls `Model.sanitize(weights)` automatically, which is where all key remapping and conv-layout transposes live.
- **Multi-component pipeline** (T2V, T2I, 3D mesh, multi-component diffusion or audio-gen) → this is the **only** case where `mlx-forge` is the answer. The companion `mlx-recipe` skill scaffolds the recipe (`classify_key`, `sanitize_key`, `convert`, `transform`). Concrete examples: LTX-2, Qwen-Image-2512, Hunyuan3D-2.1.

The **one thing you must remember** when conversion is going through anything *other* than `mlx_lm.convert` (i.e., Tier 3 or a manual non-converter script):

> **Materialize every tensor before saving.** MLX is lazy — unevaluated tensors serialize to safetensors as zeros with no error. Call `mx.eval(weight)` (or the `_materialize` helper in `mlx-forge/src/mlx_forge/quantize.py`) right before `mx.save_safetensors`. This is the silent killer of conversion recipes. **`mlx_lm.convert` and friends do this internally** — Tier-1 ports never hit it.

Other key conventions (full pattern in `references/weight-conversion.md`):
- Split safetensors per component (transformer, vae, text_encoder, …) so each can be loaded / quantized independently. Tier-1 converters produce this automatically; Tier-3 recipes do it explicitly.
- Quantize only `Linear.weight` in transformer blocks by default. Keep VAE / vocoder / connectors / vision-encoder ViT at fp16 or bf16. `mlx_vlm.convert` enforces vision-encoder exclusion automatically.
- PyTorch Conv `(O, I, *K)` → MLX `(O, *K, I)` — handle inside `Model.sanitize(weights)` for Tier 2, inside `transform` step for Tier 3. `mlx_forge.transpose.transpose_conv` is generic.

For publishing conventions (repo naming, README frontmatter, quant-suffix grammar): `references/mlx-community-conventions.md`.

## Step 5 — Parity testing (the step everyone skips)

**Do not skip this.** When end-to-end output is wrong, only layer-level parity data tells you which module is broken.

Pattern (full template in `references/parity-testing.md`):

```python
# tests/parity/test_attention_parity.py
import torch, mlx.core as mx, numpy as np

def test_attention_parity():
    x_np = np.random.randn(2, 128, 1024).astype("float32")

    pt_out = pt_attention(torch.from_numpy(x_np)).detach().numpy()
    mx_out = np.array(mx_attention(mx.array(x_np)))

    max_abs = np.max(np.abs(pt_out - mx_out))
    assert max_abs < 1e-3, f"attention diverges: max_abs={max_abs}"
```

**Target thresholds** (fp16 inference):
- Single layer: `max_abs < 1e-4` (ideal), `< 1e-3` (acceptable)
- Full transformer block: `< 5e-3`
- Full UNet / DiT pass: `< 1e-2`
- Anything `> 1e-1` is a port bug, not numerical drift.

**If parity fails:** isolate. Run the same test on smaller sub-modules until the divergent layer appears, then compare the PyTorch and MLX forward pass line-by-line.

Tests should treat PyTorch as an **optional dev dependency** (`pip install -e ".[parity]"`) so end users installing the `-mlx` fork don't need to pull torch.

Complementary invariant tests (what `mlx-arsenal` uses internally) are useful but **do not replace** PT↔MLX parity — they only catch self-consistency bugs.

**Important caveat — parity at small scale is necessary but not sufficient.** Tests with `hidden=256, num_layers=2` and random weights miss bugs that only trigger at production scale (bf16 precision accumulating over 36 layers, RoPE with real position distributions, text conditioning at trained magnitudes). Add an **end-to-end noise-path smoke test** — decode random Gaussian through the full post-DiT chain (BN inverse → unpatch → VAE.decode) — to the smoke suite. If that output has any periodic pattern, one of those spatial ops is broken at the production config, regardless of what the layer-level parity tests say. See pitfall #7 in `common-pitfalls.md` for the three-test diagnostic.

## Step 6 — End-to-end validation

Only run after layer-level parity is green.

- Run the full pipeline on a golden input (same seed, same prompt, same image as the PT reference).
- Compare output: image PSNR, text BLEU, mesh vertex delta — whatever the modality allows.
- Log peak memory (`mx.metal.get_peak_memory()`) and per-step wallclock for reference.
- If output is wrong but every layer parity passed: suspect sampler / scheduler, RNG semantics (MLX `mx.random.normal` is NOT seed-compatible with `torch.randn`), or data preprocessing.

## Step 7 — Quantize

Lock fp16 parity first. Only then apply quantization.

- `nn.quantize(model, group_size=64, bits=4)` — standard 4-bit for transformer Linears.
- `group_size=128, bits=8` — higher-quality fallback.
- MLX 0.30+: NVFP4 / MXFP8 / 3,5,6-bit QMV also available (see `references/mlx-docs.md` for current status).
- Verify post-quant with the same parity harness at relaxed thresholds (int4 typically `max_abs < 5e-2` vs fp16 ref).

## MLX-Swift consumer angle (every port ends here)

Most published mlx-community ports are eventually loaded by Swift. The Swift side has its own contract — `Module` property names can't contain dots, `Float64` crashes the GPU, `swift test` can't compile Metal shaders — and getting it wrong wastes a day per port even when the weights are correct.

- **Canonical 3-step load + safetensors dotted-key remap + `@unchecked Sendable` GPU-state classes** → `references/repo-layout.md` "MLX-Swift consumer idioms".
- **`Failed to load the default metallib` on `swift test`** → `references/repo-layout.md` "SPM CLI cannot compile Metal shaders". Build via `xcodebuild` against the workspace by default; the bundle-copy + `default.metallib` → `mlx.metallib` rename is the escape hatch for SPM-CLI lanes that genuinely need it.
- **MLX-Swift `Float64` → GPU crash, `Float * MLXArray` ambiguity** → `references/common-pitfalls.md` "Other subtler pitfalls".

If the symptom is "weights load, model.update succeeds with `.noUnusedKeys`, but inference output is garbage *only on Swift*", the bug is almost always one of these three before it's a layer-translation issue.

## Framework constraints to know

- **Metal command-buffer timeout** (≈10s): long graphs without `mx.eval` calls between steps will hit it. Insert `mx.eval(x)` at natural boundaries (end of a block, between denoising steps).
- **Lazy evaluation**: results are only computed on `mx.eval`, `.item()`, `np.array()`, `mx.save_*`, `print`. For timing, call `mx.eval` + `mx.synchronize`. For saving, always `mx.eval` first (see step 4).
- **Unified memory**: all arrays share one pool — no explicit `.to(device)`. Use `mx.metal.set_memory_limit` if OOM risk.
- **No in-place mutation**: `x[idx] = y` works but is emulated via copy — don't assume PyTorch's in-place performance characteristics.
- **bf16 support**: full on M-series GPU; some reductions still fp32 internally.
- **RNG**: `mx.random` uses a different algorithm than PyTorch. Seeds are not cross-compatible — for parity tests, generate on one side (numpy) and inject on both.

See `references/common-pitfalls.md` for a longer list including the six reading-time traps.

## Reference files

Open these only when you actually need the detail — `SKILL.md` gives the workflow, the references give the depth.

- `references/mlx-docs.md` — Curated official URLs: core framework, nn, fast, custom kernels, quantization, mlx-lm/mlx-vlm/mlx-audio/mlx-community, recent additions.
- `references/mlx-community-conventions.md` — The publishing contract: repo naming, quant-suffix grammar (`-4bit`, `-bf16`, `-mxfp4`, etc.), README frontmatter from `create_model_card()`, body snippet templates for LLM / VLM / audio, mirror policy for `xocialize-code` Swift packages.
- `references/manual-port-templates.md` — Tier-2 canonical file shape: `ModelArgs` + `Model` + `sanitize(weights)` template from `mlx_lm/models/qwen2.py`, mlx-vlm / mlx-audio sub-package layouts, mlx-lm CONTRIBUTING checklist verbatim.
- `references/weight-conversion.md` — The three-tier routing rule, canonical `mlx_*.convert` invocations, output layout the converters produce, lazy-tensor materialization rule (Tier 3 only), conv-transpose table, mlx-forge recipe skeleton (Tier 3).
- `references/common-pitfalls.md` — The ten reading-time traps (defaults, head-dim misnomer, QKV interleaving, weight layout, norms, flags, checkerboard, Pixtral BOS, structural drift, VAE numerics) with concrete past-failure examples and idiomatic mlx_lm/mlx_vlm/mlx_audio solutions.
- `references/attention-patterns.md` — Attention translation patterns + `mlx_lm.models.base` / `rope_utils` helper pointers: standard MHA, GQA, rotary, QKV interleaving detection, SDPA fast path, mask conventions.
- `references/parity-testing.md` — Test templates, threshold table, isolation strategy when a test fails, torch-as-optional-dep setup.
- `references/repo-layout.md` — `-mlx` fork standard layout (Tier 3 pipelines), monorepo vs single-package, HF auto-download wiring, Swift consumer side under `xocialize-code`, README conventions.

## Bundled scripts

- `scripts/parity_helpers.py` — Reusable helpers for PyTorch vs MLX parity tests: `make_seeded_input`, `pt_to_mx`, `mx_to_np`, `transpose_pt_conv`, `load_pt_state_into_mx`, `assert_parity`, `tensor_stats`. Copy into your `-mlx` fork at `tests/parity/_helpers.py` when setting up parity testing rather than rewriting from scratch each port.

## When to stop and ask the user

- Reference implementation has multiple branches / modes and it's unclear which one production uses — ask which checkpoint / config to match.
- A PyTorch op has no direct MLX equivalent and a custom Metal kernel is needed — confirm before writing one, since the arsenal or `mx.fast` may have added coverage.
- About to deviate from a reference config "to fix" an artifact — stop and ask. Deviations almost always hide bugs.
