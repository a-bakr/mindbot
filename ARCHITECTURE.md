# Architecture

This document maps the full architecture of the tldraw agent app so you can understand how the pieces fit together and where to extend them.

---

## 1. System Overview

Three main layers: the browser client, a Cloudflare Worker backend, and external LLM APIs.

```mermaid
graph TB
    subgraph Browser["Browser (React + tldraw)"]
        UI["UI Layer\nChatPanel · TodoList · Highlights"]
        AgentApp["TldrawAgentApp\n(app coordinator)"]
        Agent["TldrawAgent\n(per-conversation brain)"]
        Parts["Prompt Part Utils\n(assemble prompt)"]
        Actions["Action Utils\n(apply actions to canvas)"]
        Modes["Mode System\n(idling / working)"]
    end

    subgraph Worker["Cloudflare Worker"]
        Router["worker.ts\n(itty-router)"]
        StreamRoute["routes/stream.ts"]
        DO["AgentDurableObject\n(Durable Object)"]
        Service["AgentService\n(LLM caller)"]
        PromptBuilder["prompt/\nbuildSystemPrompt · buildMessages"]
    end

    subgraph LLMs["LLM APIs"]
        Anthropic["Anthropic\nClaude"]
        Google["Google\nGemini"]
        OpenAI["OpenAI\nGPT"]
    end

    UI -->|user input| Agent
    Agent -->|getPart()| Parts
    Parts -->|AgentPrompt JSON| Router
    Router --> StreamRoute
    StreamRoute -->|stub name 'anonymous'| DO
    DO --> Service
    Service --> PromptBuilder
    PromptBuilder -->|ModelMessage[]| Service
    Service -->|streamText| Anthropic & Google & OpenAI
    Service -->|Streaming AgentAction SSE| DO
    DO -->|SSE text/event-stream| Browser
    Agent -->|applyAction()| Actions
    Actions -->|tldraw editor calls| UI
    Modes -->|controls available parts & actions| Agent
```

---

## 2. Client Layer

Shows the React/agent class hierarchy and how all managers, parts, and actions hang together.

```mermaid
graph TB
    subgraph ReactTree["React Tree"]
        AppTsx["App.tsx\n(root component)"]
        TldrawCanvas["&lt;Tldraw&gt; Canvas"]
        Provider["TldrawAgentAppProvider\n(creates TldrawAgentApp)"]
        ChatPanel["ChatPanel\n+ ChatInput"]
        ChatHistory["ChatHistory\n(history items + diffs)"]
        TodoList["TodoList"]
        Highlights["Canvas Overlays\nAgentViewportBoundsHighlights\nContextHighlights"]
        CustomTools["Custom Tools\nTargetAreaTool · TargetShapeTool"]
    end

    subgraph AppClass["TldrawAgentApp (class)"]
        AgentsManager["AgentAppAgentsManager\n(creates / destroys TldrawAgent)"]
        PersistenceManager["AgentAppPersistenceManager\n(localStorage load/save)"]
    end

    subgraph AgentClass["TldrawAgent (class) — one per conversation"]
        direction TB
        M1["AgentActionManager\n(dispatch actions, update chat)"]
        M2["AgentChatManager\n(chat history atom)"]
        M3["AgentChatOriginManager\n(canvas start position)"]
        M4["AgentContextManager\n(pinned shapes / areas)"]
        M5["AgentDebugManager\n(log flags)"]
        M6["AgentLintManager\n(lint detection)"]
        M7["AgentModeManager\n(mode atom)"]
        M8["AgentModelNameManager\n(model atom)"]
        M9["AgentRequestManager\n(active/scheduled request atoms)"]
        M10["AgentTodoManager\n(todo list atom)"]
        M11["AgentUserActionTracker\n(user shape change watcher)"]
    end

    subgraph PartUtils["Prompt Part Utils (client/parts/)"]
        PU1["MessagesPartUtil"]
        PU2["ScreenshotPartUtil"]
        PU3["BlurryShapesPartUtil"]
        PU4["ContextItemsPartUtil"]
        PU5["ChatHistoryPartUtil"]
        PU6["TodoListPartUtil"]
        PU7["CanvasLintsPartUtil"]
        PU8["UserActionHistoryPartUtil"]
        PU9["SelectedShapesPartUtil"]
        PU10["PeripheralShapesPartUtil"]
        PU_etc["... 7 more"]
    end

    subgraph ActionUtils["Action Utils (client/actions/)"]
        AU1["CreateActionUtil"]
        AU2["UpdateActionUtil"]
        AU3["DeleteActionUtil"]
        AU4["MessageActionUtil"]
        AU5["ThinkActionUtil"]
        AU6["MoveActionUtil"]
        AU7["AlignActionUtil"]
        AU8["StackActionUtil"]
        AU_etc["... 16 more"]
    end

    AppTsx --> TldrawCanvas
    AppTsx --> Provider
    AppTsx --> ChatPanel
    Provider --> AppClass
    AppClass --> AgentClass
    ChatPanel --> ChatHistory
    ChatPanel --> TodoList
    TldrawCanvas --> Highlights
    TldrawCanvas --> CustomTools

    AgentClass --> PartUtils
    AgentClass --> ActionUtils
    M7 -->|mode controls available utils| M1
```

