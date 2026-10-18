# Disaster-recovery runbook

How to rebuild the production service on a fresh RTX 5090 box when the vast.ai instance dies. `/workspace` on vast.ai is container storage: a destroyed or recycled instance loses everything in it, including the venvs, the 20 GB of model weights and the job store. Plan for about an hour, most of it downloads.

Nothing on the current server is unique except the API key. Code comes from this repository, weights from Hugging Face, and jobs are disposable (24 h retention).

## Current production layout (2026-10-18)

| item | value |
|---|---|
| host | vast.ai, Ubuntu 24.04, RTX 5090 32 GB, driver 610.57 |
| public IP | 79.116.80.166 |
| SSH | port 10367 → 22 |
| backend | `/workspace/backend`, uvicorn on `0.0.0.0:10100`, public port **10171** |
| vLLM | `/workspace/detector/vllm/.venv`, vllm 0.31.0, torch 2.13.0+cu132, on `127.0.0.1:10301`, not exposed |
| weights | `/workspace/.hf_home`, `QuantTrio/Qwen3-VL-32B-Instruct-AWQ` revision `c2d44b1376c73fa120fdb4e216c2ef61c1945498` |
| process manager | supervisord 4.3.0 from the vast.ai base image, programs `fortethon` and `detector-vlm` |
| logs | `/var/log/portal/fortethon.log`, `/var/log/portal/detector-vlm.log` |
| deployed code | not a git checkout: a copied tree, commit recorded in `/workspace/backend/DEPLOYED_COMMIT`. Target version: `origin/main`, detector `5ab6ca0` |

## Before you start

- **The public URL will change.** vast.ai assigns a new IP and a new external port to each instance. The jury has the old URL. Agree with the organizers how to send the new one, and update `docs/submission.md` and `README.md`.
- Get the API key from the team password store. Reuse the same key so the jury does not need a new one. Never put it in the repo, a URL, a chat or a shell history line that gets shared.
- Keep a laptop checkout of this repo with `git fetch origin` done. The repo is private, and copying from the laptop avoids putting a GitHub token on the server.

## 1. Rent the instance

- GPU: 1x RTX 5090 32 GB. The model needs about 29 GB of VRAM at `--gpu-memory-utilization 0.92`, so nothing else may use the GPU.
- Disk: at least 80 GB (20 GB weights, about 15 GB vLLM venv, uploads).
- Image: a vast.ai base image with supervisor (same family as the current one: supervisord in `/usr/local/bin`, `/opt/supervisor-scripts`, `/var/log/portal`). Ubuntu 24.04.
- Ports: expose TCP 10100 in the instance's Docker options (`-p 10100:10100`). Do not expose 10301.
- Add your SSH public key.

After boot, note the mapping:

```bash
ssh -p <ssh_port> root@<ip> 'echo $PUBLIC_IPADDR; env | grep VAST_TCP_PORT_10100'
```

`VAST_TCP_PORT_10100=<public_port>` gives the jury URL `http://<ip>:<public_port>/`.

## 2. System packages and uv

```bash
apt-get update && apt-get install -y git curl ca-certificates ffmpeg
nvidia-smi    # must show the RTX 5090 and "CUDA Version" >= 12.8
curl -LsSf https://astral.sh/uv/install.sh | env UV_INSTALL_DIR=/usr/local/bin sh
uv --version
```

ffmpeg is only for manual debugging. The service decodes and encodes video with PyAV wheels.

## 3. Code

From the laptop, copy `main` (the detector lives in `app/detector_vlm/`):

```bash
cd ~/gitted/fortethon && git fetch origin
ssh -p <ssh_port> root@<ip> 'mkdir -p /workspace/backend'
git archive origin/main | ssh -p <ssh_port> root@<ip> 'tar -x -C /workspace/backend'
ssh -p <ssh_port> root@<ip> "echo $(git rev-parse --short origin/main) > /workspace/backend/DEPLOYED_COMMIT"
```

## 4. Backend venv

```bash
cd /workspace/backend
uv sync --frozen --no-dev        # creates /workspace/backend/.venv with Python 3.12
.venv/bin/python -c "import av, fastapi; print(av.__version__, fastapi.__version__)"
```

## 5. vLLM venv

Separate venv, because vLLM pins its own torch. The RTX 5090 (sm_120) needs wheels built for CUDA 12.8 or newer.

```bash
mkdir -p /workspace/detector/vllm
uv venv --python 3.12 /workspace/detector/vllm/.venv
uv pip install --python /workspace/detector/vllm/.venv/bin/python vllm==0.31.0 --torch-backend=auto
/workspace/detector/vllm/.venv/bin/python -c "import torch, vllm; print(vllm.__version__, torch.__version__, torch.version.cuda, torch.cuda.get_device_capability())"
# expect: 0.31.0 2.13.0+cu132 13.2 (12, 0)
```

If the driver is older than the wheel's CUDA, torch reports no GPU. Then pick a box with a newer driver rather than downgrading vLLM.

## 6. Model weights

About 20 GB. This is the only step that needs Hugging Face access; the service runs with `HF_HUB_OFFLINE=1` afterwards.

