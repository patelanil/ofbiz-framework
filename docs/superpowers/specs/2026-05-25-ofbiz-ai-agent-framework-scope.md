# OFBiz AI Agent Framework — Scope Document

> **Status:** Draft — awaiting review
> **Date:** 2026-05-25
> **Component:** `plugins/ai`

---

## 1. What We Are Building

A declarative AI agent framework built natively into the existing OFBiz `ai` plugin. Developers declare **tools** — existing OFBiz services that an LLM is permitted to call — and **agents** — LLM personas with a system prompt and a curated tool allow-list — in XML files placed in any component's `ai/` directory. The framework's startup Container scans every installed component for these declarations, validates them eagerly, and indexes them in memory. At runtime, calling the `agentRun` service kicks off an agent loop: the framework sends the user's message to the LLM over a direct HTTP connection, executes whichever OFBiz services the LLM requests using the standard service dispatcher, feeds results back, and continues until the LLM produces a final answer or an iteration cap is reached. LangChain4j is removed entirely. No third-party AI abstraction library is needed or wanted.

---

## 2. Core Design Principles

| Framework concern | OFBiz-native approach |
|---|---|
| **Startup and lifecycle** | `Container` interface (`AiContainer`), registered in `ofbiz-component.xml` under `<container>`. `start()` loads all registries; `stop()` clears them. This is the same pattern used by `JavaMailContainer`, `CatalinaContainer`, and others. |
| **Configuration** | `UtilProperties` reads named-provider settings from `config/ai.properties`. API keys that must not live in a file can be stored in the `SystemProperty` entity and retrieved at runtime via `EntityUtilProperties`. |
| **Component-level declarations** | `AiContainer.start()` iterates `ComponentConfig.getAllComponents()`, constructs the `ai/` path for each component, and parses any `*.tools.xml` and `*.agent.xml` files found there. No extra declaration in `ofbiz-component.xml` required — the scan is fully automatic, just like how `servicedef/` files are discovered. |
| **Service invocation (tools)** | `LocalDispatcher.runSync()`. Tools are plain OFBiz services; the framework calls them exactly the same way any other service calls a service. No wrappers, no adapters, no reflection. |
| **Tool schema derivation** | `DispatchContext.getModelService(serviceName)` returns a `ModelService`; its `getInModelParamList()` provides parameter names, types, and optionality — everything needed to build the JSON schema the LLM receives. No hand-written schema. |
| **Persistence** | `GenericDelegator` and `GenericValue`. Entity definitions in `entitydef/` XML, loaded via `<entity-resource>` in `ofbiz-component.xml`. Standard OFBiz entity engine. |
| **Security** | `Security.hasPermission(userLogin, permission, ...)` before any tool executes. The `UserLogin` is the one already present in the service context of the enclosing `agentRun` call. |
| **LLM HTTP calls** | `java.net.http.HttpClient` (Java 11, already in the JVM). No additional HTTP library needed. |
| **JSON** | Jackson `ObjectMapper` (already a transitive OFBiz dependency). Handles both request body construction and response parsing. |
| **Logging** | `Debug.logInfo`, `Debug.logWarning`, `Debug.logError` with the fully qualified class name as the module string, following OFBiz convention throughout. |

---

## 3. What Developers Write

### Tool declaration (`ai/*.tools.xml`)

