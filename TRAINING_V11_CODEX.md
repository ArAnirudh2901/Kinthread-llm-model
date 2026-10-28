# Next reader (v10-r9p) — training instructions for Codex

Hand this file to Codex as the task. It is self-contained: what to run, in what order, what to compare, and when to
stop. Read `AGENTS.md` (rules) and `HANDOFF.md` (latest section) first. Do not skip the measurement steps: every
change is adopted only if the real dev split says so.

## Goal and honest status

- Current model: v10-r8p step 6,000 (`runs/v10-r8p-final/best.pt`, SHA-256
  `38c289d351d6517b5cde1bc9fb64ea97bd33eee193639d6351edda52fdfd7a52`), 39.5M exact fp32 parameters.
- Real exam accuracy with the FINAL setting: dev 73.9% fair / 77.2% verified; locked test 68.8% fair / 72.2%
  verified. Target: 80% now, 85-90% after. Generated sets are already 95-99.6%, so the gap is real-exam wording,
  long inputs and twisted descriptions, not composition (the exact engine composes).
- Measured weak spots (this repo, 2026-10-28): two detours ("mother's husband's only son") flip 27% of answers
  (accuracy 80% vs 93%); 63% of long-story gold readings exceed the 96-token output cap; 2.2% of mix_r8 rows had more
  than 32 people (now dropped); formats are unbalanced in mix_r8 (data sufficiency 1.5%, nested 2.3%, possessive 2.4%).
- Gauntlet (`eval/gauntlet.py`, browser pipeline, 667 questions): 92.1% overall; claims 63%, family puzzles 70%,
  two detours 77%, data sufficiency 77%, three detours 83%; twisted hand-written 24/28. These are the cells r9p
  data must lift (the balanced supplement gives family puzzles and data sufficiency 2,500 rows each).

## Hard rules (from the owner)

- Never quantize, distil or shrink: keep exact fp32 weights. A larger model is allowed.
- Real questions (`data/real_questions`, Kaggle dataset `kinthread-real-questions`) are evaluation only, never
  training text. Tune and select on the dev split only. Score the locked test once, for the final pick only.
- Gemini keys are never printed or committed.
- Commits: author `Anirudh Aravalli <anirudh30012005@gmail.com>`; never add Co-Authored-By or session trailers;
  commit or push only when the owner asks. Python with torch on the Mac: `~/Generative_AI/LLM_from_Scratch/.venv/bin/python`.
- Code changes go to LLM-From-Scratch (`2_Training_Tiny_LLM`) and are mirrored here (see AGENTS.md "Updating this repo").
  This branch was written in the mirror: first copy the files listed at the end of HANDOFF.md into the canonical repo.

## 0. Setup (Kaggle, GPU T4 x2 or P100)

```bash
git clone -b claude/nice-hamilton-3ttcwr https://github.com/ArAnirudh2901/Kinthread-llm-model.git && cd Kinthread-llm-model
python verify_model.py                       # checkpoints and exported files match model_manifest.json
cd data && sha256sum -c SHA256SUMS && cd ..  # data files match
python -m pytest -q tests                    # all tests must pass with torch installed
```

Attach the private dataset `kinthread-real-questions` and link it as `data/real_questions/`. Put the r8p checkpoint
at `runs/v10-r8p-final/best.pt` and `verifier.json`, `FINAL.json` next to it (Kaggle run `362159765` output).

## 1. Baseline and free gains (no training)

Run once on dev with tracing; the score must equal FINAL.json (528/714 fair) or the setup is wrong.

```bash
FINAL="--adaptive 1.0 --hybrid --verifier runs/v10-r8p-final/verifier.json --cover --split_search --para 2 --k 16"
python -m eval.run_real --ckpt runs/v10-r8p-final/best.pt --split dev $FINAL \
  --trace runs/v10-r8p-final/trace_dev.jsonl --trace_shadow --out runs/v10-r8p-final/real_dev_traced.json
python -m eval.trace_report runs/v10-r8p-final/trace_dev.jsonl --out runs/v10-r8p-final/trace_dev_report.json
python -m eval.rescore_trace runs/v10-r8p-final/trace_dev.jsonl --verifier runs/v10-r8p-final/verifier.json --cover --all
```

