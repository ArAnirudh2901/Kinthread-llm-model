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
  selected model: v10-r8p, step 6,000; v10-r8p was deployed to Cloudflare Pages on 2026-10-27 at the owner's request; broad browser parity, device budgets, and the 85% accuracy target remain unmet).
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
- 2026-10-26 (website follow-up): Kinthread added release identity/manifest checks, a returning-browser update notice, SHA-256 verification for cached and downloaded files, and old-cache cleanup only after replacement initialization succeeds. Local Chrome verified success, corrupted-file rejection, and release-mismatch blocking; dark/light notice styling was checked. The active site remains v8; v10 grammar/beam/kinlogic/exam answer-path parity is still a release gate. Website `bun run check` passed (134 tests, lint, static build). No deploy, commit, or push was made.
- 2026-10-26 (unseen answer spot-check): selected `runs/v10-r8p-final/best.pt` plus Python `kinlogic` correctly answered a newly authored three-link Leena/Arvind/Meera/Kabir story as “paternal cousin.” After applying the website's exact sentence rewrite and using matching beam-16, split-search, exact-clause, verifier, coverage, and name-prior settings, Python still read `@0 daughter @1 ; @0 sister @2 ; @2 son @3 ; ask @1 @3` and returned paternal cousin (status `assumed`, confidence 0.699). Chrome local Wrangler preview returned “no family link” on the same model-format input. No exact people-masked token match was found across 597,035 rows in 10 relevant local files; the base `dataset_v7` pool is not present locally, so this is not a formally certified no-train-overlap test. This is one example, not an aggregate accuracy score; it confirms browser decoder/integration still diverges from direct checkpoint inference.
- 2026-10-26 (Kaggle/video follow-up): on Kaggle `notebookf8798397a0`, run `362159765`, verified the mounted v10-r8p checkpoint SHA `38c289d351d6517b5cde1bc9fb64ea97bd33eee193639d6351edda52fdfd7a52` and ran one CPU-only unseen three-link question; Python returned “paternal cousin” (status `assumed`, confidence 0.959). This is a smoke check, not an accuracy estimate; no training, sweep, or locked-test rerun. Removed the temporary cell and stopped the T4x2 session (Kaggle showed one GPU hour remaining); kept only the final-selection notebook, v10 training notebook, and Kinthread preview tabs. Did not rotate accounts to bypass quota. Video audit: this project uses PyTorch autograd and gradient clipping, not handwritten backprop; added fail-fast non-finite loss diagnostics and `clip_grad_norm_(..., error_if_nonfinite=True)` in canonical `train_reader.py`/`fit_health.py`, mirrored here, with three focused tests passing. Checkpoint unchanged; browser answer-path mismatch remains a v10 release blocker.
- 2026-10-26 (website v10 gate): local Chrome replaced the seeded old model cache with the verified `v10-r8p` release and initialized offline. A browser spot-check returned the correct son relation but tentative confidence 0.41 after 107,752 ms; direct pipeline parity and usable latency are not established. The locked test remains 68.8% fair / 72.2% verified, below the stated 85% target. Keep production on v8 while the site adds truthful v10 telemetry and the browser question-set comparison. No checkpoint changes, deploy, commit, or push.
- 2026-10-26 (website v10 diagnostic): mirrored website work ran in local Chrome on runtime `87cdf9f46b15` / model `c4a1d87737c6`. Warm-tab checks returned expected mother/son (`unique`, 0.69, 15.3 s) and paternal-cousin (`assumed`, 0.95, 15.3 s) answers. A fresh Chrome tab also returned the cousin answer and showed the corrected explanation for non-unique status; its first answer took about a minute by wall clock. Both cases are tentative spot checks; browser calibration and aggregate parity are unproven. The locked test remains 68.8% fair / 72.2% verified (<85% target). Keep Cloudflare production on v8 until parity and device budgets pass; no deploy, commit, or push.

