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
- **Traffic Management**: Rate limiting and request throttling.
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

### 1. Consumer Configuration
Define authorized users and their specific quotas in the `consumers` section. Setting rate limits at the consumer level ensures per-user isolation.

```yaml
consumers:
  - username: "<USERNAME>"
    plugins:
      key-auth:
        key: "${{CONSUMER_API_KEY}}"
      ai-rate-limiting:
        limit: 1000
        time_window: 3600
        limit_strategy: "total_tokens"
        rejected_code: 429
        rejected_msg: "{\"error\": \"Bro, you maxed out your token budget. Wait a bit.\"}"
```

### 2. Route Configuration
Configure routes to forward traffic to Ollama. The `ai-proxy` plugin with the `openai-compatible` provider ensures compatibility with standard AI tools.

```yaml
routes:
  - id: "1"
    name: "Ollama-Compatible-Gateway"
    uri: "/v1/chat/completions"
    plugins:
      key-auth: 
        header: "apikey"
      ai-proxy:
        provider: "openai-compatible"
        auth:
          header:
            Authorization: "Bearer ${{INTERNAL_OLLAMA_TOKEN}}"
        override:
          endpoint: "http://<OLLAMA_IP>:11434/v1/chat/completions"
        logging:
          summaries: true
      file-logger:
        path: "/dev/stdout"
```

> [!TIP]
> The `file-logger` plugin combined with `ai-proxy.logging.summaries` allows you to see detailed AI metrics (token usage, model names, latency) directly in your Docker logs.

## Scaling with Multiple Consumers

To add more consumers with independent quotas:

1. **Update `apisix.yaml`**: Add a new entry to the `consumers` list with its own `ai-rate-limiting` config.
   ```yaml
   consumers:
     - username: "user-1"
       plugins:
         key-auth:
           key: "${{USER1_API_KEY}}"
         ai-rate-limiting:
           limit: 5000  # Higher tier
           time_window: 3600
   ```

2. **Update `.env`**: Add the unique keys for each user.

3. **Update `docker-compose.yaml`**: Map the new variables to the APISIX service.

## Connecting APISIX with VSCode "Continue" Extension

The [Continue](https://www.continue.dev/) extension for VSCode allows you to use local LLMs as your coding assistant. To route requests through APISIX, follow these steps:

### 1. Update Continue configuration
Modify your `config.json` in VSCode:

```yaml
name: Local Config
version: 1.0.0
schema: v1
models:
  - name: Gemma 4 26B (Reasoning)
    provider: openai
    model: gemma4:26b-a4b-it-q4_K_M
    apiBase: http://<APISIX_IP>:9080/v1/
    requestOptions:
      headers:
        apikey: "<CONSUMER_API_KEY>"
    roles:
      - chat
      - edit
```

### 2. Benefits of this Setup
- **Per-User Quotas**: Ensure fair usage across your team by setting independent token limits.
- **Custom Error Messages**: Provide clear feedback to users when they exceed their budget.
- **Centralized Control**: Manage all local and remote models through a single gateway.

---
*Created for secure and manageable local AI development.*
