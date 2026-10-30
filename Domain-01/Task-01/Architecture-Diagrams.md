# Task 1.1 — Architecture Diagrams

Editable Mermaid diagrams. They render on GitHub, in VS Code (Mermaid extension), and in Obsidian. 📘 marks designs taken from the course; 🧠 marks illustrative designs built for learning. Back to [Study Notes](Study-Notes.md).

1. [Integration pattern map](#1-integration-pattern-map)
2. [Case study: multilingual financial assistant](#2-case-study-multilingual-financial-assistant)
3. [Bonus Assignment: claim-processing PoC](#3-bonus-assignment-claim-processing-poc)
4. [Production evolution: event-driven claim pipeline](#4-production-evolution-event-driven-claim-pipeline)
5. [Synchronous request/response sequence](#5-synchronous-requestresponse-sequence)
6. [Standardized components and GenAIOps loop](#6-standardized-components-and-genaiops-loop)

---

## 1. Integration pattern map

🧠 A decision flow for picking an integration pattern from the requirement wording.

```mermaid
flowchart TD
    A[Requirement] --> B{User waiting<br/>for answer?}
    B -- Yes --> C{Single step?}
    C -- Yes --> D[Synchronous / Streaming<br/>API GW or AppSync -> Lambda -> Bedrock Converse]
    C -- No, multi-step, state, escalation --> E[Orchestrated workflow<br/>Step Functions + RAG / Agents]
    B -- No --> F{Large volume<br/>offline?}
    F -- Yes --> G[Batch inference<br/>S3 JSONL -> Bedrock batch job]
    F -- No, event-triggered --> H[Async / Event-driven<br/>EventBridge or SQS -> Lambda -> Bedrock]
    A --> I{Needs private<br/>company facts?}
    I -- Yes --> J[RAG: Bedrock Knowledge Bases]
    A --> K{Needs actions in<br/>other systems?}
    K -- Yes --> L[Agentic: Bedrock Agents + action groups]
```

**How to read it:** first decide whether the user waits for the answer, then how many steps the work has. RAG and agents are *added* to whichever pattern you picked; they are not alternatives to it.

---

## 2. Case study: multilingual financial assistant

📘 Reconstructed from the course "Review Together" transcript.

```mermaid
flowchart TB
    U[Customers<br/>15 countries] --> GA[AWS Global Accelerator]
    GA --> R1
    GA --> R2
    subgraph R1[Region A - repeated per Region via CloudFormation]
        COG[Amazon Cognito] -.auth.-> AS[AWS AppSync<br/>GraphQL API]
        AS --> L[Lambda / resolvers]
        L --> BR[Amazon Bedrock<br/>Claude 3.5 Sonnet]
        L --> KB[Bedrock Knowledge Bases]
        KB --> DS[(S3 / SharePoint /<br/>Confluence)]
        L --> PM[Prompt Management<br/>+ Flows]
        L --> EB[Amazon EventBridge]
        EB --> CRM[Existing CRM<br/>tickets]
        KMS[AWS KMS] -.encrypt.-> DS
        CW[CloudWatch + X-Ray] -.observe.-> L
    end
    subgraph R2[Region B ...]
        X[Same standardized stack]
    end
    SMP[SageMaker Pipelines<br/>continuous evaluation] -.accuracy / bias / compliance.-> BR
```

| Component | Role |
|---|---|
| Global Accelerator | Routes users to the nearest healthy Region (latency, resilience) |
| Cognito | Authenticates customers and agents |
| AppSync | GraphQL API layer: declarative fetching, built-in security |
| Lambda / resolvers | Business logic and prompt assembly |
| Bedrock (Claude 3.5 Sonnet) | Multilingual reasoning and generation |
| Knowledge Bases | Managed RAG over financial documents |
| Prompt Management / Flows | Versioned templates per query type and language; chains multi-step queries |
| EventBridge | Sends events to the CRM to create and update tickets |
| KMS / IAM | Encryption and fine-grained access |
| CloudWatch / X-Ray | Dashboards, alarms, tracing |
| SageMaker Pipelines | Automated accuracy, bias and compliance evaluation |
| CloudFormation | Deploys the same stack in every Region |

**Failure points:**

- **Bedrock throttling:** retries with backoff; cross-Region inference.
- **KB sync lag:** scheduled ingestion jobs.
- **CRM unavailability:** EventBridge retries + DLQ.
- **Regional outage:** Global Accelerator health checks fail over to another Region.

---

## 3. Bonus Assignment: claim-processing PoC

📘 Scope from the course; 🧠 layout.

```mermaid
flowchart LR
    DEV[Python app / notebook] -->|upload| S3[(S3 bucket<br/>claim-documents-poc)]
    DEV -->|get_object| S3
    DEV --> PTM[PromptTemplateManager]
    PTM --> MI[Model invoker<br/>Converse API]
    MI -->|1. extract JSON, temp 0| BR[Amazon Bedrock]
    MI -->|2. summarize, temp 0.3| BR
    POL[(Policy docs)] --> RAG[Simple RAG<br/>or Knowledge Base]
    RAG --> PTM
    MI --> VAL[Content validator<br/>JSON schema + Guardrails]
    VAL --> OUT[Fields + summary + metrics]
```

**Data flow:**

1. Read the document.
2. Fill the template (adding retrieved policy context).
3. Bedrock call #1 extracts the fields.
4. Validate the JSON.
5. Bedrock call #2 writes the summary.
6. Return the results with token and latency metrics.

---

## 4. Production evolution: event-driven claim pipeline

🧠 How the PoC grows into a production design (the event-driven + orchestration pattern the course quiz rewards).

```mermaid
flowchart TB
    UP[Upload] --> S3[(S3 - KMS encrypted)]
    S3 -->|ObjectCreated| EB[EventBridge]
    EB --> Q[SQS queue<br/>+ DLQ]
    Q --> SF[Step Functions<br/>Standard workflow]
    subgraph SF_STEPS[State machine]
        T1[Textract / parse] --> T2[Bedrock: extract]
        T2 --> T3[Validate JSON +<br/>Guardrails]
        T3 --> C{Amount > threshold<br/>or low confidence?}
        C -- yes --> H[Human review<br/>waitForTaskToken]
        C -- no --> T4[Bedrock: summarize]
        H --> T4
    end
    SF --> SF_STEPS
    T4 --> DDB[(DynamoDB results)]
    T4 --> EB2[EventBridge -> claims system]
    VPCE[VPC endpoint<br/>bedrock-runtime] -.private path.-> T2
    CW[CloudWatch metrics:<br/>tokens, latency, throttles] -.-> SF
```

**Why this design:**

- SQS absorbs spikes and protects against Bedrock throttling.
- Step Functions gives retries, branching, human-in-the-loop and an audit trail.
- The VPC endpoint keeps traffic private.
- EventBridge decouples downstream systems.

---

## 5. Synchronous request/response sequence

🧠 What happens on the wire for one `Converse` call.

```mermaid
sequenceDiagram
    participant C as Client
    participant API as API Gateway / AppSync
    participant L as Lambda (execution role)
    participant B as Bedrock Runtime
    C->>API: POST /ask (Cognito JWT)
    API->>API: Authorize (Cognito / IAM)
    API->>L: Invoke with event
    L->>B: converse(modelId, messages, inferenceConfig) [SigV4]
    B->>B: IAM check bedrock:InvokeModel, quota check
    B-->>L: output.message, usage, metrics, stopReason
    L-->>API: JSON answer
    API-->>C: 200 OK
    Note over L,B: ThrottlingException -> retry with backoff<br/>ValidationException -> fix request
```

**Authentication chain:**

1. The user authenticates with Cognito.
2. The API authorizes the request.
3. Lambda assumes its execution role.
4. boto3 signs the request with SigV4.
5. Bedrock evaluates IAM (and SCPs/endpoint policies) before running the model.

---

## 6. Standardized components and GenAIOps loop

🧠 Built from the course's Skill 1.1.3 list.

```mermaid
flowchart TB
    subgraph LIB[Shared component library]
        IAC[IaC templates<br/>CloudFormation / CDK]
        SEC[Security: IAM roles, KMS,<br/>Cognito, VPC endpoints]
        PRM[Prompt Management<br/>versioned templates]
        INV[Model invoker<br/>retries, logging, config]
        VAL[Validator + Guardrails]
        OBS[CloudWatch dashboards,<br/>X-Ray, invocation logs]
    end
    LIB --> APP1[Use case A]
    LIB --> APP2[Use case B]
    LIB --> APP3[Region / BU N]
    subgraph OPS[GenAIOps loop]
        EXP[Experiment<br/>MLflow] --> PIPE[SageMaker Pipelines<br/>evaluate: accuracy, bias, compliance]
        PIPE --> GATE{Pass?}
        GATE -- yes --> REL[Release new prompt /<br/>model version]
        GATE -- no --> EXP
        REL --> MON[Monitor in production] --> EXP
    end
    APP1 --> MON
    WAT[WA Tool + Generative AI Lens review] -.risks / improvement plan.-> LIB
```