---

## 3. Shared Layer

Types, Zod schemas, and shape format converters used by both client and worker.

```mermaid
graph LR
    subgraph Shared["shared/"]
        subgraph Types["types/"]
            AgentRequest["AgentRequest\n(messages, bounds, data, source, contextItems)"]
            AgentAction["AgentAction\n(union of all action types)"]
            AgentMessage["AgentMessage"]
            Streaming["Streaming&lt;T&gt;\ncomplete: false | true + time"]
            ContextItem["ContextItem\n(shape | area | point)"]
            TodoItem["TodoItem"]
            ChatHistoryItem["ChatHistoryItem"]
        end

        subgraph Schema["schema/"]
            ActionSchemas["AgentActionSchemas.ts\n(Zod schemas for all 24 actions)"]
            SchemaRegistry["AgentActionSchemaRegistry.ts\n(lookup by _type string)"]
            PartDefs["PromptPartDefinitions.ts\n(priority + buildContent per part)"]
            BuildResponse["buildResponseSchema.ts\n(JSON schema sent to model)"]
        end

        subgraph Format["format/"]
            FocusedShape["FocusedShape\n(rich shape for model)"]
            BlurryShape["BlurryShape\n(overview shape)"]
            PeripheralCluster["PeripheralShapesCluster\n(off-screen clusters)"]
            Converters["convertTldrawShape ↔ FocusedShape\nconvertTldrawShape → BlurryShape\nconvertTldrawShapes → PeripheralShapes"]
        end

        Models["models.ts\n5 model definitions\n(Anthropic · Google · OpenAI)"]
    end

    ActionSchemas --> SchemaRegistry
    ActionSchemas --> BuildResponse
    ActionSchemas --> AgentAction
    PartDefs --> AgentRequest
    FocusedShape & BlurryShape & PeripheralCluster --> Converters
```

---

## 4. Worker Layer

How a request becomes LLM messages and streams back as actions.

```mermaid
graph TB
    subgraph Worker["worker/ (Cloudflare Worker)"]
        Entry["worker.ts\nWorkerEntrypoint + itty-router\nPOST /stream"]
        StreamRoute["routes/stream.ts\nresolve DO by name 'anonymous'\nforward body + pipe SSE back"]

        subgraph DO["AgentDurableObject (Durable Object)"]
            DOStream["stream()\nparse AgentPrompt\nencode Streaming&lt;AgentAction&gt; as SSE frames"]
            Service["AgentService\nstreamActions()"]
        end

        subgraph PromptLayer["prompt/"]
            BuildSys["buildSystemPrompt()\ngetSystemPromptFlags()\nintro-section · rules-section\n+ JSON schema of available actions"]
            BuildMsg["buildMessages()\niterates PromptPart[]\ncalls buildContent / buildMessages per part\nsorts by priority\nreturns ModelMessage[]"]
            ModelName["getModelName()\nreads prompt.modelName"]
        end

        Parser["closeAndParseJson()\ncloses open braces/brackets\nenables partial JSON parse\nduring streaming"]
    end

    subgraph Vercel["Vercel AI SDK (ai package)"]
        StreamText["streamText()\nmaxOutputTokens: 8192\ntemperature: 0\npre-filled assistant turn"]
    end

    Entry --> StreamRoute --> DOStream --> Service
    Service --> BuildSys & BuildMsg & ModelName
    BuildSys & BuildMsg --> StreamText
    StreamText -->|token chunks| Parser
    Parser -->|Streaming&lt;AgentAction&gt;| DOStream
```

---

## 5. End-to-End Request Data Flow

Sequence of a single user prompt through the entire system.

