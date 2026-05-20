Here's a complete guide to setting up LiteLLM proxy and connecting it to n8n's AI Agent as an OpenAI Chat Model.

---

## Overview

LiteLLM is an open-source AI Gateway that gives you a single, unified interface to call 100+ LLM providers — OpenAI, Anthropic, Gemini, Bedrock, Azure, and more — using the OpenAI format. The proxy is a self-hosted OpenAI-compatible gateway. Any client that works with OpenAI works with the proxy — no code changes needed.

---

## Step 1 — Install LiteLLM

**Option A: pip (local)**
```bash
pip install 'litellm[proxy]'
```

**Option B: Docker (recommended for n8n)**
```bash
docker pull ghcr.io/berriai/litellm:main-stable
```

---

## Step 2 — Create `config.yaml`

This is the core config file. Pick your provider below:

**OpenAI:**
```yaml
model_list:
  - model_name: gpt-4o          # name n8n will use
    litellm_params:
      model: openai/gpt-4o
      api_key: os.environ/OPENAI_API_KEY

  - model_name: gpt-3.5-turbo
    litellm_params:
      model: openai/gpt-3.5-turbo
      api_key: os.environ/OPENAI_API_KEY

general_settings:
  master_key: sk-my-master-key   # set your own secret key
```

**Anthropic (Claude):**
```yaml
model_list:
  - model_name: claude-sonnet-4   # name n8n will use
    litellm_params:
      model: anthropic/claude-sonnet-4-20250514
      api_key: os.environ/ANTHROPIC_API_KEY

general_settings:
  master_key: sk-my-master-key
```

**Ollama (local models):**
```yaml
model_list:
  - model_name: llama3
    litellm_params:
      model: ollama/llama3
      api_base: http://localhost:11434

general_settings:
  master_key: sk-my-master-key
```

You can mix multiple providers in the same config.

---

## Step 3 — Run LiteLLM Proxy

**Option A: Docker Compose (best for n8n)**

Create `docker-compose.yaml`:
```yaml
services:
  litellm:
    image: ghcr.io/berriai/litellm:main-stable
    container_name: litellm
    ports:
      - "4000:4000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - LITELLM_MASTER_KEY=sk-my-master-key
    volumes:
      - ./config.yaml:/app/config.yaml:ro
    command: ["--config", "/app/config.yaml", "--port", "4000"]
```

Create `.env`:
```env
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
```

Start it:
```bash
docker compose up -d
```

**Option B: Docker run (quick)**

Run the container mounting your config file on port 4000:
```bash
docker run \
  -v $(pwd)/config.yaml:/app/config.yaml \
  -e OPENAI_API_KEY=sk-... \
  -p 4000:4000 \
  ghcr.io/berriai/litellm:main-stable \
  --config /app/config.yaml
```

**Option C: pip CLI (simplest)**
```bash
OPENAI_API_KEY=sk-... litellm --config config.yaml --port 4000
```

---

## Step 4 — Test the Proxy

```bash
curl -X POST http://localhost:4000/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-my-master-key" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

You should get a normal OpenAI-format response back.

---

## Step 5 — Connect to n8n AI Agent

In n8n, the LiteLLM proxy appears as an **OpenAI Chat Model** credential.

1. Open your n8n workflow and add an **AI Agent** node.
2. Under **Chat Model**, select **OpenAI Chat Model**.
3. Click **Create New Credential** → choose **OpenAI API**.
4. Fill in:
   | Field | Value |
   |---|---|
   | **API Key** | `sk-my-master-key` (your master key from config) |
   | **Base URL** | `http://localhost:4000` (or your server IP/hostname) |
5. Save the credential.
6. In the model node, type the model name exactly as it appears in your `config.yaml` — e.g. `gpt-4o`, `claude-sonnet-4`, `llama3`.

> **If n8n and LiteLLM are both in Docker**, use the Docker service name instead of `localhost`:
> `http://litellm:4000`
> or your host machine IP: `http://172.17.0.1:4000`

---

## Docker Compose for n8n + LiteLLM Together

```yaml
services:
  litellm:
    image: ghcr.io/berriai/litellm:main-stable
    container_name: litellm
    ports:
      - "4000:4000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - LITELLM_MASTER_KEY=sk-my-master-key
    volumes:
      - ./config.yaml:/app/config.yaml:ro
    command: ["--config", "/app/config.yaml", "--port", "4000"]

  n8n:
    image: n8nio/n8n
    container_name: n8n
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=password
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - litellm

volumes:
  n8n_data:
```

In this setup, use `http://litellm:4000` as the Base URL in n8n.

---

## Summary

