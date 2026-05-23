# Routiq

[![PyPI version](https://badge.fury.io/py/routiq.svg)](https://pypi.org/project/routiq)
[![Python 3.9+](https://img.shields.io/badge/python-3.9+-blue.svg)](https://python.org)
[![Docker](https://img.shields.io/badge/docker-ghcr.io%2Fyashbafna%2Froutiq-blue.svg)](https://ghcr.io/yashbafna/routiq)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

> **The LLM gateway that picks the right model, right provider, right price — every time.**  
> This repo is the public home for bug reports, feature requests, and discussion.  

---

## What is Routiq?

Routiq is a local-first LLM gateway you run yourself. It sits between your app and LLM providers and reduces costs automatically — through caching, smart routing, and cost tracking. Your prompts never leave your machine.

**Supported providers:** OpenAI · Anthropic · DeepSeek · Gemini

**How it reduces costs:**

| Mechanism | What it does | Typical saving |
|-----------|-------------|----------------|
| Exact cache | Returns cached response for identical requests — zero provider cost | 20–40% |
| Semantic cache | Matches similar prompts using embeddings — catches paraphrases | 20–40% |
| Complexity routing | Sends simple prompts to cheap models, hard ones to powerful models | 40–60% |
| Provider cache injection | Leverages native caching where providers support it | 5–15% |

**Real numbers from testing:**
- 84% cache hit rate on repeated workloads
- 9ms p50 / 67ms p95 overhead added by Routiq
- Sub-50ms semantic cache responses

---

## Dashboard Preview

| Light Mode | Dark Mode |
|-----------|-----------|
| ![Light Mode Dashboard](https://github.com/user-attachments/assets/9ef05e10-32f6-4635-9903-dad2b5504678) | ![Dark Mode Dashboard](https://github.com/user-attachments/assets/042d4c64-3f8c-4e21-a25e-5a0710c70c21) |
| ![Light Mode Detail](https://github.com/user-attachments/assets/0f98574d-bebe-4e8e-b530-5b60094c48b7) | ![Dark Mode Detail](https://github.com/user-attachments/assets/0bb75a38-b2b9-4b94-b4ca-08ba72ae860b) |

---

## Quick Start

### Option A — pip (Python 3.9+, no Docker needed)

```bash
# 1. Install
pip install routiq

# 2. First run — creates .env in current directory
routiq start
# → .env created. Edit it and run again.

# 3. Add your keys to .env
ROUTIQ_API_KEY=any-secret-you-choose
GEMINI_API_KEY=...        # at least one provider key

# 4. Start the gateway
routiq start
# → Listening on http://localhost:8001

# 5. Health check
curl http://localhost:8001/health
# → {"status": "ok", "version": "0.1.4"}
```
![CLI Screenshot](https://github.com/user-attachments/assets/246761c0-522a-4b42-ba08-9ddb486f397e)

Redis is optional — caching is disabled gracefully if Redis isn't running.  
To enable caching: `brew install redis && redis-server` (Mac) or `apt install redis` (Linux).

---

### Option B — Docker (recommended for teams, includes Redis)

**Prerequisites:** [Docker Desktop](https://www.docker.com/products/docker-desktop/) (Mac/Windows) or Docker Engine (Linux)

```bash
# 1. Download compose file and env template
curl -O https://raw.githubusercontent.com/yashbafna/routiq-feedback/main/docker-compose.yml
curl -O https://raw.githubusercontent.com/yashbafna/routiq-feedback/main/.env.example
mv .env.example .env

# 2. Edit .env — add at least one provider key
# ROUTIQ_API_KEY=any-secret-you-choose
# GEMINI_API_KEY=...

# 3. Start (includes Redis automatically)
docker compose up -d

# 4. Health check
curl http://localhost:8001/health
# → {"status": "ok", "version": "0.1.0"}
```

---

## Connecting Your App

```python
from routiq import RoutiqClient

client = RoutiqClient(
    api_key="your-ROUTIQ_API_KEY",      # the key you set in .env
    base_url="http://localhost:8001"    # optional, this is the default
)

# Let Routiq pick the model automatically based on prompt complexity
response = client.chat(messages=[{"role": "user", "content": "Hello"}])
print(response.choices[0].message.content)

# Pin a specific provider and model
client.chat(model="claude-latest", messages=[...])
client.chat(model="gemini-flash-latest", messages=[...])
client.chat(model="deepseek-latest", messages=[...])
client.chat(model="gpt-4o-latest", messages=[...])

# Async support
response = await client.achat(messages=[...])
```

Routiq handles provider routing internally — you never change your code when switching providers, only the `model` string.

**Already using the OpenAI SDK?** Point it directly at Routiq without installing anything extra:
```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8001/v1", api_key="your-ROUTIQ_API_KEY")
```

`ROUTIQ_API_KEY` is a secret you choose yourself — it never leaves your machine.  
Provider keys (`OPENAI_API_KEY` etc.) live only in your `.env` and are never exposed to callers.

---

## Model Aliases

Aliases are pinned — provider releases won't silently break your app.

| Alias | Resolves To | Provider |
|-------|-------------|----------|
| `gpt-4o-latest` | gpt-4o-2024-11-20 | OpenAI |
| `gpt-4o-mini-latest` | gpt-4o-mini | OpenAI |
| `claude-latest` | claude-sonnet-4-20250514 | Anthropic |
| `claude-haiku-latest` | claude-haiku-4-5-20251001 | Anthropic |
| `deepseek-latest` | deepseek-chat | DeepSeek |
| `deepseek-reasoner-latest` | deepseek-reasoner | DeepSeek |
| `gemini-flash-latest` | gemini-2.5-flash | Gemini |
| `gemini-flash-lite-latest` | gemini-2.5-flash-lite | Gemini |
| `gemini-pro-latest` | gemini-2.5-pro | Gemini |
| `auto` | complexity-based selection | — |

Canonical model names (`gpt-4o`, `gpt-4o-mini`, `claude-sonnet-4-20250514` etc.) also work directly.

---

## Complexity Routing

When you use `model="auto"`, Routiq classifies each prompt and routes it to the cheapest model that can handle it:

| Complexity | Triggers | Default model |
|------------|----------|--------------|
| Simple | Short prompt, no tools | `gemini-flash-latest` |
| Medium | Moderate length or reasoning | `gpt-4o-mini` |
| Complex | Tools present, long context, keywords like `debug`, `implement`, `analyze` | `claude-latest` |

Override tiers in `.env`:
```env
SIMPLE_MODEL=gemini-flash-latest,gpt-4o-mini
MEDIUM_MODEL=gemini-flash-latest,gpt-4o-mini,claude-haiku-latest
COMPLEX_MODEL=gemini-pro-latest,claude-latest,gpt-4o
```

Models are tried left to right — if the first fails, the next is used.

---

## Dashboard

```
http://localhost:8001/dashboard
```

Shows 24h / 7d / 30d windows: total requests, cache hit rate (exact + semantic), total spend, total saved, Routiq overhead (p50/p95/p99), breakdown by model and complexity, recent request table with MISS / HIT / SIMILAR badges.

---

## Environment Variables

**Required:**
```env
ROUTIQ_API_KEY=any-secret-you-choose   # your gateway auth token
GEMINI_API_KEY=...                     # at least one provider key
```

**Optional:**
```env
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
DEEPSEEK_API_KEY=...
REDIS_URL=redis://localhost:6379       # enables caching (fail-open if absent)
SQLITE_PATH=./data/routiq.db          # cost tracking database
LOG_LEVEL=INFO
SEMANTIC_CACHE_ENABLED=true           # enable semantic similarity cache
SEMANTIC_THRESHOLD=0.82               # similarity threshold (0.0–1.0)
SIMPLE_MODEL=gemini-flash-latest,gpt-4o-mini
MEDIUM_MODEL=gemini-flash-latest,gpt-4o-mini,claude-haiku-latest
COMPLEX_MODEL=gemini-pro-latest,claude-latest,gpt-4o
```

Only providers with a key set are active. Adapters for unconfigured providers are skipped at startup.

---

## How to Use This Repo

| Use this repo for | Don't use this repo for |
|-------------------|------------------------|
| 🐛 Bug reports | Security issues (see below) |
| 💡 Feature requests | General LLM questions |
| 🗳️ Voting on what gets built | Getting started help (use Discussions) |
| 💬 Discussing roadmap | |

### Filing a Bug
[→ Open a Bug Report](../../issues/new?template=bug_report.md)

Please include:
- Routiq version (`curl http://localhost:8001/health`)
- Install method (pip or Docker)
- Provider and model string used
- Steps to reproduce
- Expected vs actual behaviour
- Relevant logs (sanitise your API keys before pasting)

### Requesting a Feature
[→ Open a Feature Request](../../issues/new?template=feature_request.md)

### Discussions
Use [GitHub Discussions](../../discussions) for questions and ideas.

---

## Security Issues

**Do not open a public issue for security vulnerabilities.**  
Email: yashbafnanitrr@gmail.com  
We aim to respond within 48 hours.

---

## Roadmap Transparency

Upvote issues (👍) to influence priority. We tag issues with:
- `planned` — on the roadmap
- `considering` — under evaluation
- `not-planned` — with an explanation why

---
## Learn More

**📖 Technical deep dive:**  
[Building Routiq: a local-first LLM gateway that proves its savings](https://yashbafna.hashnode.dev/building-routiq-local-first-llm-gateway#the-pipeline-end-to-end)

A 3,800-word breakdown of how Routiq works under the hood — the pipeline architecture, caching algorithms, complexity routing, cost recording, 22 security fixes, and what I got wrong on the first try.

---
**Built by [Yash Bafna](https://linkedin.com/in/yashbafna)**  
