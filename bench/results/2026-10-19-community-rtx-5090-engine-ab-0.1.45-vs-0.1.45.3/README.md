# Engine A/B on an RTX 5090: 0.1.45 (release build) vs 0.1.45.3 (local build)

Measured on 2026-10-19 by **IVoyt** on one RTX 5090, with `tools/strata_bench.py`'s standard suite
(chat / 4K / 16K / agent / 32K, three runs each). The same model, the same run configuration and the
same six `STRATA_*` switches on both sides; only the engine binary differs.

**Main limitation, stated up front:** the two engines are not built by the same toolchain, so this
compares *code and toolkit together* (see *Limitations*). The prompt-path gain is consistent across
two independent runs of the new engine; the decode figures are not resolvable at this machine's
noise level, and this report says so instead of picking the nicer number.

## Hardware and software

- GPU and VRAM: NVIDIA GeForce RTX 5090, 32 GiB VRAM
- CPU: 12th Gen Intel(R) Core(TM) i7-12700KF (12 cores, 20 threads, no AVX-512); installed RAM: 95.8 GiB
- Storage: NVMe SSD (model not recorded); the experts were read from the SSD into RAM at 0.52 GiB/s
- OS: Windows 11 (10.0.26300); GPU driver: 617.42 (CUDA UMD 13.4)
- Strata: commit `3fa1b524` (v0.2.0-beta-7), source checkout, Python 3.13.12
- Engine A ("old"): **0.1.45**, the official Windows release build (CUDA 13.0, machine code for
  sm_75/86/89/120 + PTX)
- Engine B ("new"): **0.1.45.3**, a local source build of the same checkout with **CUDA 13.1**
  (nvcc 13.1.115), sm_120 only, `STRATA_PORTABLE=OFF`
- Background workloads and power limits: not recorded; the box was otherwise idle

## Model and configuration

- Model: Swift 1.5 Qwen3.8-Flash-Next (IQ3_S), `Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf`
  plus its second shard for the PLE table; a prepared native pack (`D:\Strata-data\packs\swift-iq3_s`)
  and the engine's expert profile
- Vision encoder: off; reasoning off for the suite; temperature 0
- Context: 524,288 tokens; `--kv int8`, `--kv-resident 32768`; `--conversation-cache-mib 8192`;
  `--conversation-cache-park-gain 2048`; `--expert-cache auto`; `--prefill auto`; `--spec 4`;
  MTP draft layer at `D:\Strata-data\mtp\rt`; rope scaling yarn, factor 2
- `--spec-min-p 0.5`, `--pool-workers 3`, `--adapt-every 1`, `--adapt-swaps 160`, `--adapt-decay 0.97`
- Environment: `STRATA_HC_Q8=1`, `STRATA_EXCHANGE_ROTATE=1`, `STRATA_ATTN_DAILY=1`, `STRATA_EXPDEC=1`,
  `STRATA_HC_UPMIX=1`, `STRATA_QSA_SPLIT_ATTN=1` (all six exist in the 0.1.45 tree as well)
- **No calibration was applied to this measurement**: the run config is the install's own, unchanged
- Exact configs: `run-config-engine-0.1.45.json` and `run-config-engine-0.1.45.3.json`
  (they differ only in `exe`, `port` and `log`; the `args` are byte-identical)

```text
python serve/server.py --engine strata --config <config> --host 127.0.0.1 --port <port> --api-key <key>
```

## Method

