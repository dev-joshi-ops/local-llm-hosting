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

This repository provides a professional setup for hosting and managing Local Large Language Models (LLMs) using **Ollama** as the model execution engine and **Apache APISIX** as a high-performance cloud-native API gateway.

## Project Overview

The goal of this project is to create a robust infrastructure for local AI deployments. By using Apache APISIX as a proxy for Ollama, you gain enterprise-grade features such as:

- **Security**: Authentication and authorization layers.
- **Traffic Management**: Model-aware rate limiting and request throttling.
- **Observability**: Metrics, logging, and tracing for AI requests.
- **Scalability**: Seamlessly routing to multiple Ollama instances.

> [!NOTE]
> This project uses **APISIX Standalone Mode**, where the configuration is managed declaratively via `apisix.yaml`. This follows Infrastructure-as-Code (IaC) principles.

## Getting Started

### Prerequisites

- [Docker](https://www.docker.com/)
- [Docker Compose](https://docs.docker.com/compose/)
- [Ollama](https://ollama.com/) (installed locally or running as a separate service)

### Environment Configuration

For security, sensitive API keys and tokens are managed via environment variables and are not committed to the repository.

1. **Create your environment file:**
   ```bash
   cp .env.example .env
   ```

2. **Configure your secrets:**
   Open `.env` and fill in your actual consumer keys and internal tokens.

### Initial Setup

1. **Clone the repository:**
   ```bash
   git clone <repository-url>
   cd local-llm-hosting
   ```

2. **Configure Your Infrastructure:**
   The gateway configuration is stored in `apisix.yaml`. It uses the `${{VAR}}` syntax to pull values from your `.env` file via Docker Compose.

3. **Start the Infrastructure:**
   ```bash
   docker-compose up -d
   ```

4. **Verify APISIX:**
   The gateway will be available at `http://127.0.0.1:9080`.

## Declarative Configuration (IaC)

In standalone mode, all configurations are defined in `apisix.yaml`. APISIX automatically monitors this file for changes and performs hot reloads.

### Model-Wise Rate Limiting

This project implements model-specific token quotas at the **Route Level**. This ensures that different models have appropriate limits based on their complexity and host resource usage.

```yaml
routes:
  - plugins:
      ai-rate-limiting:
        instances:
          - name: "gemma4:e4b"
            limit: 10000
            time_window: 3600
          - name: "gemma4:26b-a4b-it-q4_K_M"
            limit: 5000
            time_window: 3600
        rejected_msg: "{\"error\": \"Model quota exceeded.\"}"
```

## Advanced: Tiered Service (Premium Quotas)

While this repository defaults to global model-wise limits, APISIX supports **Premium Service Tiers** using plugin precedence.

### Implementing a Premium Tier
To grant a specific user higher limits (overriding the defaults above), add the `ai-rate-limiting` plugin directly to their **Consumer** profile in `apisix.yaml`:

```yaml
consumers:
  - username: "premium-user"
    plugins:
      key-auth:
        key: "${{PREMIUM_KEY}}"
      ai-rate-limiting:
        limit: 50000        # Much higher limit for high-value users
        time_window: 3600
```

> [!TIP]
> APISIX precedence follows the order: **Consumer > Route > Global**. Any plugin defined on the Consumer will completely replace the same plugin's settings from the Route for that specific user.

## Scaling and Management

### Adding Models and Rules
Update the `routes` section in `apisix.yaml` to add new models or endpoints. Standardize on the `ai-proxy` plugin for OpenAI compatibility.

```yaml
routes:
  - id: "1"
    name: "Ollama-Compatible-Gateway"
    uri: "/v1/chat/completions"
    plugins:
      ai-proxy:
        provider: "openai-compatible"
        auth:
          header:
            Authorization: "Bearer ${{INTERNAL_OLLAMA_TOKEN}}"
        override:
          endpoint: "http://<OLLAMA_IP>:11434/v1/chat/completions"
      file-logger:
        path: "/dev/stdout"
```

> [!TIP]
> The `file-logger` plugin combined with `ai-proxy.logging.summaries` allows you to see detailed AI metrics (token usage, model names, latency) directly in your Docker logs.

---
*Created for secure and manageable local AI development.*
