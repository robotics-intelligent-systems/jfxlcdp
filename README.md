# jfxlcdp Local AI and Astra Max Integration

## Overview

`jfxlcdp` can be extended with a local-first artificial intelligence plane for
specification-driven development, low-code application generation, Modelica
multidomain modeling, Scala integration middleware, and containerized delivery.

The proposed design combines:

- `gpt-oss` as the local or private inference engine;
- a policy-based Model Router;
- Retrieval-Augmented Generation (RAG) with PostgreSQL and Qdrant;
- Model Context Protocol (MCP) as a bounded tool interface;
- the proposed `Astra Max` plugin for ChatGPT Work;
- isolated Modelica, code-generation, and deployment workers.

> **Naming note:** `Astra Max` is used in this document as the project name for
> the ChatGPT Work integration. It is an integration layer, not a separate
> inference model and not a replacement for ChatGPT's internal model.

## Goals

The integration is designed to provide:

1. private inference for source code, specifications, and enterprise models;
2. offline or low-connectivity development with `gpt-oss-20b`;
3. scalable private-server inference with `gpt-oss-120b`;
4. traceable answers grounded in project documents and model artifacts;
5. controlled tool execution through MCP;
6. approval gates for changes, deployments, and other consequential actions;
7. an extensible architecture that does not depend on one inference runtime.

## Source-backed `gpt-oss` capabilities

The official `gpt-oss` project provides two open-weight models:

| Profile | Model | Intended deployment |
|---|---|---|
| Local and specialized | `gpt-oss-20b` | Workstations, development devices, and edge-oriented services |
| Production and complex reasoning | `gpt-oss-120b` | Private GPU servers and multi-user workloads |

The official repository describes `gpt-oss-20b` as a lower-latency local model
and `gpt-oss-120b` as a production model that can fit on a single 80 GB GPU.
It also documents MXFP4 quantization, tool calling, structured outputs,
configurable reasoning effort, and the required Harmony response format.

The model license and runtime requirements must be reviewed independently from
the license and policies of the `jfxlcdp` repository.

## Reference architecture

```mermaid
flowchart TD
    A[ChatGPT Work / Astra Max] -->|HTTPS and OAuth| B[Astra Max MCP Gateway]
    B -->|mTLS or secure outbound tunnel| C[jfxlcdp Agent Gateway]
    C --> D[Policy Model Router]
    D --> E[gpt-oss-20b Local]
    D --> F[gpt-oss-120b Private]
    C --> G[PostgreSQL and Qdrant RAG]
    C --> H[Modelica Scala Docker Workers]
```

### Responsibility boundaries

| Component | Responsibility |
|---|---|
| Astra Max plugin | ChatGPT Work discovery, workflow guidance, authorization context, and user-facing interaction |
| MCP Gateway | Exposes only approved domain tools and structured results |
| jfxlcdp Agent Gateway | Normalizes requests, applies policy, records audit identifiers, and coordinates tools |
| Model Router | Selects local or private inference according to policy and runtime conditions |
| `gpt-oss-20b` | Local reasoning, specification assistance, code drafts, and offline workflows |
| `gpt-oss-120b` | Complex reasoning, larger contexts, batch generation, and private multi-user workloads |
| PostgreSQL | Structured project data, permissions, versions, metadata, and execution records |
| Qdrant | Semantic index for specifications, source code, architecture documents, and model artifacts |
| Workers | Sandboxed Modelica simulations, Scala generation, tests, packaging, and deployment preparation |

## Local inference service

The application must communicate with an OpenAI-compatible LLM Gateway rather
than calling Ollama, vLLM, llama.cpp, or LM Studio directly. This keeps the
application independent from the selected runtime.

### Ollama development profile

```bash
ollama pull gpt-oss:20b
ollama run gpt-oss:20b
```

### vLLM private-server profile

```bash
vllm serve openai/gpt-oss-120b
```

The gateway should normalize:

- chat and response requests;
- streaming responses;
- reasoning effort;
- tool calls and structured JSON outputs;
- request correlation and audit identifiers;
- health checks, timeouts, retries, and concurrency limits;
- Harmony-compatible message handling.

## Model Router policy

The Model Router evaluates the following attributes before selecting a model:

- data sensitivity and tenant policy;
- local connectivity and offline mode;
- available CPU, RAM, GPU, and VRAM;
- latency target and request priority;
- context size and reasoning complexity;
- cost and concurrency limits;
- required tools and structured-output constraints.

Recommended default policy:

| Condition | Route |
|---|---|
| Private data, local device available, or offline mode | `gpt-oss-20b` locally |
| Complex architecture analysis or large batch workload | `gpt-oss-120b` on the private server |
| Local model unavailable | Fail closed or use an explicitly configured private route |
| Optional cloud escalation | Disabled by default and permitted only by policy |

The router should return the selected profile, policy decision, model version,
and execution identifier as metadata. It should not expose hidden model
reasoning to end users.

## Retrieval-Augmented Generation

The RAG layer provides grounded context to the selected model. Indexed sources
may include:

- software requirements specifications and acceptance criteria;
- SysML and UML models;
- Modelica contracts and simulation parameters;
- architecture decision records and design notes;
- source code, API definitions, and Scala middleware;
- Docker and Kubernetes manifests;
- test results, simulation outputs, and generated artifacts.

PostgreSQL stores structured entities, access rules, versions, and provenance.
Qdrant stores vector representations and retrieval metadata. A dedicated
embedding model is used for vectorization; `gpt-oss` is used for reasoning and
generation, not for embeddings.

Every grounded response should preserve:

