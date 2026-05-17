# run-notebooks

Helper repo that uses GitHub Actions to execute Jupyter notebooks on a CI runner and commit the executed outputs (and any artifacts they produce) back to the repository.

---

## How it works

1. You drop a notebook into [`input/`](input/).
2. A push to `main` (or a manual run from the Actions tab) triggers the [`Run and Save Notebook`](.github/workflows/run-and-save-notebook.yml) workflow.
3. The workflow installs dependencies, executes every notebook in `input/` with [papermill](https://papermill.readthedocs.io/), and commits the results under [`output/`](output/) back to `main`.

### Trigger

```yaml
on:
  push:
    branches: [main]
    paths:
      - "input/**"
      - "requirements.txt"
      - ".github/workflows/run-and-save-notebook.yml"
  workflow_dispatch:
```

- Only pushes that touch a notebook, the requirements file, or the workflow itself fire the action — commits that only update `output/` won't re-trigger it.
- `workflow_dispatch` lets you run it manually from the Actions tab.

> **Note:** `main` is _not_ a protected branch in this repo, so the `GITHUB_TOKEN` with `permissions: contents: write` can push back. Pushes made by `GITHUB_TOKEN` do **not** re-trigger workflows, so there is no infinite loop.

---

## Repository layout

```
.
├── .github/workflows/run-and-save-notebook.yml   # the CI workflow
├── input/                                        # source notebooks you want executed
│   └── *.ipynb
├── output/                                       # executed notebooks + artifacts (auto-committed)
│   ├── *.ipynb
│   ├── models/...
│   └── results/...
├── data/                                         # cached datasets + intermediates (cached in CI)
├── requirements.txt                              # Python deps installed on the runner
└── README.md
```

### Notebook conventions

Notebooks are executed from the **repository root** (papermill's CWD). When writing paths inside a notebook:

- Read inputs from `data/` (e.g. `Path('data')`).
- **Write all outputs under `output/`** (e.g. `Path('output/models/...')`, `Path('output/results/...')`). Anything outside `output/` will not be committed.
- Caches that you do not want committed (e.g. raw dataset downloads, conformer pickles) go under `data/` — that folder is cached between runs but never committed.

---

## The workflow, step by step

The file: [.github/workflows/run-and-save-notebook.yml](.github/workflows/run-and-save-notebook.yml)

1. **Checkout** the repo.
2. **Set up Python 3.11** with `actions/setup-python@v5` and `cache: pip` (caches pip's download directory).
3. **Restore the cached `.venv/`** keyed on `requirements.txt` hash.
4. **Install dependencies** — only runs `pip install -r requirements.txt` if the venv cache missed. Always (re)installs the `python3` ipykernel.

   PyG compiled extensions (`torch-cluster`, `torch-scatter`, `torch-sparse`) are installed in the same step from the PyG wheel index that matches the installed torch version (`https://data.pyg.org/whl/torch-<ver>+cpu.html`). These are required by 3D GNN ops such as `radius_graph` used by `SchNet`. `torch` itself is pinned to a CPU build in `requirements.txt` (`torch==2.4.1` from PyTorch's CPU index) so the PyG wheels always match.

5. **Restore the cached `data/` folder** (TDC raw downloads, conformer pickles, etc.).
6. **Execute every notebook in `input/`** via papermill, writing executed copies to `output/<filename>.ipynb`.
7. **Commit and push** everything under `output/` back to `main` as the `github-actions[bot]` user.

---

## Caching strategy (why subsequent runs are fast)

Three layered caches keep CI time low:

| Cache                 | Path           | Key                                                                              | What it saves                                                                                     |
| --------------------- | -------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| **pip downloads**     | `~/.cache/pip` | (managed by `setup-python`)                                                      | Wheel downloads for first venv build                                                              |
| **Python virtualenv** | `.venv/`       | `venv-<os>-py3.11-<hash(requirements.txt)>`                                      | The fully installed env — `pip install` is skipped on cache hit                                   |
| **Notebook data**     | `data/`        | `notebook-data-v1-<hash(requirements.txt)>` with restore-key `notebook-data-v1-` | TDC dataset downloads, conformer pickles, anything intermediate the notebooks write under `data/` |

### What each run looks like

- **Edit a notebook only** → venv + data both cache-hit. Restore + execute only. Typical end-to-end: ~1–2 min for a small notebook.
- **Add or bump a package in `requirements.txt`** → venv cache invalidates (full reinstall), data cache still restores.
- **Change conformer-generation / data-prep code** → bump `v1` to `v2` in the data-cache key inside the workflow to force regeneration.

### Limits & gotchas

- GitHub Actions caches have a **10 GB per-repo limit** and entries are evicted after **7 days** of no access.
- Caches are scoped per branch with fallback to the default branch. Heavy use of many branches can evict the main-branch caches.
- The data cache is intentionally never committed; if it is evicted, the next run regenerates it (slower one-time hit).

---

## Adding a new notebook

1. Place the `.ipynb` in `input/`.
2. Inside the notebook, write artifacts to `output/...` and intermediates to `data/...`.
3. Make sure every import is covered by `requirements.txt`.
4. Commit + push to `main` (or trigger the workflow manually).
5. Watch the run in the **Actions** tab. On success, the executed notebook and any artifacts appear in `output/` on `main` via a `github-actions[bot]` commit.

### Tips for fast iteration

- Keep a "smoke" version of expensive notebooks (small subsample, few epochs) so a full CI run finishes in a couple of minutes — see [`input/conformer_ensemble_smoke.ipynb`](input/conformer_ensemble_smoke.ipynb) for the pattern.
- Temporarily move heavy notebooks _out of_ `input/` while iterating on a small one.
- Bump cache keys deliberately when you change cache-invalidating code rather than disabling caching globally.

---

## Possible further optimizations (not yet wired in)

- **Skip unchanged notebooks**: hash each notebook and only papermill the ones whose hash differs from the last committed output.
- **Parallel execution**: use a `matrix` strategy so each notebook runs on its own runner.
- **CPU-only torch wheels**: pin `torch==<version>+cpu` from `https://download.pytorch.org/whl/cpu` for smaller download/install.
- **Prebuilt container image** with torch + PyG + RDKit baked in, used via `container:` in the job — skips installs entirely.
- **Self-hosted runner with GPU** for notebooks where training time dominates.
