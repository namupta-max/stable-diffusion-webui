# AGENTS.md

AUTOMATIC1111 Stable Diffusion web UI: a Gradio app (plus a FastAPI API at `/sdapi/v1/*`) for txt2img/img2img/inpainting, built by monkey-patching upstream SD, SDXL, SD3 and k-diffusion code at runtime. Extensions plug in through a script/callback system.

## Branches
- Work on **`dev`**. `master` is a stale release branch (last commit July 2024), and a CI job fails any PR that targets it. Branch from `dev` and open PRs against `dev`.

## Architecture
- `launch.py` → `modules/launch_utils.py:prepare_environment()` pip-installs torch and `requirements_versions.txt`, then clones pinned repos into `repositories/`. After that it imports `webui.py`, which runs `webui()` (Gradio UI on :7860) or `api_only()` (with `--nowebui`, on :7861).
- `modules/` is flat, grouped by filename prefix:
  - `processing.py`: the generation pipeline (`StableDiffusionProcessingTxt2Img/Img2Img`, the `p` object passed everywhere).
  - `sd_models*`: checkpoint loading. `sd_samplers*`: samplers and schedulers. `sd_hijack*`: patches applied to upstream model code.
  - `shared.py` / `shared_options.py` / `shared_state.py`: global state and settings. `cmd_args.py`: every CLI flag.
  - `ui*.py`: Gradio UI. `api/api.py` + `api/models.py`: REST API.
- `scripts/`: `scripts.Script` subclasses that appear in the "Script" dropdown (e.g. `xyz_grid.py`).
- `extensions-builtin/`: first-party extensions (Lora, SwinIR, soft-inpainting, …), each laid out like a third-party extension: `scripts/`, `javascript/`, `preload.py`, `style.css`.
- `javascript/`: plain browser scripts loaded into one global scope. There is no bundler and no `import`/`export`.

## Commands
Python is **3.10** (the only version supported on Windows). Pinned stack: `gradio==3.41.2`, `fastapi==0.94.0` (pydantic v1).
- Set up the environment once: `python launch.py --skip-torch-cuda-test --exit`. On a CPU-only machine, set `TORCH_INDEX_URL=https://download.pytorch.org/whl/cpu` first.
- Run: `webui.bat` / `./webui.sh`, or `python launch.py --skip-prepare-environment` if the venv is already set up.
- Python lint: `uvx ruff@0.3.3 check .` (or `pip install ruff==0.3.3`). **Pin the version.** Newer ruff reports ~670 errors on clean code, and `ruff .` without `check` is removed syntax.
- JS lint: `npm i` then `npm run lint`. On a Windows clone with `core.autocrlf=true`, every file fails `linebreak-style`. Use `npx eslint . --rule "linebreak-style: off"`, and don't run `npm run fix`.
- Tests need a **running server**, because they send HTTP requests to `http://127.0.0.1:7860` (`base_url` in `pyproject.toml`):
  1. Install test deps: `pip install -r requirements-test.txt`
  2. Start the server and leave it running in its own terminal: `python launch.py --skip-prepare-environment --skip-torch-cuda-test --test-server --do-not-download-clip --no-half --disable-opt-split-attention --use-cpu all --api-server-stop`
  3. Once it's serving: `python -m pytest -vv --verify-base-url test`
  4. When done: `curl -XPOST http://127.0.0.1:7860/sdapi/v1/server-stop`
  - `--test-server` loads the dummy checkpoint `test/test_files/empty.pt`, so no real model is needed. Without a server, only `test/test_torch_utils.py` can pass.

## Conventions
- **Settings**: add them in `shared_options.py` as `"key": OptionInfo(default, "Label", component, component_args)` inside an `options_section(...)` block, and read them as `shared.opts.key`. **CLI flags** go only in `cmd_args.py`.
- **Imports**: import the module, not names from it: `from modules import shared, devices, errors`.
- **Errors**: use `errors.report("msg", exc_info=True)` or `errors.display(e, "task")`, not bare `print`/`traceback`.
- **Changing upstream model behaviour**: patch it from `sd_hijack*.py` using `sd_hijack_utils.CondFunc` or `modules.patches.patch`.
- **Scripts**: subclass `scripts.Script` and implement `title()`, `show()`, `ui()` and `run()` (a dropdown script) or `process()` (a script that returns `scripts.AlwaysVisible` from `show()`). Give UI components `elem_id=self.elem_id("name")`. Record generation parameters in `p.extra_generation_params[...]` so they appear in the infotext.
- **Hooks**: register through `script_callbacks.on_*` rather than editing core call sites.
- **Tests**: use plain `requests` against `/sdapi/v1/...`, with a `url_<endpoint>(base_url)` fixture and tiny payloads (64×64, `steps: 3`). Shared image fixtures live in `test/conftest.py`, and their files in `test/test_files/`.
- **JS**: 4-space indent, semicolons, `{a: 1}` (no spaces inside braces). Any new global that another file uses must be added to `globals` in `.eslintrc.js`.
- **Vendored code**: keep the "original source / license / credit" header, and add the license to `html/licenses.html`.

## Do not touch
- `repositories/`: pinned upstream clones that `launch.py` re-checks out. Patch them via `sd_hijack*` instead.
- Generated or user state: `venv/`, `models/**` (multi-GB weights), `extensions/` (user-installed third-party code; it's excluded from lint), `outputs/`, `tmp/`, `config.json`, `ui-config.json`, `styles.csv`, `cache.json`.
- `webui.sh` / `webui.bat`: set options in `webui-user.sh` / `webui-user.bat` (`COMMANDLINE_ARGS`) instead.
- Don't bump pinned versions in `requirements_versions.txt` or the commit hashes in `launch_utils.py` as a side effect of another change. Gradio 4 / pydantic 2 APIs won't work here.