| What | Value |
|---|---|
| Proxy URL | `http://localhost:4000` |
| n8n Base URL | `http://litellm:4000` (Docker) |
| API Key | your `master_key` from config |
| Model name | matches `model_name` in config.yaml |
| n8n node type | OpenAI Chat Model |


## **Set Custom Headers***
Yes, absolutely! LiteLLM supports custom headers in multiple ways. Here's a complete breakdown:

---

## Method 1 — `extra_headers` in `litellm_params` (most common)

This is the primary way to set **static custom headers** sent to your custom model API on every request.

You can set `extra_headers` directly under `litellm_params` for any model in your config:

```yaml
model_list:
  - model_name: my-custom-model       # name n8n uses
    litellm_params:
      model: openai/my-model          # openai/ prefix for OpenAI-compatible APIs
      api_base: https://my-custom-api.com/v1
      api_key: os.environ/MY_API_KEY
      extra_headers:
        X-Custom-Auth: "my-secret-token"
        X-Tenant-ID: "tenant-123"
        X-App-Name: "my-n8n-app"
        Authorization: "Bearer my-custom-bearer-token"
```

You can also load header values from environment variables:

```yaml
model_list:
  - model_name: my-custom-model
    litellm_params:
      model: openai/my-model
      api_base: https://my-custom-api.com/v1
      api_key: os.environ/MY_API_KEY
      extra_headers:
        X-Custom-Auth: os.environ/MY_CUSTOM_AUTH_TOKEN
        X-Tenant-ID: os.environ/MY_TENANT_ID
```

Then in your `.env`:
```env
MY_API_KEY=sk-...
MY_CUSTOM_AUTH_TOKEN=secret-value
MY_TENANT_ID=tenant-abc
```

---

## Method 2 — Forward headers from the client (n8n → LiteLLM → your API)

You can selectively enable header forwarding for specific model groups so that headers sent by the client (n8n) are forwarded to your backend API. When enabled, LiteLLM forwards any header starting with `x-`.

```yaml
model_list:
  - model_name: my-custom-model
    litellm_params:
      model: openai/my-model
      api_base: https://my-custom-api.com/v1
      api_key: os.environ/MY_API_KEY

litellm_settings:
  model_group_settings:
    forward_client_headers_to_llm_api:
      - my-custom-model        # enable for this model only
```

Then when n8n calls your proxy, any `x-*` headers it sends get forwarded to your custom API.

---

## Method 3 — Multiple models with different headers

This is useful if you have different environments or tenants:

```yaml
model_list:
  # Production environment
  - model_name: my-model-prod
    litellm_params:
      model: openai/my-model
      api_base: https://api.mycompany.com/v1
      api_key: os.environ/PROD_API_KEY
      extra_headers:
        X-Environment: "production"
        X-Tenant-ID: "tenant-prod"
        X-Version: "v2"

  # Staging environment
  - model_name: my-model-staging
    litellm_params:
      model: openai/my-model
      api_base: https://staging-api.mycompany.com/v1
      api_key: os.environ/STAGING_API_KEY
      extra_headers:
        X-Environment: "staging"
        X-Tenant-ID: "tenant-staging"
        X-Version: "v2"

general_settings:
  master_key: sk-my-master-key
```

---

## Method 4 — Override the Authorization header entirely

If your custom API uses a non-standard auth scheme (e.g. not `Bearer`):

```yaml
model_list:
  - model_name: my-custom-model
    litellm_params:
      model: openai/my-model
      api_base: https://my-custom-api.com/v1
      api_key: "dummy"               # set dummy so LiteLLM doesn't complain
      extra_headers:
        Authorization: "ApiKey my-real-secret-key"   # override auth header
        X-Custom-Header: "value"
```

---

## Real-world example: API behind a gateway with multiple auth headers

```yaml
model_list:
  - model_name: company-llm
    litellm_params:
      model: openai/company-model
      api_base: https://internal-gateway.mycompany.com/llm/v1
      api_key: os.environ/GATEWAY_API_KEY
      extra_headers:
        X-Gateway-Token: os.environ/GATEWAY_TOKEN
        X-Service-Name: "n8n-automation"
        X-Region: "us-east-1"
        X-Request-Source: "litellm-proxy"

general_settings:
  master_key: sk-my-master-key
```

---

## Quick Reference

| Scenario | Solution |
|---|---|
| Static header always sent to your API | `extra_headers` in `litellm_params` |
| Header value from environment variable | `os.environ/VAR_NAME` as the value |
| Forward headers dynamically from n8n | `forward_client_headers_to_llm_api` |
| Override Authorization header | `extra_headers: Authorization: "..."` |
| Per-model different headers | Separate `model_name` entries each with their own `extra_headers` |

The `extra_headers` approach in `litellm_params` is the most reliable and secure for connecting to a custom model API since the headers are server-side and never exposed to clients like n8n.
