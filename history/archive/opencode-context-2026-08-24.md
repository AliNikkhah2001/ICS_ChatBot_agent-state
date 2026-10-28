# Archived: .opencode/context.md (2026-08-24)

Source: /splunk-data/v1/.opencode/context.md
Status: STALE — historical reference only, not needed for project running.

## Environment (dated Aug 2026)
- ai-gpu1 2xH200 NVL 139.8GiB SM90 driver580 CUDA13, 1Ti RAM 30c Xeon, proxy 192.168.203.2:3128, dir /splunk-data/v1/Work_RAG-Server-Setup
- Python: 3.11.15 via deadsnakes PPA, 3.12.3 default
- Venvs: venv-deepseek (py3.12 torch2.12 nightly), venv-deepseek11 (py3.11)
- Models: DS-V4-Flash FP4/int8/bf16, HF 46 shards 148GB
- nvidia-smi: 88763/54394 MiB used/free gpu0, 61811/81346 gpu1

## Mission status (dated Aug 2026)
- M1: S1.1.1/S1.1.2 done, S1.2.1 venv done, S1.2.2 torch downloading, S1.2.3 tilelang pending
- M2: T2.2 checkpoint audit done (35219/35351 keys), T2.1 kernels pending
- M3: generate.py CPU patch + config_1x1024 + smoke pending
- M4: README/docs/daemon pending

> Note: This was a DeepSeek-V4 mission on ai-gpu1, NOT the current RAG project.
> Current agent context: agent-state/context/context.md