`rescore_trace` replays the vote under `--closure`, `--core_repair`, `--lever`, `--trace_pow 1` and their
combinations, with McNemar against base. Keep a flag only if it wins on dev with p < 0.05. Then measure the output
budget, which needs re-reading:

```bash
python -m eval.run_real --ckpt runs/v10-r8p-final/best.pt --split dev $FINAL --max_new 192 --reserve 8 \
  --out runs/v10-r8p-final/real_dev_budget192.json
```

Record the r8p numbers on the hard and repeatability suites (they are the bar r9p must clear):

```bash
python -m eval.gauntlet --ckpt runs/v10-r8p-final/best.pt --k 16 --out runs/v10-r8p-final/gauntlet.json
python -m eval.consistency --groups 300 --ckpt runs/v10-r8p-final/best.pt --k 16 --out runs/v10-r8p-final/consistency.json
python -m eval.evaluate --ckpt runs/v10-r8p-final/best.pt --sets unseen   # as in the 2026-10-27 check (99.57%)
```

Write all numbers into HANDOFF.md before training.

## 2. Freeze the new locked test before any new work looks at new questions

```bash
python -m eval.sites --sites <new sites only>        # scrape sites that are not in data/real_questions
python -m eval.locked --freeze --pattern 'data/fresh_online/*.jsonl'
```

If fewer than 3 new sites are available, skip this step and say so; the v1 locked test then stays the final measure.

## 3. Training data for r9p

Keep everything r8p trained on and add the balanced supplement (exact generator facts, engine-checked):

| File | Rows | Use |
| --- | --- | --- |
| `data/reworded.jsonl.gz` | as before | `*2`, as r8p |
| `data/mix_r8.jsonl.gz` | as before | as r8p (rows with > 32 people are now dropped by `reader.encode_example`) |
| `star_r3.jsonl.gz` (r8p notebook output) | as before | `*2`, as r8p |
| `data/balanced_mix_v1.jsonl.gz` | 22,500 (9 formats x 2,500) | new: `*2` (see the manifest) |
| `data/gemini_synthetic_v1.jsonl.gz` | 1,015 | optional, variant B only |

`balanced_mix_v1` includes LogiQAte-style detours (`q_obfuscated`), lines of descent up to ten generations and deep
cousins (`q_generations`), long and two-family stories, and every format at the same count. Rebuild it with:
`python build_balanced.py --per_format 2500 --seed 11 --out data/balanced_mix_v1.jsonl.gz --extra data/balanced_question_formats_v1.jsonl.gz`.
`train_reader` applies the leak guard to every row (real questions, held-out rewrites, the unseen set and the gauntlet).

## 4. Train r9p (warm start from r8p, same architecture)

Same recipe as r8p (`colab/r8p_kaggle.ipynb`), with the new data and a longer block so long stories fit with a
192-token reading. `colab/r9p_kaggle.ipynb` is ready (Save & Run All; it checks the r8p SHA-256). It differs from r8p only in:

```python
EXTRA = [f'{DATA}/reworded.jsonl.gz*2', 'data/mix_r8.jsonl.gz', f'{S}/star_r3.jsonl.gz*2', 'data/balanced_mix_v1.jsonl.gz*2']
```

```bash
PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True python train_reader.py --arch v10 --run v10-r9p --out runs/v10-r9p \
  --pool {DATA} --init runs/v10-r8p-final/best.pt --prefix_lm --of_form --style_p 0.8 --steps 15000 --batch 64 \
  --lr 3e-4 --warmup 500 --optim muon --amp --workers 2 --resume --eval_every 1500 --save_every 1000 \
  --reworded {EXTRA} --reworded_dev data/reworded_dev.jsonl.gz --p_reworded 0.75 --ramp_steps 2000 --block 512 \
  --real_dev 575 --select_real
```

