# Context engineering

Problems:

## Memory management

Most valuable tasks solved agentically take more than a short, simple chat. They take days, multiple sessions, and require remembering past decisions, constraints, and current progress. \
With naive memory:
- Agents remember **outdated facts**
- **Context is bloated**, which leads to higher costs and hallucinations
- Agents **lose constraints**

### Examples

#### 1. Coding agent on real repo

- Day 1: the agent implements “task X” and stores decisions in chat (files, approach, test results)
- Overnight, reality changes (new commits/CI issues), so stored details become **outdated facts**
- Day 2: the agent uses the old chat info and edits the wrong place or targets the wrong CI failure
- Debugging adds big CI logs, repeated file reads, and multiple diffs to the same session
- This **context bloat** raises token cost and hides the real code issues
- Previous rules (e.g., “use library Y”) get buried in the long session
- The agent then **loses constraints** and reintroduces forbidden changes
- Outcome: repeated fixes, higher cost with lower trust

#### 2. Customer support ticket thread

- Day 1: customer reports issue (“feature X fails”); agent drafts reply, asks for logs, notes account tier
- Day 2–3: customer sends partial info (screenshot, truncated logs); agent adds more questions, suggests a workaround
- Meanwhile, reality changes: ticket status changes, internal incident is opened, etc.; earlier notes become **outdated facts**
- Day 4–5: agent resumes using old assumptions (“still investigating Y”, “need logs”) and follows the wrong hypothesis
- Context grows fast: multiple customer replies + internal notes + pasted logs + screenshots → **context bloat**
- With bloated context, the agent mixes old vs new, and makes confident wrong statements (“we confirmed Z”)
- Critical rules get buried: “don’t request PII, “refund requires approval”; agent **loses constraints**
- Result: incorrect promises, higher handling time, lower customer trust

### Solutions

#### 1. OpenAI Agents SDK sessions with context trimming/context summarization [link](https://developers.openai.com/cookbook/examples/agents_sdk/session_memory)

A session layer that keeps short-term continuity across agent runs, plus patterns for trimming and compression to keep prompts small. \

Pros:
- Simple way to avoid re-sending full chat history while preserving continuity
- Built-in patterns for trimming/compressing long threads (targets **context bloat**)

Cons:
- Summaries can drop details
- Remembers **outdated facts** when the world outside changes (e.g. CI change, ticket status)
- No clear built-in rules for what should become long-term memory (a sessions is essentially a transcript)

#### 2. LangGraph persisted state [link](https://docs.langchain.com/oss/python/langgraph/memory)

Workflows modeled as graphs with persisted state (checkpoints) that can pause and later resume from the saved checkpoint.

Pros:
- Good for long-running workflows (clean pause/resume mechanism)
- State is structured (better than just a "transcript") 

Cons:
- More engineering: must design the state schemas and transitions
- Bad state design can still cause **context bloat** and **outdated assumptions** (you can choose to persist the wrong thing)

#### 3. Claude Code memory [link](https://code.claude.com/docs/en/memory)

Coding-agent that has durable instruction files (e.g. CLAUDE.md / rules) plus auto memory (notes Claude writes to a persistent per-project store) to keep constraints and repo conventions consistent across sessions.

Pros:
- Practical for teams (project rules live in files)
- Auto memory helps continuity without keeping a huge single conversation

Cons:
- Risk of outdated facts if “memory” captures changing details (paths, decisions)
- Rule files can be poorly written (which dillutes **important constraints**)
- Auto memory quality can still vary

#### 4. Mem0 [link](https://docs.mem0.ai/open-source/python-quickstart)

A standalone memory layer you add to an agent/chat app: you write memories (`add`) and retrieve them (`search`) later; optionally adds Graph Memory (entity/relationship extraction).

Pros:
- Reduces **context bloat** by retrieving only relevant memories instead of repeating a "transcript"
- Graph memory preserves relationships
- Works as plug-in across many agent stacks (memory as a service)

Cons:
- Doesn’t automatically prevent **outdated facts**; you still need verification policies
- You can put garbage in the memory (have bad writes which create bad retrievals later)

#### 5. MemGPT paper [link](https://arxiv.org/abs/2310.08560)

A system design that adds hierarchical memory tiers (the LLM is like a CPU with limited RAM) to move information in/out of the prompt.

Pros:
- Directly addresses **context bloat**: keep a smaller working set in-context, page the rest
- More principled than ad-hoc summarization (explicit memory tiers + control flow)

