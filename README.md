# Survey-FWI

Repository for references and auxiliary files for the "Survey FWI" paper.

It contains an **LLM-assisted bibliography filtering pipeline**: each paper
(title + abstract) is classified `include` / `exclude` for the survey by a local
LLM served with [Ollama](https://ollama.com), using the criteria in
[`criteria.txt`](criteria.txt). Models are pulled directly from HuggingFace
(GGUF) and run locally — no data leaves the machine.

## Repository contents

| File | Purpose |
|------|---------|
| `tableFilter.ipynb` | Notebook pipeline: classifies rows of an Excel table (`tabela.xlsx`). |
| `bibFilter.py` | CLI pipeline: classifies entries of a `.bib` file. |
| `bibParser.py` | BibTeX → dict parser used by `bibFilter.py`. |
| `criteria.txt` | Inclusion/exclusion criteria fed to the model. |
| `filteringImage.def` | Apptainer image definition (Ollama + Python + JupyterLab). |
| `tabela.xlsx` | Input table of papers (`title`, `abstract`, …). |

## Requirements

- [Apptainer](https://apptainer.org/) (or Singularity) to build/run the container.
- An NVIDIA GPU + drivers for `--nv` (recommended; large models are very slow on CPU).
- Disk space for the model (e.g. ~43 GB for the 70B Q4 model) under `ollama_models/`.

## 1. Build the container image

```bash
cd /path/to/bibRevIA
apptainer build filteringImage.sif filteringImage.def
```

The image is based on `ollama/ollama` and adds Python, JupyterLab, and a named
Jupyter kernel (`bibRevIA (Ollama)`). Rebuild whenever you change `filteringImage.def`.

## 2a. Notebook workflow (`tableFilter.ipynb`)

Classifies the rows of `tabela.xlsx` and writes two new columns per model:
`IA suggestion - <model>` (the verdict) and `IA comment - <model>` (the reason).

### Start the container (Ollama + JupyterLab)

```bash
cd /path/to/bibRevIA
apptainer run --nv filteringImage.sif
```

This starts Ollama on port `11434`, then JupyterLab on port `8888` (token
`bibrevia`). Leave it running. The current directory is mounted, so the notebook
and data are visible inside the container, and pulled models persist in
`./ollama_models`.

### Connect from VSCode

1. Open `tableFilter.ipynb`.
2. **Select Kernel → Select Another Kernel… → Existing Jupyter Server…**
3. URL: `http://localhost:8888`
4. Token: `bibrevia`  (this is required — an empty token makes VSCode hang).
5. Pick the **bibRevIA (Ollama)** kernel.

### Run

Execute the cells top-to-bottom:

1. **Config** — set `MODEL`, `CATEGORIES`, output column names.
2. **Load table** — reads `tabela.xlsx`, adds the output columns.
3. **Ollama functions** then **start/pull/warmup** — starts the server, pulls
   `MODEL` from HuggingFace (first run downloads the weights), loads it on the GPU.
4. **`IaSuggestion`** — the classifier prompt/function.
5. **Main loop** — classifies up to `MAX_entries` rows (set to `None` for all).
6. **Save** — writes `tabela_with_IA_<model>.xlsx`.

## 2b. CLI workflow (`bibFilter.py`)

Classifies a `.bib` file and writes a verdict CSV. Resumable: re-running skips
entries already present in the output CSV.

```bash
apptainer exec --nv filteringImage.sif \
    python3 bibFilter.py input.bib criteria.txt verdict.csv --model qwen2.5:7b
```

- `input.bib` — bibliography to filter.
- `criteria.txt` — filtering criteria.
- `verdict.csv` — output (`key, title, accepted, reason`).
- `--model` — any Ollama model name, including `hf.co/<user>/<repo>:<quant>`.

## Configuration

- **Model** — in the notebook, set `MODEL`. Pull GGUF models straight from
  HuggingFace with `hf.co/<user>/<repo>:<quantization>`, e.g.
  `hf.co/unsloth/DeepSeek-R1-Distill-Llama-70B-GGUF:Q4_K_M` or
  `hf.co/bartowski/Qwen2.5-7B-Instruct-GGUF:Q4_K_M`. The repo must contain GGUF
  files. Output column names derive from the model, so different models don't
  overwrite each other.
- **Criteria** — edit [`criteria.txt`](criteria.txt). The notebook re-reads it on
  each run of the main loop.
- **How many rows** — `MAX_entries` in the main-loop cell.

## Notes & troubleshooting

- **Port 11434 already in use** — if Ollama is already running on the host,
  the container shares the host network and reuses it; that's fine. The notebook
  talks to whichever Ollama is on `11434`.
- **VSCode spins forever / "password" prompt** — enter the token `bibrevia`.
  An empty token leaves Jupyter's XSRF protection on and blocks kernel creation.
- **VRAM** — a 70B Q4 model needs ~43 GB of GPU memory for full offload; otherwise
  Ollama spills to CPU/RAM and inference is slow.
- **Custom token** — override at launch:
  `apptainer run --nv filteringImage.sif --ServerApp.token=YOURTOKEN`.
