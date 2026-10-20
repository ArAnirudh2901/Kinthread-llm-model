# Kinthread LLM model: handbook for any agent or model

Keep this file current. When a checkpoint, result, command or decision changes, update it here (and the Claude
memory noted in CLAUDE.md) and add a line to the handoff log, so the next model can continue cold.

## What it is

The kinship QA model behind the Kinthread website: model code (create, train, test, export), the final
checkpoints, the CLUTRR training data and the exported browser model. See README.md for usage and results.

## Related repos

| Repo | Local path | Holds |
| --- | --- | --- |
| [Kinthread-llm-model](https://github.com/ArAnirudh2901/Kinthread-llm-model) (this) | `~/Generative_AI/Kinthread-llm-model` | model code + `runs/` checkpoints + `data/` + `web/model/` |
| [LLM-From-Scratch](https://github.com/ArAnirudh2901/LLM-From-Scratch) | `~/Generative_AI/LLM_from_Scratch` | canonical source of this code (`2_Training_Tiny_LLM`); assets git-ignored there |
| [Kinthread](https://github.com/ArAnirudh2901/Kinthread) | `~/Web_Dev/kinthread` | the website; ships `web/model` + runtime in `vendor/kinship/` |

## Updating this repo

Code changes are made in LLM-From-Scratch and brought here, keeping history:

```bash
cd ~/Generative_AI/LLM_from_Scratch && git subtree split --prefix=2_Training_Tiny_LLM -b model-split
cd ~/Generative_AI/Kinthread-llm-model && git pull --no-rebase ~/Generative_AI/LLM_from_Scratch model-split
```

After a retrain or export, copy `runs/clutrr-qa-v8/best.pt`, `runs/name-tagger-v4/tagger.pt`, `web/model/*` and
any changed data from LLM-From-Scratch, regenerate `data/SHA256SUMS` if data changed, then run
`python verify_model.py` before committing.

## Integrity (don't weaken)

- `.gitattributes` marks `*.pt *.bin *.gz *.wasm *.parquet data/** web/model/**` binary: no line-ending conversion.
- `verify_model.py` checks checkpoints and exported files against `model_manifest.json` (size + SHA-256);
  `cd data && shasum -a 256 -c SHA256SUMS` checks data. Verified on a fresh clone with `core.autocrlf=true`.
- No Git LFS (clones without LFS would get pointer files). Largest files: `best.pt` and `web/model/qa.bin`, 52 MB
  each (GitHub warns above 50 MB, rejects above 100 MB).
- `.gitignore` keeps only the two final checkpoints under `runs/`.

## Owner's hard rules

- No quantization, no distillation or smaller model: exact fp32 v8 weights.
- Commit or push only when asked. Commits are authored as `Anirudh Aravalli <anirudh30012005@gmail.com>`
  (ArAnirudh2901; set in git config). Never add `Co-Authored-By: Claude` or `Claude-Session` lines. If `gh` is
  signed in as another account, push with the owner's token (see Kinthread's AGENTS.md).
- Python with torch: `~/Generative_AI/LLM_from_Scratch/.venv/bin/python`. 8 GB Mac: batches at 64,
  `PYTORCH_MPS_LOW_WATERMARK_RATIO=0.35`.
- CLUTRR data is CC BY-NC 4.0: keep the attribution in README.

## Handoff log

- 2026-10-20: created from a subtree split of LLM-From-Scratch/2_Training_Tiny_LLM plus checkpoints, data
  (SHA256SUMS) and the exported model; README written; fresh-clone verify and a real `ask_qa.py` run passed;
  history authored only by ArAnirudh2901.