```bash
HF_HOME=/workspace/.hf_home /workspace/detector/vllm/.venv/bin/hf download \
  QuantTrio/Qwen3-VL-32B-Instruct-AWQ --revision c2d44b1376c73fa120fdb4e216c2ef61c1945498
du -shL /workspace/.hf_home/hub/models--QuantTrio--Qwen3-VL-32B-Instruct-AWQ/snapshots/*   # about 20G
```

## 7. Environment file

Create `/workspace/backend/.env.local`, mode 600. Variable names only here; values come from the password store and from this table.

| variable | value |
|---|---|
| `API_KEY` | jury key from the password store |
| `DETECTOR` | `app.detector_vlm.detector` |
| `VLM_ACTIONS` | `1` |

```bash
chmod 600 /workspace/backend/.env.local
```

`DATA_DIR` is set by the start script. Other variables keep their defaults (see `README.md`).

## 8. Supervisor

`/opt/supervisor-scripts/fortethon.sh`:

```bash
#!/bin/bash
cd /workspace/backend
set -a; . ./.env.local; set +a
export DATA_DIR=/workspace/backend/var
exec /workspace/backend/.venv/bin/uvicorn app.main:create_app --factory --host 0.0.0.0 --port 10100 --workers 1 2>&1
```

`/etc/supervisor/conf.d/fortethon.conf`:

```ini
[program:fortethon]
environment=PROC_NAME="%(program_name)s"
command=/opt/supervisor-scripts/fortethon.sh
autostart=true
autorestart=true
stopsignal=TERM
stopwaitsecs=30
stdout_logfile=/var/log/portal/fortethon.log
redirect_stderr=true
stdout_logfile_maxbytes=50MB
```

`/etc/supervisor/conf.d/detector-vlm.conf`. The old server ran the script from a second checkout in `/workspace/detector`; pointing at the backend copy is equivalent, since `vlm.sh` only depends on the vLLM venv path and `/workspace/.hf_home`.

```ini
[program:detector-vlm]
environment=PROC_NAME="%(program_name)s"
command=/workspace/backend/app/detector_vlm/deploy/vlm.sh
autostart=true
autorestart=true
startsecs=10
stopasgroup=true
killasgroup=true
stopsignal=TERM
stopwaitsecs=30
stdout_logfile=/var/log/portal/detector-vlm.log
redirect_stderr=true
stdout_logfile_maxbytes=50MB
```

```bash
chmod +x /opt/supervisor-scripts/fortethon.sh /workspace/backend/app/detector_vlm/deploy/vlm.sh
mkdir -p /var/log/portal
supervisorctl reread && supervisorctl update
supervisorctl status fortethon detector-vlm
```

Both start together. The backend's detector process waits up to 15 min (`VLM_WAIT`) for vLLM to come up, so start order does not matter.

## 9. Verify

```bash
# vLLM is up (first load takes several minutes)
tail -f /var/log/portal/detector-vlm.log            # wait for "Application startup complete"
curl -s http://127.0.0.1:10301/v1/models            # lists "vlm"
nvidia-smi                                          # about 29 GB used by vllm

# backend reports the model loaded
curl -s http://127.0.0.1:10100/healthz
# 200 {"status":"ok","detector":"app.detector_vlm.detector","model_loaded":true,"worker_alive":true,...}
# 503 "model_not_loaded" means vLLM is still loading or the detector failed: see fortethon.log
```

From the laptop, end to end through the public port, with a dataset video:

```bash
curl -s http://<ip>:<public_port>/healthz
curl -s -X POST http://<ip>:<public_port>/api/v1/analyze -H "X-API-Key: <API_KEY>" -F "file=@assets/videos/X-2.mov"
curl -s http://<ip>:<public_port>/api/v1/jobs/<job_id> -H "X-API-Key: <API_KEY>"
# within 285 s: "status":"done", "verdict":"violation", first item «Деньги приняты без проверки документа»
# a «Требуется ручная проверка» item with an "error" field means the analysis did not finish: check the logs
curl -s -o /dev/null -w "%{http_code}\n" -X POST http://<ip>:<public_port>/api/v1/analyze -F "file=@assets/videos/X-2.mov"
# 401 without the key
```

Then open `http://<ip>:<public_port>/` in a browser, upload a video and check that the result page shows the finding, the evidence frame and the legal reference.

Finally update the URL in `docs/submission.md` and `README.md` and send the new URL to the organizers.

## Troubleshooting

| symptom | check |
|---|---|
| `detector-vlm` restarts in a loop | `detector-vlm.log`. CUDA out of memory: another process holds the GPU (`nvidia-smi`), stop it. "No CUDA GPUs": driver too old for the wheel |
| `/healthz` stays 503 `model_not_loaded` | vLLM not answering on 10301, or `DETECTOR` misspelled. `fortethon.log` has the load error |
| results are «Требуется ручная проверка» with `error` `timeout` or `deadline` | GPU shared with an experiment, or a long queue. One video at a time is the design |
| 401 with the right key | `.env.local` not loaded: check the file name and that `fortethon.sh` sources it, then `supervisorctl restart fortethon` |
| public URL does not answer | port 10100 not exposed at instance creation. It cannot be added later on vast.ai; recreate the instance |
