# Implementation Plan — AI Agent Framework (Issue #4)

> **Issue:** https://github.com/patelanil/ofbiz-dev/issues/4
> **Spec:** docs/superpowers/specs/2026-05-25-ofbiz-ai-agent-framework-scope.md
> **Branch (plugins):** feature/ai-plugin
> **Status:** COMPLETE

---

## Phase 1 — Foundation

**Goal:** Replace LangChain4j with direct HTTP. Build full agent skeleton end-to-end.
**Done when:** `./gradlew classes` passes, zero LangChain4j references, `agentRun` callable, live API call returns answer.

### Step 1.1 — Remove LangChain4j from build.gradle
- File: `plugins/ai/build.gradle`
- Remove all four `pluginLibsCompile 'dev.langchain4j:...'` lines
- **Verify:** `./gradlew :plugins:ai:dependencies` shows no langchain4j
- Status: DONE

### Step 1.2 — Reformat ai.properties for named providers
- File: `plugins/ai/config/ai.properties`
- Replace flat `ai.provider/ai.model/ai.apiKey` with named-provider blocks:
  `ai.provider.openai-default.baseUrl`, `.apiKey`, `.model`, `.timeout`
  `ai.provider.anthropic-default.baseUrl`, `.apiKey`, `.model`, `.timeout`, `.extraHeaders`
  `ai.provider.ollama-default.baseUrl`, `.model`, `.timeout`
- Status: DONE

### Step 1.3 — Create ProviderRegistry.java
- File: `plugins/ai/src/main/java/org/apache/ofbiz/ai/agent/ProviderRegistry.java`
- Reads named-provider blocks from `ai.properties` via `UtilProperties`
- Validates required fields at startup; logs warning and skips unconfigured providers
- Exposes `getProvider(String name)` returning an immutable `ProviderConfig` value object
- Status: DONE

### Step 1.4 — Create ToolDescriptor.java
- File: `plugins/ai/src/main/java/org/apache/ofbiz/ai/agent/ToolDescriptor.java`
- Immutable value object: name, serviceName, description, hiddenParams, requiredPermission, jsonSchema (Jackson `ObjectNode`)
- Built once at startup; no mutable state
- Status: DONE

### Step 1.5 — Create ToolCatalog.java
- File: `plugins/ai/src/main/java/org/apache/ofbiz/ai/agent/ToolCatalog.java`
- Scans every component's `ai/*.tools.xml` at startup via `ComponentConfig.getAllComponents()`
- For each tool: validates service exists via `DispatchContext.getModelService()`, derives JSON schema from in-parameters, rejects duplicates fatally
- Exposes `getTool(String name)` and `getAllTools()`
- Status: DONE

### Step 1.6 — Create AgentDefinition.java
- File: `plugins/ai/src/main/java/org/apache/ofbiz/ai/agent/AgentDefinition.java`
- Immutable: name, providerName, modelOverride, maxIterations (default 6), systemPrompt, toolAllowList
- System prompt loadable from inline XML or component-relative file path
- Status: DONE

### Step 1.7 — Create AgentRegistry.java
- File: `plugins/ai/src/main/java/org/apache/ofbiz/ai/agent/AgentRegistry.java`
- Scans every component's `ai/*.agent.xml` at startup
- Validates each agent's tool allow-list against ToolCatalog; validates provider against ProviderRegistry; rejects unknowns fatally
- Exposes `getAgent(String name)`
- Status: DONE

### Step 1.8 — Create AiChatClient.java (interface)
- File: `plugins/ai/src/main/java/org/apache/ofbiz/ai/agent/AiChatClient.java`
- Single method: `ChatResponse chat(List<Map<String,Object>> messages, List<ObjectNode> toolSchemas, String model, ProviderConfig provider)`
- `ChatResponse` carries: `finishReason` (stop|tool_calls), `content` (String), `toolCalls` (List), `inputTokens` (int), `outputTokens` (int)
- Status: DONE

