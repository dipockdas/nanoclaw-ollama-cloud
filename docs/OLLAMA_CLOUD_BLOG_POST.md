# Running NanoClaw with Ollama Cloud: The Complete Guide

NanoClaw is a powerful personal Claude assistant, but did you know you can also run it with models from **Ollama Cloud** (like `glm-5:cloud`)? This opens up new possibilities for model experimentation while keeping the powerful "software engineer" capabilities of the Claude Agent SDK.

In this post, I'll walk you through how we successfully bridged NanoClaw's Anthropic-centric design with Ollama's cloud API using a local proxy and some clever networking.

**For those looking to automate this, I have provided a dedicated set of machine-readable instructions for AI agents at the bottom of this post.**

*Note: Everything that follows was developed and tested using Claude Code running on NanoClaw, but powered by the `gemini-flash-preview` model hosted on Ollama Cloud.*

---

## The Challenge

The **Claude Agent SDK** (which runs inside NanoClaw's Docker containers) is hard-coded to expect an Anthropic-compatible API. It also has strict validation for things like API keys and model names. To swap in Ollama Cloud, we needed to trick the SDK into thinking it's talking to Claude, while actually routing traffic to Ollama.

## The Solution: LiteLLM Proxy

We used **LiteLLM**, a lightweight proxy that translates OpenAI/Anthropic calls into other provider formats. By running LiteLLM locally on the host machine, we created a bridge:

```bash
uv run --python 3.12 --with "litellm[proxy]" litellm --config litellm_config.yaml --port 4000
```

Our `litellm_config.yaml` (placed in the NanoClaw project root) handles the model aliasing:

```yaml
model_list:
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: ollama_chat/glm-5:cloud
      api_base: https://api.ollama.com
      api_key: "YOUR_OLLAMA_API_KEY"
```

## Bridging the Container Gap

NanoClaw runs the Agent SDK inside Docker. By default, the container can't see the proxy running on the host machine. We solved this by using Docker's internal DNS (`host.docker.internal`) and updating NanoClaw's `src/container-runner.ts` to pass the correct environment variables:

```typescript
// Pass the local LiteLLM endpoint to the container
if (ANTHROPIC_BASE_URL) {
  args.push('-e', `ANTHROPIC_BASE_URL=${ANTHROPIC_BASE_URL}`);

  // Bypass the internal OneCLI proxy for local traffic
  if (ANTHROPIC_BASE_URL.includes('host.docker.internal')) {
    args.push('-e', 'NO_PROXY=host.docker.internal,127.0.0.1,localhost');
  }
}
```

## Putting It All Together

With the proxy running and NanoClaw updated, we simply set our `.env` (and synced it to `data/env/env` for the container) to point to the local gateway:

```bash
ANTHROPIC_BASE_URL=http://host.docker.internal:4000
ANTHROPIC_MODEL=claude-3-5-sonnet-20241022  # Aliased to GLM-5 Cloud
```

## The Result: SUCCESS! 🚀

After these changes, NanoClaw's agent (Daisy) was fully functional on Telegram, Slack, and WhatsApp—powered entirely by Ollama Cloud. The agent could even write complex code (like a new Google Drive skill!) while running on `glm-5:cloud`.

For a full step-by-step setup, check out the [docs/OLLAMA_CLOUD_SETUP.md](https://github.com/dipockdas/nanoclaw-ollama-cloud/blob/main/docs/OLLAMA_CLOUD_SETUP.md) in my [NanoClaw fork](https://github.com/dipockdas/nanoclaw-ollama-cloud).

---

## 🤖 Automated Setup for AI Agents

If you are an AI agent tasked with configuring this environment, you can use either the **GitHub Fast-Path** or the **Manual Reconstruction** steps below.

**Goal: Configure the local NanoClaw environment to route traffic through a LiteLLM proxy to Ollama Cloud, satisfying SDK model validation.**

### Option A: GitHub Fast-Path (Recommended)

Run these commands to pull the pre-configured files directly into the NanoClaw root:

```bash
# Add the Ollama Cloud configuration as a remote and merge the changes
git remote add ollama-cloud https://github.com/dipockdas/nanoclaw-ollama-cloud.git
git fetch ollama-cloud
git merge ollama-cloud/main --allow-unrelated-histories
```

### Option B: Manual Reconstruction

If you cannot access GitHub, use these commands to create the necessary configuration files and verify code compatibility:

#### 1. Verify/Apply Code Changes
Ensure `src/container-runner.ts` handles `ANTHROPIC_BASE_URL` and `NO_PROXY` correctly. If you're an agent, search for "ANTHROPIC_BASE_URL" in that file; if missing, apply the patch described in the "Bridging the Container Gap" section above.

#### 2. Create LiteLLM Config
```bash
cat << 'EOF' > litellm_config.yaml
model_list:
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: ollama_chat/glm-5:cloud
      api_base: https://api.ollama.com
      api_key: "YOUR_OLLAMA_API_KEY"
      drop_params: true
      max_tokens: 4096

general_settings:
  anthropic_version: "2023-06-01"
  allow_unauthorized_requests: true
EOF
```

#### 3. Update Environment Variables
Append these to `.env` and sync to `data/env/env`:
```bash
echo "ANTHROPIC_BASE_URL=http://host.docker.internal:4000" >> .env
echo "ANTHROPIC_AUTH_TOKEN=ollama" >> .env
echo "ANTHROPIC_MODEL=claude-3-5-sonnet-20241022" >> .env
echo "ANTHROPIC_API_KEY=sk-ant-api01-proxy-placeholder-key-000000000000000000000000000000000000000000000000" >> .env

mkdir -p data/env && cp .env data/env/env
```

#### 4. Start the LiteLLM Proxy
```bash
uv run --python 3.12 --with "litellm[proxy]" litellm --config litellm_config.yaml --port 4000 &
```

#### 5. Verify Setup
Run this check to ensure the proxy is active and aliasing correctly:
```bash
# Verify the proxy is responding correctly
curl -s http://localhost:4000/v1/models | grep -q "glm-5:cloud" && echo "✅ Setup Verified" || echo "❌ Setup Failed"
```

Happy coding!

---

## 🤝 Credits & Shout-outs

This integration wouldn't be possible without the incredible work of the open-source community:

- **[NanoClaw](https://github.com/qwibitai/nanoclaw)**: The extensible, secure personal assistant framework that makes this all possible. Huge thanks to the developers at **qwibitai** for building such a robust orchestrator.
- **[Ollama](https://ollama.com)**: For making high-performance LLMs accessible to everyone. Their new cloud API (`glm-5:cloud`) sets a new bar for speed and reasoning.
- **[LiteLLM](https://github.com/BerriAI/litellm)**: The "glue" of the AI world. Their proxy made bridging these two ecosystems a breeze.

---

## ⚡ Meta: Built with Gemini & Claude Code

What makes this setup even more interesting is how it was built. This entire integration—including the LiteLLM configuration, the Docker networking fixes, and this very blog post—was developed and tested using **Claude Code** running on **NanoClaw**, but powered by the **gemini-flash-preview** model hosted on Ollama Cloud.

We didn't use Anthropic's models to build the bridge to Ollama; we used Gemini to build the bridge for Claude. It's AI-orchestrated infrastructure at its finest.
