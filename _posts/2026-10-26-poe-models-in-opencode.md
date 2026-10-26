---
layout: post
title:  "Adding Poe's newest models to OpenCode"
date:   2026-10-26 12:00:00
description: Why OpenCode's Poe catalog lags behind Poe itself, and how to add any missing model by hand through the OpenAI-compatible API — with Kimi-K3.1-Pro as the worked example
tags: opencode poe llm tooling
categories: tooling
---

> **TL;DR** You do not have to wait for OpenCode's Poe integration to catch up. Poe exposes an [OpenAI-compatible API](https://creator.poe.com/docs/external-applications/openai-compatible-api) at `https://api.poe.com/v1`, so any bot in Poe's catalog can be wired into OpenCode by hand — same endpoint, same key, no extra setup.

I use OpenCode as my terminal coding agent and pay for Poe, which gives me one subscription covering frontier models from every lab. Logging into Poe inside OpenCode works, but the model picker only knows about the models that were in OpenCode's [models.dev](https://models.dev) catalog the last time that catalog was regenerated. Poe ships new models far faster than that catalog is updated.

Concretely, while writing this: Poe's `/v1/models` returned **302** models, while OpenCode's Poe entry offered **58** selectable ones. Kimi-K3.1-Pro was in the first list and not in the second.

The fix is a few lines of config, plus one API call to read the metadata OpenCode needs. I'll use Kimi-K3.1-Pro as the running example throughout, but nothing below is specific to it — swap the ID and the metadata for any other bot Poe serves.

## Step 1 — Check that Poe actually has the model

Grab an API key from <https://poe.com/api/keys>, then list the catalog:

```bash
export POE_API_KEY="sk-..."   # https://poe.com/api/keys

curl -s https://api.poe.com/v1/models \
  -H "Authorization: Bearer $POE_API_KEY" \
  | jq -r '.data[].id' | grep -i kimi
```

```text
kimi-k3.1-pro
kimi-k3
kimi-k3-el
kimi-k2.6
```

The IDs are lowercase and are exactly what you pass as `model` in a request — Poe's web URLs (`poe.com/kimi-k3.1-pro`) use the same string.

Before touching any config, confirm the model really answers:

```bash
curl -s https://api.poe.com/v1/chat/completions \
  -H "Authorization: Bearer $POE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"kimi-k3.1-pro","messages":[{"role":"user","content":"Reply with exactly: ok"}]}' \
  | jq -r '.choices[0].message.content'
```

If that returns `ok`, the model is live on your account and everything after this is just bookkeeping.

## Step 2 — Read the metadata OpenCode needs

OpenCode wants context size, input modalities, tool support, and pricing, and it will happily invent defaults if you omit them (200k context, 32k output, text+image, tools on). Those guesses are wrong often enough to matter, so pull the real values:

```bash
curl -s https://api.poe.com/v1/models \
  -H "Authorization: Bearer $POE_API_KEY" \
  | jq '.data[] | select(.id=="kimi-k3.1-pro")
       | {id, context_length, architecture, supported_features, pricing, parameters}'
```

```json
{
  "id": "kimi-k3.1-pro",
  "context_length": 1048576,
  "architecture": {
    "input_modalities": ["text", "image"],
    "output_modalities": ["text"]
  },
  "supported_features": ["tools"],
  "pricing": {
    "prompt": "0.0000020202",
    "completion": "0.000010101",
    "input_cache_read": "0.000000202"
  },
  "parameters": [
    {
      "name": "reasoning_effort",
      "schema": { "enum": ["low", "high", "max"] },
      "default_value": "high"
    }
  ]
}
```

The translation into OpenCode fields is mechanical:

| Poe | OpenCode |
| --- | --- |
| `id` | key in the `models` map (add `modelID` only if you want an alias) |
| `metadata.display_name` | `name` |
| `context_length` | `limit.context` |
| *(absent)* | `limit.output` — set it yourself, otherwise 32k |
| `architecture.input_modalities` | `capabilities.input` |
| `supported_features` contains `tools` | `capabilities.tools` |
| `pricing.prompt` × 10⁶ | `cost.input` |
| `pricing.completion` × 10⁶ | `cost.output` |
| `pricing.input_cache_read` × 10⁶ | `cost.cache.read` |
| `parameters[name=reasoning_effort]` | `variants[].body.reasoning_effort` |

Poe reports prices **per token**; OpenCode wants **per million tokens**, hence the ×10⁶.

## Step 3 — Point OpenCode at Poe

