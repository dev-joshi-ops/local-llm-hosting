# Local LLM Hosting with Ollama & Apache APISIX

<div align="center">

![Local LLM Infrastructure](https://img.shields.io/badge/Infrastructure-Local--LLM-blue?style=for-the-badge&logo=ai)
![Ollama](https://img.shields.io/badge/Model--Runner-Ollama-white?style=for-the-badge&logo=ollama)
![APISIX](https://img.shields.io/badge/API--Gateway-Apache--APISIX-red?style=for-the-badge&logo=apache-apisix)
![Docker](https://img.shields.io/badge/Container-Docker-blue?style=for-the-badge&logo=docker)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](http://makeapullrequest.com)
[![GitHub Stars](https://img.shields.io/github/stars/dev-joshi-ops/local-llm-hosting.svg?style=social)](https://github.com/dev-joshi-ops/local-llm-hosting/stargazers)

</div>

---

This repository provides a professional setup for hosting and managing Local Large Language Models (LLMs) using **Ollama** and **Apache APISIX**.

## Project Overview

The goal of this project is to create a robust infrastructure for local AI deployments. By using Apache APISIX as a proxy for Ollama, you gain enterprise-grade features such as:

- **Security**: Authentication and authorization layers.
- **Traffic Management**: Model-aware rate limiting (10M tokens/hr) and 5-minute response timeouts.
- **Performance**: High-speed inference using NVIDIA GPU acceleration.
- **Observability**: Metrics, logging, and tracing for AI requests.
- **Scalability**: Seamlessly routing to multiple Ollama instances.

> [!NOTE]
> This project uses **APISIX 3.16.0** (Standalone Mode) and **Ollama 0.21.0**. The entire stack is managed via Docker Compose following Infrastructure-as-Code (IaC) principles.

## Getting Started

### Prerequisites

- [Docker](https://www.docker.com/) & [Docker Compose](https://docs.docker.com/compose/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) (for GPU acceleration on Ubuntu/Linux)

### Environment Configuration

For security and persistence, sensitive keys and model paths are managed via environment variables.

1. **Create your environment file:**
   ```bash
   cp .env.example .env
   ```

2. **Configure your secrets:**
   Open `.env` and set:
   - `CONSUMER_API_KEY`: Your gateway access key.
   - `INTERNAL_OLLAMA_TOKEN`: Token for upstream authorization.
   - `OLLAMA_ENDPOINT`: The internal Docker endpoint for Ollama (e.g., `http://ollama:11434/v1/chat/completions`).
   - `LOCAL_OLLAMA_MODELS`: Path to your local models (e.g., `/usr/share/ollama/.ollama`).

### Initial Setup

1. **Clone the repository:**
   ```bash
   git clone <repository-url>
   cd local-llm-hosting
   ```

2. **Start the Stack:**
   ```bash
   docker-compose up -d
   ```

3. **Verify the Stack:**
   - APISIX: `http://localhost:9080`
   - Ollama: `curl http://localhost:11434/api/tags`

## Declarative Configuration (IaC)

### Model-Wise Rate Limiting
This project implements high-capacity token quotas (10M tokens/hr) using the `ai-proxy` and `ai-rate-limiting` plugins. The gateway reads the requested `model` from the JSON body and uses it as the APISIX AI instance name for model-specific quotas.

```yaml
routes:
  - uris:
      - "/v1/chat/completions"
      - "/v1/completions"
    plugins:
      serverless-pre-function:
        phase: rewrite
        functions:
          - |
            return function(conf, ctx)
                local core = require("apisix.core")
                ngx.req.read_body()
                local body_str = ngx.req.get_body_data()
                if body_str then
                    local ok, body = pcall(core.json.decode, body_str)
                    if ok and body and body.model then
                        ctx.picked_ai_instance_name = body.model
                    end
                end
            end
      ai-proxy:
        provider: "openai-compatible"
        timeout: 300000          # 5-minute timeout for reasoning models
        override:
          endpoint: "${{OLLAMA_ENDPOINT}}"
      ai-rate-limiting:
        instances:
          - name: "gemma4:e4b"
            limit: 10000000      # 10M tokens/hr
          - name: "gemma4:26b-a4b-it-q4_K_M"
            limit: 10000000
```

### Models List Endpoint

The gateway also exposes `/v1/models` and `/models` through APISIX's `mocking` plugin so OpenAI-compatible clients can discover the local models without calling Ollama directly.

## Advanced: Implementing a Premium Tier
To grant a specific user higher limits, add the `ai-rate-limiting` plugin directly to their **Consumer** profile in `apisix.yaml` using the same instance names as the request `model` values:

```yaml
consumers:
  - username: "premium-user"
    plugins:
      ai-rate-limiting:
        instances:
          - name: "gemma4:e4b"
            limit: 50000000      # 50M tokens for premium tiers
          - name: "gemma4:26b-a4b-it-q4_K_M"
            limit: 20000000
```

## Adding Custom Plugins

You can extend APISIX with custom Lua plugins from the local `plugins/` directory. Docker Compose mounts this directory into the APISIX container at `/opt/apisix-custom-plugins`, and `config.yaml` adds that mount to APISIX's Lua search path.

1. **Add your plugin file** under the APISIX module path:
   ```text
   plugins/
     apisix/
       plugins/
         my-plugin.lua
   ```
2. **Configure APISIX**:
   - Add your plugin to the `plugins` list in `config.yaml`.
   - Apply it to routes in `apisix.yaml`.

> [!IMPORTANT]
> Defining a `plugins:` list in `config.yaml` replaces APISIX's default enabled plugin list. Include the built-in plugins this gateway already uses, such as `key-auth`, `serverless-pre-function`, `ai-proxy`, `ai-rate-limiting`, `file-logger`, and `mocking`, along with your custom plugin.

## Connecting with VSCode "Continue" Extension

Modify your `config.json` in VSCode:

```yaml
name: Local Config
version: 1.0.0
schema: v1
models:
  - name: Gemma 4 26B (Reasoning)
    provider: openai
    model: gemma4:26b-a4b-it-q4_K_M
    apiKey: "<CONSUMER_API_KEY>"
    apiBase: http://<SERVER_IP>:9080/v1/
```

## Connecting with Open WebUI (External)

If you are running Open WebUI (e.g., via their standalone Docker image or hosted), follow these steps to connect:

1.  **Open Settings** in Open WebUI.
2.  **Navigate to Connections** > **OpenAI API**.
3.  **Configure the following:**
    *   **OpenAI Base URL**: `http://<YOUR_SERVER_IP>:9080/v1`
    *   **OpenAI API Key**: `${CONSUMER_API_KEY}` (The value defined in your `.env`)
4.  **Important Note on Authentication**:
    The gateway is configured for OpenAI-compatible clients that send `Authorization: Bearer <CONSUMER_API_KEY>`. In Open WebUI, enter the raw `CONSUMER_API_KEY`; Open WebUI will send it with the `Bearer` prefix.

### Why use the Gateway instead of direct Ollama?
Connecting through the gateway (`port 9080`) instead of direct Ollama (`port 11434`) gives you:
- **Token Rate Limiting**: Prevents any single user/session from exhausting your GPU.
- **Audit Logs**: See all requests passing through APISIX in the Docker logs.
- **Unified Endpoint**: Manage multiple models through a single API base.

---
*Created for secure and manageable local AI development.*
