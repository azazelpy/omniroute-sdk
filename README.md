# 🔀 OmniRoute SDK — Multi-Provider LLM Gateway Client

> **One API key, 200+ models, 16 providers** — TypeScript, Python, and Go clients for the OmniRoute gateway.

## Overview

OmniRoute is a lightweight Go gateway (10.0.0.110:20128) that aggregates 16+ LLM providers behind a single OpenAI-compatible API. This SDK provides idiomatic clients for the three languages used in the MINIX stack.

## Providers & Models (Live)

| Provider | Models | Speciality |
|----------|--------|------------|
| **NVIDIA** | Nemotron-3-Ultra, Laguna (Poolside), Nemotron-3-55B | Coding, reasoning |
| **Fireworks** | DeepSeek-V4, Llama 3.3, Mixtral, Qwen2.5 | Speed + variety |
| **Groq** | Llama 3.3 70B, Mixtral, Gemma 2 | LPU inference (sub-second) |
| **xAI** | Grok-2, Grok-2-mini | Reasoning |
| **Google** | Gemini 1.5 Pro/Flash | Multimodal, long context |
| **DeepInfra** | 50+ open models (Llama, Qwen, Yi, Phi) | Breadth |
| **OpenRouter** | Aggregator (Claude, GPT, etc.) | Fallback |
| **Auggie** | Code-specific models | Coding |
| **Chipotle** | Specialized fine-tunes | Niche |
| **Combo** | Router ensembles | Best-of-N |
| **Ollama Cloud** | Local models via cloud | Privacy |

## Model Routing (Auto-Selection)

```yaml
# Server-side config.yaml
routing:
  auto:
    - task: coding
      model: nvidia/poolside/laguna-xs-2.1
      fallback: [fireworks/deepseek-v4-flash, groq/llama-3.3-70b]
    - task: chat
      model: z.ai/glm-5.2
      fallback: [nvidia/nemotron-3-ultra, xai/grok-2]
    - task: vision
      model: openai/gpt-4o
      fallback: [google/gemini-1.5-pro]
    - task: fast
      model: groq/llama-3.1-8b-instant
      fallback: [fireworks/llama-3.2-3b]
```

**Usage:** `model: "auto:coding"` → routes to Laguna with automatic failover.

## Quick Start

### TypeScript / Node.js

```bash
npm install @minix/omniroute-sdk
```

```typescript
import { OmniRoute } from "@minix/omniroute-sdk";

const client = new OmniRoute({
  apiKey: process.env.OMNIROUTE_API_KEY,
  baseURL: "http://10.0.0.110:20128/v1"
});

// Streaming chat completion
const stream = await client.chat.completions.create({
  model: "auto:coding",
  messages: [{ role: "user", content: "Write a fibonacci function in Go" }],
  stream: true,
});

for await (const chunk of stream) {
  process.stdout.write(chunk.choices[0]?.delta?.content || "");
}
```

### Python

```bash
pip install omniroute-sdk
```

```python
from omniroute import OmniRoute
import os

client = OmniRoute(
    api_key=os.getenv("OMNIROUTE_API_KEY"),
    base_url="http://10.0.0.110:20128/v1"
)

# Non-streaming
resp = await client.chat.completions.create(
    model="auto:chat",
    messages=[{"role": "user", "content": "Explain RPO vs RTO"}]
)
print(resp.choices[0].message.content)

# Streaming
async for chunk in await client.chat.completions.create(
    model="auto:coding",
    messages=[{"role": "user", "content": "Python async context manager"}],
    stream=True
):
    print(chunk.choices[0].delta.content or "", end="", flush=True)
```

### Go

```bash
go get github.com/minix/omniroute-sdk-go
```

```go
package main

import (
    "context"
    "fmt"
    "os"

    omniroute "github.com/minix/omniroute-sdk-go"
)

func main() {
    client := omniroute.NewClient(&omniroute.Config{
        APIKey:  os.Getenv("OMNIROUTE_API_KEY"),
        BaseURL: "http://10.0.0.110:20128/v1",
    })

    stream, err := client.Chat.Completions.CreateStream(context.Background(), omniroute.ChatCompletionRequest{
        Model: "auto:coding",
        Messages: []omniroute.Message{
            {Role: "user", Content: "Go HTTP server with graceful shutdown"},
        },
    })
    if err != nil {
        panic(err)
    }
    defer stream.Close()

    for {
        chunk, err := stream.Recv()
        if err != nil { break }
        fmt.Print(chunk.Choices[0].Delta.Content)
    }
}
```

## Features

| Feature | TS | Python | Go |
|---------|----|--------|----|
| Chat completions | ✅ | ✅ | ✅ |
| Streaming (SSE) | ✅ | ✅ | ✅ |
| Tool/function calling | ✅ | ✅ | ✅ |
| Auto-retry + backoff | ✅ | ✅ | ✅ |
| Request/response hooks | ✅ | ✅ | ✅ |
| Typed responses | ✅ | ✅ | ✅ |
| Zero dependencies | ✅ | ✅ | ✅ |

## Configuration

| Env Var | Required | Default |
|---------|----------|---------|
| `OMNIROUTE_API_KEY` | Yes | — |
| `OMNIROUTE_BASE_URL` | No | `http://10.0.0.110:20128/v1` |
| `OMNIROUTE_TIMEOUT` | No | `30s` |
| `OMNIROUTE_MAX_RETRIES` | No | `3` |

## Production Use Cases

| Service | Model | Purpose |
|---------|-------|---------|
| **Karina Bot** | `auto:coding` / `nvidia/poolside/laguna-xs-2.1` | Invoice JSON extraction |
| **Code Review Bot** | `auto:coding` | PR analysis, security issues |
| **Incident Summarizer** | `auto:chat` / `z.ai/glm-5.2` | Alert → narrative |
| **Runbook Generator** | `auto:coding` | Postmortem → runbook |
| **This Portfolio** | `auto:fast` | Blog content, skill tagging |

## Management API

```bash
# Health
curl -H "Authorization: Bearer $OMNIROUTE_API_KEY" http://10.0.0.110:20128/api/health

# List models
curl -H "Authorization: Bearer $OMNIROUTE_API_KEY" http://10.0.0.110:20128/api/models

# List providers
curl -H "Authorization: Bearer $OMNIROUTE_API_KEY" http://10.0.0.110:20128/api/providers
```

## Architecture

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  Your App   │ ──► │ OmniRoute    │ ──► │ 16 Providers│
│  (TS/Py/Go) │     │ Gateway      │     │ (NVIDIA,    │
└─────────────┘     │ :20128/v1    │     │  Fireworks, │
                    │ Auto-router  │     │  Groq, etc.)│
                    └──────────────┘     └─────────────┘
```

## License

MIT — Part of MINIX stack 🇵🇾