- 2026-10-27 (threaded v10 follow-up): canonical/model source now precomputes RoPE sin/cos values and builds SIMD/scalar pthread reader modules. The website gates threads to isolated devices reporting at least 8 GB and leaves low/unknown-memory devices single-threaded. The threaded build needs 384 MiB after 192/256 MiB variants exhausted KV slots on deep beam search. 50 generated prompts pass wrapper parity (max log-prob difference `2.27e-4`, probability difference `7.82e-5`). Website Node/Pyodide end-to-end answer regression passes, and local Chrome runtime `f608eed15e9b` returned paternal cousin (`assumed`, uncalibrated score 0.95) in `9,632 ms` with SIMD/two threads. Full website `bun run check`: 139 pass, one intentional skip, lint/build passed. This is one answer, not aggregate accuracy or a controlled latency benchmark. Locked test remains 68.8% fair / 72.2% verified; production remains v8. No deploy, commit, or push.

- 2026-10-27 (authorized v10 production release): pushed canonical source commit 11508ba to LLM-From-Scratch/main, model mirror merge 57455fc to Kinthread-llm-model/v9-reader, and website commit 6f32924 to Kinthread/main. Ran bun run deploy for Cloudflare Pages project kinthread under the owner's account. Production manifest reports v10-r8p, checkpoint SHA-256 38c289d351d6517b5cde1bc9fb64ea97bd33eee193639d6351edda52fdfd7a52, content c4a1d87737c6, runtime f608eed15e9b. Live production hashes matched all 66 model files (127,187,553 bytes) and 20 runtime files (14,012,183 bytes). Locked test remains 68.8% fair / 72.2% verified, below 85%; one Chrome answer smoke check is not broad parity or accuracy evidence.

- 2026-10-27 (v10-r8p accuracy check): checkpoint SHA-256 38c289d351d6517b5cde1bc9fb64ea97bd33eee193639d6351edda52fdfd7a52 evaluated with eval.evaluate --sets unseen on frozen generated data SHA-256 42d1a89c1a6bc4215b0b1c9e8faf6e50be0a98a84e26f627db77c8e5a14bad0d. Result: 1,394/1,400 = 99.57%; misses were 5/95 story and 1/83 sufficiency questions. The manifest says question wording was excluded from training/tuning; this set was previously reported for v10-r3 (99.21%), so do not treat it as project-blind or real-world accuracy. The official locked real-exam result remains 474/689 fair = 68.8% and 462/640 verified = 72.2%, scored once for selection. No local real-question files are available to rerun; preserve the locked-test rule. Full JSON is in runs/v10-r8p-final/unseen_eval.json.

- 2026-10-27 (overfitting diagnostic): read checkpoint health history at steps 1,500/3,000/4,500/6,000: training loss 0.0893→0.0170; held-out loss 0.0127→0.0125 (intermediate max 0.0136); synthetic accuracy 96.7%→96.6%; real-dev score 71.2%→72.9%. This shows no clear classical train/held-out divergence in saved points. Re-evaluated separate available sets: fresh 38/40 (95.0%), gender 23/24 (95.8%), reworded 329/340 same-answer (96.8%) but only 258/340 exact fact parses (75.9%). These are small or synthetic/reworded benchmarks, not real-world exam accuracy. Combined with the generated-unseen 1,394/1,400 result, the near-perfect generated scores demonstrate strong generator-distribution performance; the official real-exam locked score is still 68.8% fair / 72.2% verified. Best diagnosis: synthetic-to-real distribution gap and template-family bias are evident risks; 99.57% alone does not prove classical overfitting. Full details: runs/v10-r8p-final/accuracy_diagnostic.json.

- 2026-10-28 (balanced question-format supplement): added a deterministic, code-generated candidate with 3,750 rows: 1,250 each direct speech, indirect speech, and long stories; 25 answer-relation buckets have exactly 50 rows in every format. Long stories span 80–187 words in this run, across the configured 80–240 word bands. Indirect wording now shifts first-person references to the speaker's name and possessive. All rows were independently parsed and solved; all 2,500 speech MCQs have five parsed choices with the gold answer exactly once, and each direct/indirect pair matches facts, truth, and answer. Data SHA-256: 315dccb5d2a9bdf0b3f32441601babb6b22d21eb73ad8b4545a51e7311e5cd88. No Gemini requests, training, checkpoint changes, commit, or push. The supplement is a candidate, not a model-accuracy claim.

