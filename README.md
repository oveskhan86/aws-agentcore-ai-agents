# Customer Support AI Agent on Amazon Bedrock AgentCore

An end-to-end build that takes a customer support agent from a local prototype to a secure, observable, customer-facing application using **Amazon Bedrock AgentCore**.

The agent answers product, troubleshooting, warranty and return-policy questions, remembers each customer across sessions, calls shared tools through a secured gateway, runs on a managed runtime, and is traced in CloudWatch. A Streamlit web app with login sits on top.

![Architecture](images/architecture.png)

> **Note:** This is a learning and portfolio project. It shows how the AgentCore services fit together when moving an agentic use case from prototype to production. It is a reference implementation, not a hardened production system.

---

## What I Built

| Layer | What it does | Built with |
|---|---|---|
| **Agent** | Customer support agent with tool routing and a system prompt | Strands Agents, Amazon Bedrock |
| **Knowledge** | Product and troubleshooting guides retrieved on demand | Amazon S3, Bedrock Knowledge Bases |
| **Memory** | Short-term conversation history and long-term customer preferences | AgentCore Memory |
| **Shared tools** | Tools exposed once and reused by any agent, secured with JWT | AgentCore Gateway, AgentCore Identity, AWS Lambda, Amazon Cognito |
| **Deployment** | Fully managed hosting with session isolation and continuity | AgentCore Runtime |
| **Observability** | Traces, sessions, token usage and cost per request | AgentCore Observability, CloudWatch GenAI Observability, OpenTelemetry |
| **Interface** | Login, streaming chat and session handling | Streamlit |

**Journey overview**

1. Build a working agent prototype with tools
2. Add memory so it remembers customers across conversations
3. Move tools behind a secure, shared gateway
4. Deploy to a managed runtime with full observability
5. Ship a customer-facing web app

---

## Architecture

```mermaid
sequenceDiagram
    actor C as Customer
    participant UI as Streamlit App
    participant CG as Cognito
    participant RT as AgentCore Runtime (Strands Agent)
    participant MEM as AgentCore Memory
    participant GW as AgentCore Gateway
    participant LM as Lambda tools
    participant KB as Bedrock Knowledge Base
    participant OBS as CloudWatch Observability

    C->>UI: Log in
    UI->>CG: Authenticate
    CG-->>UI: JWT access token
    C->>UI: Ask a support question
    UI->>RT: Invoke (bearer token + session ID)
    RT->>MEM: Retrieve customer context
    RT->>KB: Retrieve support guides
    RT->>GW: Call shared tool (JWT)
    GW->>LM: Invoke Lambda
    LM-->>GW: Result
    GW-->>RT: Tool result
    RT->>MEM: Save interaction
    RT-->>UI: Streamed answer
    RT-->>OBS: OpenTelemetry traces
```

---

## Prerequisites

- AWS account with Amazon Bedrock access and a foundation model enabled
- Python 3.10+
- AWS CLI configured
- Amazon SageMaker AI Studio (JupyterLab) or any Jupyter environment with AWS credentials

---

## Steps

### Foundation: environment and knowledge base

Set up a SageMaker AI domain with a JupyterLab space, store six support guides in S3, and index them in a Bedrock Knowledge Base (fixed-size chunking). The sync job scanned and added all six documents.

| SageMaker AI domain | S3 buckets |
|---|---|
| ![SageMaker domain](images/01-sagemaker-domain.png) | ![S3 buckets](images/02-s3-buckets.png) |

| Knowledge base files in S3 | Knowledge base data source sync |
|---|---|
| ![KB objects](images/04-s3-kb-objects.png) | ![KB sync](images/05-bedrock-kb-data-source-sync.png) |

![Downloading KB files in JupyterLab](images/03-jupyter-download-kb-files.png)

---

### Step 1: Create the agent prototype

A customer support agent with four tools:

- Return policy lookup
- Product information search
- Technical support from the knowledge base
- Web search for troubleshooting

**What I learned:** building an agent with Strands Agents, writing a system prompt, and integrating tools, including retrieval from a Bedrock Knowledge Base.

| Return policy tool | Technical support (Knowledge Base) | Web search |
|---|---|---|
| ![Return policy](images/06-step1-agent-return-policy.png) | ![Technical support](images/07-step1-technical-support-tool.png) | ![Web search](images/08-step1-web-search-tool.png) |

---

### Step 2: Add memory

Turn a "goldfish agent" into one that remembers customers across conversations.

- Persistent conversation history
- Customer preference extraction (for example brand, budget, RAM, OS and use case)
- Cross-session context awareness

**What I learned:** AgentCore Memory for both short-term and long-term persistence.

| Memory-enhanced agent | Preference recall |
|---|---|
| ![Memory agent](images/09-step2-memory-agent.png) | ![Preference recall](images/10-step2-preference-recall.png) |

---

### Step 3: Scale with Gateway and Identity

Move from local tools to shared, enterprise-ready services.

