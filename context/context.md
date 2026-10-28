# Project Context

## Environment
- ai-gpu1 2xH200 NVL 139.8GiB SM90 driver580 CUDA13, 1Ti RAM 30c Xeon, proxy 192.168.203.2:3128, dir /splunk-data/v1/Work_RAG-Server-Setup
- Python: 3.11.15 installed via deadsnakes PPA (verified 2026-08-24 05:50Z), 3.12.3 still default
- Venvs: venv-deepseek py3.12 torch2.12.0.dev20260408+cu128 tilelang0.1.8 (TVM fail on 3.12), venv-deepseek11 py3.11 pip26.2.1 (25M, torch install in_progress PID3006102 41M/900M)
- Models: DS-V4-Flash FP4 89GBx2, int8 88GBx2, bf16 154GBx2, HF 46 shards 148GB
- nvidia-smi 88763/54394 MiB used/free gpu0, 61811/81346 gpu1

## Current Status
- M1 in_progress: S1.1.1/S1.1.2 done (verified), S1.2.1 venv creation done (pyvenv.cfg 3.11.15), S1.2.2 torch nightly downloading (41M cudnn), S1.2.3 tilelang pending
- M2: T2.2 checkpoint audit S2.2.1-3 done (35219/35351 keys, F4 16896, du 89Gx2, config fp4), T2.1 kernels pending on venv11
- M3: generate.py CPU patch + config_1x1024 + smoke pending (ses_3 started prematurely, will re-queue after M1)
- M4: README/docs/daemon pending

## Pending Tasks (simplified)
- M1: wait S1.2.2 completion (~10m), then S1.2.3 tilelang==0.1.8
- M2: verify float4 copy kernels + tilelang import on py3.11, cache docs
- M3: patch generate.py cpu init, create tuned config, torchrun MP2 2q smoke
- M4: README Phase1, daemon 30m, gemma/manager verify, Reviewer gate