- 2026-10-28 (long-story reader baseline): Long-story baseline diagnostic (same fixed 15 rows: five each from 80–120, 121–170, and 171–240 configured word bands): current v10-r8p checkpoint SHA remains 38c289d351d6517b5cde1bc9fb64ea97bd33eee193639d6351edda52fdfd7a52. Python reader top-1 matched 5/15 gold answers at the default 96 generated-token cap and 12/15 when only the diagnostic cap was raised to 192. All 1,250 long-story prefixes fit within 236 tokens, below the 320-token prefix budget with max_new=192, so the test did not truncate their input. Three medium/extended cases still failed. This tiny same-generator sample is diagnostic, not an accuracy estimate; no model weights or production runtime changed.

- 2026-10-28 (synthetic-data candidates and long-story baseline): added `generate_balanced_question_types.py` and forced direct/indirect modes in `kinlogic/generate.py`; `kinlogic/speech.py` now parses an explicit named speaker in reported clauses. The deterministic graph-generated candidate contains 3,750 rows: 1,250 each direct speech, indirect speech, and long stories; 25 canonical answer buckets have exactly 50 rows per format. All rows were accepted by the exact readers/solver; the 2,500 speech MCQs each have five parsed options with one gold answer, and paired direct/indirect examples preserve facts, query, and answer. Data SHA-256: `315dccb5d2a9bdf0b3f32441601babb6b22d21eb73ad8b4545a51e7311e5cd88`. Separately added a paced, resumable Gemini rewrite generator and stronger prompts/structured-output validation in `kinlogic/reword.py`; the committed Gemini candidate has 1,015 accepted rows from 2,069 attempts (49.1%), 204 focused rows, and 275 tracked API requests at an 8-second minimum interval. It is not relation/format balanced. The Gemini candidate and balanced candidate are separate; neither has been trained on. The v10-r8p checkpoint is unchanged (SHA `38c289d351d6517b5cde1bc9fb64ea97bd33eee193639d6351edda52fdfd7a52`). A fixed 15-story diagnostic was 5/15 at the 96-token generation cap and 12/15 at 192; this is a tiny same-generator diagnostic, not an accuracy estimate. The locked exam remains 68.8% fair / 72.2% verified. See `SYNTHETIC_DATASET_PLAN.md`, `HANDOFF.md`, and `RESEARCH.md` before deciding what to test next; do not infer a training gain or tune against the locked test.

- 2026-10-28 (dev trace instrumentation): added `eval/trace.py` (observing hooks on the exact readers and
  `exam.candidates`), `eval/trace_report.py`, and `--trace` / `--trace_shadow` in `eval.run_real`; tests in
  `tests/test_trace.py` (6 pass without torch). Not yet run on the private dev set; see HANDOFF.md for the Kaggle
  command and how to read the buckets. Answers are unchanged by tracing. Written in this mirror (cloud session without
  the canonical LLM-From-Scratch checkout): copy `eval/trace*.py`, the `eval/run_real.py` change and the test there.
- 2026-10-28 (answer-stage flags, balance, repeatability; cloud mirror, no torch): added off-by-default flags
  `--closure --core_repair --lever --trace_pow --max_new --reserve` to `eval.run_real` with an offline replay
  (`eval.rescore_trace`, McNemar vs base), `eval.balance_report`, `build_balanced.py`, detour questions
  (`kinlogic.generate.obfuscation_set` / `q_obfuscated`, opt-in) and `eval.consistency` (rename, option shuffle,
  reorder, detours; gold facts 100% agree, rule readers 96.7%). Fixed: "my. wife" stray period; rows with > 32 people
  are dropped instead of encoded as `<unk>` (2.2% of mix_r8). 117 non-torch tests pass. Website decoder parity fixes
  are in Kinthread (runtime a171d14ed9a4). Checkpoint unchanged; nothing deployed. See HANDOFF.md (2026-10-28 decoder
  parity section) for commands and what still needs a Kaggle run. Locked test v2 (`eval/locked.py`, `--split locked2`):
  held-out new sites, hashes only, scored once with a ledger; not frozen yet (needs new scraped sites).
- 2026-10-28 (gauntlet and r9p plan): `eval/gauntlet.py` (1,140 generated hard questions in 19 categories, twins,
  plus 28 hand-written twisted ones in `eval/twisted_set.py`; gold 100%, rules alone 45.7%). Browser production
  pipeline on 667 of them: 92.1% (claims 63%, puzzles 70%, two detours 77%, sufficiency 77%). Deep lineages up to ten
  generations (`q_generations`). `colab/r9p_kaggle.ipynb` and `TRAINING_V11_CODEX.md` hold the next training run.
