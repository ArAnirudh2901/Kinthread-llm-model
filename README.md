# Kinthread LLM model

**The from-scratch language model behind [Kinthread](https://github.com/ArAnirudh2901/Kinthread): it reads a short
family story and works out how two people are related, showing its reasoning step by step.**

Everything here was written and trained from scratch in PyTorch: a 13M-parameter GPT that answers kinship questions
with a chain of thought, a 5.1M-parameter GPT that finds person names, and a small C engine compiled to WebAssembly
that runs both in a browser. The trained checkpoints, the training data and the exported browser model are included,
so you can ask questions, evaluate, retrain or export straight after cloning.

## Contents

```text
model_gpt.py         the GPT (decoder-only transformer) used by both models
tokenizer.py         byte tokenizer
qa_data.py           kinship QA: data loading, entity ids, chain-of-thought sequences, constrained beam search
train_qa.py          trains the relation model
name_tagger.py       trains and runs the name model
ask_qa.py            ask a question about a story
eval_pipeline.py     end-to-end evaluation (names + reasoning + "unknown")
fast_qa.py           reference for the browser engine: the same beam search with a shared key/value cache
export_web.py        exports both models for the browser (exact fp32, byte-shuffled gzip) and writes model_manifest.json
verify_model.py      checks checkpoints and exported files against model_manifest.json
train.py, dataset.py, orchestrator.py, utils.py   the base tiny-LLM training setup
runs/clutrr-qa-v8/best.pt        final relation model checkpoint
runs/name-tagger-v4/tagger.pt    final name model checkpoint
data/                CLUTRR training data (parquet) and the base text corpus, with SHA256SUMS
web/                 browser engine: engine.c (WebAssembly, six builds), kinship.js, exported model, tests
```

## How it works

1. **Names become entity ids.** The name model marks every person in the text; each name is replaced by `@0`, `@1`,
   … so the relation model learns from the wording ("her son", "his sister"), not from what names usually mean.
2. **Chain of thought.** For "how is B related to A?", the model first writes the family facts along the path from
   A to B (`@0 father @1 <sep> @1 wife @2 <sep> …`), then the answer.
3. **Constrained beam search.** Only relation words and people from the story can be chosen, and the path must end
   at B. A step checker (`<nofact>`), trained on deliberately wrong chains mined from the model's own search, lowers
   the weight of facts the story doesn't state.
4. **Knowing when to stop.** If A and B are never mentioned together, or through people who are, the answer is
   "unknown" without running the model; low agreement between paths gives a tentative answer.

## Results

| Measure | Score |
| --- | --- |
| CLUTRR validation | 89.8% |
| CLUTRR validation with distractor sentences | 78.2% |
| CLUTRR test (longer chains than training) | 44.8% |
| Name model: held-out places wrongly tagged as people | about 7% |
| Browser speed (Apple M2, Chrome, WebAssembly, 4 threads) | about 26 ms per question |

Short chains of two to four steps are reliable; long chains are the main weakness.

## Quick start

```bash
git clone https://github.com/ArAnirudh2901/Kinthread-llm-model.git
cd Kinthread-llm-model
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

python verify_model.py      # all checkpoints and exported files match model_manifest.json
python ask_qa.py --story "Ashley's daughter, Lillian, asked her mom to read her a story. Nicholas's sister Lillian asked him for some help." --query Ashley Nicholas
```

`--query A B` answers "B is A's ___" (here: son). `python ask_qa.py --help` lists the search options.

## Training and evaluation

```bash
python name_tagger.py --out runs/my-tagger            # name model
python train_qa.py --out_dir runs/my-qa                # relation model (see --help for every setting)
python eval_pipeline.py --ckpt runs/my-qa/best.pt --tagger runs/my-tagger/tagger.pt
```

Use new output folders so the included checkpoints stay untouched. Training ran on an 8 GB Apple-silicon Mac with
MPS; on similar machines keep beam and mining batches at 64 and set `PYTORCH_MPS_LOW_WATERMARK_RATIO=0.35`.

## Browser engine

```bash
python export_web.py --qa runs/clutrr-qa-v8/best.pt --tagger runs/name-tagger-v4/tagger.pt --out web/model
cd web && ./build.sh          # needs Emscripten (emcc): relaxed SIMD / SIMD / scalar, each with and without threads
node test/test_node.mjs       # the engine must match the Python reference on test/golden.json
```

The export is lossless: the original 32-bit weights, not quantized. In the browser the outputs match PyTorch to
within 1.5 × 10⁻⁵. Kinthread ships this exported model.

## Integrity

Weights, checkpoints and data are marked `binary` in `.gitattributes`, so git never converts a byte on any OS.
`python verify_model.py` checks every checkpoint and exported file against `model_manifest.json` (size and
SHA-256), and `cd data && shasum -a 256 -c SHA256SUMS` checks the data. The browser engine also checks a CRC-32 of
the weights every time it loads them.

## Data

The kinship questions come from **CLUTRR** (Sinha et al., *CLUTRR: A Diagnostic Benchmark for Inductive Reasoning
from Text*, EMNLP 2019, Facebook AI Research), distributed under CC BY-NC 4.0: free to use with attribution, not
for commercial use.

## Related

- [Kinthread](https://github.com/ArAnirudh2901/Kinthread): the website that runs this model in the browser.
- [LLM-From-Scratch](https://github.com/ArAnirudh2901/LLM-From-Scratch): the full learning project, from attention
  to this model. This repo is its `2_Training_Tiny_LLM` folder, history included.

## License

Code: [Apache License 2.0](LICENSE). CLUTRR data: CC BY-NC 4.0 (see Data).