Any component contributes tools by placing a `*.tools.xml` file in its `ai/` subdirectory. A tool points at an existing OFBiz service, gives the LLM a description, and optionally marks some service parameters as hidden (excluded from the LLM's JSON schema view) or restricts access to users who hold a named OFBiz permission.

```xml
<tools>
    <tool name="findOrders"
          service="mantle.order.OrderServices.findOrders"
          required-permission="AI_TOOL_ORDER_VIEW">
        <description>Find open orders by status or customer.
Returns a summary list — use getOrderDetail for full order data.</description>

        <parameter name="statusId">
            <description>Filter by order status, e.g. OrderApproved.</description>
        </parameter>
        <parameter name="pageIndex" hidden="true"/>

        <example user="Show approved orders for ACME">
            { "customerPartyId": "ACME", "statusId": "OrderApproved" }
        </example>
    </tool>
</tools>
```

**Key rules:**
- `name` is globally unique across all components. A duplicate is a fatal startup error.
- `service` must be a defined OFBiz service. An undefined service is a fatal startup error.
- `hidden="true"` parameters are stripped from the LLM's JSON schema but still passed when the service is called (with their default values).
- `required-permission` is optional. When present, the framework checks it before invoking the service.
- `<example>` content is appended to the description string that the LLM sees.

### Agent declaration (`ai/*.agent.xml`)

An agent is a named LLM persona: a provider, a model, a system prompt, and the exact set of tools it is allowed to call. The tool list is an explicit allow-list — the agent cannot reach any tool not named here, regardless of what other tools exist in the catalog.

```xml
<agents>
    <agent name="OrderAssistant"
           provider="openai-default"
           model="gpt-4.1"
           max-iterations="6">

        <system-prompt><![CDATA[
You are an order operations assistant.
Use the available tools to answer questions. Be concise.
        ]]></system-prompt>

        <tool name="findOrders"/>
        <tool name="getOrderDetail"/>
    </agent>
</agents>
```

**Key rules:**
- `name` is globally unique across all components. A duplicate is a fatal startup error.
- `provider` must resolve to a configured provider in `ai.properties`. An unconfigured provider is a fatal startup error.
- Every `<tool>` name must exist in the tool catalog. An unknown tool is a fatal startup error.
- `max-iterations` defaults to 6 if not specified.
- The system prompt can alternatively be loaded from a file using a `<system-prompt-location>` element with a component-relative path.

### Invoking an agent

Agents are invoked as ordinary OFBiz services. No agent-specific API; no factory lookup:

```java
Map<String, Object> ctx = new HashMap<>();
ctx.put("agentName", "OrderAssistant");
ctx.put("userMessage", "Show me approved orders for ACME Corp");
ctx.put("userLogin", userLogin);

Map<String, Object> result = dispatcher.runSync("agentRun", ctx);
String answer = (String) result.get("assistantMessage");
```

---

## 4. Component Layout

All new and updated files live within `plugins/ai/`:

```
plugins/ai/
├── ofbiz-component.xml            updated: adds <entity-resource> and <data-resource> entries
├── build.gradle                   updated: removes LangChain4j dependency declarations
├── config/
│   └── ai.properties              updated: named-provider format (openai-default, anthropic-default, ollama-default)
├── entitydef/                     NEW
│   └── AiAgentEntities.xml        AiAgentRun, AiAgentToolCall, AiProviderCost, AiConversationThread, AiConversationMessage, AiAgentProposal, AiAgentProposalTool
├── data/                          NEW
│   └── AiAgentSeedData.xml        StatusType, StatusItem seed rows; AiProviderCost seed rows
├── servicedef/
│   └── services.xml               updated: adds agentRun, getUsageSummary, archiveConversationThread, getConversationHistory, approveAgentProposal, rejectAgentProposal
├── src/
│   └── main/java/org/apache/ofbiz/ai/
│       ├── AiServices.java        keep: aiGenerate, aiGenerateStructured (refactored to use direct HTTP)
│       ├── AiWorker.java          refactored alongside new agent layer
│       ├── container/
│       │   └── AiContainer.java   refactored: no LangChain4j imports; holds ToolCatalog and AgentRegistry as fields
│       └── agent/                 NEW package
│           ├── ProviderRegistry.java
│           ├── ToolDescriptor.java
│           ├── ToolCatalog.java
│           ├── AgentDefinition.java
│           ├── AgentRegistry.java
│           ├── AiChatClient.java  (interface)
│           ├── AiHttpClient.java  (production implementation)
│           └── AgentRunner.java
└── testdef/
    └── AgentTests.java            updated: agent integration and offline unit tests
```

---

## 5. Phase Overview

| # | Name | Delivers | Done when |
|---|---|---|---|
| **1** | Foundation | LangChain4j removed. Tool and agent XML declarations load at startup. Agents callable as OFBiz services. | `AgentIntegrationTest` passes: ToolCatalog loads, AgentRegistry loads, `agentRun` is defined and invocable. Live API call returns a coherent answer. `./gradlew classes` shows zero LangChain4j references. |
| **2** | Observability | Every run persisted to the database. LLM layer injectable for offline unit tests. | `AiAgentRun` and `AiAgentToolCall` records written after every run. `AgentRunnerTest` passes with no network access or API key. |
| **3** | Security and Cost | Tool permissions enforced before execution. Token costs estimated and queryable. | Permission-denied tool call produces a graceful LLM explanation, not an exception. `getUsageSummary` returns correct estimates for known models and null (without crashing) for unknown models. |
| **4** | Conversation Memory | Agent remembers prior turns within a named thread. | Two-turn integration test passes: second call carries context from first and the LLM's answer demonstrates it. Calls without a threadId are unaffected. |
| **5** | Human Approval Gate | Agent proposes actions before executing them. Human approves or rejects. | End-to-end test passes: `agentRun` with `approvalRequired=true` returns a proposalId with no tool executed; `approveAgentProposal` executes tools and returns the final answer; double-approval is rejected. |
| **6** | Advanced Capabilities | Prompt caching, model fallback, agent composition, structured output, admin screens. | Future sprints, tracked individually. |

---

## 6. Phase Descriptions

### Phase 1 — Foundation

This phase replaces LangChain4j with a direct HTTP implementation and builds the full agent framework skeleton. When it completes, the framework is operational end-to-end: tools and agents declared in XML, agents callable as services, real LLM calls succeeding.

**What is being built and why:**

**ProviderRegistry** reads named-provider configuration from `ai.properties` using `UtilProperties`. A named provider is a set of related settings — base URL, API key, model, timeout, and any extra HTTP headers the provider requires (Anthropic, for example, requires a version header on every request). By grouping these as named sets of properties, the framework can support any number of providers without branching logic anywhere in the call path. Configuration is validated at startup, not deferred to call time.

**ToolDescriptor** is an immutable value object that carries everything about one tool: its name, which OFBiz service it wraps, the LLM-facing description (assembled from the `<description>` and any `<example>` blocks), the list of parameters that are hidden from the LLM, the required permission if any, and a pre-built JSON schema object ready to send to the LLM. It is built once at startup and read many times at runtime with no allocation.

**ToolCatalog** scans every component's `ai/` directory at startup, parses each `*.tools.xml` file it finds, and builds one `ToolDescriptor` per declared tool. For each tool it validates that the referenced OFBiz service is actually defined (via `DispatchContext.getModelService()`), reads that service's in-parameters to generate the LLM-facing JSON schema automatically (no hand-written schema), and rejects duplicate tool names with a fatal error. The catalog is a global index — any agent in any component can reference any tool by name.

**AgentDefinition** is an immutable value object for one agent: its name, provider, model override, maximum loop iterations, system prompt, and tool allow-list. The system prompt can be declared inline in the XML or loaded from a file using a component-relative path, which keeps large prompts out of the XML.

**AgentRegistry** scans `*.agent.xml` files the same way ToolCatalog scans tools. It validates eagerly at startup: every tool name in an agent's allow-list must exist in the ToolCatalog, and every provider name must resolve in the ProviderRegistry. These failures are fatal — the system refuses to start with a misconfigured agent. This surfaces errors at deploy time, not during a live user interaction.

**AiHttpClient** is the LLM HTTP layer. It uses `java.net.http.HttpClient` and Jackson to construct an OpenAI-compatible request body (message list, tool schema list, model name), POST it to `{baseUrl}/chat/completions`, and parse the response. The response is either a final answer (finish reason `stop`) or a list of tool calls (finish reason `tool_calls`). It extracts token usage counts from the response for later cost tracking. Extra headers from the provider configuration — such as Anthropic's required version header — are applied without any provider-specific branching; the provider config carries them as a key-value string.

**AgentRunner** is the agent loop. It loads the `AgentDefinition` and resolves the `ToolDescriptor` list for the agent's allow-list. It builds the initial message list — system prompt followed by the user message — and enters the loop. On each iteration it calls `AiHttpClient`, checks the finish reason, and either exits with the final answer or dispatches each requested tool call via `LocalDispatcher.runSync()`. Tool results are serialized to JSON, capped at a configurable character limit to prevent context window overflow, and appended as tool-role messages before the next LLM call. The loop runs until the LLM stops, the iteration cap is reached, or an unrecoverable error occurs.

**`agentRun` service** is the public entry point — a standard service definition in `services.xml`. It accepts `agentName`, `userMessage`, and `userLogin`, constructs an `AgentRunner`, calls it, and returns `assistantMessage`, `stopReason`, and `iterationsUsed`. Callers use no agent-specific API.

**`AiContainer` refactoring** removes all LangChain4j imports and build dependencies. The container holds the `ToolCatalog` and `AgentRegistry` as fields populated in `start()`, and exposes them via static accessors so the `agentRun` service implementation can reach them without a dependency injection framework.

**Done when:** `AgentIntegrationTest` passes — ToolCatalog loads tools, AgentRegistry loads agents, `agentRun` is defined, and a live call with a real API key returns a coherent answer. `./gradlew classes` succeeds. `grep -r "langchain4j"` inside the plugin source returns zero matches.

---

### Phase 2 — Observability

This phase makes every agent run queryable and the framework testable offline. Before adding security or cost controls, the underlying data must exist. Before growing the codebase further, the test seam must exist.

**What is being built and why:**

**`AiAgentRun` entity** records one row per `agentRun` invocation. It captures: who ran it (the UserLogin), which agent, when it started and ended, the original user message, the final assistant response, how many iterations the loop took, the token counts extracted from LLM responses, and a status (started, completed, failed). This is the audit record on which all future observability, cost tracking, and debugging rests.

**`AiAgentToolCall` entity** records one row per tool invocation within a run. It links to the parent `AiAgentRun`, names the tool, stores the arguments the LLM sent and the serialized result the OFBiz service returned, the call timestamp, and a status (completed, failed). Together with `AiAgentRun` it gives a complete replay of what happened in any given run.

**`AiAgentEntities.xml`** defines both entities using standard OFBiz entity definition syntax. A new `<entity-resource>` entry in `ofbiz-component.xml` makes them visible to the entity engine at startup.

**`AiChatClient` interface** extracts the LLM call into a one-method contract so `AgentRunner` no longer has a hard dependency on `AiHttpClient`. This single seam is what makes offline testing possible. The interface is minimal — one method that accepts the message list, tool schema list, model, and provider config, and returns a typed response carrying the finish reason, content or tool calls, and token counts. `AiHttpClient` implements this interface; nothing else changes in the production path.

**`MockAiChatClient`** is a scripted test double. A test builds a sequence of scripted responses — "first call returns this tool call, second call returns this final answer" — and passes the mock directly to `AgentRunner`, bypassing the Container entirely. If the script is exhausted before the loop finishes, the mock throws, which is a test failure signal rather than a silent pass.

**`AgentRunner` persistence** adds run record lifecycle around the existing loop logic. Before the loop: create an `AiAgentRun` record with status started, capture its ID. At each tool dispatch: create an `AiAgentToolCall` record. After the loop: update the run record with end time, final message, token totals, and completion status. All persistence uses `GenericDelegator` directly.

**Done when:** After any live `agentRun` call, querying `AiAgentRun` and `AiAgentToolCall` returns populated records. `AgentRunnerTest` runs entirely offline — no API key, no network — and verifies loop dispatch logic, tool result truncation at the character cap, and the max-iterations stop condition. All Phase 1 tests still pass.

---

### Phase 3 — Security and Cost

This phase adds two production-readiness concerns that must be in place before the framework is used in anything beyond developer testing: preventing unauthorized tool calls, and giving operators visibility into token spend.

**What is being built and why:**

**Permission enforcement** checks `Security.hasPermission(userLogin, requiredPermission, ...)` before every tool dispatch inside the agent loop. The `UserLogin` is already in the outer service context. If the check fails, the framework does not throw an exception — that would crash the conversation and produce a bad user experience. Instead, it writes an error JSON payload back to the LLM as a tool-role message (the same wire format as a successful tool result), and creates an `AiAgentToolCall` record with failed status for the audit trail. The LLM receives the error as data and produces a human-readable explanation: "I don't have permission to cancel orders." The permission name lives on the `ToolDescriptor`, set when the ToolCatalog parses the `required-permission` attribute.

**`AiProviderCost` entity** is a seed-data table with one row per model, holding the cost per million input tokens and per million output tokens in USD, plus an effective date. Seed data ships in `data/AiAgentSeedData.xml` with rows for common OpenAI and Anthropic models. Pricing is in a queryable entity rather than config properties because it changes over time (price updates become data changes, not code deploys) and because it needs to be joined against run data to compute costs.

**`getUsageSummary` service** queries `AiAgentRun`, applies optional filters (agent name, user, date range), sums token counts across matching runs, looks up the `AiProviderCost` row for the model used, and returns total runs, total input tokens, total output tokens, and an estimated cost in USD. The return field is named `estimatedCostUsd` to make explicit that it is derived from token counts and published per-token rates, which may not match the provider invoice exactly (batch discounts, prompt caching credits, etc.).

**Done when:** A tool with `required-permission` is blocked when the calling user lacks it; the `AiAgentToolCall` record shows failed status; the LLM reply is a graceful explanation with no exception surfaced to the caller. `getUsageSummary` returns a non-null cost for known models, null (not an error) for unknown models, and filters correctly by agent, user, and date range. All Phase 1 and Phase 2 tests still pass.

---

### Phase 4 — Conversation Memory

This phase enables multi-turn conversations. Without it, every `agentRun` call is completely stateless — the LLM has no memory of what was said before, making it useless for any interactive use case.

**What is being built and why:**

**`AiConversationThread` entity** is a named context container. A caller that wants conversation continuity passes a `threadId` to `agentRun`. The entity records which agent the thread belongs to, who owns it, when it was created, and when it was last active. A `statusId` field supports archiving: an archived thread is never loaded.

**`AiConversationMessage` entity** stores one row per exchanged message within a thread. Only user inputs and assistant final responses are stored — not the internal tool call messages that fly between `AgentRunner` and the LLM during a single turn. Tool messages are implementation detail; storing them would inflate context with noise that confuses the LLM on future turns.

**Thread loading and saving in `AgentRunner`** is conditionally activated when a `threadId` is present in the service context. Before building the initial message list, the runner loads all `AiConversationMessage` rows for the thread in chronological order and prepends them after the system prompt. After the loop completes, the new user message and the final assistant response are written as new `AiConversationMessage` rows. When no `threadId` is provided, behavior is identical to Phase 1 — nothing is loaded or written.

**Token budget enforcement** prevents the accumulated history from overflowing the LLM's context window. Before prepending history, the runner estimates token usage using a character-count heuristic and drops the oldest message pairs first until the estimate is within the configured budget. The system prompt and the current user message are always included regardless of budget pressure.

**`archiveConversationThread` and `getConversationHistory` services** let callers manage threads — archiving one so future calls with the same ID start fresh, or retrieving the message history for display in a UI.

**Done when:** A two-turn integration test passes where the second call references context established in the first, and the LLM's answer demonstrates awareness of that context. A call with no `threadId` is behaviorally identical to Phase 1.

---

### Phase 5 — Human Approval Gate

This phase adds a mandatory checkpoint before any irreversible tool executes. The agent proposes its intended actions and suspends; a human reviews and either approves or rejects before execution proceeds.

**What is being built and why:**

**`AiAgentProposal` entity** captures a suspended agent run. It stores the full conversation state at the point of suspension (the message list, serialized as JSON), who triggered the run, the current review status (pending, approved, rejected), who reviewed it and when, and the rejection reason if applicable. Serializing conversation state to the database — rather than holding it in memory — is essential for correctness in a clustered deployment and ensures the proposal survives a server restart.

**`AiAgentProposalTool` entity** records each individual tool call in the proposal: the tool name, the LLM-assigned call identifier, and the arguments the LLM intended to pass. This gives the reviewer a human-readable description of exactly what the agent wants to do before they approve.

**Loop suspension in `AgentRunner`** detects when any tool in the current `tool_calls` response requires approval — either because the tool itself is declared `requires-approval="true"` in its XML, or because the `agentRun` call was invoked with `approvalRequired=true`. On detection, the runner creates the `AiAgentProposal` and `AiAgentProposalTool` records, and returns immediately from `agentRun` with the proposal ID and a summary of proposed actions. No tool has executed at this point.

**`approveAgentProposal` service** loads the proposal, verifies it is still pending (a second approval attempt is rejected, preventing double-execution), deserializes the conversation state, executes each proposed tool call via `LocalDispatcher.runSync()`, appends the results to the message list, and resumes the agent loop to completion. The final assistant message and updated `AiAgentRun` record are returned to the approver.

**`rejectAgentProposal` service** loads the proposal, marks it rejected, appends the rejection reason to the message list as a tool-role message, calls the LLM once for an acknowledgment response, and returns the acknowledgment to the caller.

**Done when:** End-to-end test: `agentRun` with `approvalRequired=true` returns a `proposalId` with zero tools executed; `approveAgentProposal` executes tools and returns the final answer; `rejectAgentProposal` returns a graceful LLM acknowledgment; a second `approveAgentProposal` call on the same proposal is rejected with an error.

---

### Phase 6 — Advanced Capabilities

Roadmap items for future sprints. Each is independent of the others; they can be sequenced in any order based on priority.

- **Prompt caching** — mark the system prompt and tool schema prefix as cacheable in the HTTP request body for providers that support it (Anthropic's `cache_control` field; OpenAI's automatic prefix caching). Significantly reduces cost for high-volume agents where the same prefix repeats across calls.
- **Model fallback** — on provider rate-limit (429) or service-unavailable (503) responses, retry once with a configured fallback model before surfacing an error to the caller. The actual model used is recorded on `AiAgentRun`.
- **Agent composition** — declare another agent as a tool in `*.tools.xml`. The framework invokes `agentRun` recursively rather than a regular service. Cycle detection enforced at startup; delegation chains longer than a configurable depth are rejected.
- **Structured output** — accept an optional JSON Schema in the `agentRun` call and return a parsed map rather than a plain string. Useful for classification, extraction, and scoring use cases.
- **Admin screens** — OFBiz screens for run history, tool call drill-down, estimated cost dashboard, pending proposal review, and conversation thread explorer. All built on the entities added in Phases 2–5.
- **Batch runs** — process a list of inputs through an agent in one service call, each input producing an independent `AiAgentRun` record. Concurrency controlled via a configurable property.