`run-ab.py` starts the server with one of the two configs, waits for the model, runs
`python tools/strata_bench.py --url http://127.0.0.1:<port> --api-key <key> --runs 3 --long`
against the HTTP API, and stops the server. The bench is pointed at a running server with `--url`
because its own "start the server" path raises a `NameError` (issue #3188); the measurements
themselves are the standard suite.

- model loading (114 s per side) is **not** timed
- one warm-up chat and a 1K prompt are discarded before the suite
- every prompt carries a unique tag, so the conversation cache cannot help the fresh-prompt tests
- thinking is off, output caps 100 (chat), 24 (prompt tests) and 64 (agent turns) tokens
- the runs were done in the order **old, new, new** on purpose: this machine moves 8-28 % between
  engine starts at an unchanged setting (#1332), so a single old-vs-new pair cannot separate drift
  from a real difference. The two new runs bracket the old one; a real effect shows up in both,
  drift shows up as a disagreement between them.

## Results

Medians with the range over the runs; `n` is the number of requests.

| Test | old 0.1.45 | new 0.1.45.3 (run 1) | new 0.1.45.3 (run 2) |
| --- | ---: | ---: | ---: |
| chat decode tok/s (n=9) | 149.00 (121.90-158.70) | 140.60 (113.00-162.00) | 152.50 (126.70-161.80) |
| chat first token (s) (n=9) | 0.32 (0.28-0.38) | 0.29 (0.26-0.38) | 0.29 (0.27-0.36) |
| 4K prompt tok/s (n=3) | 2967.80 (2702.80-2987.40) | 3107.10 (2800.50-3130.50) | 3047.90 (2723.10-3058.10) |
| 4K first token (s) (n=3) | 1.39 (1.38-1.50) | 1.33 (1.32-1.46) | 1.36 (1.35-1.49) |
| 16K prompt tok/s (n=3) | 5412.40 (5330.50-5476.10) | 5871.30 (5854.00-6026.90) | 5823.60 (5774.40-5888.30) |
| 16K first token (s) (n=3) | 3.02 (2.96-3.06) | 2.77 (2.70-2.77) | 2.80 (2.75-2.82) |
| 32K prompt tok/s (n=3) | 5564.30 (5559.30-5592.30) | 6111.90 (6109.00-6149.10) | 6032.80 (6021.30-6093.90) |
| 32K first token (s) (n=3) | 5.81 (5.79-5.81) | 5.30 (5.26-5.30) | 5.36 (5.30-5.37) |
| agent cache hit % (n=3) | 69.27 | 69.34 | 69.28 |
| agent turn first token (s) (n=15) | 1.31 (1.24-1.63) | 1.25 (1.14-1.59) | 1.24 (1.20-1.58) |
| agent decode tok/s (n=15) | 193.10 (136.20-232.00) | 194.40 (123.50-252.50) | 224.90 (133.10-257.90) |

### What the bracket says

**Prompt reading is faster with 0.1.45.3, and the gain grows with the prompt length.** Both
independent runs of the new engine are above the old engine on every prompt test:

| prompt | old | new run 1 | new run 2 | change |
| --- | ---: | ---: | ---: | --- |
| 4,019 tokens | 2967.8 | 3107.1 | 3047.9 | +2.7 % / +4.7 % |
| 16,058 tokens | 5412.4 | 5871.3 | 5823.6 | +7.6 % / +8.5 % |
| 32,049 tokens | 5564.3 | 6111.9 | 6032.8 | +8.4 % / +9.8 % |

Time to first token improves in the same direction, in both runs: 4K −2.4…−4.3 %, 16K −7.2…−8.3 %,
32K −7.7…−8.8 %, chat −7.6…−8.2 %, agent turns −4.6…−5.6 %.

**Decode is not established.** The two new runs disagree with each other by 8.5 % (chat decode) and
by 30 tok/s (agent decode), and the old value lies between them. On this machine the decode
difference is below the noise, so this report claims no decode change in either direction.

**The conversation cache is unchanged**: 69.27 % (old) against 69.34 % and 69.28 % (new), with the
same number of tokens re-read per session (74 / 73 / 74).

### Files

- `results-engine-0.1.45.json`, `results-engine-0.1.45.3-run1.json`, `results-engine-0.1.45.3-run2.json`
  — every request with its timings, plus the machine facts (already redacted)
- `README-engine-0.1.45.md`, `README-engine-0.1.45.3-run1.md`, `README-engine-0.1.45.3-run2.md` —
  the bench tool's own report for each run
- `engine-log-0.1.45.txt`, `engine-log-0.1.45.3.txt` — the engine logs of the two sides
- `run-ab.py` — the script that started the server, ran the suite and stopped it
- `run-config-engine-0.1.45.json`, `run-config-engine-0.1.45.3.json` — the run configs

## Correctness and limitations

- **Speed only.** No needle test, no HumanEval, no tool-call check was run on either side, and the
  answers of the two engines were **not** compared token by token. 0.1.45.3 changes the default
  prompt path on sm_120 (fused int8 prompt experts, the fused gate/up SwiGLU launch), which the
  project's own notes describe as slightly different answers at the same quality; this report does
  not measure that difference.
- **Code and toolkit are confounded.** The old side is the official 0.1.45 release build (CUDA
  13.0); the new side is a local build with CUDA 13.1 for sm_120 only. A control that isolates the
  compiler (the same source built with 13.0, or the 0.1.45 source built with 13.1) was **not** run,
  so a part of the prompt-path gain could be the toolkit rather than the code. The two new runs
  agreeing with each other only rules out run-to-run drift; it does not rule the compiler out.
- **One machine, one model, one configuration.** A 24 GiB card with a different expert-cache size, or
  a model with a different quantisation, may land elsewhere.
- **Machine facts are from the server** (`nvidia-smi`, the OS); background load and power limits were
  not recorded, and the drive model is not in the report.
- The engine logs show the PCIe share probe at 30.8 GB/s and the engine's own share of 0.55 on both
  sides, so the two runs share the same PCIe behaviour.
