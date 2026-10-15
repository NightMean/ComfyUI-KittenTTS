# AGENTS.md


## Project

A single ComfyUI custom node wrapping [KittenTTS](https://github.com/KittenML/KittenTTS), an ultra-lightweight ONNX text-to-speech model. Runs on CPU or CUDA/ROCm. No build step — ComfyUI imports the package directly.

## Commands

There is no test suite, linter, or build. Development happens inside a live ComfyUI install:

```bash
# Install as a ComfyUI custom node
cd ComfyUI/custom_nodes
git clone <repo> ComfyUI-KittenTTS

# The node auto-installs its Python deps on first import (see __init__.py).
# To force-reinstall the bundled runtime wheel manually:
pip install --force-reinstall --no-deps whl/kittentts-0.8.0-py3-none-any.whl
```

To test a change, restart ComfyUI and run a workflow containing the "😻 Kitten-TTS" node. `example_workflow/kittenTTS_example_workflow.json` is a ready-made graph.

**Prerequisite that is not pip-installable:** `espeak-ng` must be installed system-wide and the `PHONEMIZER_ESPEAK_LIBRARY` env var must point at `libespeak-ng.dll` / `.so`. Phonemization fails at runtime without it — this is the most common "works on my machine" gotcha.

## Architecture

Three layers, only two of which run:

1. **`__init__.py` — ComfyUI entrypoint.** On import it calls `install_dependencies()`, which pip-installs missing deps (`onnxruntime` vs `onnxruntime-gpu` chosen by `has_cuda()`), then force-installs the bundled wheel if the installed `kittentts` version != `0.8.0`. Only after that succeeds does it import and register the node into `NODE_CLASS_MAPPINGS`. Import-time side effects are intentional — that's how ComfyUI custom nodes bootstrap their deps.

2. **`nodes/` — the actual node (the code you edit).**
   - `kittentts_node.py`: the `KittenTTS` node class. `INPUT_TYPES` dynamically restricts the `device` dropdown based on detected GPU support. `generate()` calls `tts_model.model.generate(...)` — note `.model` reaches through the wrapper to the ONNX class, because only the inner class accepts `clean_text`.
   - `utils.py`: model caching (`_shared_model` module-global, keyed on a config dict so switching model/device reloads), ONNX provider selection (`get_onnx_providers`), and `format_audio_for_comfyui` (numpy → `{waveform, sample_rate}` dict, mono→stereo via repeat). After the wheel loads a model, `get_cached_model` **rebuilds the ONNX `InferenceSession`** with the requested providers + full graph optimization — the wheel otherwise defaults to CPU only.

3. **`KittenTTS/` — vendored upstream source, reference only.** This is a copy of the upstream package for reading. The code that actually executes is the installed wheel in `whl/`. Editing `KittenTTS/*.py` changes nothing at runtime.

### Voice name mapping (non-obvious)

The node exposes friendly voice names (`Jasper`, `Bella`, …) in `AVAILABLE_VOICES` (`utils.py`). These are **not** translated locally — they are passed straight to the model, which resolves them via a `voice_aliases` map loaded from each model's HuggingFace `config.json` (see `KittenTTS_1_Onnx._prepare_inputs`). The raw model voices are `expr-voice-{2..5}-{m,f}`. If a friendly name stops working, the alias table lives server-side in the HF repo, not in this codebase.

### Model selection

`AVAILABLE_MODELS` holds display strings with size suffixes (e.g. `"KittenML/kitten-tts-mini-0.8 (80M)"`); `MODEL_MAP` strips them to real HF repo IDs. A non-empty `custom_model` input bypasses `MODEL_MAP` entirely. Models download to `ComfyUI/models/kittentts/<safe_name>/` via `get_models_dir()`.

## Release / publishing

Version lives in **two** places that must stay in sync: `__init__.py` (`__version__`) and `pyproject.toml` (`version`). `.github/workflows/publish_action.yml` publishes to the ComfyUI registry on any push to `main` that touches `pyproject.toml` — so bumping the version there triggers a release.

## Decisions
- 2026-07-10: `KittenTTS/` dir kept as vendored upstream reference; runtime uses the wheel in `whl/`. Don't "fix" bugs there expecting runtime effect.

## Repository policy

The repository is permanently private. The owner authorized tracking this AGENTS.md.
The owner authorizes automatic pushes of verified changes to origin/main while GitHub confirms this repository is private. Check visibility before pushing; do not automatically push if it is public or visibility cannot be confirmed.