Poe lives in the global config, `~/.config/opencode/opencode.jsonc`, so the model becomes available in every project. OpenCode's current format is the `providers` map (`provider`/`npm`/`options` still work but are the deprecated V1 spelling):

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "providers": {
    "poe": {
      "name": "Poe",
      "package": "aisdk:@ai-sdk/openai-compatible",
      "settings": {
        "baseURL": "https://api.poe.com/v1",
        "apiKey": "{file:~/.secrets/poe-key}"
      },
      "models": {}
    }
  }
}
```

`{file:...}` keeps the key out of the config file. If you prefer an environment variable, use `{env:POE_API_KEY}` instead.

## Step 4 — Add the model

Drop the entry into that `models` map:

```jsonc
"kimi-k3.1-pro": {
  "name": "Kimi-K3.1-Pro",
  "releaseDate": "2026-10-23",
  "capabilities": {
    "tools": true,
    "input": ["text", "image"],
    "output": ["text"],
    "reasoning": { "supported": true }
  },
  "limit": { "context": 1048576, "output": 131072 },
  "cost": {
    "input": 2.0202,
    "output": 10.101,
    "cache": { "read": 0.202, "write": 0 }
  },
  "variants": [
    { "id": "low",  "body": { "reasoning_effort": "low" } },
    { "id": "high", "body": { "reasoning_effort": "high" } },
    { "id": "max",  "body": { "reasoning_effort": "max" } }
  ]
}
```

Two details worth calling out:

- **`body` for reasoning, not `settings`.** Poe treats `reasoning_effort` as a pass-through body parameter (the OpenAI SDK's `extra_body`), so putting it in `body` guarantees it reaches the wire.
- **`limit.output` is an assumption.** Poe reports `max_output_tokens: null` for this model, so I used 131072 from the Kimi K3 family. Leaving it out silently caps you at 32k.

## Step 5 — Reload and verify

OpenCode watches its config and reloads without a restart. Open `/models` and you should see `poe/kimi-k3.1-pro`, plus `poe/kimi-k3.1-pro#low`, `#high`, and `#max`. Then:

1. Send a normal prompt — checks auth, streaming, and the model ID.
2. Ask for a tool call ("list the files in this directory") — Poe's tool support is best-effort, so verify it with your own eyes before trusting it on a real task.

## Why not just log into Poe?

Because the login path and the API path read the same stale catalog. Logging in gives you credentials; the *list* of models still comes from models.dev, which is regenerated on its own schedule and, at the time of writing, keyed its Poe entries with vendor prefixes (`poe/anthropic/claude-opus-4.8`, `poe/novita/kimi-k2.6`) rather than the bare IDs Poe's API returns. Hand-written entries bypass that entirely: you read the truth from `/v1/models` and write it into your config.

## Caveats

- **Billing is Poe points**, not per-token invoices. Prices above are the reference rates Poe publishes, and they show up in OpenCode's cost display.
- **Poe ignores several OpenAI parameters**: `response_format` (including `json_schema`), `presence_penalty`, `frequency_penalty`, `seed`, `logit_bias`, and `n > 1`. Structured output is not enforced and the `strict` flag on tools is ignored.
- **Rate limit is 500 requests/minute** per key, with no token-based limiting.
- **Media bots** (image, video, audio) are not chat models — don't add them to `models`.
- If a hand-added model stops resolving after a Poe rename, re-run Step 1; the ID is the contract.

## Bulk-adding the rest

Once the pattern is clear, generating entries for the missing models is a small script. This one keeps only models that advertise tool support, which filters out most media bots:

```python
import json, os, urllib.request

req = urllib.request.Request(
    "https://api.poe.com/v1/models",
    headers={"Authorization": f"Bearer {os.environ['POE_API_KEY']}"},
)
models = json.load(urllib.request.urlopen(req))["data"]

entries = {}
for m in models:
    if "tools" not in m.get("supported_features", []):
        continue
    p = m["pricing"]
    entries[m["id"]] = {
        "name": m["metadata"]["display_name"],
        "capabilities": {
            "tools": True,
            "input": m["architecture"]["input_modalities"],
            "output": m["architecture"]["output_modalities"],
        },
        # output limit omitted: OpenCode falls back to 32k unless you set it
        "limit": {"context": m["context_length"]},
        "cost": {
            "input": round(float(p["prompt"]) * 1e6, 4),
            "output": round(float(p["completion"]) * 1e6, 4),
            "cache": {
                "read": round(float(p["input_cache_read"] or 0) * 1e6, 4),
                "write": 0,
            },
        },
    }

print(json.dumps(entries, indent=2))
```

Paste the output into `providers.poe.models` and you have Poe's whole text catalog in OpenCode — no waiting on an upstream catalog refresh.
