# factory-cursor-bridge

A unified BYOK proxy that wires any OpenAI-compatible model into **Cursor IDE** via the "Override OpenAI Base URL" setting. Supports multiple providers, protocol translation (OpenAI ↔ Anthropic), and automatic model prefixing.

## How it works

```
Cursor IDE
    │
    ├─ Override OpenAI Base URL ──► https://your-tunnel-url.trycloudflare.com
    │   (OpenAI API Key = bearer token)
    │
    ▼
factory-cursor-bridge (:8316)
    │
    ├─ strips custom prefix (e.g. fx-) from model name
    ├─ looks up base URL & API key from config file
    ├─ translates protocol (OpenAI ↔ Anthropic) if needed
    │
    ▼
Your actual API providers
```

## Quick Start

### 1. Configure your models

Create a `config.json` file with your models:

```json
{
  "custom_models": [
    {
      "model": "my-gpt-4",
      "model_display_name": "My GPT-4",
      "base_url": "https://api.openai.com/v1",
      "api_key": "sk-...",
      "provider": "openai"
    },
    {
      "model": "my-claude",
      "model_display_name": "My Claude",
      "base_url": "https://api.anthropic.com/v1",
      "api_key": "sk-ant-...",
      "provider": "anthropic"
    },
    {
      "model": "my-local-model",
      "model_display_name": "Local Model",
      "base_url": "http://localhost:11434/v1",
      "api_key": "local",
      "provider": "openai"
    }
  ]
}
```

### 2. Start the proxy

```bash
node factory-cursor-bridge.mjs --config ./config.json --port 8316
```

Or set the environment variable:

```bash
export FACTORY_CONFIG=/path/to/your/config.json
node factory-cursor-bridge.mjs
```

### 3. Expose via tunnel

Cursor blocks `127.0.0.1` (SSRF protection). You need a public tunnel:

```bash
cloudflared tunnel --url http://127.0.0.1:8316
```

Use the `.trycloudflare.com` URL it gives you.

### 4. Configure Cursor

1. Settings → Models
2. Set **OpenAI API Key** to the bearer token from your config
3. Enable **Override OpenAI Base URL** → your tunnel URL
4. Click **+ Add Custom Model** and add your model names with the `fx-` prefix (e.g., `fx-my-gpt-4`, `fx-my-claude`)

## Config File Format

```json
{
  "custom_models": [
    {
      "model": "model-id-in-upstream",
      "model_display_name": "Display Name in Cursor",
      "base_url": "https://api.provider.com/v1",
      "api_key": "your-api-key",
      "provider": "openai | anthropic | generic-chat-completion-api",
      "supports_images": true,
      "extra_headers": { "X-Custom-Header": "value" },
      "extra_args": { "reasoning_effort": "high" }
    }
  ]
}
```

### Provider types

| Provider | Protocol | Notes |
|----------|----------|-------|
| `openai` | OpenAI | Standard chat completions |
| `anthropic` | Anthropic | Auto-converts to Anthropic Messages API |
| `generic-chat-completion-api` | OpenAI | Generic OpenAI-compatible backend |

## Model naming

Each model in your config gets a `fx-` prefix in Cursor:

| Config `model` | Cursor model name |
|-----------------|------------------|
| `gpt-4o` | `fx-gpt-4o` |
| `claude-3-5-sonnet` | `fx-claude-3-5-sonnet` |
| `qwen-72b` | `fx-qwen-72b` |

## Full example with multiple providers

```json
{
  "custom_models": [
    {
      "model": "gpt-5.4",
      "model_display_name": "GPT-5.4",
      "base_url": "http://localhost:8318/v1",
      "api_key": "local",
      "provider": "openai"
    },
    {
      "model": "claude-opus-4-6",
      "model_display_name": "Claude Opus",
      "base_url": "http://localhost:8318/v1",
      "api_key": "local",
      "provider": "anthropic"
    },
    {
      "model": "llama-3.1-70b",
      "model_display_name": "Llama 70B",
      "base_url": "https://api.groq.com/openai/v1",
      "api_key": "gsk_...",
      "provider": "openai"
    },
    {
      "model": "gemini-1.5-pro",
      "model_display_name": "Gemini Pro",
      "base_url": "https://generativelanguage.googleapis.com/v1beta",
      "api_key": "AI...",
      "provider": "generic-chat-completion-api"
    },
    {
      "model": "deepseek-coder",
      "model_display_name": "DeepSeek Coder",
      "base_url": "http://localhost:11434/v1",
      "api_key": "local",
      "provider": "openai"
    }
  ]
}
```

## CLI Options

```bash
node factory-cursor-bridge.mjs [options]

Options:
  --config <path>   Path to config.json (default: ~/.factory/config.json)
  --port <port>     Port to listen on (default: 8316)
  --host <host>     Host to bind to (default: 127.0.0.1)
  --token <token>   Bearer token for auth (default: from config)
```

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | Health check |
| `GET` | `/v1/models` | List available models |
| `POST` | `/v1/chat/completions` | Chat completions |

## Troubleshooting

**"Provider was unable to process your request" error**
→ Cursor blocks `127.0.0.1`. You must use a public tunnel URL.

**"Model not found" error**
→ Make sure you're using the `fx-` prefixed name in Cursor.

**Anthropic models not working**
→ The proxy auto-converts OpenAI format to Anthropic format for `anthropic` provider type.
