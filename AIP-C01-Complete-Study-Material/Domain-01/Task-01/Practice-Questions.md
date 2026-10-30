# Task 1.1 — Practice Questions

> I wrote these **12 original** certification-style questions for learning. They are **not** official AWS exam questions. Part B analyses the course's own self-checks and knowledge-check items in my own words and does not reproduce them.
>
> Format matches AIP-C01: *multiple choice* has one correct answer; *multiple response* has two or more correct answers, and you must select all of them. Answers sit inside collapsible blocks, so try each question first.

Back to [Study Notes](Study-Notes.md).

---

## Part A — Original questions

### Q1. Network boundary

A pharmaceutical company wants an internal Q&A assistant over research reports stored in S3. Security policy forbids any traffic to AWS services over the public internet. The team has no ML infrastructure staff. Which design meets the requirements with the LEAST operational overhead?

- A. Host an open-weight model on EC2 GPU instances in a private subnet.
- B. Use Amazon Bedrock Knowledge Bases, with Bedrock and S3 accessed through VPC endpoints and data encrypted with AWS KMS.
- C. Deploy a SageMaker real-time endpoint with model parallelism, behind a NAT gateway.
- D. Use Amazon Bedrock over its public endpoint with TLS 1.2 enforced.

<details><summary>Answer</summary>

**B.** PrivateLink interface endpoints keep traffic on the AWS network. Knowledge Bases provides managed RAG, and KMS covers encryption at rest.

- **A** works technically, but it carries a heavy operational burden for a team with no ML infrastructure staff.
- **C** adds unnecessary complexity, and a NAT gateway still routes traffic to public endpoints.
- **D** uses TLS, but the traffic still goes to a public endpoint, which violates the policy.

*Concept:* private connectivity plus managed RAG.
</details>

### Q2. Integration pattern

An insurer receives 40,000 claim PDFs each night. Summaries must be ready by 7 a.m. Cost should be minimized and nobody waits on individual results. Which approach is MOST cost-effective?

- A. A synchronous API Gateway → Lambda → Bedrock call per document.
- B. Amazon Bedrock batch inference with JSONL input and output in S3.
- C. Bedrock Provisioned Throughput with a 6-month commitment.
- D. A streaming integration with Kinesis Data Streams.

<details><summary>Answer</summary>

**B.** Batch inference is designed for large offline jobs and is priced lower than on-demand for supported models.

- **A** wastes concurrency and invites throttling.
- **C** pays for capacity 24/7 for a nightly job.
- **D** is built for continuous streams, not scheduled bulk jobs.

*Concept:* matching the pattern to latency tolerance.
</details>

### Q3. Multiple response: orchestration (select TWO)

A bank's support assistant must keep conversation context, look up account data in a CRM, and route disputes above $5,000 to a human agent for approval before replying. Which TWO components best implement this?

- A. AWS Step Functions Standard workflow with a callback task for human approval.
- B. Amazon Kinesis Data Analytics for real-time scoring.
- C. Amazon Bedrock Knowledge Bases or an action that retrieves CRM context for grounding.
- D. Amazon SQS FIFO queue as the only orchestration layer.
- E. Bedrock batch inference.

<details><summary>Answer</summary>

**A, C.** Step Functions handles state, branching, and human approval through `.waitForTaskToken`. Retrieval grounds answers in CRM and policy data.

- **B** is stream analytics, the wrong tool.
- **D** queues messages but provides no branching or approval logic.
- **E** is not real-time.

*Concept:* orchestrated workflow plus RAG (mirrors the course self-check).
</details>

### Q4. RAG vs fine-tuning

A legal firm's product data changes weekly. Answers must cite the source paragraph, and client documents must not be embedded into model weights. Which approach fits?

- A. Fine-tune a model monthly on all client documents.
- B. Continued pre-training on the firm's corpus.
- C. RAG with Bedrock Knowledge Bases using `RetrieveAndGenerate`.
- D. Few-shot prompting with five example documents in each prompt.

<details><summary>Answer</summary>

**C.** RAG handles frequently changing facts, returns citations, and keeps data out of model weights.

