# Agentic Framework Architecture Directory

> A comprehensive analysis of leading agent orchestration frameworks to inform Erlang/OTP-based agent design.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Framework Deep Dives](#framework-deep-dives)
   - [LangChain / LangGraph](#1-langchain--langgraph)
   - [Microsoft AutoGen](#2-microsoft-autogen)
   - [CrewAI](#3-crewai)
   - [OpenAI Swarm / Agents SDK](#4-openai-swarm--agents-sdk)
   - [AutoGPT / Autonomous Agents](#5-autogpt--autonomous-agents)
3. [Actor-Model Frameworks](#actor-model-frameworks)
   - [Akka (JVM)](#akka-jvm)
   - [Microsoft Orleans (.NET)](#microsoft-orleans-net)
   - [Ray (Python)](#ray-python)
   - [Dapr (Language-Agnostic)](#dapr-language-agnostic)
4. [Cross-Framework Pattern Analysis](#cross-framework-pattern-analysis)
5. [Mapping to Erlang/OTP Concepts](#mapping-to-erlangotp-concepts)
6. [Architecture Toolbox Summary](#architecture-toolbox-summary)

---

## Executive Summary

This directory analyzes six major agentic frameworks and four actor-model systems to extract architectural patterns, identify common threads, and inform the design of an Erlang/OTP-based agent orchestration system.

### Key Findings

| Dimension | Industry Consensus | Erlang/OTP Advantage |
|-----------|-------------------|----------------------|
| **Concurrency** | Async/await, thread pools | Lightweight processes (millions) |
| **Fault Tolerance** | Retry policies, fallbacks | Supervision trees, "let it crash" |
| **State Management** | External stores, checkpointing | Built-in ETS/Mnesia, process state |
| **Communication** | Message passing, pub/sub | Native message passing, pg groups |
| **Distribution** | gRPC, HTTP, custom protocols | Native distribution, location transparency |

---

## Framework Deep Dives

### 1. LangChain / LangGraph

**Philosophy**: Composable chains with graph-based orchestration for complex workflows.

#### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        LangGraph                                 │
├─────────────────────────────────────────────────────────────────┤
│  StateGraph (Builder) ──compile()──▶ CompiledStateGraph        │
│                                            │                    │
│                                      Pregel Runtime             │
│                                    (BSP Supersteps)             │
├─────────────────────────────────────────────────────────────────┤
│                        LangChain Core                           │
├──────────────┬──────────────┬──────────────┬───────────────────┤
│   Runnables  │    Chains    │    Agents    │    Retrievers     │
│   (LCEL)     │              │              │                   │
└──────────────┴──────────────┴──────────────┴───────────────────┘
```

#### Salient Features

| Feature | Description | OTP Parallel |
|---------|-------------|--------------|
| **LCEL (Expression Language)** | Declarative composition via pipe operator | Erlang behaviours + pipelines |
| **Pregel/BSP Execution** | Superstep-based parallel execution | gen_statem state transitions |
| **Reducer-Driven State** | Custom reducers per state key | Process dictionaries, ETS |
| **Checkpointing** | Thread-based state persistence | Mnesia transactions |
| **Interruptible Execution** | Human-in-the-loop via interrupts | Selective receive, monitors |

#### State Management (Reducers)

```python
# LangGraph: Annotated state with custom reducers
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]  # Accumulates
    context: str                              # Overwrites (default)
```

**Key Insight**: State channels with explicit merge semantics. Maps to Erlang's process mailbox + ETS for shared state.

#### Orchestration Patterns

1. **Supervisor Pattern**: Central router delegates to specialists
2. **Peer-to-Peer**: Direct agent communication
3. **Pipeline**: Sequential handoffs with context passing

#### Memory Architecture

| Type | Backend | Scope |
|------|---------|-------|
| Short-term | In-memory | Thread/conversation |
| Long-term | PostgresStore | Cross-thread |
| Semantic | pgvector | Similarity search |

---

### 2. Microsoft AutoGen

**Philosophy**: Actor-model-based multi-agent framework with layered architecture.

#### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         AgentChat                                │
│          (High-level teams: RoundRobin, Swarm, GraphFlow)       │
├─────────────────────────────────────────────────────────────────┤
│                          Core API                                │
│              (Actor Model + Typed Message Passing)               │
├─────────────────────────────────────────────────────────────────┤
│                       Extensions API                             │
│            (LLM clients, code executors, plugins)                │
└─────────────────────────────────────────────────────────────────┘
```

#### Salient Features

| Feature | Description | OTP Parallel |
|---------|-------------|--------------|
| **Actor Model Core** | Independent agents, async messages | Native Erlang processes |
| **Factory Registration** | Runtime creates agents on first message | Dynamic child specs |
| **Topic-Based Pub/Sub** | Indirection for broadcast | pg (process groups) |
| **Typed Messages** | Schema-validated communication | Records, type specs |
| **AgentId Addressing** | Type + Key identification | Registered names, pids |

#### Agent Lifecycle

```python
# AutoGen: Factory-based agent registration
await MyAgent.register(
    runtime,
    "my_agent",           # Type string
    lambda: MyAgent()     # Factory function
)
# Agents created lazily on first message delivery
```

**Key Insight**: Lazy instantiation with factory pattern. Maps to Erlang's `simple_one_for_one` supervisor with dynamic children.

#### Team Orchestration

| Team Type | Mechanism | Erlang Equivalent |
|-----------|-----------|-------------------|
| RoundRobinGroupChat | Fixed order cycling | Round-robin process pool |
| SelectorGroupChat | LLM/function selects next | Dynamic routing gen_server |
| Swarm | HandoffMessage delegation | Process handoff via message |
| GraphFlow | Directed graph edges | State machine transitions |

#### Communication Types

```
Direct (RPC-style)     ───▶  call/cast to specific pid
Broadcast (Pub/Sub)    ───▶  pg:send/2 to process group
Handoff                ───▶  Transfer ownership + state
```

---

### 3. CrewAI

**Philosophy**: Role-based agent orchestration with Flows + Crews architecture.

#### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                          Flows                                   │
│        (Event-driven backbone, state management)                 │
├─────────────────────────────────────────────────────────────────┤
│                          Crews                                   │
│              (Teams of role-based agents)                        │
├─────────────┬─────────────┬─────────────┬───────────────────────┤
│   Agents    │    Tasks    │   Tools     │      Memory           │
│  (Roles)    │  (Work)     │  (Actions)  │   (Context)           │
└─────────────┴─────────────┴─────────────┴───────────────────────┘
```

#### Salient Features

| Feature | Description | OTP Parallel |
|---------|-------------|--------------|
| **Role-Based Agents** | Persona defines behavior | Parameterized gen_server |
| **ReAct Framework** | Thought-Action-Observation loop | gen_statem with explicit states |
| **Process Types** | Sequential vs Hierarchical | Supervision strategies |
| **Flows + Crews** | Scaffold + workers separation | Application + supervision tree |
| **Delegation** | Agents delegate to specialists | Message forwarding |

#### ReAct Implementation

```
┌──────────────────────────────────────────────────────────────┐
│  THOUGHT: LLM determines action needed                       │
│     │                                                        │
│     ▼                                                        │
│  ACTION: Tool selection + input specification                │
│     │                                                        │
│     ▼                                                        │
│  EXECUTION: CrewAI parses output, calls Python function      │
│     │                                                        │
│     ▼                                                        │
│  OBSERVATION: Tool return fed back into prompt               │
│     │                                                        │
│     ▼                                                        │
│  LOOP: Until Final Answer produced                           │
└──────────────────────────────────────────────────────────────┘
```

#### Process Types

| Process | Description | Erlang Mapping |
|---------|-------------|----------------|
| **Sequential** | Tasks execute in order, output → context | Chain of gen_server calls |
| **Hierarchical** | Manager delegates, validates | Supervisor with children |
| **Consensual** | Democratic decision-making | Quorum-based messaging |

#### Memory Architecture

| Memory Type | Backend | Purpose |
|-------------|---------|---------|
| Short-Term | ChromaDB (RAG) | Session context |
| Long-Term | SQLite3 | Cross-session persistence |
| Entity | ChromaDB (RAG) | Named entity knowledge |
| External | Custom | Cross-application sharing |

---

### 4. OpenAI Swarm / Agents SDK

**Philosophy**: Lightweight, stateless agent coordination with minimal abstractions.

#### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     Agents SDK                                   │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │  Agents   │  │ Handoffs  │  │Guardrails │  │ Sessions  │    │
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘    │
├─────────────────────────────────────────────────────────────────┤
│                    Runner (Agent Loop)                           │
│        LLM Call → Tool Execution → Handoff → Repeat             │
├─────────────────────────────────────────────────────────────────┤
│                   OpenAI Chat Completions API                    │
└─────────────────────────────────────────────────────────────────┘
```

#### Salient Features

| Feature | Description | OTP Parallel |
|---------|-------------|--------------|
| **Stateless Runs** | No state between calls | Functional message handling |
| **Handoff Pattern** | Agent → Agent control transfer | Process message forwarding |
| **Context Variables** | Explicit state threading | Process state in gen_server |
| **Guardrails** | Input/output validation | Guards, pattern matching |
| **Built-in Tracing** | Hierarchical spans | OTP SASL, observer |

#### The Handoff Pattern

```
┌─────────────┐    handoff     ┌─────────────┐    handoff     ┌─────────────┐
│   Triage    │ ─────────────▶ │   Sales     │ ─────────────▶ │  Support    │
│   Agent     │                │   Agent     │                │   Agent     │
└─────────────┘                └─────────────┘                └─────────────┘
       │                              │                              │
       └──────────────────────────────┴──────────────────────────────┘
                    Conversation History Preserved
```

**Key Insight**: Handoffs preserve full conversation context. Maps to Erlang process state transfer or shared ETS table.

#### Runner Execution Modes

| Mode | Method | Use Case |
|------|--------|----------|
| Async | `Runner.run()` | FastAPI, notebooks |
| Sync | `Runner.run_sync()` | Simple scripts |
| Streaming | `Runner.run_streamed()` | Real-time feedback |

#### Design Philosophy

- **Minimalism**: Two primitives (Agent, Handoff)
- **Statelessness**: Client-side execution, no server state
- **Transparency**: Traceable execution flow
- **Specialization**: Many small agents > one large agent

---

### 5. AutoGPT / Autonomous Agents

**Philosophy**: Goal-driven autonomous loops with self-correction.

#### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Goal / Objective                              │
├─────────────────────────────────────────────────────────────────┤
│                   Task Decomposition                             │
│              (Planning Agent / Chain of Thought)                 │
├─────────────────────────────────────────────────────────────────┤
│                    Execution Loop                                │
│     ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐   │
│     │  Plan   │───▶│Criticize│───▶│   Act   │───▶│ Observe │   │
│     └─────────┘    └─────────┘    └─────────┘    └─────────┘   │
│          ▲                                              │        │
│          └──────────────────────────────────────────────┘        │
├─────────────────────────────────────────────────────────────────┤
│  Memory Layer    │    Tool Layer     │    Plugin System          │
└─────────────────────────────────────────────────────────────────┘
```

#### Salient Features

| Feature | Description | OTP Parallel |
|---------|-------------|--------------|
| **Autonomous Loop** | Self-directed execution | gen_server loop with self-sends |
| **Self-Critique** | Evaluate own outputs | Validation + retry logic |
| **Task Decomposition** | Break goals into subtasks | Supervisor child creation |
| **Plugin Architecture** | Extensible tool system | Behaviour callbacks |
| **Feedback Loop** | Learn from failures | Error handling + state update |

#### The Feedback Loop

```
Plan ──▶ Criticize ──▶ Act ──▶ Read Feedback ──▶ Plan (adjusted)
                         │
                         ▼
                   Observation / Error
```

#### BabyAGI Three-Agent Pattern

```
┌────────────────────────────────────────────────────────────┐
│                    Task Queue                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Task 1 │ Task 2 │ Task 3 │ ... │ Task N             │   │
│  └─────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────┘
        │                    │                    │
        ▼                    ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Execution   │    │    Task      │    │Prioritization│
│    Agent     │───▶│   Creation   │───▶│    Agent     │
│              │    │    Agent     │    │              │
└──────────────┘    └──────────────┘    └──────────────┘
```

**Key Insight**: Separation of execution, creation, and prioritization. Maps to specialized gen_servers coordinated by supervisor.

#### Memory Evolution

Industry trend: Moving away from monolithic vector databases toward:
- Task-oriented memory per agent
- Multiple specialized stores
- Cooperation-based information sharing

---

## Actor-Model Frameworks

### Akka (JVM)

**Philosophy**: Classic actor model with explicit supervision hierarchies.

#### Key Concepts

| Concept | Description | Erlang Equivalent |
|---------|-------------|-------------------|
| ActorRef | Opaque handle to actor | pid() |
| Props | Actor configuration | Child spec |
| Behavior | Message handler definition | gen_server callbacks |
| Supervision | Parent manages child failures | OTP supervisors |

#### Supervision Strategies

```
┌─────────────────────────────────────────────────────────────┐
│                    Supervisor Actor                          │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Directive on Failure:                                │   │
│   │   • Resume   - Keep state, continue processing       │   │
│   │   • Restart  - Clear state, create new instance      │   │
│   │   • Stop     - Terminate actor permanently           │   │
│   │   • Escalate - Pass failure to parent supervisor     │   │
│   └─────────────────────────────────────────────────────┘   │
│          │              │              │                     │
│          ▼              ▼              ▼                     │
│     ┌────────┐    ┌────────┐    ┌────────┐                  │
│     │Child 1 │    │Child 2 │    │Child 3 │                  │
│     └────────┘    └────────┘    └────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

#### Event Sourcing Pattern

```
Events (Journal)     Snapshots         Actor State
    │                    │                  │
    ▼                    ▼                  ▼
┌────────────┐    ┌────────────┐    ┌────────────┐
│ Event 1    │    │ Snapshot   │    │ Current    │
│ Event 2    │ +  │ @ Event    │ =  │ State      │
│ Event 3    │    │ 1000       │    │            │
│ ...        │    │            │    │            │
└────────────┘    └────────────┘    └────────────┘
```

**Applicability to Agents**: Event sourcing enables conversation replay, audit trails, and debugging.

---

### Microsoft Orleans (.NET)

**Philosophy**: Virtual actors with automatic lifecycle management.

#### Key Concepts

| Concept | Description | Erlang Equivalent |
|---------|-------------|-------------------|
| Grain | Virtual actor (always exists conceptually) | gen_server + auto-restart |
| Silo | Host process for grains | Erlang node |
| Grain Reference | Strongly-typed proxy | Registered name |
| Activation | In-memory grain instance | Running process |

#### Virtual Actor Model

```
┌─────────────────────────────────────────────────────────────┐
│                   Grain Lifecycle                            │
│                                                              │
│  Conceptually Exists Always                                  │
│         │                                                    │
│         ▼                                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ First Message Arrives → Activate → Load State       │    │
│  │                              │                       │    │
│  │                              ▼                       │    │
│  │                     Process Messages                 │    │
│  │                              │                       │    │
│  │                              ▼                       │    │
│  │              Idle Timeout → Deactivate → Save State  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Key Insight**: Agents activated on-demand, deactivated when idle. Perfect for conversational agents with variable activity.

#### Placement Strategies

| Strategy | Description | Use Case |
|----------|-------------|----------|
| Random | Any available silo | Default, simple |
| PreferLocal | Same silo as caller | Reduce latency |
| HashBased | Consistent hashing | Predictable placement |
| ResourceOptimized | Based on CPU/memory | Load balancing |

---

### Ray (Python)

**Philosophy**: Distributed computing for AI/ML with tasks and actors.

#### Key Concepts

| Concept | Description | Erlang Equivalent |
|---------|-------------|-------------------|
| Task | Stateless remote function | spawn + receive |
| Actor | Stateful remote object | gen_server |
| ObjectRef | Future/promise for result | Ref from make_ref() |
| Placement Group | Co-located resources | Node affinity |

#### Hybrid Task/Actor Model

```python
# Ray: Stateless task
@ray.remote
def process_message(msg):
    return transform(msg)

# Ray: Stateful actor
@ray.remote
class Agent:
    def __init__(self, config):
        self.state = initialize(config)

    def handle(self, msg):
        self.state = update(self.state, msg)
        return respond(self.state, msg)
```

**Applicability to Agents**: Tasks for stateless tool calls, Actors for stateful agent conversations.

#### Resource Specification

```python
@ray.remote(num_cpus=2, num_gpus=1, memory=4*1024*1024*1024)
class LLMAgent:
    pass
```

**Key Insight**: Explicit resource requirements per actor. Maps to Erlang's process limits and scheduling.

---

### Dapr (Language-Agnostic)

**Philosophy**: Building blocks for distributed applications via sidecar pattern.

#### Key Concepts

| Concept | Description | Erlang Equivalent |
|---------|-------------|-------------------|
| Sidecar | Separate process handling distribution | Distribution protocol |
| Building Block | Capability (state, pub/sub, actors) | OTP behaviour |
| Virtual Actor | Auto-managed actor lifecycle | gen_server + supervisor |
| Placement Service | Actor location management | Global registry |

#### Sidecar Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Application Pod                         │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │                 │    │         Dapr Sidecar             │ │
│  │   Your App      │◀──▶│  ┌─────┐ ┌─────┐ ┌─────────┐   │ │
│  │  (Any Language) │    │  │State│ │Pub/ │ │ Actors  │   │ │
│  │                 │    │  │Store│ │ Sub │ │         │   │ │
│  └─────────────────┘    │  └─────┘ └─────┘ └─────────┘   │ │
│                         └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Applicability to Agents**: Language-agnostic agent implementations with consistent distributed semantics.

---

## Cross-Framework Pattern Analysis

### Common Orchestration Patterns

| Pattern | LangGraph | AutoGen | CrewAI | Swarm | AutoGPT |
|---------|-----------|---------|--------|-------|---------|
| **Supervisor/Manager** | ✅ | ✅ | ✅ (Hierarchical) | ❌ | ✅ |
| **Peer-to-Peer** | ✅ | ✅ | ❌ | ✅ | ❌ |
| **Sequential Pipeline** | ✅ | ✅ (GraphFlow) | ✅ | ❌ | ✅ |
| **Handoff/Delegation** | ✅ (Command) | ✅ (HandoffMessage) | ✅ | ✅ (Primary) | ❌ |
| **Parallel Fan-Out** | ✅ (Send API) | ✅ | ❌ | ✅ (asyncio) | ❌ |

### Common State Management Approaches

| Approach | Frameworks Using It | Characteristics |
|----------|--------------------| ----------------|
| **Explicit Threading** | Swarm, LangGraph | State passed explicitly between calls |
| **Session/Checkpoint** | LangGraph, Agents SDK | Persistent conversation state |
| **Actor State** | AutoGen, Orleans, Akka | Per-actor isolated state |
| **Shared Memory** | LangGraph (reducers) | Coordinated state updates |
| **Event Sourcing** | Akka Persistence | Append-only event log |

### Common Communication Patterns

| Pattern | Description | Frameworks |
|---------|-------------|------------|
| **Direct Messaging** | Point-to-point calls | All |
| **Pub/Sub Broadcast** | Topic-based distribution | AutoGen, Dapr |
| **Request/Response** | Synchronous call semantics | All |
| **Fire-and-Forget** | Async without waiting | AutoGen, Akka |
| **Streaming** | Progressive results | LangGraph, Agents SDK |

### Common Fault Tolerance Strategies

| Strategy | Description | Frameworks |
|----------|-------------|------------|
| **Retry with Backoff** | Exponential backoff on transient failures | All |
| **Fallback Chain** | Try alternatives on failure | LangGraph, Agents SDK |
| **Supervision** | Parent handles child failures | Akka, Orleans, OTP |
| **Checkpointing** | Save progress for recovery | LangGraph, AutoGPT |
| **Circuit Breaker** | Stop cascading failures | Dapr, production systems |

---

## Mapping to Erlang/OTP Concepts

### Direct Mappings

| Agentic Concept | OTP Equivalent | Notes |
|-----------------|----------------|-------|
| Agent | gen_server / gen_statem | Stateful process with message handling |
| Supervisor Agent | supervisor | Manages child agent lifecycles |
| Handoff | Message + state transfer | Send state to new process owner |
| Tool | Function call / RPC | gen_server:call/2 to tool process |
| Memory (Short-term) | Process state | State in gen_server loop data |
| Memory (Long-term) | Mnesia / ETS | Persistent storage |
| Pub/Sub | pg (process groups) | Subscribe processes to topics |
| Checkpointing | Mnesia transactions | Durable state snapshots |
| Guardrails | Guards + pattern matching | Validate before processing |
| Tracing | SASL + observer | Built-in logging and monitoring |

### Supervision Tree Design

```
                    ┌───────────────────┐
                    │  Application      │
                    │  Supervisor       │
                    └─────────┬─────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ Agent Pool      │ │ Tool Registry   │ │ Memory Manager  │
│ Supervisor      │ │ Supervisor      │ │ Supervisor      │
│ (one_for_one)   │ │ (one_for_all)   │ │ (rest_for_one)  │
└────────┬────────┘ └────────┬────────┘ └────────┬────────┘
         │                   │                   │
    ┌────┼────┐         ┌────┼────┐         ┌────┼────┐
    │    │    │         │    │    │         │    │    │
    ▼    ▼    ▼         ▼    ▼    ▼         ▼    ▼    ▼
┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐
│Agent││Agent││Agent││Tool ││Tool ││Tool ││Short││Long ││Cache│
│  1  ││  2  ││  N  ││  1  ││  2  ││  N  ││Term ││Term ││     │
└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘
```

### gen_statem for Agent Workflows

```erlang
%% Agent states (like CrewAI's ReAct loop)
-type state() :: idle | thinking | tool_calling | awaiting_response | finished.

%% State machine callbacks
callback_mode() -> [state_functions, state_enter].

idle(enter, _OldState, Data) ->
    {keep_state, Data};
idle({call, From}, {new_task, Task}, Data) ->
    {next_state, thinking, Data#{task => Task}, [{reply, From, ok}]}.

thinking(enter, _OldState, Data) ->
    %% Invoke LLM for reasoning
    {keep_state, Data, [{state_timeout, 30000, llm_timeout}]};
thinking(cast, {llm_response, Response}, Data) ->
    case needs_tool_call(Response) of
        true -> {next_state, tool_calling, Data#{response => Response}};
        false -> {next_state, finished, Data#{result => Response}}
    end.

tool_calling(enter, _OldState, Data) ->
    %% Execute tool and return to thinking
    Tool = extract_tool(Data),
    gen_server:cast(tool_registry, {execute, Tool, self()}),
    {keep_state, Data};
tool_calling(cast, {tool_result, Result}, Data) ->
    {next_state, thinking, Data#{observation => Result}}.
```

### Process Groups for Agent Discovery

```erlang
%% Join agents to topic groups (like AutoGen pub/sub)
pg:join(agent_pool, self()),
pg:join({specialist, coding}, self()),
pg:join({specialist, research}, self()).

%% Broadcast to all coding specialists
[gen_server:cast(Pid, {task, CodingTask})
 || Pid <- pg:get_members({specialist, coding})].

%% Round-robin selection (like AutoGen RoundRobinGroupChat)
select_next(Pool) ->
    Members = pg:get_members(Pool),
    Index = persistent_term:get({pool_index, Pool}, 0),
    Next = lists:nth((Index rem length(Members)) + 1, Members),
    persistent_term:put({pool_index, Pool}, Index + 1),
    Next.
```

---

## Architecture Toolbox Summary

### Patterns to Adopt

| Pattern | Source | OTP Implementation |
|---------|--------|-------------------|
| **Supervisor Pattern** | LangGraph, AutoGen, CrewAI | OTP supervisor with children as agents |
| **Handoff** | Swarm, AutoGen | Message passing with state transfer |
| **ReAct Loop** | CrewAI, AutoGPT | gen_statem state machine |
| **Checkpointing** | LangGraph, AutoGPT | Mnesia transactions |
| **Virtual Actors** | Orleans | simple_one_for_one + lazy activation |
| **Event Sourcing** | Akka | Append-only Mnesia tables |
| **Process Groups** | AutoGen | pg module for pub/sub |
| **Typed Messages** | AutoGen | Records + type specs |

### Features to Implement

1. **Agent Registry**: Named process registration with metadata
2. **Tool Executor**: gen_server pool for tool execution
3. **Memory Manager**: ETS for short-term, Mnesia for long-term
4. **Checkpoint System**: Periodic state snapshots
5. **Tracing/Observability**: SASL + custom event logging
6. **Guardrails**: Input/output validation modules
7. **Handoff Protocol**: Standardized state transfer format

### Unique OTP Advantages

| Capability | Benefit for Agents |
|------------|-------------------|
| **Millions of processes** | One process per conversation |
| **Location transparency** | Distribute agents across nodes |
| **Hot code loading** | Update agent logic without restart |
| **Built-in distribution** | Native clustering |
| **Selective receive** | Priority message handling |
| **Links and monitors** | Detect agent failures |
| **Binary efficiency** | Fast message passing |

---

## Next Steps

1. **Define Agent Behaviour**: Create `agent` behaviour with standard callbacks
2. **Design Message Protocol**: Structured message types for inter-agent communication
3. **Implement Supervision Tree**: Hierarchical agent management
4. **Build Tool Registry**: Dynamic tool registration and execution
5. **Create Memory Manager**: Short-term (ETS) + Long-term (Mnesia) storage
6. **Add Checkpointing**: Periodic state persistence
7. **Implement Handoff Protocol**: Standard state transfer mechanism
8. **Build Observability**: Tracing, logging, metrics

---

## Sources

### Agentic Frameworks
- LangChain/LangGraph Documentation and Architecture
- Microsoft AutoGen Documentation and GitHub
- CrewAI Documentation and Technical Analysis
- OpenAI Swarm GitHub and Agents SDK Documentation
- AutoGPT Documentation and Architecture Analysis

### Actor-Model Frameworks
- Akka Documentation (Scala/Java)
- Microsoft Orleans Documentation (.NET)
- Ray Documentation (Python)
- Dapr Documentation (CNCF)

### Research and Analysis
- Various technical deep-dives and architectural comparisons
- Industry best practices for multi-agent systems