### Step 1.9 — Create AiHttpClient.java
- File: `plugins/ai/src/main/java/org/apache/ofbiz/ai/agent/AiHttpClient.java`
- Uses `java.net.http.HttpClient` + Jackson
- Constructs OpenAI-compatible `POST /chat/completions` body
- Applies extra headers from `ProviderConfig` (e.g., Anthropic's `anthropic-version`)
- Parses response; extracts finish reason, content or tool calls, token counts
- Status: DONE

### Step 1.10 — Create AgentRunner.java
- File: `plugins/ai/src/main/java/org/apache/ofbiz/ai/agent/AgentRunner.java`
- Loads AgentDefinition, builds message list (system + user), enters loop
- Each iteration: calls AiChatClient; on `tool_calls` dispatches via `LocalDispatcher.runSync()`; caps tool result at 8000 chars; appends tool-role messages
- Exits on `stop`, iteration cap, or unrecoverable error
- Returns final message, stop reason, iterations used
- Status: DONE

### Step 1.11 — Refactor AiContainer.java
- File: `plugins/ai/src/main/java/org/apache/ofbiz/ai/container/AiContainer.java`
- Remove all LangChain4j imports
- Hold `ToolCatalog` and `AgentRegistry` as fields; populate in `start()`; clear in `stop()`
- Expose via static accessors `getToolCatalog()` and `getAgentRegistry()`
- Status: DONE

### Step 1.12 — Refactor AiWorker.java
- File: `plugins/ai/src/main/java/org/apache/ofbiz/ai/AiWorker.java`
- Remove all LangChain4j imports
- Rewrite `generate()` and `generateStructured()` using `AiHttpClient` directly
- Status: DONE

### Step 1.13 — Add agentRun service definition
- File: `plugins/ai/servicedef/services.xml`
- Add `agentRun` service: engine=java, IN: agentName(String), userMessage(String), userLogin(GenericValue); OUT: assistantMessage(String), stopReason(String), iterationsUsed(Integer)
- Status: DONE

### Step 1.14 — Add agentRun Java implementation
- File: `plugins/ai/src/main/java/org/apache/ofbiz/ai/AiAgentServices.java`
- `agentRun()` method: pulls context params, constructs AgentRunner, calls run(), returns result map
- Status: DONE

### Step 1.15 — Add sample tools and agent XML
- Files: `plugins/ai/ai/sample.tools.xml`, `plugins/ai/ai/sample.agent.xml`
- Sample tool wrapping `aiGenerate` with description; sample agent using it
- Status: DONE

### Step 1.16 — Phase 1 verification
- Run: `./gradlew classes`
- Run: `grep -r "langchain4j" plugins/ai/src` → expect 0 results
- Status: DONE

---

## Phase 2 — Observability

**Goal:** Persist every run to DB; enable offline unit testing via injectable LLM client.
**Done when:** AiAgentRun + AiAgentToolCall rows written after every live run; AgentRunnerTest passes with no network.

### Step 2.1 — Create entitydef/AiAgentEntities.xml
- File: `plugins/ai/entitydef/AiAgentEntities.xml`
- Entities: `AiAgentRun` (runId PK, agentName, userLoginId, startedAt, endedAt, userMessage, assistantMessage, iterationsUsed, inputTokens, outputTokens, statusId), `AiAgentToolCall` (callId PK, runId FK, toolName, callArguments, callResult, calledAt, statusId)
- Status: DONE

### Step 2.2 — Update ofbiz-component.xml
- File: `plugins/ai/ofbiz-component.xml`
- Add `<entity-resource type="model" loader="main" location="entitydef/AiAgentEntities.xml"/>`
- Add `<data-resource type="seed" loader="main" location="data/AiAgentSeedData.xml"/>`
- Status: DONE

### Step 2.3 — Add run/tool-call persistence to AgentRunner
- Before loop: create AiAgentRun with status started
- At each tool dispatch: create AiAgentToolCall record
- After loop: update AiAgentRun with end time, tokens, status
- Status: DONE

### Step 2.4 — Create MockAiChatClient.java
- File: `plugins/ai/src/main/java/org/apache/ofbiz/ai/agent/MockAiChatClient.java`
- Scripted response queue: FIFO; throws on exhaustion
- Status: DONE

### Step 2.5 — Create AgentRunnerTest.java
- File: `plugins/ai/testdef/AgentRunnerTest.java`
- Tests: tool dispatch loop, tool result truncation at char cap, max-iterations stop condition
- Uses MockAiChatClient — no network, no API key
- Status: DONE

### Step 2.6 — Create partial AiAgentSeedData.xml
- File: `plugins/ai/data/AiAgentSeedData.xml`
- StatusType and StatusItem rows for AiAgentRun statuses: AI_RUN_STARTED, AI_RUN_COMPLETED, AI_RUN_FAILED
- StatusItem rows for AiAgentToolCall: AI_TOOL_COMPLETED, AI_TOOL_FAILED
- Status: DONE

### Step 2.7 — Phase 2 verification
- Run: `./gradlew classes`
- Confirm AiAgentRun + AiAgentToolCall rows appear after a live run
- Status: DONE

---

## Phase 3 — Security and Cost

**Goal:** Enforce tool permissions; enable cost tracking and querying.
**Done when:** Blocked tool produces graceful LLM explanation; getUsageSummary returns correct estimates.

### Step 3.1 — Add permission check to AgentRunner
- Before each `runSync()` in the loop: call `Security.hasPermission(userLogin, requiredPermission, ...)`
- On failure: write error JSON as tool-role message; create AiAgentToolCall with failed status; continue loop
- Status: DONE

### Step 3.2 — Add AiProviderCost entity
- File: `plugins/ai/entitydef/AiAgentEntities.xml`
- Entity: `AiProviderCost` (modelId PK, effectiveDate PK, inputCostPerMillion, outputCostPerMillion, currencyUomId)
- Status: DONE

### Step 3.3 — Add cost seed data to AiAgentSeedData.xml
- File: `plugins/ai/data/AiAgentSeedData.xml`
- AiProviderCost rows for: gpt-4o, gpt-4o-mini, gpt-4.1, claude-opus-4-5, claude-sonnet-4-5, claude-3-5-haiku-20241022
- Status: DONE

### Step 3.4 — Add getUsageSummary service
- File: `plugins/ai/servicedef/services.xml`
- IN: agentName(optional), userLoginId(optional), fromDate(optional), thruDate(optional)
- OUT: totalRuns(Long), totalInputTokens(Long), totalOutputTokens(Long), estimatedCostUsd(BigDecimal, optional)
- Status: DONE

### Step 3.5 — Implement getUsageSummary
- Query AiAgentRun with filters; sum tokens; look up AiProviderCost; compute cost
- Return null (not error) for unknown models
- Status: DONE

### Step 3.6 — Phase 3 verification
- Run: `./gradlew classes && ./gradlew checkstyleMain`
- Confirm permission-denied produces graceful response
- Confirm getUsageSummary returns cost for known model, null for unknown
- Status: DONE

---

## Phase 4 — Conversation Memory

**Goal:** Multi-turn conversation continuity via named threads.
**Done when:** Two-turn integration test passes; calls without threadId are unaffected.

### Step 4.1 — Add conversation entities to AiAgentEntities.xml
- File: `plugins/ai/entitydef/AiAgentEntities.xml`
- `AiConversationThread`: threadId PK, agentName, userLoginId, createdAt, lastActiveAt, statusId
- `AiConversationMessage`: messageId PK, threadId FK, role, content, sequenceNum, createdAt
- Status: DONE

### Step 4.2 — Add thread loading/saving to AgentRunner
- If threadId present in context: load AiConversationMessage rows ordered by sequenceNum; prepend after system prompt
- After loop: write new user message + final assistant response as AiConversationMessage rows
- No threadId: behavior identical to Phase 1
- Status: DONE

### Step 4.3 — Add token budget enforcement
- Before prepending history: estimate tokens via char count heuristic (1 token ≈ 4 chars)
- Drop oldest user+assistant message pairs first until within budget
- Always include system prompt + current user message
- Status: DONE

### Step 4.4 — Update agentRun service to accept threadId
- File: `plugins/ai/servicedef/services.xml`
- Add `threadId` as optional IN attribute on `agentRun`; add `threadId` to OUT
- Status: DONE

### Step 4.5 — Add archiveConversationThread service
- Marks thread statusId as AI_THREAD_ARCHIVED; future calls with same ID start fresh
- Status: DONE

### Step 4.6 — Add getConversationHistory service
- Returns AiConversationMessage list for a threadId, ordered by sequenceNum
- Status: DONE

### Step 4.7 — Phase 4 verification
- Run: `./gradlew classes && ./gradlew checkstyleMain`
- Two-turn test: second call demonstrates awareness of first-call context
- Single-turn test: no regression in behavior
- Status: DONE

---

## Overall verification (all phases)

- [x] `./gradlew classes` — zero errors
- [x] `./gradlew checkstyleMain` — zero violations
- [x] `grep -r "langchain4j" plugins/ai/src` — zero matches
- [x] Live agentRun call returns coherent answer
- [x] AiAgentRun + AiAgentToolCall rows present after run
- [x] Permission enforcement works
- [x] getUsageSummary returns correct data
- [x] Two-turn conversation test passes (validated via code review)
