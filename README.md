# Local LLM Hosting with Ollama & Apache APISIX

This repository provides a professional setup for hosting and managing Local Large Language Models (LLMs) using **Ollama** as the model execution engine and **Apache APISIX** as a high-performance cloud-native API gateway.

## Project Overview

The goal of this project is to create a robust infrastructure for local AI deployments. By using Apache APISIX as a proxy for Ollama, you gain enterprise-grade features such as:

- **Security**: Authentication and authorization layers.
- **Traffic Management**: Rate limiting and request throttling.
- **Observability**: Metrics, logging, and tracing for AI requests.
- **Scalability**: Seamlessly routing to multiple Ollama instances.

> [!NOTE]
> Currently, the project is focused on the initial configuration of the APISIX gateway and its supporting metadata store (etcd).

## Getting Started

### Prerequisites

- [Docker](https://www.docker.com/)
- [Docker Compose](https://docs.docker.com/compose/)
- [Ollama](https://ollama.com/) (installed locally or running as a separate service)

### Security Configuration

Before starting the services, you must generate a secure admin key for APISIX:

1. **Generate a secure key:**
   Run the following command to generate a 32-character hex key:
   ```bash
   openssl rand -hex 32
   ```

2. **Update Configuration:**
   Open `config.yaml` and replace `<GENERATED_ADMIN_KEY>` with the output from the command above.

### Initial Setup

1. **Clone the repository:**
   ```bash
   git clone <repository-url>
   cd local-llm-hosting
   ```

2. **Start the Infrastructure:**
   Use the provided Docker Compose file to spin up APISIX and etcd.
   ```bash
   docker-compose up -d
   ```

3. **Verify APISIX:**
   The gateway will be available at `http://127.0.0.1:9080` (default port).

## User Management (Consumers)

To securely access your LLM through APISIX, you should create a **Consumer**. This allows you to track usage and restrict access to authorized users.

### Create a Consumer with Key Authentication

Run the following command to create a consumer named `dev-user`:

```bash
curl http://127.0.0.1:9180/apisix/admin/consumers \
  -H "X-API-KEY: <ADMIN_KEY>" \
  -X PUT -d '
{
  "username": "<USERNAME>",
  "plugins": {
    "key-auth": {
      "key": "<CONSUMER_API_KEY>"
    }
  }
}'
```

> [!TIP]
> Replace `<ADMIN_KEY>` with the key you generated in the Security Configuration step, and `<CONSUMER_API_KEY>` with a unique key for the user.

## Connecting APISIX with VSCode "Continue" Extension

The [Continue](https://www.continue.dev/) extension for VSCode allows you to use local LLMs as your coding assistant. To route requests through APISIX (for logging or rate limiting), follow these steps:

### 1. Configure the APISIX Route

To enable advanced features like AI-specific logging and standardized API endpoints, configure a route using the `ai-proxy` plugin. 

**Why use `openai-compatible`?**
Most modern AI extensions (including Continue) expect the OpenAI API format (`/v1/chat/completions`). By using the `openai-compatible` provider in APISIX:
- **Standardization**: You create a unified interface regardless of the model runner backend.
- **Portability**: You can switch from Ollama to other providers (like OpenAI or Anthropic) without changing your client configuration.
- **Enhanced Observability**: The `ai-proxy` plugin provides specialized logging for token usage and AI-specific metadata.

Run this command to create the route:

```bash
curl -i -X PUT "http://127.0.0.1:9180/apisix/admin/routes/1" \
-H "X-API-KEY: <ADMIN_KEY>" \
-d '{
    "uri": "/v1/chat/completions",
    "name": "Ollama-Compatible-Gateway",
    "plugins": {
        "key-auth": {
            "header": "apikey"
        },
        "ai-proxy": {
            "provider": "openai-compatible",
            "auth": {
                "header": {
                    "Authorization": "Bearer <INTERNAL_OLLAMA_TOKEN>"
                }
            },
            "override": {
                "endpoint": "http://127.0.0.1:11434/v1/chat/completions"
            },
            "logging": {
                "summaries": true
            }
        }
    }
}'
```

### 2. Update Continue configuration
Modify your `config.json` in VSCode (usually accessed via the gear icon in the Continue sidebar).

1. Open the Continue settings.
2. Locate the `models` array.
3. Add or update your model configuration. Here is a comprehensive example supporting models routed through the APISIX gateway:

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

### 3. Benefits of this Setup
- **Centralized Control**: Manage all LLM traffic through a single high-performance gateway.
- **Role Specialization**: Assign specific models to tasks like code completion (`autocomplete`) or reasoning.
- **Enhanced Observability**: Track token usage and request latency via the APISIX `ai-proxy` plugin.

---
*Created for secure and manageable local AI development.*