- **A** and **B** bake data into weights and go stale between training runs.
- **D** cannot cover a large corpus.

*Concept:* RAG vs customization.
</details>

### Q5. PoC scoping

A retailer wants to know whether an LLM can answer product questions in 6 languages before investing. The deadline is three weeks. Which plan best validates feasibility?

- A. Build the production system for all 6 languages, then measure.
- B. A Bedrock prototype covering the 2 highest-volume languages and the top query types, with predefined accuracy, latency, and cost-per-answer targets, compared across 2–3 models.
- C. Train a custom multilingual model on SageMaker.
- D. Deploy a generic chatbot template and survey users.

<details><summary>Answer</summary>

**B.** A narrow scope, explicit metrics, and a model comparison on managed services is the PoC best practice (the same approach as the course case study).

- **A** does full production work before proving feasibility.
- **C** is high effort for a feasibility test.
- **D** cannot test the specific requirements.

*Concept:* PoC methodology.
</details>

### Q6. Best-practice review

An architecture board must check that 12 GenAI workloads follow AWS best practices and must track remediation over time. What is the MOST efficient approach?

- A. Write a custom checklist in a spreadsheet.
- B. Use the AWS Well-Architected Tool with the Generative AI Lens applied to each workload, and track improvement plans and milestones.
- C. Run AWS Trusted Advisor only.
- D. Hire a third party to audit each workload yearly.

<details><summary>Answer</summary>

**B.** The WA Tool records reviews, highlights high and medium risks, saves milestones, and applies the GenAI-specific Lens.

- **A** and **D** are non-standard and slow.
- **C** checks account-level best practices, not GenAI architecture.

*Concept:* WA Tool plus Lens.
</details>

### Q7. API layer

A mobile banking app needs one request to return the user's profile (DynamoDB), recent transactions (an HTTP microservice), and an LLM-generated spending insight. The app also wants real-time push when a long insight finishes. Which API layer fits best?

- A. Amazon API Gateway REST API with three endpoints.
- B. AWS AppSync GraphQL API with multiple data sources and subscriptions.
- C. Application Load Balancer with Lambda targets.
- D. Amazon SNS mobile push only.

<details><summary>Answer</summary>

**B.** AppSync combines multiple data sources in one declarative query, and GraphQL subscriptions push results to the client.

- **A** needs multiple round trips and a separate push mechanism.
- **C** has no aggregation or subscriptions.
- **D** is notification-only.

*Concept:* AppSync for GenAI front-ends.
</details>

### Q8. Throttling at scale

A document service calls Bedrock synchronously from Lambda. During monthly peaks, many requests fail with `ThrottlingException` and users must resubmit. Which change MOST improves reliability with minimal redesign?

- A. Increase the Lambda memory size.
- B. Put an SQS queue between the upload event and the Lambda consumer, with a DLQ, limited concurrency, and SDK retries with exponential backoff.
- C. Switch to a larger model.
- D. Disable retries to reduce load.

<details><summary>Answer</summary>

**B.** Queues smooth bursts, limited consumer concurrency keeps requests within quotas, backoff absorbs transient throttles, and the DLQ captures failures. Cross-Region inference or a quota increase are further options.

- **A** does not affect Bedrock quotas.
- **C** usually lowers throughput.
- **D** makes failures worse.

*Concept:* event-driven decoupling.
</details>

### Q9. Code: request format

A developer's code sends `{"prompt": ..., "max_tokens_to_sample": 500}` with `invoke_model` to a current Claude model and receives `ValidationException`. What is the BEST fix that also makes switching models easier?

- A. Rename `max_tokens_to_sample` to `max_tokens`.
- B. Call `bedrock-runtime` `converse()` with `messages` and `inferenceConfig={"maxTokens": 500}`.
- C. Use the `bedrock` control-plane client instead.
- D. Base64-encode the body.

<details><summary>Answer</summary>

**B.** Converse gives one consistent message schema for all Bedrock text models.

- **A** alone is not enough, because current Claude models also require `messages` and `anthropic_version` in `InvokeModel` bodies.
- **C** is the wrong client: the control plane does not run inference.
- **D** is irrelevant.