- If memory runs out at `--block 512`, use `--accum 2` (same effective batch), not a smaller model.
- Variant B (only if the Kaggle quota allows a second run): add `'data/gemini_synthetic_v1.jsonl.gz*2'` to EXTRA.
- Do not change the tokenizer, vocabulary or architecture in r9p: that would be v11.
- Watch `fit_health` output: a non-finite loss stops the run (by design); restart from the last checkpoint with a
  lower lr (2e-4), never by disabling the check.

## 5. Select (dev only)

1. Retrain the verifier for the new checkpoint: `python train_verifier.py --ckpt runs/v10-r9p/best.pt --n 1500 --out runs/v10-r9p/verifier.json`.
2. Run the same setting sweep as `colab/final_kaggle.ipynb` (base, cover, para2, split, verifier, k16, all) on dev,
   plus the flags that won in step 1 and `--max_new 192 --reserve 8`.
3. Adopt r9p only if its best dev fair score beats r8p's 73.9% by at least 2 points (McNemar p < 0.05 on the paired
   questions), and the gauntlet, consistency and unseen scores are not worse than r8p's.
4. Ensemble check (allowed: no weights change): `--ensemble runs/v10-r8p-final/best.pt` on the r9p run; keep it only
   if it wins on dev.

## 6. Locked score (once)

For the single selected checkpoint and setting only:

```bash
python -m eval.run_real --ckpt <pick> --split test <selected flags> --out runs/<pick>/real_test.json          # v1
python -m eval.run_real --ckpt <pick> --pattern 'data/fresh_online/*.jsonl' --split locked2 <selected flags>  # v2, if frozen
```

Report both with the dev numbers and the r8p baseline. Never re-tune after seeing them.

## 7. After training (only when the owner asks)

Export with `python export_reader_web.py --ckpt <pick> --verifier ... --final ... --name-table ...`, run
`python verify_model.py`, update `model_manifest.json`, then follow Kinthread's AGENTS.md for `bun run sync`,
`bun run check` and a browser check before any deploy.

## Hypotheses to test toward 85-90% (in this order; each is kept only if dev says so)

1. Answer-stage flags (step 1): free, from one traced run.
2. Output budget 192 + reserve: long stories lose their question statement at 96 tokens.
3. r9p data: balanced formats, detours, deep generations, two-family stories (step 4).
4. Consistency-filtered self-training (Stratified Consistency Distillation, arXiv 2608.30258): on generated or
   Gemini-rewritten questions only (never real ones), take 16 readings, group them by the engine's answer, keep rows
   whose largest group holds at least 75% of the weight and whose answer matches the generator's, and add them as
   STaR rows (`star.py`) for r10p.
5. Minimal pairs (counterfactual augmentation, arXiv 2605.04330): up-weight `kinlogic.generate.flip_pair` rows,
   which change one word and the answer, so the reader cannot answer from surface statistics.
6. Paraphrase voting at inference (`--para`), already in FINAL; try `--para 4` with the detour suite.
7. v11, only if 1-6 plateau below 85% on dev: a larger reader (e.g. 24 x 512) trained with the same data; it
   doubles browser download and memory, so it needs the owner's decision first.

Open-source material worth reading: KinshipQA generator and data (github.com/TiandaSun/Kinship-Data-Benchmark-for-Multi-hop-Reasoning,
CC BY 4.0; its descriptive-system categories 1-3 are a usable external test), LogiQAte (huggingface.co/datasets/abhilekhborah/LogiQAte,
blood-relation obfuscation), CLUTRR (github.com/facebookresearch/clutrr), and the implicit-deduction code
(github.com/envomp/Implicit-Deductive-Reasoning-in-Transformers).
