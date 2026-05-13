# Brazil - Real-Time Processing and Concurrency (2015-2017)

> I began my strong technical phase in a project in Brazil focused on real-time image processing (eye-tracking).
> It was a highly demanding environment where I learned to deal with low-level concurrency. We separated data capture from analysis using Python threads to ensure the user was never blocked.
> There, I faced synchronization and performance issues for the first time. The real challenge was avoiding race conditions, which we solved by designing thread-safe structures and strictly controlling access to shared data to ensure the system never crashed under any circumstances.

_Technical notes for the interview:_

- _Tools: OpenCV, MediaPipe, scikit-learn, scipy, NumPy (optimization and signal processing)._
- _GIL Handling: Not a critical problem because many operations were I/O-bound or supported by C-optimized libraries, which gave us enough parallelism._

# AWS and Distributed Architectures (Transactional Platform)

> Later, I moved to a more full-stack role in a high-demand transactional platform. Here, the ecosystem was more heterogeneous: the backend core was built in Node.js, interacting with user interfaces and control panels developed in Angular, while we integrated Python microservices for specific tasks.
> In this project, the challenge changed: the main focus was the resilience and scalability of the entire system. We designed this architecture based on containerized microservices in AWS (ECS).
> We focused heavily on decoupling. For strict synchronous communication between services, we used REST APIs, but we designed most of the flow to be event-driven to prevent the backend from blocking the Angular frontend. For example, we used RabbitMQ for complex (Pub/Sub) transactional state routing between the Node core and Python services. On the other hand, we delegated heavier asynchronous workloads or deferred processing, like background report generation or mass notification sending, to AWS SQS.
> With such a distributed system, observability was critical. We centralized metrics and logs in CloudWatch and set up alarms integrated directly with our Slack channels. This gave us early alerts for any latency spikes, queue failures, or increased error rates, allowing the engineering team to react proactively before it escalated to a user-facing incident.
> At the infrastructure level, we managed traffic using Load Balancers. We configured load balancing to distribute requests to containers using routing algorithms like Round Robin. This allowed us to scale horizontal capacity automatically according to the system's real load. For intermittent processes, we continued to delegate to Serverless (Lambda) to optimize resources.
> CI/CD was implemented using AWS CDK as Infrastructure as Code (IaC) integrated with Jenkins pipelines. This allowed us to define infrastructure as code and automate the entire cycle: from building Docker images and pushing to ECR, to automatic deployment configuring Task Definitions and services in ECS. This ensured that both provisioning and deployments were always immutable, predictable, and versioned.

_Technical notes for the interview:_

- _Stack: Node.js (Core APIs), Angular (Frontend / Control Panels), Python/FastAPI (Microservices)._
- _Communication: REST (synchronous), RabbitMQ (core events / Pub-Sub), SQS (heavy background tasks), Redis (cache)._
- _Observability: CloudWatch integrated with Slack webhooks for early alerts on latencies and consumption errors._
- _CI/CD: Jenkins + AWS CDK (IaC). Pipeline: Build Docker -> Push to ECR -> Generate/Update Task Definitions -> Deploy to ECS._
- _ECS vs Lambda: Preferred ECS for constant traffic and resource control; Lambda for event-driven asynchronous loads._

# Microservices Architecture and Fault Tolerance (Onboarding System)

> In my most recent stage, I co-led the architecture of a fairly complex microservices system for an onboarding platform. We had a Node.js core and an orchestration ecosystem in Python with FastAPI. To scale development and ensure maintainability, we structured services based on Hexagonal Architecture. This allowed us to isolate business logic from infrastructure and have well-defined domains, which was key for different teams to work on new features in parallel and fully decoupled.
> The main focus was not just solving logic, but building a highly predictable and fault-tolerant system. We implemented dynamic routing: if a downstream service failed, we had automatic retries, circuit breakers, and graceful degradation (fallback) strategies. Additionally, we standardized error handling across the board: no internal exception or stack trace ever reached the end user. Everything was intercepted, mapped to consistent HTTP responses, and injected with enriched context into our logging platform.
> All this routing and error handling was supported by a strong monitoring layer that allowed us to trace every request, audit in real time when the system activated a fallback, and react to any problem.
> Finally, security was a non-negotiable pillar. We protected microservice entry points with strict authentication (JWT) and rate limiting from an API Gateway. At the data level, we applied the principle of least privilege: we completely decoupled orchestration from databases using standardized protocols (like MCP), interacting only through validated, read-only interfaces to isolate operational risks.

_Technical notes for the interview:_

- _Architecture: Hexagonal Architecture (Ports and Adapters) for team and domain decoupling. Supervisor orchestrator (LangGraph) for AI._
- _Resilience and Errors: Circuit breakers, retry policies, fallbacks, and Global Exception Handlers to standardize error responses._
- _Observability: Alerts on activated fallbacks, structured logs, and active monitoring of inter-service latencies._
- _Security: API Gateway (JWT, Rate Limiting) and total database isolation (PostgreSQL/pgvector) using MCP with Read-Only roles._
- _Performance: Asynchronous flows and streaming responses (SSE) via FastAPI to optimize latency._

