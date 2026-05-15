# Deploy Crawl4AI on Dokploy

Use `dokploy-compose.yml` as the Raw provider compose file.

## Dokploy variables

Minimum:

```env
APP_NAME=crawl4ai
PORT=11235
TAG=0.8.6
SHM_SIZE=1gb
```

Optional LLM keys:

```env
LLM_PROVIDER=openai/gpt-4o-mini
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=
GROQ_API_KEY=
DEEPSEEK_API_KEY=
TOGETHER_API_KEY=
MISTRAL_API_KEY=
GEMINI_API_TOKEN=
```

The API will listen inside the container on port `11235`.

Useful URLs after deployment:

- `/playground`
- `/dashboard`
- `/health`
- `/metrics`

For production, keep `CRAWL4AI_HOOKS_ENABLED=false` unless you explicitly need hooks.
