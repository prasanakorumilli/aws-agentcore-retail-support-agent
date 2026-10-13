# RetailCustomerSupport

A Python customer-support agent built with the Strands Agents SDK and deployed to Amazon Bedrock AgentCore Runtime. It answers product and return-policy questions, can use an MCP web-search server for broader research, and retains customer context with AgentCore Memory.

`RetailCustomerSupport` documents a hands-on implementation based on the AWS Builder Center's **Getting Started with Amazon Bedrock AgentCore** workshop:

- [Prerequisites: self-paced setup](https://catalog.workshops.aws/agentcore-getting-started/en-US/10-prereqs/12-self-paced)
- [Hands-on 1: Build the agent prototype](https://catalog.workshops.aws/agentcore-getting-started/en-US/20-lab1-agent)
- [Hands-on 2: Add memory to the agent](https://catalog.workshops.aws/agentcore-getting-started/en-US/30-lab2-memory)

The AgentCore CLI is the workflow tool used in this hands-on implementation to scaffold, run, deploy, and manage the application; it is not a separate application dependency. See the [official workshop overview](https://catalog.workshops.aws/agentcore-getting-started/en-US) and [AgentCore CLI](https://github.com/aws/agentcore-cli).

## Architecture

```text
Customer
   │
   ▼
AgentCore Runtime (HTTP)
   │
   ▼
Strands retail-support agent ──► Amazon Bedrock model
   │                │
   │                ├── get_product_info
   │                ├── get_return_policy
   │                ├── Exa MCP web search
   │                └── AgentCore Memory
   │                       ├── short-term session events
   │                       ├── semantic customer facts
   │                       └── conversation summaries
   ▼
Streaming response
```

## What this project demonstrates

- A Strands agent hosted by Amazon Bedrock AgentCore Runtime.
- Two Python tools: `get_product_info` and `get_return_policy`.
- A streamable HTTP MCP client for Exa web search.
- A Bedrock-hosted Claude model selected in `app/RetailCustomerSupport/model/load.py`.
- Durable customer context through AgentCore Memory.
- Semantic memory for facts and summarization memory for conversation continuity.

## Prerequisites

The workshop's self-paced setup calls for the following local tools:

- Node.js 20.x or later
- `uv` 0.4 or later for Python environments and dependencies
- AWS CLI v2, authenticated to the AWS account and Region you will use
- Git 2.x or later
- An AWS account with permissions to use Amazon Bedrock and provision the AgentCore resources required by the deployment
- Access to the Bedrock model configured in `app/RetailCustomerSupport/model/load.py`

Install and verify the AgentCore CLI:

```bash
npm install -g @aws/agentcore
agentcore --version
```

Verify the other tools and AWS identity before continuing:

```bash
node --version
uv --version
aws --version
git --version
aws sts get-caller-identity
```

> This hands-on implementation creates billable AWS resources, including Bedrock model invocations and AgentCore Runtime/Memory resources. Follow your organization's cleanup and cost-management practices when you finish.

## Project layout

```text
RetailCustomerSupport/
├── agentcore/
│   ├── agentcore.json                 # Runtime and memory declarations
│   ├── aws-targets.json               # Local deployment target: account and Region
│   └── cdk/                           # CLI-managed deployment infrastructure
└── app/RetailCustomerSupport/
    ├── main.py                        # Agent, tools, MCP client, Runtime entrypoint
    ├── memory/session.py              # AgentCore Memory session-manager setup
    ├── mcp_client/client.py           # Streamable HTTP MCP client
    └── model/load.py                  # Bedrock model configuration
```

## 1. Scaffold RetailCustomerSupport locally

The first hands-on activity uses the AgentCore CLI to scaffold a Python Strands application backed by Amazon Bedrock. The equivalent non-interactive command is:

```bash
agentcore create \
  --name RetailCustomerSupport \
  --framework Strands \
  --model-provider Bedrock \
  --memory none
```

The CLI creates the application skeleton, `agentcore/` project configuration, a local Python environment, and the deployment infrastructure. This repository is the resulting project after the customizations described below.

## 2. Add customer-support tools and MCP web search

In `app/RetailCustomerSupport/main.py`, the agent is given two local tools:

| Tool | Purpose |
| --- | --- |
| `get_product_info` | Searches the demo product catalog by ID, name, category, or keyword. |
| `get_return_policy` | Returns the policy for audio, electronics, or accessories. |

The agent also adds the MCP client from `mcp_client/client.py`. It connects to Exa's streamable HTTP MCP endpoint, allowing the model to search the web when a question needs information outside the local demo catalog.

After installing dependencies, run the agent locally from the project root:

```bash
cd RetailCustomerSupport
agentcore dev
```

To send a prompt to the local runtime from the CLI, pass the custom user ID as a header:

```bash
agentcore dev \
  --header "X-Amzn-Bedrock-AgentCore-Runtime-Custom-User-Id: customer-123" \
  "What is the return policy for wireless headphones?"
```

Try product lookup as well:

```bash
agentcore dev \
  --header "X-Amzn-Bedrock-AgentCore-Runtime-Custom-User-Id: customer-123" \
  "Tell me about PROD-002"
```

## 3. Deploy the runtime to AWS

Review the deployment target in `agentcore/aws-targets.json`, then validate and deploy from the project root:

```bash
agentcore validate
agentcore deploy
agentcore status
```

The CLI synthesizes and deploys the AgentCore resources needed for the configured CodeZip runtime. Once deployment completes, invoke the deployed runtime with a session and user ID:

```bash
agentcore invoke \
  --session-id deployed-demo-001 \
  --user-id customer-123 \
  "What is the return policy for a smart watch?"
```

## 4. Add and wire AgentCore Memory

The memory hands-on activity distinguishes between session persistence and durable memory:

- **Short-term memory** keeps raw conversation events for a session.
- **Semantic memory** extracts customer facts that can be retrieved across sessions.
- **Summarization memory** stores compressed conversation context for continuity.

This project declares a `SharedMemory` resource in `agentcore/agentcore.json` with a 30-day event expiry and two long-term strategies:

```text
/users/{actorId}/facts                 semantic customer facts
/summaries/{actorId}/{sessionId}       conversation summaries
```

`app/RetailCustomerSupport/memory/session.py` reads the deployed memory ID from `MEMORY_SHAREDMEMORY_ID`, creates an `AgentCoreMemorySessionManager`, and attaches it to the Strands agent. The runtime uses the AgentCore session ID and custom user ID as the session and actor identity.

Deploy the configuration change after adding memory:

```bash
agentcore validate
agentcore deploy
```

### Validate memory across sessions

Use the same user ID, but a new session ID for the second request. That confirms the returned fact comes from durable AgentCore Memory rather than only the live session context.

```bash
# Store a customer fact.
agentcore invoke \
  --session-id memory-test-1 \
  --user-id customer-123 \
  "My name is Alex and I prefer audio products."

# Retrieve it from a different session.
agentcore invoke \
  --session-id memory-test-2 \
  --user-id customer-123 \
  "What do you remember about me?"
```

## Security and repository hygiene

- Never commit AWS access keys, API keys, tokens, or `agentcore/.env.local`.
- Treat `agentcore/aws-targets.json` as environment-specific because it contains an AWS account ID. For a public portfolio, publish a sanitized example file and keep the real target file local.
- Review `git diff --cached` before each commit.
- Do not use production customer data in tool fixtures, prompts, memory-validation inputs, screenshots, or documentation.

## Next steps

The hands-on sequence continues with AgentCore Gateway, security and observability, evaluation, a customer interface, and policy-based governance. Good next enhancements for `RetailCustomerSupport` are an order-status integration backed by authenticated data, evaluation cases, observability dashboards, and a web chat interface.
