---
name: infer-lite
description: Drive the `infer` lite gateway — a stdlib-only Python LLM proxy (no Docker, no LiteLLM) that aggregates ~150 free LLM models behind a single OpenAI-compatible endpoint at http://localhost:4000. One Python file. `./infer serve` runs the gateway in the foreground; `./infer keys add` stores provider keys; `./infer probe` smoke-tests upstreams; `./infer wire <cc|opencode|pi|openclaw|hermes>` configures coding agents. Trigger when the user asks to add API keys, start/check the gateway, list models, test a model, add a new provider, refresh discovery, debug routing, wire a coding agent, or mentions "infer", "free models", "free LLMs", "local proxy", "lite gateway".
---

# infer-lite — Minimum Viable Free-LLM Gateway

A stripped-down sibling of `bradAGI/fi-gateway`. Same wire/probe/discovery surface; the LiteLLM-in-Docker proxy is replaced with a stdlib `http.server` forwarder.

## When to use this skill

Trigger phrases:

- "add my **<provider>** key" / "store my **<provider>** key"
- "start / boot / spin up the gateway / proxy / infer"
- "what free models do I have"
- "test **<model>**" / "smoke test"
- "add provider **<x>**"
- "refresh / re-sync OpenRouter / NVIDIA / Gemini"
- "wire **<tool>** to infer"
- "why is **<model>** failing"

Use the heavier sibling `bradAGI/fi-gateway` if the user needs `/v1/messages` (Anthropic shape), Cohere/Vertex/Bedrock providers, or LiteLLM's rate-limit-aware router.

## Layout

The skill lives at the repo root:

- `infer` — the CLI + HTTP server (single Python file, stdlib only).
- `providers.toml` — committed catalog of providers and models.
- `SKILL.md` — this file.

User data in `~/.config/infer/`:

- `keys.env` — chmod 600, gitignored.
- `.discovery-cache.json` — 24h cache of auto-discovered model lists.
- `.probe-cache.json` — 24h cache of `./infer probe` results.

## Architecture

```
infer (single Python file, stdlib only)
 ├── http.server bound to 127.0.0.1:4000
 │    ├── GET  /v1/models                → list catalog aliases
 │    ├── POST /v1/chat/completions      → forward to <upstream>/chat/completions
 │    ├── POST /v1/embeddings            → forward to <upstream>/embeddings
 │    └── GET  /health                   → liveness
 ├── ~/.config/infer/keys.env
 ├── providers.toml
 └── per-request: lookup alias → grab key → forward → stream SSE
```

`./infer serve` runs in foreground. There is **no** persistent background daemon, no Docker container, no config.yaml regeneration. The script IS the gateway.

## Commands

```
./infer serve [--bind ADDR] [--port N]   Run the gateway (foreground, blocks)
./infer doctor                           Health overview
./infer install [--name N]               Symlink into a $PATH dir
./infer uninstall

./infer keys add <provider> <key>
./infer keys list | remove <provider> [--index N]

./infer providers                        Active vs inactive
./infer models [-g GROUP] [-w | --broken]
./infer sync                             Refresh auto-discovered catalogs
./infer probe [-g G] [-p P] [-c N] [-t SEC]   Direct upstream smoke test

./infer detect | wire <tool> | unwire <tool>
```

`./infer wire <cc|opencode|pi|openclaw|hermes>` configures agent CLIs to send requests to `http://localhost:4000/v1` with a placeholder API key. The gateway accepts any auth on localhost.

## Catalog

Each provider needs `base_url` (the OpenAI-compatible endpoint root) and `key_env`:

```toml
[[provider]]
name = "scaleway"
key_env = "SCALEWAY_API_KEY"
base_url = "https://api.scaleway.ai/v1"
optional_key = false   # true for keyless providers (LLM7)

[[provider.model]]
alias = "qwen3.5-397b-scaleway"
upstream = "qwen3.5-397b-a17b"
groups = ["smart", "code"]
```

Or use `discovery = "openrouter_free"` / `"nvidia_nim"` / `"gemini"` to auto-fetch from a `/v1/models` endpoint.

## Workflows

### Onboarding

1. Ask which keys the user has. Easy wins: Gemini, OpenRouter, NVIDIA NIM.
2. `./infer keys add <name> <key>` for each.
3. `./infer sync` to populate auto-discovery caches.
4. `./infer serve` (in a separate terminal) to start the gateway.
5. `./infer probe` to confirm what works.
6. `./infer wire <tool>` for any coding agent CLIs they have installed.

### "Server is up but my client gets 404 on /v1/messages"

This is lite — it only implements `/v1/chat/completions` and `/v1/embeddings`. If the user's client requires Anthropic shape (e.g., Claude Code's full feature set), point them at the full `bradAGI/fi-gateway` instead.

### Backgrounding

`./infer serve` blocks. To run it in the background:

```bash
nohup ./infer serve > ~/.config/infer/serve.log 2>&1 &
echo $! > ~/.config/infer/serve.pid
```

Stop with `kill "$(cat ~/.config/infer/serve.pid)"`. For supervised mode, recommend systemd / launchd.

## Hard rules

- Never push to git unless the user explicitly asks.
- Never commit `~/.config/infer/*` — those are user secrets.
- Never add Co-Authored-By or Claude/Anthropic attribution to commits.
- Never edit `infer` to add third-party Python deps — stdlib-only is the whole point of "lite".
- Never bind the server to `0.0.0.0` without putting auth in front; the gateway accepts any Authorization header on localhost.
