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

- No quantization, no distillation or smaller model: preserve the selected model's exact fp32 weights (current
  release candidate: v10-r8p, step 6,000; v8 remains the website's current deployed model until the reader runtime
  passes browser parity).
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
- 2026-10-20: no changes to code or weights since creation; the Kinthread website ships `web/model` from here.
- 2026-10-21 (branch v9-reader): v9 reader pipeline merged (kinlogic engine, generator, train_reader.py, eval/, tests/). Train with colab/train_reader_colab.ipynb (clones this branch); see AGENTS.md in LLM-From-Scratch for the layout.

- 2026-10-26: Kaggle final selection finished (`notebookf8798397a0`, run `362159765`). Full dev (748 total; 714 fair / 650 verified): r8p base 73.0% / 76.5%; r7 72.7% / 76.5%; r8 72.1% / 75.5%. `r8p@notebooks` with `all` (`--verifier verifier.json --cover --split_search --para 2 --k 16`, plus notebook-fixed `--adaptive 1.0 --hybrid`) won at 73.9% / 77.2%. Locked test, scored once after selection: 68.8% / 72.2%; r7 baseline 66.2% / 70.5% (+2.6 / +1.7 points). `/Users/anirudharavalli/Downloads/best.zip` is v10-r8p step 6,000/15,000 (SHA-256 `38c289d351d6517b5cde1bc9fb64ea97bd33eee193639d6351edda52fdfd7a52`); confirmed selected output: Kaggle `best.pt` and local `best.zip` are byte-identical (158,748,883 bytes, same SHA-256). `FINAL.json` and `verifier.json` are downloaded to `~/Downloads`. Chrome extension is installed/enabled (`ChatGPT` 1.26.1020.11845, all-sites access); Codex's browser list still omits a Chrome surface, but native Chrome computer use now works.

- 2026-10-26: final reader selection completed in Kaggle run `362159765`; selected `runs/v10-r8p-final/best.pt`
  (SHA-256 `38c289d351d6517b5cde1bc9fb64ea97bd33eee193639d6351edda52fdfd7a52`). Website migration started; canonical
  source added `2_Training_Tiny_LLM/export_reader_web.py`. Export smoke check: 39.50M exact fp32 parameters, 158.0 MB
  raw, 126.1 MB gzip. The site still uses v8 until v10 runtime parity/device limits pass. Do not commit or push unless
  the owner asks.
- 2026-10-26 (continued): canonical source has a v10 reader WASM prototype with exact-fp32 prefix attention, KV slot cache and copy head. One golden prompt matched PyTorch within 1.15e-5; this is not full browser parity, and website remains on v8. Continue the website upgrade plan before deployment.

- 2026-10-26 (continued): v10 export smoke passed from this repo's final checkpoint; output at `/tmp/kinthread-v10-final` includes exact fp32 model weights, grammar, `FINAL.json`, `verifier.json`, name table, and SHA-256 manifest. This temporary export has not been copied to either tracked repository.

- 2026-10-26 (continued): canonical v10 SIMD/scalar WASM parity suite passes on 50 deterministic generated prompts (13–101 tokens). Maximum differences: hidden 2.86e-5, decode 1.38e-5, logits 1.53e-5, copy keys 2.38e-5, mixture probabilities 7.82e-5. Full question-pipeline parity remains incomplete.

- 2026-10-26 (continued): the JS `ReaderWasm` wrapper also passes those 50 prompts on both SIMD/scalar builds through prefill, cache growth and one cached decode (max logp 2.27e-4; probability 7.82e-5). Grammar beam and answer pipeline remain incomplete.
