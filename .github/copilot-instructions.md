# vLLM embedding service

This is the standalone vLLM embedding Compose repository inside `private-ai-stack/services/vllm-embedding`.

- The deployment skill is `.github/skills/deploy-bare-metal/SKILL.md`.
- Never read or print `env/.env.vllm-embedding` or secrets such as `HF_TOKEN`. Read only `env/.env.vllm-embedding.example` and report variable names.
- Do not add `ports:` or `environment:` entries, change the image tag, or add services without approval.
- Preserve the NVIDIA GPU reservation. Do not start containers or download the model until the user approves it.
- Validate both prod and dev Compose configurations with `config -q` so resolved environment values are never printed.
- The service is reachable only on `private-ai-stack_backend` at `http://vllm-embedding:8000/v1`.