- Centralised tool management through AgentCore Gateway
- JWT-based authentication with AgentCore Identity and Amazon Cognito
- Integration with AWS Lambda functions

**What I learned:** exposing tools once and securing access so multiple agents can reuse them.

| Testing the agent against shared tools | Lambda functions created for this step |
|---|---|
| ![Shared tools testing](images/11-step3-shared-tools-testing.png) | ![Lambda functions](images/12-step3-lambda-functions.png) |

---

### Step 4: Deploy to production

Deploy the agent to handle real traffic with full observability.

- Fully managed deployment on AgentCore Runtime
- Session continuity and session isolation
- JWT bearer token invocation
- CloudWatch observability integration

**What I learned:** AgentCore Runtime with production-grade observability.

| Invoking the deployed agent | AgentCore Gateway (Ready) | Cognito user pools |
|---|---|---|
| ![Runtime invocation](images/13-step4-runtime-invocation.png) | ![Gateway](images/14-step4-agentcore-gateway.png) | ![Cognito](images/15-step4-cognito-user-pools.png) |

#### Observability

CloudWatch GenAI Observability shows agents, sessions, traces, spans, token usage and cost. In my test runs it recorded **3 sessions, 28 traces, about 11.4K tokens and about $0.007 total cost**, with per-trace span trees showing model calls and tool execution.

![Observability overview](images/16-cloudwatch-observability-overview.png)

| Trace detail (23 spans) | Trace input and system prompt | OTEL sessions |
|---|---|---|
| ![Trace detail](images/17-cloudwatch-trace-detail.png) | ![Trace input](images/18-cloudwatch-trace-input.png) | ![Sessions](images/19-cloudwatch-otel-sessions.png) |

---

### Step 5: Build the customer interface

A web app customers can actually use.

- Streamlit-based chat interface
- Login backed by Cognito
- Session management
- Context-aware answers that use remembered preferences

**What I learned:** frontend integration with secure agent endpoints.

![Launching the Streamlit app](images/20-step5-launch-streamlit.png)

| Login | Chat | Memory-aware greeting |
|---|---|---|
| ![Login](images/21-step5-streamlit-login.png) | ![Empty chat](images/22-step5-streamlit-chat-empty.png) | ![Greeting](images/23-step5-memory-aware-greeting.png) |

| Troubleshooting answer | Follow-up conversation |
|---|---|
| ![Troubleshooting](images/24-step5-troubleshooting-answer.png) | ![Follow-up](images/26-step5-followup-conversation.png) |

| Return policy answer | Context recall across questions |
|---|---|
| ![Return policy](images/27-step5-return-policy-answer.png) | ![Context recall](images/28-step5-context-recall.png) |

<details>
<summary>Full troubleshooting response</summary>

![Troubleshooting detail](images/25-step5-troubleshooting-detail.png)

</details>

---

## Architecture Evolution

| Step | Architecture |
|---|---|
| **1** | Local agent with embedded tools |
| **2** | Agent + AgentCore Memory for persistence |
| **3** | Agent + Memory + AgentCore Gateway and Identity for shared tools |
| **4** | Deployment on AgentCore Runtime with AgentCore Observability |
| **5** | Customer-facing application with authentication |

---

## Getting Started

1. Clone this repository
   ```bash
   git clone https://github.com/oveskhan86/aws-agentcore-ai-agents.git
   cd aws-agentcore-ai-agents
   ```
2. Install dependencies
   ```bash
   pip install -r requirements.txt
   ```
3. Configure AWS credentials and confirm Bedrock model access in your region
4. Open the notebooks in order, starting with Step 1. Each step builds on the previous one.
5. For Step 5, install the frontend dependencies and start the Streamlit app
   ```bash
   cd <frontend-folder>
   pip install -r requirements.txt
   streamlit run main.py
   ```

> Create your own Cognito test user. Never commit credentials, tokens or account IDs to the repository.

---

## Clean Up

To avoid ongoing charges, delete the resources when you finish, roughly in reverse order: AgentCore Runtime, AgentCore Gateway, Lambda functions and their stack, Cognito user pools, the Bedrock Knowledge Base, S3 buckets, and the SageMaker AI domain.

---

## Skills Demonstrated

`Agentic AI` · `Amazon Bedrock` · `Amazon Bedrock AgentCore` · `Strands Agents` · `RAG / Knowledge Bases` · `Agent Memory` · `Tool Use and Gateways` · `JWT / OAuth with Cognito` · `AWS Lambda` · `OpenTelemetry and CloudWatch` · `Streamlit` · `Python`

---

## Acknowledgements

Built by following and extending the AWS reference tutorial for moving a customer support agent from prototype to production with Amazon Bedrock AgentCore. Screenshots are from my own deployment, with account identifiers redacted.

## Author

**Oves Khan** · AI Engineer
[LinkedIn](https://www.linkedin.com/in/oves-khan) · [GitHub](https://github.com/oveskhan86)
