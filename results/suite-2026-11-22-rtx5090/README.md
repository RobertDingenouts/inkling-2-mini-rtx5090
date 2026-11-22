# Community benchmark on NVIDIA GeForce RTX 5090

Measured on 2026-11-22 with `tools/strata_bench.py` (suite 1, 3 runs per test, medians with ranges). Speeds are the engine's own timings; first-token times are measured by the client over HTTP. Speed only: no correctness check was run.

## Hardware and software

- GPU and VRAM: NVIDIA GeForce RTX 5090, 32 GiB VRAM
- CPU: Intel(R) Core(TM) Ultra 9 285K (? cores, 24 threads); installed RAM: 60.9 GiB; storage: nvme0n1 KINGSTON SNV3S4000G nvme
- OS: Linux 7.0.0-43-generic; GPU driver: 595.99.02; CUDA (driver supports up to): 13.2
- Strata: project 0.1.62, commit not a git checkout; engine not read
- GPU detail (after the run, near idle): power_limit_w=575.00, power_draw_w=569.36, pcie=gen5 x16, pstate=P1, temp_c=71
- Background workloads: not recorded (GPU power limit/draw read above)

## Model and configuration

- Model and quantization: inkling-2-mini-abliterated
- Context: not read tokens; parallel slots: 1
- Settings: the install's own configuration, unchanged; thinking off for the suite; temperature 0

## Method

Run by `python tools/strata_bench.py --long` against the HTTP API of a Strata server (already running); model loading is not timed. One warm-up chat and a 1K prompt are discarded first. Every prompt starts with a unique tag, so the conversation cache cannot help the fresh-prompt tests.

- short chat: the three questions of `tools/ab_engine.py`, up to 100 new tokens, 3 runs
- 4K / 16K prompts: generated word lists sized by a calibration request, 24 new tokens, 3 fresh prompts each
- agent session: system prompt and four tools, then 5 turns that each add about 1500 words of a project file and continue from the model's own earlier answers (the way agent clients resend a conversation), 64 new tokens per turn, 3 sessions; the cache numbers come from the answer's usage (`cached_tokens`) and timings (`prompt_n`)
- concurrency (when run): N requests sent at the same moment, total tokens over the wall time of the burst, latency over all requests

## Results

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median and range | Decode tok/s median and range | TTFT seconds median and range |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| short chat | 44 (43-50) | 0 | 100 (100-100) | 9 | not measured | 391.2 (351.1-622.2) | 0.03 (0.03-0.03) |
| 4K prompt, fresh | 4075 (4075-4075) | 0 (0-0) | 24 cap | 3 | not measured | not measured | 0.33 (0.33-0.33) |
| 16K prompt, fresh | 16128 (16128-16128) | 0 (0-0) | 24 cap | 3 | not measured | not measured | 1.32 (1.32-1.32) |
| 32K prompt, fresh | 32208 (32208-32208) | 0 (0-0) | 24 cap | 3 | not measured | not measured | 2.77 (2.77-2.77) |
| agent session, follow-up turns | 11639 (11638-11690) at the last turn | see below | up to 64 | 3 | see below | 771.9 (375.9-872.2) | 0.19 (0.18-0.19) |

Ranges are the lowest and highest of the runs; `Runs` counts requests that returned a token.

### Agent session (conversation cache)

Cache hit over the follow-up turns: 0.0 % (0.0-0.0); earlier-turn tokens read again per session: 25112 (25107-25365) (earlier turns that the cache did not hold; the -25112 (-25365--25107) new tokens of the files are read once); first turn (nothing cached) first token: 0.06 s (0.06-0.07).

| Turn (run 1) | Prompt tokens | Reused | Read | Prompt tok/s | First token (s) | Decode tok/s |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 555 | None | None | - | 0.067 | 307.9 |
| 1 | 2731 | None | None | - | 0.18 | 404.6 |
| 2 | 4958 | None | None | - | 0.184 | 789.8 |
| 3 | 7185 | None | None | - | 0.187 | 871.8 |
| 4 | 9412 | None | None | - | 0.189 | 872.2 |
| 5 | 11639 | None | None | - | 0.193 | 842.1 |

Per-request data: `results.json`. Engine log lines from the start: `engine-log-excerpt.txt` (when this run started the server).

## Correctness and limitations

Not measured: this is a speed suite. The conversation uses the model's own replies, so token counts differ a little between runs. Machine details come from the server, `nvidia-smi`/`rocm-smi` and the OS; background load is not recorded. Total time of the suite: 21 s.