*Concept:* Converse vs model-specific bodies.
</details>

### Q10. Multiple response: standardization (select TWO)

A company will roll out the same GenAI claims assistant to 9 subsidiaries in 4 Regions. Which TWO practices MOST improve consistency?

- A. Each subsidiary writes its own prompts and IAM policies.
- B. CloudFormation StackSets / CDK constructs that deploy a reviewed reference stack, including IAM roles, KMS keys, and VPC endpoints.
- C. A shared, versioned prompt library in Bedrock Prompt Management, with a model-invoker library that logs the prompt version.
- D. Copy prompts into application code at each site.
- E. Store templates in S3 without versioning to save cost.

<details><summary>Answer</summary>

**B, C.** Both are reusable, governed, versioned components.

- **A** and **D** cause drift between sites.
- **E** loses audit history.

*Concept:* standardized technical components.
</details>

### Q11. Least important component

A field-service company wants technicians to photograph a broken part and receive repair steps from manuals. Which component is LEAST necessary?

- A. A multimodal FM that accepts images.
- B. Retrieval over repair manuals.
- C. A fine-tuned text-to-speech voice-cloning model.
- D. Guardrails to block unsafe instructions.

<details><summary>Answer</summary>

**C.** No requirement mentions voice.

- **A** is needed because the input is a photo.
- **B** is needed for grounding in the manuals.
- **D** is needed for safety.

*Concept:* spotting components that don't map to a requirement. Compare the course item, where the multimodal model was the unnecessary one; here image input *is* a requirement.
</details>

### Q12. Multi-Region global design

A global assistant must give low latency worldwide. EU customer data must be stored and processed in the EU. Which design is MOST appropriate?

- A. Single Region in us-east-1 with CloudFront.
- B. Regional stacks deployed from the same IaC, AWS Global Accelerator routing to the nearest Region, EU Knowledge Bases holding only EU data, and Bedrock inference in EU Regions (any cross-Region inference profile limited to the EU geography).
- C. A global DynamoDB table replicating all customer data to all Regions.
- D. A cross-Region inference profile with a global scope for all traffic.

<details><summary>Answer</summary>

**B.** It provides low latency and data residency, and the shared IaC keeps the Regional stacks consistent.

- **A** gives EU users high latency for dynamic calls and breaks residency.
- **C** replicates EU data globally.
- **D** may process EU data outside the EU.

*Concept:* deployment strategy under compliance constraints.
</details>

---

## Part B — Course item analysis (concepts tested)

| Course item (paraphrased) | Correct idea | Distractor lesson |
|---|---|---|
| Support chatbot with CRM, context and escalation | Step Functions orchestration + RAG | Sync = too simple; batch is not real-time; streaming analytics is the wrong problem |
| Financial summarization with privacy | Knowledge Bases + Claude | Fine-tuning exposes data and is costlier; few-shot is not grounded |
| Email drafts: brand voice, accuracy, speed (LEAST important) | Multimodal model is unnecessary | Filtering, HITL and KB all map to requirements |
| Sensitive docs must stay in network | Bedrock + VPC endpoint + encryption | SageMaker parallelism / Inferentia = overkill; cross-Region replication is irrelevant |
| Healthcare standardization (TWO) | Reusable IAM/KMS security + standardized Step Functions with compliance monitoring | Isolated components; no version control; ad-hoc Lambda |
| Feasibility of multilingual automation | Focused Bedrock prototype | Full production first; custom ML; rigid templates |
| Validate best practices (TWO) | WA Tool + GenAI Lens; WA Framework | Custom reviews; third-party; generic guidelines |
| Large-scale FM integration | Event-driven SQS + Step Functions | Sync with basic retry; Lambda only; direct access |
| Token cost management (TWO) | CloudWatch token monitoring + smart caching; per-facility budgets + prompt optimization | No controls; fixed 30-day cache; model per doc type |

**Pattern across the course items:** the correct answer is usually the *managed, standardized, observable* option that maps one-to-one to the stated requirements.