```mermaid
sequenceDiagram
    actor User
    participant Chat as ChatPanel
    participant Agent as TldrawAgent
    participant Parts as PromptPartUtils
    participant Worker as Cloudflare Worker
    participant DO as AgentDurableObject
    participant LLM as LLM API

    User->>Chat: types message, hits send
    Chat->>Agent: agent.prompt("Draw a snowman")

    Note over Agent: ModeChart: idling.onPromptStart → setMode('working')

    Agent->>Parts: preparePrompt(request)
    Parts->>Parts: screenshots, blurry shapes, context items,<br/>chat history, todos, lints, user actions...
    Parts-->>Agent: AgentPrompt (flat object of typed parts)

    Agent->>Worker: POST /stream (AgentPrompt JSON)
    Worker->>DO: AGENT_DURABLE_OBJECT.fetch('/stream')
    DO->>DO: buildSystemPrompt(prompt)
    DO->>DO: buildMessages(prompt) sorted by priority
    DO->>LLM: streamText(model, messages)

    loop token chunks arriving
        LLM-->>DO: raw token
        DO->>DO: closeAndParseJson(buffer)
        DO-->>Agent: SSE: Streaming<AgentAction> {complete:false}
        Agent->>Agent: sanitizeAction()
        Agent->>Agent: applyAction() → speculative shape on canvas (locked)
    end

    LLM-->>DO: action complete
    DO-->>Agent: SSE: Streaming<AgentAction> {complete:true}
    Agent->>Agent: revert speculative diff, reapply final action
    Agent->>Agent: update chat history, run lint detection

    Note over Agent: ModeChart: working.onPromptEnd

    alt incomplete todos OR unsurfaced lints
        Agent->>Agent: agent.schedule(continuation)
        Agent->>Parts: preparePrompt (next loop iteration)
    else all done
        Agent->>Agent: setMode('idling')
    end
```

---

## 6. Mode State Machine

How the agent transitions between modes and what hooks fire.

```mermaid
stateDiagram-v2
    [*] --> idling : app starts

    idling --> working : onPromptStart\n(user sends a message)

    working --> working : onPromptEnd\n[incomplete todos OR unsurfaced lints]\nautomatically schedules next request

    working --> idling : onPromptEnd\n[all todos done, lints clear]

    working --> idling : onPromptCancel\n(user cancels)

    note right of idling
        active: false
        Resets todos
        Clears user action history
    end note

    note right of working
        active: true
        17 prompt parts available
        24 action types available
        Shapes created by agent are locked during streaming
        Locked shapes released on onExit
    end note
```

---

## 7. How to Extend

Use this table to know which file(s) to touch for each type of extension.

| Goal | File(s) to edit |
|---|---|
| Add a new **action** the agent can take | `shared/schema/AgentActionSchemas.ts` (schema) + `client/actions/YourActionUtil.ts` (behavior) + `client/modes/AgentModeDefinitions.ts` (enable in mode) |
| Add new **information** the agent can see | `shared/schema/PromptPartDefinitions.ts` (server render) + `client/parts/YourPartUtil.ts` (data collection) + `client/modes/AgentModeDefinitions.ts` (enable in mode) |
| Add a new **mode** with different capabilities | `client/modes/AgentModeDefinitions.ts` (definition) + `client/modes/AgentModeChart.ts` (lifecycle hooks) |
| Change **system prompt** wording or rules | `worker/prompt/buildSystemPrompt.ts` + `worker/prompt/sections/` |
| Add a new **LLM model** | `shared/models.ts` + `worker/do/AgentService.ts` |
| Add a **custom tldraw shape** the agent can create | `shared/format/FocusedShape.ts` + `shared/format/convertTldrawShapeToFocusedShape.ts` + `shared/format/convertFocusedShapeToTldrawShape.ts` |
| Add agent-side **state** (new manager) | `client/agent/managers/YourManager.ts` (extend `BaseAgentManager`) + `client/agent/TldrawAgent.ts` (instantiate) |
| Add app-level **state** (new app manager) | extend `BaseAgentAppManager` + add to `TldrawAgentApp.ts` |
| Add a new **canvas tool** | `client/tools/YourTool.tsx` + register in `App.tsx` `tools` array |
| Persist per-user DO state | change the stub name in `worker/routes/stream.ts` from `'anonymous'` to a user ID |
| Change how actions **stream/speculate** | `client/agent/TldrawAgent.ts` → `requestAgentActions()` |