Cons:
- Just a pattern, not a plug-in product (you still implement storage, retrieval, policies etc.)
- Still needs freshness checks for **outdated facts** (can page back stale info)

#### 6. Extra

The CoALA paper ([link](https://arxiv.org/abs/2309.02427)) outlines a useful framework for memory in agents: explicit memory modules (working + episodic/semantic/procedural) analogous to human memory. 

### Wedge

Freshness-aware memory layer. Similar to Mem0, but with an **outdated fact** prevention mechanism.

# AI safety

Problems:

## Sensitive data leakage

Using LLM apps can lead to **sensitive data** (PII, credentials, internal documents) reaching places it shouldn't: the **model vendor**, **logs/traces**, **vector databases**, or **another user’s response**.

### Examples

#### 1. Simple LLM chat

- A user adds **internal documents**, **PII** etc. to their prompt.
- Result: the **sensitive data** reaches the **model vendor**.

#### 2. Coding agent + leaked secrets via logging

- A dev pastes .env or a stack trace that includes **secrets**.
- The app **logs** prompts/responses to an observability tool for debugging.
- Those logs persist and are accessible to many engineers.
- Result: **secrets/PII** spread across systems (chat DB, logs, backups).

#### 3. Customer support bot with multi-tenant setup 

- You build a support assistant for many customers (tenants).
- You store all embeddings in one place and forget to filter retrieval by tenant.
- A user asks: “show my last invoice,” and the bot retrieves a chunk from **another tenant’s** invoice.
- Result: direct privacy breach.

#### 4. Internal support bot + permission-blind retrieval

- You connect Google Drive and bulk-index documents into a shared RAG index (including restricted HR/payroll files).
- Retrieval is not enforced with permission/role checks.
- An employee asks an innocent question (“What’s our parental leave policy?”) and the bot retrieves a chunk from a **restricted HR/payroll document**.
- Result: unauthorized disclosure of **sensitive internal information**.

### Solutions

#### 1. Google Sensitive Data Protection (DLP API) [link](https://cloud.google.com/security/products/sensitive-data-protection)

A managed Google Cloud service/API to discover/classify **sensitive data** and apply de-identification.

Pros:
- Strong control: stops **PII** from reaching the **model/logs/storage**
- Built-in de-identification methods

Cons:
- Cost, latency, vendor dependency
- False positives remove **useful context**, false negatives can still happen

#### 2. Microsoft Persidio [link](https://microsoft.github.io/presidio/anonymizer/)

Open-source toolkit to detect **PII** and then anonymize it.

Pros:
- Deployable on-prem/offline
- Customizable anonymization operators

Cons:
- Not true permission control (it only transforms content)
- Requires tuning/maintenance
- **Context loss** (e.g. aggressive redaction)

#### 3. Datadog Sensitive Data Scanner [link](https://docs.datadoghq.com/security/sensitive_data_scanner/)

A product feature to detect and hash/mask **sensitive sequences** in **telemetry (logs)** at ingestion.

Pros: 
- Directly addresses one of the most common leak paths: observability pipelines

Cons:
- Only protects what reaches the logs (does not stop leakage into the **model**, **vector DB** etc.)

#### 4. Amazon Kendra [link](https://docs.aws.amazon.com/kendra/latest/dg/what-is-kendra.html)

An AWS enterprise search service that indexes document sets and supports filtering search results based on user/group access.

Pros:
- Strong mitigation for RAG leakage from permission-blind retrieval
- Connectors for common sources (SharePoint, Drive)

Cons:
- Integration complexity: identity, groups, connectors etc.
- Not a full RAG safety solution: sensitive data can still leak through **prompts/logs/tools**
- Vendor constraints

#### 5. Pinecone namespaces [link](https://docs.pinecone.io/guides/index-data/implement-multitenancy)

Namespaces let you partition vectors so queries operate on one subset (commonly one per tenant).

Pros:
- Tenant data separation

Cons:
- Does not solve permissions within a tenant

All of the above are only partial solutions to the sensitive data leakage issue in LLM apps. Most of them fit inside one step in the following framework: 1) User input → 2) RAG retrieval (search docs) → 3) Build prompt → 4) LLM → 5) Answer → 6) Logs/analytics

### Wedge

A complete privacy + compliance layer for LLM/agentic apps with features like:
- Pre-processing: detect and redact PII/secrets before sending to the model
- Permission-aware retrieval/tool use
- Safe logging
- Tenant isolation defaults
