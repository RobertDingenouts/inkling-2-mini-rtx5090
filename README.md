# Inkling-2-Mini — on a single RTX 5090 (with ingine, no Strata)

**TL;DR** — A ~20 GB, abliterated mini MoE (Thinking Machines, 2026-11-20) runs as a text-only chat assistant on one RTX 5090 (32 GB VRAM) with 63 GB system RAM, 262K context, through an **entirely different engine** than the Strata stack of the 177B MoE next door: [ingine](https://github.com/satellitedown/inkling-2-mini-abliterated-ingine) (satellitedown), a purpose-built C++/CUDA server. It is not the 177B daily driver — it is a different box, a different architecture, and a different engine family, measured on this exact machine and posted with the honest limits attached.

*Numbers below are this box's own measurements (2026-11-22) — the community suite of `tools/strata_bench.py`, client-side timing. See the "honest limits" section before quoting them.*

---

## What this is

- **Model**: `inkling-2-mini-abliterated` — abliterated mini MoE from Thinking Machines (the original: `thinkingmachines/Inkling-2-Mini`, Apache-2.0, arch `inkling2_loop`), ~20 GB of NVFP4/MXFP8 weights, text-only in this configuration.
- **Engine**: **ingine** ([satellitedown](https://github.com/satellitedown/inkling-2-mini-abliterated-ingine)) — a custom C++/CUDA inference server (`ingine-serve`, built for sm_120a), not Python and not PyTorch, with its own private CUDA 13.4 toolchain (`.cuda-toolkit/`); the system CUDA is not used at inference time.
- **Key features**: 262K context, FP8 KV-cache, DSpark spec-decode (threshold 0.8), text-only chat (no image input).

## Hardware and setup (this measurement)

| Component | Spec |
|---|---|
| GPU | NVIDIA GeForce RTX 5090 (32 GiB VRAM, sm_120a) |
| CPU | Intel Core Ultra 9 285K (24 threads) |
| RAM | 63 GB DDR5 (60.9 GiB reported) |
| Storage | NVMe Gen4 (`nvme0n1`) |
| OS | Linux 7.0.0-43-generic |
| GPU driver | 595.99.02 (CUDA 13.2 in the driver; engine built against 13.4) |

Start it with `scripts/ingine start` / the `ink` alias; the API is at `http://127.0.0.1:8003/v1`.

## Measured performance (this box, 2026-11-22)

Suite 1 — the community suite from upstream Strata's `tools/strata_bench.py` (3 runs per test, medians with ranges; thinking off, temperature 0, `--long`). _The tool is Strata-framed; the raw artifact carries a Strata-template README — see its honesty banner._

| Configuration | Requests | Decode tok/s | First token (s) | Prompt tok/s |
|---|---|---|---|---|
| short chat | 9 | **407.7** (371.4–632.4) | **0.03** (0.03–0.03) | not measured |
| 4K prompt (fresh) | 3 | not measured | **0.33** (0.33–0.34) | not measured |
| 16K prompt (fresh) | 3 | not measured | **1.31** (1.31–1.32) | not measured |
| 32K prompt (fresh) | 3 | not measured | **2.76** (2.76–2.76) | not measured |
| agent session (6 turns, ~11.6K tok) | 3 sessions | **753.5** (417.7–838.2) | **0.18** (0.18–0.19) | not measured |

From the same run (agent session, earlier-turn processing): the tool reports the full earlier-turn context being read again per session — ~25.1K tokens (≈25.1K reported as "read" per session, and the tool's "reused tokens" breakdown is empty). There is no cached-token usage from ingine to split that into cached vs. fresh, so the "0.0 % cache hit" figure in the raw artifact is a reporting gap, not a measured fact.

Raw artifact: [`results/suite-2026-11-22-rtx5090/`](results/suite-2026-11-22-rtx5090/) (`README.md` + `results.json`).

### The honest limits — read this before quoting the numbers

- **Timings are client-side.** ingine exposes no engine telemetry (`/v1/status` = 404; `/health` carries only `status/waiting/running/max_queue`). The suite reports decode from `(gen-1)/(total-first-token)`, labelled `client` per turn. There is no engine-clock number to cross-check against.
- **Prompt tok/s is not measured.** The engine never reports one (or `prompt_tokens_details.cached_tokens`), so only first-token times exist — the report explicitly shows `not measured` / `-` for that column.
- **No cached-token accounting from ingine.** Both "cache hit 0.0 %" and "0 reused tokens" in the table are the tool reporting an absence, not a measured fact. Do not read them as "the cache is broken".
- **Spec-decode acceptance is not visible** (no draft stats in the API). The decode numbers above are therefore *not* verified against a drafting denominator.
- **Not an official/protocol-approved run.** This is the community suite, not the frozen-prompt/thinking-on "standard" protocol behind Strata's own results — directionally useful for speed, not comparable as "certified" performance.
- Numbers are single-machine, single-session (not multiple repeat runs).

### The author's own numbers (not measured here)

For reference — the upstream author reports on their RTX 5090: decode **490–560 t/s** (was 490–570 before the 22-11 revision; DSpark spec-decode, temp 1; plain ~300 t/s), prefill **6.2–10.9K t/s**, **~258 t/s at 254K context**, ~23.8 GiB VRAM + 3 GiB conversation cache, FP8 KV (16 KiB/token, ~4 GiB at 262K), sliding-window 1024 tok BF16. These are *author* claims; this box's own numbers are in the table above and are the ones to trust for this machine.

## Comparison with the Strata daily driver (same box, same day pattern)

The same suite against the 177B model on the Strata engine (2026-11-20) — the "official" comparison:

| Metric | Strata (177B, engine clock) | ingine (mini MoE, client clock) | Change |
|---|---|---|---|
| chat decode tok/s | 78.1 | **407.7** | +423% |
| chat first token (s) | 0.37 | **0.03** | −93% |
| 4K fresh first token (s) | 4.56 | **0.33** | −93% |
| 16K fresh first token (s) | 4.75 | **1.31** | −72% |
| 32K fresh first token (s) | 6.20 | **2.76** | −55% |
| agent turn first token (s) | 1.57 | **0.18** | −88% |
| agent decode tok/s | 174.1 | **753.5** | +333% |
| agent cache hit % | 71.0 | not reported | — |
| agent re-read tok/session | 5 | ~25092 | — |

Read this honestly: different engines, different models, different timing sources — the direction (much faster turn-one and turn-continuation latency) is the real story, not the precise ratios.

## Provenance

- Tree reviewed: [satellitedown/inkling-2-mini-abliterated-ingine](https://github.com/satellitedown/inkling-2-mini-abliterated-ingine) at commit `306eaa8` (no release tag; the repo README is what ships), then re-measured on 2026-11-22 against commit `d8a1bc5` (Hugging Face revision `9dcfb4f`); the pre-update figures were 391.2/771.9. The new installer's P0 profile works: the run ends in pstate P0 (P1 before the update).
- Engine built locally from source — 62 ninja steps (CXX/CUDA/main), producing the 37.5 MB `ingine-serve` binary. Build time is not recorded here; a local source build takes minutes, not hours.
- Model: abliterated single-shard `model-00001-of-00001.safetensors` (~19 GB on disk; unchanged across the revision move) plus tokenizer/chat-template/config/abliteration support files from the pinned Hugging Face release `afd44de` (the 19:13 pre-update run) (~20.2 GB download; per-file SHA-256 verified on download). The abliteration file `inkling-2-mini-abliterated.safetensors` was replaced with the detector-gated revision `9dcfb4f` (1,771,852 bytes); the base weights and `mtp.safetensors` did not change.
- The suite ran against the already-running `ink` server (same tool family as upstream Strata's community suite; the tool used here is a derived copy of `tools/strata_bench.py` — regenerated on every run so it cannot drift, and it fails closed if the upstream tool changes — with three placement-only patches: (1) hardcoded model-id → `inkling-2-mini-abliterated` with `stream_options.include_usage=true` (ingine validates this id and only streams usage if asked), (2) model-name fallback to the `/v1/models` id (ingine has no `/v1/status`/`/metrics`), (3) the preflight's `service=="strata"` check relaxed to any `/health` 200 (ingine labels itself differently). No functionality is otherwise changed.

## Credits

- Serving engine (ingine) and the abliteration file (community fine-tune): [satellitedown](https://github.com/satellitedown) — author of ingine and of the fine-tune (HF: [satellitedown/Inkling-2-Mini-abliterated-ingine](https://huggingface.co/satellitedown/Inkling-2-Mini-abliterated-ingine), revision `9dcfb4f`).
- Model: Thinking Machines — Inkling-2-Mini (Apache-2.0), abliterated by satellitedown without their involvement.
- Benchmark tooling: a derived copy of the upstream Strata community suite (see Provenance), made for this showcase.
- Hardware: owned by the author of this repo. Notes from the upstream abliteration card are the author's, not independently re-verified here.
- License: MIT (same as the author's other repos).
