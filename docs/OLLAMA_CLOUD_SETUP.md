# Ollama Cloud Setup for NanoClaw

This guide describes how to configure NanoClaw to use Ollama Cloud models (like `glm-5:cloud`) as the primary LLM backend instead of the default Anthropic models.

## Architecture

To bridge the Claude Agent SDK's requirements with Ollama Cloud, we use a local **LiteLLM proxy** that acts as an Anthropic-compatible gateway. This allows the SDK to communicate with a local endpoint while actually routing requests to the Ollama Cloud API.

1.  **NanoClaw Orchestrator**: Manages messaging channels (WhatsApp, Telegram, Slack).
2.  **Claude Agent SDK**: Runs inside isolated Docker containers to perform tasks.
3.  **LiteLLM Proxy**: Converts Anthropic-formatted requests into Ollama Cloud API calls and aliases the model name.
4.  **OneCLI Gateway**: NanoClaw's internal security gateway is configured to *bypass* traffic destined for the local LiteLLM proxy to avoid proxy-on-proxy collision errors.

---

## 1. Prerequisites

-   **uv**: A fast Python package manager. Install it via `curl -fsSL https://astral.sh/uv/install.sh | sh`.
-   **Docker**: For running agent containers.
-   **Ollama Cloud API Key**: Get your key from the Ollama Cloud dashboard.

---

## 2. LiteLLM Proxy Configuration

Create a `litellm_config.yaml` in the project root:

```yaml
model_list:
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: ollama_chat/glm-5:cloud
      api_base: https://api.ollama.com
      api_key: "YOUR_OLLAMA_API_KEY"
      drop_params: true
      max_tokens: 4096

general_settings:
  # Force LiteLLM to accept Anthropic requests without strict validation
  anthropic_version: "2023-06-01"
  allow_unauthorized_requests: true
```

### Starting LiteLLM

Run the proxy using `uv` (it's recommended to use an isolated environment):

```bash
uv run --python 3.12 --with "litellm[proxy]" litellm --config litellm_config.yaml --port 4000
```

---

## 3. NanoClaw Environment Setup

Update your `.env` (and sync it to `data/env/env`) with the following variables:

```bash
# Point NanoClaw to the local LiteLLM proxy
ANTHROPIC_BASE_URL=http://host.docker.internal:4000
ANTHROPIC_AUTH_TOKEN=ollama
# A placeholder key to satisfy the Agent SDK's initial validation
ANTHROPIC_API_KEY=sk-ant-api01-proxy-placeholder-key-000000000000000000000000000000000000000000000000
# The alias name defined in litellm_config.yaml
ANTHROPIC_MODEL=claude-3-5-sonnet-20241022
```

---

## 4. Code Changes

To ensure the agent container can reach the host-level proxy and bypass internal proxy conflicts, the following changes were applied to NanoClaw:

### `src/config.ts`

Exports the new `ANTHROPIC_*` variables from `.env`:

```typescript
export const ANTHROPIC_BASE_URL = process.env.ANTHROPIC_BASE_URL || envConfig.ANTHROPIC_BASE_URL;
export const ANTHROPIC_API_KEY = process.env.ANTHROPIC_API_KEY || envConfig.ANTHROPIC_API_KEY;
export const ANTHROPIC_AUTH_TOKEN = process.env.ANTHROPIC_AUTH_TOKEN || envConfig.ANTHROPIC_AUTH_TOKEN;
export const ANTHROPIC_MODEL = process.env.ANTHROPIC_MODEL || envConfig.ANTHROPIC_MODEL;
```

### `src/container-runner.ts`

Injects these variables into the Docker `run` command and adds `NO_PROXY` rules for local traffic:

```typescript
if (ANTHROPIC_BASE_URL) {
  args.push('-e', `ANTHROPIC_BASE_URL=${ANTHROPIC_BASE_URL}`);

  // Prevent OneCLI proxy from intercepting traffic to the local LiteLLM proxy
  if (ANTHROPIC_BASE_URL.includes('host.docker.internal')) {
    args.push('-e', 'NO_PROXY=host.docker.internal,127.0.0.1,localhost');
    args.push('-e', 'no_proxy=host.docker.internal,127.0.0.1,localhost');
  }
}
```

---

## 5. Troubleshooting

-   **API Error 404 (`/v1/v1/messages`)**: Ensure `ANTHROPIC_BASE_URL` in `.env` does **not** end with `/v1`. The SDK appends it automatically.
-   **API Error 400 (Database issues)**: Ensure `allow_unauthorized_requests: true` is set in `litellm_config.yaml` to run in stateless mode.
-   **HPE_INVALID_CONSTANT**: This occurs when OneCLI attempts to proxy traffic to the local LiteLLM proxy. Ensure the `NO_PROXY` logic is active in `src/container-runner.ts`.
-   **Invalid API Key**: LiteLLM might still be trying to validate keys. Verify the "Open Mode" settings in your YAML.
