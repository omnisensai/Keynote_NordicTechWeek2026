# Carwash Reproducibility

> **From the AI Summit 2026 Keynote:**
> *"Substrate Engineering — What it takes to control a confidently wrong
> large language model"*

> This repository accompanies the talk presented at KTH Arena (KTH Innovation) during Nordic Tech Week september 11th 2026.
> This repo lets you reproduce the carwash demonstration on your own hardware. You can send the same prompt to 19 models across 6 vendors and see them all argmax the wrong answer, then send the same prompt with the 6-line substrate from the keynote as a system message and watch the answers flip to correct. These experiments show that reasoning can become an engineered specification, one you write, test, and deploy. Output correctness and reproducibility become achievable without relying on fine-tuning, frontier models, or constantly managing hardware numerical jitter.

- **Watch the full keynote**: https://www.youtube.com/watch?v=ExY2qNCbPzw&t=3s
- **More on our research**: https://www.omnisensai.com/research
- **For questions**: elin@omnisensai.com

## What this experiment shows

**One specification. Six lines.** Deployed as system prompt that produces consistent
object-appropriate reasoning across:

- **Different orders of magnitude in parameter count** — from Llama 3.1-8B up
  to frontier-scale models (GPT-4, Claude Opus 4.7, Kimi K2)
- **Vendors with different training pipelines** (Anthropic, OpenAI,
  Meta, Alibaba, Mistral, DeepSeek, Moonshot)
- **Different quantizations and serving stacks** (bf16, int8, int4)
- **Different training year generations** — from GPT-3.5-turbo (2022) to
  Claude Opus 4.7 (2026), a 4-year span across independent labs


---

## What this repo does

For each model in the list below, we send:

- **Baseline** (no system prompt) → all 19 argmax `walk` on carwash
- **Treatment** (system = substrate) → 15/19 fully flip to `drive` (10/10),
  1 borderline (9/10), 3 remain at boundary and don't fully cross
- **Anti-test** (same substrate, library-book prompt) → all 19 stay `walk`
  (a book is not a vehicle → the operate constraint doesn't fire →
  the model correctly walks)

The anti-test is what proves the substrate isn't hardcoding "drive" — the
same 6 lines produce `drive` for the car and `walk` for the book, because
the substrate specifies a state machine, not an answer.

## The 19 models

**Anthropic (5)** — via Anthropic API:
- `claude-opus-4-7`
- `claude-haiku-4-5`
- `claude-sonnet-5`
- `claude-sonnet-4-6`
- `claude-sonnet-4-5`

**OpenAI (5)** — via OpenAI API:
- `gpt-4`
- `gpt-4o`
- `gpt-4.1`
- `gpt-4.1-mini`
- `gpt-3.5-turbo`

**Meta / Llama (5)** — via OpenRouter:
- `meta-llama/llama-3.2-3b-instruct`
- `meta-llama/llama-3.1-8b-instruct`
- `meta-llama/llama-3.3-70b-instruct`
- `meta-llama/llama-4-scout`
- `meta-llama/llama-4-maverick`

**Other open-source (4)** — via OpenRouter:
- `qwen/qwen3-max`
- `mistralai/mistral-large`
- `deepseek/deepseek-chat`
- `moonshotai/kimi-k2`

## The prompts

- `carwash_baseline.txt` — user message for the flip test (car → drive)
- `library_baseline.txt` — user message for the anti-test (book → walk)
- `substrate.txt` — the 6-line semantic substrate (system prompt)

## Setup

```bash
pip install anthropic openai
export ANTHROPIC_API_KEY=sk-ant-...
export OPENAI_API_KEY=sk-...
export OPENROUTER_API_KEY=sk-or-...
```

## Run — the flip test (carwash)

```bash
# Baseline (no substrate) → expected: 10/10 walk on all 19
python3 run.py --baseline carwash_baseline.txt --substrate NONE --n 10

# Treatment (substrate) → expected: 10/10 drive on 15 of 19 models
python3 run.py --baseline carwash_baseline.txt --substrate substrate.txt --n 10
```

## Run — the anti-test (library book)

```bash
# Baseline → expected: walk (correct)
python3 run.py --baseline library_baseline.txt --substrate NONE --n 10

# Treatment (same substrate) → expected: still walk (book is not a vehicle)
python3 run.py --baseline library_baseline.txt --substrate substrate.txt --n 10
```

If the library test flipped to `drive` under the substrate, the substrate
would be hardcoding "drive". It doesn't — it stays `walk`. That is the
evidence that the 6 lines specify a state machine, not an answer.

Each response is printed live and appended to `runs.jsonl` with provenance
(response_id, returned_model, substrate SHA, baseline SHA, timestamp).

## Reproducibility notes

- Temperature = 1.0 (explicit), everything else provider-default
- N = 10 samples per (model, condition, task)
- Under `substrate.txt` on carwash:
  - **15/19 fully flip → 10/10 Drive**
  - **1 borderline** (Llama 3.3-70B at 9/10, M-margin ≈ +6 nats)
  - **3 boundary cases** that don't fully flip under this substrate:
    - Llama 3.2-3B: ~4/10 Drive
    - Llama 4-Scout: ~1/10 Drive
    - Qwen 3 Max: ~3/10 Drive
  These 3 can be rescued by prepending an explicit **ontology block** to
  the substrate (e.g., `Object = car`, `User = human`, `Service location = car wash`).
  With this addition, Llama 4-Scout and Qwen 3 Max flip to 10/10 Drive;
  Llama 3.2-3B improves to ~8/10 but remains a boundary case. Extended explicit ontology substrates are out of scope
  for this repo.
- Library anti-test: **19/19 stay walk** in both conditions
- Argmax at t=1.0 is provider-sensitive when M-margin is small; results
  may vary run-to-run for the boundary cases

## License

MIT
