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
> This project uses **APISIX Standalone Mode**, where the configuration is managed declaratively via `apisix.yaml`. This follows Infrastructure-as-Code (IaC) principles, removing the need for an external database like etcd.

## Getting Started

### Prerequisites

- [Docker](https://www.docker.com/)
- [Docker Compose](https://docs.docker.com/compose/)
- [Ollama](https://ollama.com/) (installed locally or running as a separate service)

### Initial Setup

1. **Clone the repository:**
   ```bash
   git clone <repository-url>
   cd local-llm-hosting
   ```

2. **Configure Your Infrastructure:**
   The gateway configuration is stored in `apisix.yaml`. Ensure you have defined your routes and consumers there.

3. **Start the Infrastructure:**
   Use the provided Docker Compose file to spin up APISIX.
   ```bash
   docker-compose up -d
   ```

4. **Verify APISIX:**
   The gateway will be available at `http://127.0.0.1:9080` (default port).

## Declarative Configuration (IaC)

In standalone mode, all configurations are defined in `apisix.yaml`. APISIX automatically monitors this file for changes and performs hot reloads.

### 1. Consumer Configuration
Define authorized users and their API keys in the `consumers` section:

```yaml
consumers:
  - username: "<USERNAME>"
    plugins:
      key-auth:
        key: "<CONSUMER_API_KEY>"
```

### 2. Route Configuration
Configure routes to forward traffic to Ollama in the `routes` section. Using the `ai-proxy` plugin with the `openai-compatible` provider ensures compatibility with standard AI tools.

```yaml
routes:
  - id: "ollama-route"
    name: "Ollama-Compatible-Gateway"
    uri: "/v1/chat/completions"
    plugins:
      key-auth: 
        header: "apikey"
      ai-proxy:
        provider: "openai-compatible"
        auth:
          header:
            Authorization: "Bearer <INTERNAL_OLLAMA_TOKEN>"
        override:
          endpoint: "http://<OLLAMA_IP>:11434/v1/chat/completions"
        logging:
          summaries: true
```

## Connecting APISIX with VSCode "Continue" Extension

The [Continue](https://www.continue.dev/) extension for VSCode allows you to use local LLMs as your coding assistant. To route requests through APISIX, follow these steps:

### 1. Update Continue configuration
Modify your `config.json` in VSCode (usually accessed via the gear icon in the Continue sidebar).

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

  - name: Gemma 4 E4B (Fast)
    provider: openai
    model: gemma4:e4b
    apiBase: http://<APISIX_IP>:9080/v1/
    requestOptions:
      headers:
        apikey: "<CONSUMER_API_KEY>"
    roles:
      - chat
      - edit
      - autocomplete
```

> [!TIP]
> Using the `requestOptions.headers` as shown above ensures your `apikey` is sent correctly to APISIX's `key-auth` plugin.

### 2. Benefits of this Setup
- **Standardization**: Your tools talk to a standard OpenAI API, making it easy to swap backends.
- **Centralized Control**: Manage all local and remote models through a single gateway.
- **AI-Specific Analytics**: APISIX can log token usage and model performance via the `ai-proxy` plugin.

---
*Created for secure and manageable local AI development.*
