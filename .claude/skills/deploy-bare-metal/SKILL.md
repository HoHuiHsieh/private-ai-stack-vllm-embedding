---
name: deploy-bare-metal
description: Configure the env files and deploy the vllm-embedding service (vLLM embedding server on the private-ai-stack backend network) on a bare-metal NVIDIA host. Use when asked to set up, configure, fill in, or deploy vllm-embedding, or its env/.env.vllm-embedding file, on a server.
---

# Deploy vllm-embedding on bare metal

Goal: `services/vllm-embedding` runs on a host, `env/.env.vllm-embedding` is filled in correctly, and
`http://vllm-embedding:8000/v1/embeddings` answers from another container on the `backend` network.

This skill covers this service only. Host setup (Docker, the NVIDIA container toolkit, firewall, cloning the
root repo) is in the root skill `.claude/skills/deploy-bare-metal/SKILL.md`. Run its Steps 1 to 3 first, or
confirm they are already done. Do not repeat them here.

## Ground rules

- **Do not read secrets.** Never open `env/.env.vllm-embedding` or the root `.env`. Read only
  `env/.env.vllm-embedding.example`, and check variable names with `cut`, never with a grep that prints lines.
- **Never print a secret value.** `HF_TOKEN` is entered by the user in the file, not in chat.
- **Config goes in env files.** Do not add `environment:` entries, literal values, or `ports:` to
  `docker-compose*.yml`. Do not add services, and do not change the image tag without asking.
- **Start containers only after approval.** Validation and env setup do not need approval. The first
  `./start.sh` that starts containers does.
- **No destructive commands.** Do not run `docker compose down -v`, `docker volume rm`, or prune commands
  without asking. The `hf_cache` volume holds the model weights.
- **Report faithfully.** If a step fails, show the exact error and stop there. Do not claim the service is
  healthy until the healthcheck says so.

## Step 1: Check the host (read-only)

```sh
nvidia-smi --query-gpu=name,memory.total,memory.used,memory.free --format=csv
docker info --format '{{json .Runtimes}}' | grep -q nvidia && echo "nvidia runtime: yes" || echo "nvidia runtime: no"
docker network inspect private-ai-stack_backend >/dev/null 2>&1 && echo "backend network: up" || echo "backend network: MISSING"
```

- No GPU, or no nvidia runtime: stop. The compose file reserves a GPU and cannot run without one. Ask the
  user how to proceed. Do not edit the compose file.
- Backend network missing: the root stack has not started. Run the root skill's Step 7 first.
- Note the free GPU memory. It sets `VLLM_EMBED_GPU_MEM_UTIL` (Step 3).

## Step 2: Create the env file

```sh
cd <repo-root>
cp services/vllm-embedding/env/.env.vllm-embedding.example services/vllm-embedding/env/.env.vllm-embedding
chmod 600 services/vllm-embedding/env/.env.vllm-embedding
```

If the file already exists, do not overwrite it. Ask the user before changing anything in it.

Check that the names match the template, without printing values:

```sh
diff <(grep -oE '^#?\s*[A-Z_][A-Z_0-9]*=' services/vllm-embedding/env/.env.vllm-embedding.example | sed 's/^#\s*//;s/=$//' | sort) \
     <(grep -oE '^#?\s*[A-Z_][A-Z_0-9]*=' services/vllm-embedding/env/.env.vllm-embedding | sed 's/^#\s*//;s/=$//' | sort)
```

Lines that differ are a missing or extra name. Fix them by name only.

## Step 3: Fill in the values

| Variable | Who sets it | How to decide |
| --- | --- | --- |
| `VLLM_EMBED_MODEL` | Agent, with the user's confirmation | Default `Qwen/Qwen3-Embedding-0.6B`. Must be an embedding model. Confirm the model ID on Hugging Face. |
| `VLLM_EMBED_GPU_MEM_UTIL` | Agent, from Step 1 | Fraction of GPU memory (0 to 1). Keep it at or below the free share, and low if the chat LLM shares the GPU. Default `0.10`. |
| `VLLM_EMBED_MAX_MODEL_LEN` | Agent, with the user's confirmation | Max tokens per input. Default `8192`. |
| `HF_TOKEN` | User only | Needed only for gated or private models. Leave the line commented out otherwise. |

Write the values with a helper that prints nothing:

```sh
set_val() { sed -i "s|^$2=.*|$2=$3|" "$1"; }   # set_val <file> <KEY> <value>
set_val services/vllm-embedding/env/.env.vllm-embedding VLLM_EMBED_GPU_MEM_UTIL 0.10
```

Only set the value the user has confirmed. For `HF_TOKEN`, ask the user to edit the file themselves. Do not
ask for the token in chat.

## Step 4: Validate

```sh
docker compose --env-file .env -f services/vllm-embedding/docker-compose.yml config -q && echo "prod ok"
docker compose --env-file .env -f services/vllm-embedding/docker-compose.yml -f services/vllm-embedding/docker-compose.dev.yml config -q && echo "dev ok"
```

Use `-q` every time. Without it, `config` prints resolved env values. If a file is missing, Compose
reports it. An env file with `required: false` is optional, so a missing file here is not an error.

## Step 5: Start (after approval)

Ask the user to approve starting the container. Then:

```sh
./start.sh prod vllm-embedding
docker compose --env-file .env -f services/vllm-embedding/docker-compose.yml ps
```

The first start downloads the model, which can take several minutes. The healthcheck allows 120 seconds
before it starts counting failures. Watch the logs:

```sh
docker compose --env-file .env -f services/vllm-embedding/docker-compose.yml logs --tail=100 vllm-embedding
```

If the container exits with an argument error, check the flags in `docker-compose.yml` against the vLLM
version in the image. Report the exact message. Do not guess a fix.

## Step 6: Verify

Run the checks inside the container. No host port exists, and no other container is needed:

```sh
docker compose --env-file .env -f services/vllm-embedding/docker-compose.yml exec vllm-embedding \
  python3 -c "import urllib.request,json; print(json.load(urllib.request.urlopen('http://127.0.0.1:8000/v1/models'))['data'][0]['id'])"

docker compose --env-file .env -f services/vllm-embedding/docker-compose.yml exec vllm-embedding \
  python3 -c "import urllib.request,json; r=urllib.request.Request('http://127.0.0.1:8000/v1/embeddings', data=json.dumps({'model':'<model id>','input':['hello']}).encode(), headers={'Content-Type':'application/json'}); print(len(json.load(urllib.request.urlopen(r))['data'][0]['embedding']))"
```

Replace `<model id>` with the `VLLM_EMBED_MODEL` value. The second command prints the vector dimension
(1024 for the default model). The Compose healthcheck must show `healthy` in `ps`.

From another container on `backend`, the same request goes to `http://vllm-embedding:8000/v1/embeddings`.
Run it only if the user asks for a cross-container check.

## Step 7: Report

End with a short report:
- Host checks: GPU, runtime, backend network (pass or fail).
- Env file: created, names checked, values set by the agent, values left for the user (`HF_TOKEN` if needed).
- Validation: `config -q` results for prod and dev.
- Start: approved or not, container state, healthcheck state.
- Verification: model ID, embedding dimension, or the exact error.
- What is left: anything the user must do.
