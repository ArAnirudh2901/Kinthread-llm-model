# Handoff (2026-10-26): continue here with any agent / account

Read in order: this file -> `2_Training_Tiny_LLM/JOURNEY.md` (story, intuition, results, lessons) -> `AGENTS.md`
(handbook + full log) -> `~/Web_Dev/kinthread/MODEL_UPGRADE_PLAN.md` (website). Claude memory (if on this Mac):
`~/.claude/projects/-Users-anirudharavalli-Generative-AI-LLM-from-Scratch/memory/` (MEMORY.md index). On another machine
use the private bundle `~/kinthread_handoff.tgz` (memory, Gemini keys, real questions; never commit or share it).

## Completed final selection (Kaggle, owner's account)
- r8-final: `notebookf6d1e34e7f` version 6 (causal reader). r8p-final: `notebook99ae0586bd` version 7
  (prefix-LM). The final selection in `notebookf8798397a0`, run `362159765`, completed and wrote `FINAL.json`.
- Full dev (748 total; 714 fair / 650 verified): r8p base 73.0% fair / 76.5% verified; r7 72.7% / 76.5%; r8
  72.1% / 75.5%. The winner was `r8p@notebooks` with `all` reading settings (`--verifier verifier.json --cover
  --split_search --para 2 --k 16`), scoring 73.9% fair / 77.2% verified on dev.
- Locked test, scored once after the dev pick: winner 68.8% fair / 72.2% verified; r7 baseline 66.2% / 70.5%
  (+2.6 / +1.7 points). Do not re-score locked test for selection.

## Next steps
1. Candidate, verifier and final report are staged locally at `runs/v10-r8p-final/` (git-ignored). Exact checkpoint
   hash matches Kaggle. The downloaded `verifier.json` has 1,466 questions, 1,172 training examples and 22 weights.
2. In the canonical `LLM-From-Scratch/2_Training_Tiny_LLM` source, wire the selected setting as a model preset or
   defaults (`--adaptive 1.0 --hybrid --verifier runs/v10-r8p-final/verifier.json --cover --split_search --para 2
   --k 16`), then mirror the code here using the documented subtree workflow. Keep paths valid after the final
   artifacts are included. Do not commit or push unless asked.
3. For a release, split `best.pt` into <100 MB chunks, preserve `best.pt.sha256`, and update `.gitignore` explicitly;
   the current run folder is ignored. Website: follow `MODEL_UPGRADE_PLAN.md` (browser downloads the model; Cloudflare
   Pages only serves files).

## Rules that must hold
- Own from-scratch model only; fp32; no quantization/distillation; scraped real questions are evaluation only; locked
  test scored once for selection; decide on the full dev, never a subset.
- Commits authored by the owner (git config in repo); never add Claude co-author/session trailers. Push the model
  repo as the owner's GitHub account (see AGENTS.md "Owner's hard rules"); never act as another gh account.
- Never type passwords or create accounts; secrets (Gemini keys) only in `~/.config/kinthread/*.env`, never printed.
- Heavy jobs on Kaggle/Colab, not the 8 GB Mac. Keep tool calls and replies short (owner's usage limit).

## 2026-10-26 update
- Final selection completed in Kaggle notebook `notebookf8798397a0`, run `362159765`. Full dev (748 total; 714 fair / 650 verified): r8p base 73.0% fair / 76.5% verified; r7 72.7% / 76.5%; r8 72.1% / 75.5%. The dev winner is `r8p@notebooks` with setting `all`: fixed `--adaptive 1.0 --hybrid` plus `--verifier verifier.json --cover --split_search --para 2 --k 16`, scoring 73.9% fair / 77.2% verified.
- Locked test was scored once after the dev pick: winner 68.8% fair / 72.2% verified; r7 baseline 66.2% / 70.5%, improvement +2.6 / +1.7 points. Do not re-score this locked test for selection.
- `/Users/anirudharavalli/Downloads/best.zip` is v10-r8p step 6,000/15,000 (SHA-256 `38c289d351d6517b5cde1bc9fb64ea97bd33eee193639d6351edda52fdfd7a52`), the selected Kaggle checkpoint, byte-identical to Kaggle `best.pt` (158,748,883 bytes, SHA-256 `38c289d351d6517b5cde1bc9fb64ea97bd33eee193639d6351edda52fdfd7a52`). `FINAL.json` and the 628-byte `verifier.json` are in `~/Downloads`; all three artifacts are copied into local, git-ignored `runs/v10-r8p-final/` with `best.pt.sha256`. Remaining: decide and implement the CLI defaults in the canonical source repo, then mirror them here; do not commit or push unless asked.
- Chrome extension status checked: ChatGPT extension v1.26.1020.11845 is installed, enabled, and allowed on all sites. Codex's browser inventory still does not expose a Chrome surface, but native Chrome computer use works now.

## Update 2026-10-26
Final selection completed: v10-r8p, step 6,000, `all` boosters won full dev and was scored once on locked test. Details and integrity SHA are in AGENTS.md/JOURNEY.md. Export work has started in canonical LLM-From-Scratch source (`export_reader_web.py`), exact fp32, 39.50M parameters, 126.1 MB gzip. Website still runs v8 while v10 WASM/Pyodide parity and mobile budgets remain unverified. No commits or pushes requested.

## Update 2026-10-26 (v10 browser runtime prototype)
Canonical LLM source now has an exact-fp32 v10 export and separate SIMD/scalar WASM forward runtime. The golden test's maximum absolute difference was 1.15e-5 across prefix, cached decode, tied logits and copy head outputs for one short prompt. This is not full parity and has not been synced into Kinthread. Current site remains v8; follow `~/Web_Dev/kinthread/MODEL_UPGRADE_PLAN.md` and do not deploy until pipeline/device tests pass.

## 2026-10-26 export follow-up
The canonical source exporter produced `/tmp/kinthread-v10-final` from `runs/v10-r8p-final/best.pt` plus the final settings, verifier, and name table. Manifest generated; do not treat this as a website release. Kinthread published the separate low-power v8 runtime update at `https://af2acf01.kinthread.pages.dev`; continue website v10 integration and parity before replacing v8.

## 2026-10-26 parity follow-up
The canonical LLM source has a 50-prompt deterministic C/WASM parity check passing in SIMD and scalar against PyTorch (13–101 tokens). Maximum: prefix 2.86e-5, decode 1.38e-5, logits 1.53e-5, copy keys 2.38e-5, mixed probabilities 7.82e-5. This does not validate the JS wrapper, beam search, end-to-end answers, or mobile budgets. Site's separate resource-aware v8 update is live at `https://kinthread.pages.dev`.

The 50-prompt C/WASM parity cases also now pass through `ReaderWasm` JS wrapper prefill/cache/decode in scalar and SIMD (max logp 2.27e-4; probability 7.82e-5). Full Python beam and answer parity remain pending.
