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
- **Traffic Management**: Model-aware rate limiting and request throttling.
- **Observability**: Metrics, logging, and tracing for AI requests.
- **Scalability**: Seamlessly routing to multiple Ollama instances.

> [!NOTE]
> This project uses **APISIX Standalone Mode** and containerized **Ollama**. The entire stack is managed via Docker Compose following Infrastructure-as-Code (IaC) principles.

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
This project implements model-specific token quotas using the `ai-proxy-multi` and `ai-rate-limiting` plugins. This ensures that different models have appropriate limits based on their complexity.

```yaml
routes:
  - uri: "/v1/*"
    plugins:
      ai-proxy-multi:
        instances:
          - name: "gemma4:e4b"
            weight: 1
            override:
              model: "gemma4:e4b"
          - name: "gemma4:26b-a4b-it-q4_K_M"
            weight: 1
            override:
              model: "gemma4:26b-a4b-it-q4_K_M"
      ai-rate-limiting:
        instances:
          - name: "gemma4:e4b"
            limit: 10000
          - name: "gemma4:26b-a4b-it-q4_K_M"
            limit: 5000
```

## Advanced: Implementing a Premium Tier
To grant a specific user higher limits (overriding the defaults above), add the `ai-rate-limiting` plugin directly to their **Consumer** profile in `apisix.yaml` using the matching instance names:

```yaml
consumers:
  - username: "premium-user"
    plugins:
      ai-rate-limiting:
        instances:
          - name: "gemma4:e4b"
            limit: 50000
          - name: "gemma4:26b-a4b-it-q4_K_M"
            limit: 20000
```

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
    apiBase: http://<SERVER_IP>:9080/v1/
    requestOptions:
      headers:
        apikey: "<CONSUMER_API_KEY>"
```

---
*Created for secure and manageable local AI development.*