- document or artifact identifiers;
- repository revision or model version;
- retrieved source references;
- validation status;
- generated artifact metadata;
- the correlation identifier for the execution.

## Astra Max ChatGPT Work plugin

The plugin is packaged as a reusable ChatGPT Work integration containing an
MCP server and a workflow skill. The MCP server exposes business-level
capabilities instead of a raw LLM endpoint.

### Plugin package

```text
plugins/astra-max-jfxlcdp/
├── plugin.json
├── mcp.json
├── skills/
│   └── jfxlcdp-ai-workflow/
│       └── SKILL.md
├── assets/
│   ├── icon.png
│   └── logo.png
└── README.md
```

The plugin skill should guide the following workflow:

1. understand and classify the request;
2. verify project and tenant authorization;
3. retrieve relevant specifications and architecture context;
4. select the appropriate `gpt-oss` profile through the Model Router;
5. call only the tools required for the task;
6. return citations, validation results, and artifact metadata;
7. request explicit approval before write or deployment actions.

### Read-only tools

- `search_specifications`
- `retrieve_architecture_context`
- `explain_modelica_contract`
- `inspect_component`
- `get_simulation_result`
- `check_service_health`

### Controlled execution tools

- `validate_modelica_model`
- `run_simulation`
- `generate_scala_service`
- `generate_kubernetes_manifest`
- `create_architecture_document`
- `generate_test_plan`

### Approval-required tools

- `create_patch`
- `modify_specification`
- `publish_artifact`
- `deploy_service`
- `execute_migration`

Write operations must be independently authorized, validated, logged, and
approved. The plugin must not expose unrestricted shell access, credentials, or
unvalidated deployment commands.

## Connectivity and privacy model

ChatGPT Work cannot directly call a user's private `localhost` inference
endpoint. For Work-based integration, the local worker should establish an
outbound authenticated connection to the Astra Max MCP Gateway, or the gateway
should be deployed inside the same private network as the inference service.

For strict offline operation, users should interact with the local jfxlcdp
interface or CLI. The Astra Max Work plugin is used when a governed remote
connection is available.

Recommended controls:

- HTTPS for all remote MCP traffic;
- OAuth or equivalent identity-based authorization;
- mTLS or workload identity between gateway and local/private workers;
- tenant and project-level access control;
- prompt and document redaction for sensitive values;
- allowlisted tools and parameter validation;
- isolated containers for simulations and code execution;
- approval gates for all write and deployment operations;
- audit logs with request, tool, model, and artifact identifiers;
- concise explanations and citations instead of raw hidden reasoning.

## Configuration example

```dotenv
JFXLLM_DEFAULT_PROFILE=local
JFXLLM_LOCAL_MODEL=gpt-oss:20b
JFXLLM_PRIVATE_MODEL=openai/gpt-oss-120b
JFXLLM_LOCAL_BASE_URL=http://localhost:11434/v1
JFXLLM_PRIVATE_BASE_URL=https://private-llm.example/v1
JFXLLM_REASONING_EFFORT=medium

JFXRAG_DATABASE_URL=postgresql://user:password@postgres:5432/jfxlcdp
JFXRAG_QDRANT_URL=http://qdrant:6333

ASTRA_MAX_MCP_URL=https://astra-max.example/mcp
ASTRA_MAX_AUTH_MODE=oauth
ASTRA_MAX_WRITE_APPROVAL_REQUIRED=true
```

Production secrets must be injected through a secret manager or protected
runtime configuration. They must not be committed to the repository.

## Suggested repository layout

```text
jfxlcdp/
├── README.md
├── docs/
│   ├── ai-architecture.md
│   ├── model-routing-policy.md
│   └── astra-max-plugin.md
├── services/
│   ├── llm-gateway/
│   ├── model-router/
│   ├── rag-service/
│   └── mcp-server/
├── workers/
│   ├── modelica-runner/
│   ├── scala-generator/
│   └── deployment-preparer/
├── plugins/
│   └── astra-max-jfxlcdp/
├── config/
├── deploy/
└── tests/
```

## MVP delivery plan

1. Run `gpt-oss-20b` locally through Ollama.
2. Add the provider-neutral LLM Gateway.
3. Implement policy-based routing for local and private profiles.
4. Index project specifications in PostgreSQL and Qdrant.
5. Implement read-only jfxlcdp MCP tools.
6. Package the Astra Max plugin and its workflow skill.
7. Add OAuth, project authorization, audit logging, and approval gates.
8. Isolate Modelica simulation and code-generation workers.
9. Add automated tests for routing, retrieval, tool permissions, and outputs.
10. Measure latency, context-grounding accuracy, tool reliability, resource
    consumption, and simulation validation quality.

## Acceptance criteria

- Local inference works without sending project documents to a cloud model.
- The same application code can use the local or private `gpt-oss` profile.
- Model routing decisions are policy-driven and auditable.
- RAG responses identify the source documents and versions used.
- MCP tools expose bounded, schema-validated capabilities.
- Astra Max can search, explain, validate, and generate jfxlcdp artifacts.
- Write and deployment operations require explicit approval.
- Modelica and code execution run in isolated workers.
- No credentials or unrestricted shell operations are exposed to the plugin.

## References

- [OpenAI `gpt-oss` repository](https://github.com/openai/gpt-oss)
- [OpenAI plugin architecture](https://developers.openai.com/plugins/concepts/plugins)
- [OpenAI plugin packaging guide](https://developers.openai.com/plugins/build/plugins)
- [OpenAI MCP server guide](https://developers.openai.com/api/docs/mcp)
