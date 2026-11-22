# vllm-embedding

> This directory is a sub-project of [`private-ai-stack`](https://github.com/HoHuiHsieh/private-ai-stack).

vLLM OpenAI-compatible embedding server for `Qwen/Qwen3-Embedding-0.6B`. Internal only: no ports are
published. Other containers on the root stack's `backend` network call it at
`http://vllm-embedding:8000/v1/embeddings`.

## Run

```
cp services/vllm-embedding/env/.env.vllm-embedding.example services/vllm-embedding/env/.env.vllm-embedding
./start.sh prod vllm-embedding     # or: ./start.sh dev vllm-embedding (same containers, no hot reload)
```

The root stack must be up first (it creates the `private-ai-stack_backend` network).

Logs:

```
docker compose --env-file .env -f services/vllm-embedding/docker-compose.yml logs -f vllm-embedding
```

## Settings

| Variable | Default | Meaning |
| --- | --- | --- |
| `VLLM_EMBED_MODEL` | `Qwen/Qwen3-Embedding-0.6B` | Embedding model to serve |
| `VLLM_EMBED_GPU_MEM_UTIL` | `0.10` | Share of GPU memory vLLM may use |
| `VLLM_EMBED_MAX_MODEL_LEN` | `8192` | Max sequence length (tokens) |
| `HF_TOKEN` | unset | Only for gated or private models |

Values are read from `env/.env.vllm-embedding` at container start, so change them and run
`./start.sh prod vllm-embedding` again to recreate the container.

The image tag (`vllm/vllm-openai:latest`) is set in `docker-compose.yml`. Pin it there if you want
reproducible restarts.

## Checks

```
# From another container on the backend network (no host port exists):
curl -s http://vllm-embedding:8000/v1/models
curl -s http://vllm-embedding:8000/v1/embeddings \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-Embedding-0.6B","input":["hello"]}'
```

## Notes

- Needs an NVIDIA GPU. The `deploy.resources` reservation fails on a host without one. The old
  `scripts/start-vllm-embedding.sh` ran the same server with `docker run` and could run on CPU
  (`VLLM_GPUS=`). This compose file does not support CPU.
- No access control: any container on `backend` can call it. It has no nginx route.
- The model cache is the `hf_cache` volume in this project, so the model is downloaded once.