---

**Question:** Your agent system is costing a lot of money per month. Your boss asks you to reduce costs without degrading quality. What do you do?
**Answer:** First, I optimize the system, then the model if necessary.
I investigate the costs of:

- Inference model: Can I switch to a cheaper model (e.g., Claude 3.5 Sonnet) without losing quality?
- Tokens: Can I reduce the number of tokens sent (e.g., summarizing context, using smart chunking)?
- Frequency: Can I reduce the frequency of calls (e.g., caching responses, using triggers instead of periodic calls)?
- Tool usage: Can I make the agent use external tools (e.g., a vector database) to reduce the need for direct inference?
- Embeddings: Can I use a cheaper embedding model for semantic searches, instead of sending all context to the LLM?
- Vector DB: Can I optimize vector database queries to reduce tokens and LLM calls?
- Computational cost: Can I do preprocessing or postprocessing locally to reduce the load on the LLM?

**Question:** Where should guardrails be placed in an agent system?
**Answer:** Guardrails (restrictions) should be implemented at multiple levels:

- **Prompt Level:** Include clear restrictions in the prompt (e.g., "Do not make external API calls", "Respond only with JSON", negative example injection, PII - Personally Identifiable Information). This avoids unnecessary token usage and thus additional costs.
- **Middleware Level:** Implement validations in the code that receives the LLM response (e.g., verify that the JSON is valid, that certain limits are not exceeded).
- **Model Level:** Use models with enhanced security capabilities or low temperature settings to reduce hallucinations.
- **Tool Level:** If the agent can use tools, implement restrictions in the tools themselves (e.g., limit which endpoints can be called, validate inputs/outputs).

**Question:** What happens if a tool or agent fails? What would be the fallback mechanism?
**Answer:** In a multi-agent architecture with LangGraph, we handle failures at several levels to ensure resilience and graceful degradation:

- **Retries and Auto-correction (Agent Level):** If a tool fails (for example, Text2SQL generates an invalid query or a required parameter is missing), we capture the exception. The Supervisor Agent receives the error message and asks the sub-agent to reflect and correct its call (limited to a maximum of N retries to avoid infinite loops).
- **Fallback Routing (Graph Level):** LangGraph allows defining alternative routes (conditional edges) and fallback nodes. If a sub-agent or MCP server does not respond after the allowed retries, the flow is automatically redirected to a contingency strategy.
- **Graceful Degradation:** If, for example, the system that answers SQL queries is down, the Supervisor can "fallback" to the RAG Agent to try to answer using only documented knowledge, or return a friendly message to the user ("I can't access your progress data right now, but...") instead of showing a 500 error and breaking the experience.
- **Circuit Breaker and Timeouts (Infrastructure Level):** In FastAPI connections to AWS Bedrock or MCP servers, we apply strict timeouts and a Circuit Breaker pattern. If an external service fails repeatedly, the circuit opens to fail fast and not exhaust server resources (threads/connections).

# Questions:

- What are the main technical challenges for the team today?
- How do you handle reliability and observability in production?
- What level of involvement do developers have in deployments and operational support?

---

# Why LangGraph and not traditional LangChain or a sequential pipeline?

Standard LangChain is excellent for linear chains (simple DAGs), but our onboarding flow required cycles and dynamic decision-making. LangGraph allowed us to define the logic as a finite state machine. With a Supervisor Agent, if the Text2SQL Agent generated an incorrect query, the supervisor detected it and forced an internal loop for the sub-agent to correct the SQL by reading the database schema before returning the response to the user—something very difficult to do cleanly with traditional chains.

# How did you manage memory in this multi-agent architecture?

We used LangGraph's _checkpointer_ system backed by PostgreSQL. The graph state was persisted at the end of each execution node. To guarantee data separation and privacy, we associated memory with a unique identifier per session and user. When a user sent a message, the frontend sent a `session_id` linked to their authentication token. We loaded the state using this `session_id` as `thread_id` in LangGraph, which gave access to strictly session-limited historical context. Additionally, we injected the `user_id` and their role (RBAC) into the graph state, so agents, especially Text2SQL, could only access information permitted for their access level.

# How did you optimize Bedrock latency in the FastAPI microservice?

With multiple agents interacting before giving a final answer, latency (Time to First Token) could increase. We mitigated this in two ways:

1. Routing and internal validation agents used faster, lighter models, reserving the more powerful (and slower) Bedrock models only for complex reasoning tasks.
2. We implemented asynchronous endpoints in FastAPI returning responses via _streaming_ using Server-Sent Events (SSE), so the frontend could start rendering the LLM response while it was still being generated.

# How to act in the face of latency problems?

1. Locate the bottleneck: Is it model inference, database query, or orchestration logic?
2. What optimization solves the problem: Can I use a faster model, reduce tokens, cache results, parallelize tasks, compress context?
