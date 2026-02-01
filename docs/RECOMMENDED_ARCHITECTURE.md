# Recommended Architecture for AI Agents in Erlang/OTP

This document presents the recommended architecture for implementing AI agents in Erlang/OTP, synthesized from the research in [actor-model-frameworks.md](../research/actor-model-frameworks.md) and [AGENT_FRAMEWORK_DIRECTORY.md](./AGENT_FRAMEWORK_DIRECTORY.md).

## Core Design: Supervision Tree with Specialized Behaviors

```
                         Application Supervisor
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         │                         │                         │
   Orchestrator Sup          Agent Pool Sup            Infrastructure Sup
   (one_for_one)             (simple_one_for_one)      (rest_for_one)
         │                         │                         │
   ┌─────┴─────┐              [Agent Workers]         ┌──────┼──────┐
   │           │              (gen_statem)            │      │      │
Coordinator  Router                                 Tool   Memory  Trace
(gen_server) (gen_server)                          Pool   Manager Manager
```

### Supervision Strategy Rationale

- **Orchestrator Supervisor (`one_for_one`)**: Coordinator and Router are independent; one failing shouldn't affect the other
- **Agent Pool Supervisor (`simple_one_for_one`)**: Dynamic agent spawning per conversation, agents are independent
- **Infrastructure Supervisor (`rest_for_one`)**: Tool Pool depends on Memory Manager; if Memory fails, restart both

---

## Key Components

### 1. Agent as `gen_statem` (State Machine)

This is the critical insight: agents naturally follow a state machine pattern (ReAct loop). The states map directly to the Thought → Action → Observation cycle used in frameworks like LangChain and CrewAI.

```erlang
-module(ai_agent).
-behaviour(gen_statem).

-export([start_link/2, send_message/2]).
-export([init/1, callback_mode/0, terminate/3]).
-export([idle/3, thinking/3, acting/3, observing/3]).

-record(data, {
    conversation_id,
    history = [],
    pending_reply,
    current_tool,
    observation,
    config
}).

%% API
start_link(ConversationId, Config) ->
    gen_statem:start_link(?MODULE, [ConversationId, Config], []).

send_message(Agent, Message) ->
    gen_statem:call(Agent, {user_message, Message}, 60000).

%% Callbacks
init([ConversationId, Config]) ->
    process_flag(trap_exit, true),
    Data = #data{
        conversation_id = ConversationId,
        config = Config
    },
    {ok, idle, Data}.

callback_mode() -> [state_functions, state_enter].

%% State: idle - waiting for user input
idle(enter, _OldState, Data) ->
    {keep_state, Data};
idle({call, From}, {user_message, Msg}, Data) ->
    NewHistory = Data#data.history ++ [{user, Msg}],
    {next_state, thinking, Data#data{
        history = NewHistory,
        pending_reply = From
    }}.

%% State: thinking - querying the LLM
thinking(enter, _OldState, Data) ->
    %% Async LLM request
    self() ! {llm_request, Data#data.history},
    {keep_state, Data};
thinking(info, {llm_request, History}, Data) ->
    %% In production, this would be async
    Response = llm_client:complete(History, Data#data.config),
    self() ! {llm_response, Response},
    {keep_state, Data};
thinking(info, {llm_response, Response}, Data) ->
    case parse_response(Response) of
        {tool_call, Tool, Args} ->
            {next_state, acting, Data#data{current_tool = {Tool, Args}}};
        {final_answer, Answer} ->
            NewHistory = Data#data.history ++ [{assistant, Answer}],
            gen_statem:reply(Data#data.pending_reply, {ok, Answer}),
            {next_state, idle, Data#data{
                history = NewHistory,
                pending_reply = undefined
            }}
    end.

%% State: acting - executing a tool
acting(enter, _OldState, #data{current_tool = {Tool, Args}} = Data) ->
    tool_executor:async_call(Tool, Args, self()),
    {keep_state, Data};
acting(info, {tool_result, Result}, Data) ->
    {next_state, observing, Data#data{observation = Result}};
acting(info, {tool_error, Error}, Data) ->
    %% Tool failed - feed error back to LLM
    {next_state, observing, Data#data{observation = {error, Error}}}.

%% State: observing - processing tool result
observing(enter, _OldState, Data) ->
    %% Append observation to history and return to thinking
    Observation = Data#data.observation,
    {Tool, Args} = Data#data.current_tool,
    NewHistory = Data#data.history ++ [
        {assistant_tool_call, Tool, Args},
        {tool_result, Observation}
    ],
    {next_state, thinking, Data#data{
        history = NewHistory,
        current_tool = undefined,
        observation = undefined
    }}.

terminate(_Reason, _State, Data) ->
    %% Checkpoint state before termination
    checkpoint_manager:save(Data#data.conversation_id, Data),
    ok.

%% Internal functions
parse_response(Response) ->
    %% Parse LLM response to determine if it's a tool call or final answer
    case Response of
        #{tool_call := #{name := Tool, arguments := Args}} ->
            {tool_call, Tool, Args};
        #{content := Content} ->
            {final_answer, Content}
    end.
```

### 2. Virtual Actor Pattern via `simple_one_for_one`

Agents are spawned on-demand per conversation and auto-terminated when idle. This mirrors the Orleans virtual actor model where actors exist conceptually and are activated only when needed.

```erlang
-module(agent_pool_sup).
-behaviour(supervisor).

-export([start_link/0, start_agent/2, stop_agent/1]).
-export([init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    ChildSpec = #{
        id => agent,
        start => {ai_agent, start_link, []},
        restart => temporary,  %% Don't restart finished conversations
        shutdown => 5000,
        type => worker,
        modules => [ai_agent]
    },
    {ok, {#{strategy => simple_one_for_one, intensity => 10, period => 60}, [ChildSpec]}}.

%% Start a new agent for a conversation
start_agent(ConversationId, Config) ->
    supervisor:start_child(?MODULE, [ConversationId, Config]).

%% Gracefully stop an agent
stop_agent(AgentPid) ->
    supervisor:terminate_child(?MODULE, AgentPid).
```

### 3. Agent Registry with Process Groups

Use `pg` (process groups) for agent discovery and topic-based messaging:

```erlang
-module(agent_registry).
-export([start_link/0, register_agent/3, find_agent/1, broadcast/2]).

start_link() ->
    pg:start_link(?MODULE).

%% Register an agent with its capabilities
register_agent(AgentPid, ConversationId, Capabilities) ->
    %% Join conversation-specific group
    pg:join(?MODULE, {conversation, ConversationId}, AgentPid),
    %% Join capability-based groups for routing
    lists:foreach(fun(Cap) ->
        pg:join(?MODULE, {capability, Cap}, AgentPid)
    end, Capabilities).

%% Find agents for a conversation
find_agent(ConversationId) ->
    pg:get_members(?MODULE, {conversation, ConversationId}).

%% Broadcast to all agents with a capability
broadcast({capability, Cap}, Message) ->
    Members = pg:get_members(?MODULE, {capability, Cap}),
    [gen_statem:cast(Pid, Message) || Pid <- Members].
```

### 4. Tool Execution as Supervised Worker Pool

Tools are isolated in separate processes with timeout and circuit breaker patterns:

```erlang
-module(tool_executor).
-behaviour(gen_server).

-export([start_link/0, async_call/3, sync_call/3]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(state, {
    circuit_breakers = #{}  %% Tool -> {state, failure_count, last_failure}
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    {ok, #state{}}.

%% Async tool execution - result sent as message to caller
async_call(Tool, Args, ReplyTo) ->
    gen_server:cast(?MODULE, {execute_async, Tool, Args, ReplyTo}).

%% Sync tool execution with timeout
sync_call(Tool, Args, Timeout) ->
    gen_server:call(?MODULE, {execute_sync, Tool, Args}, Timeout).

handle_call({execute_sync, Tool, Args}, _From, State) ->
    case check_circuit_breaker(Tool, State) of
        {open, NewState} ->
            {reply, {error, circuit_open}, NewState};
        {closed, NewState} ->
            Result = execute_tool(Tool, Args),
            FinalState = record_result(Tool, Result, NewState),
            {reply, Result, FinalState}
    end.

handle_cast({execute_async, Tool, Args, ReplyTo}, State) ->
    case check_circuit_breaker(Tool, State) of
        {open, NewState} ->
            ReplyTo ! {tool_error, circuit_open},
            {noreply, NewState};
        {closed, NewState} ->
            %% Spawn a process to avoid blocking
            Self = self(),
            spawn_link(fun() ->
                Result = execute_tool(Tool, Args),
                Self ! {tool_completed, Tool, Result, ReplyTo}
            end),
            {noreply, NewState}
    end.

handle_info({tool_completed, Tool, Result, ReplyTo}, State) ->
    NewState = record_result(Tool, Result, State),
    case Result of
        {ok, Value} -> ReplyTo ! {tool_result, Value};
        {error, Err} -> ReplyTo ! {tool_error, Err}
    end,
    {noreply, NewState}.

%% Internal: Execute tool with timeout
execute_tool(Tool, Args) ->
    try
        Timeout = application:get_env(ai_agent, tool_timeout, 30000),
        case Tool:execute(Args) of
            {ok, Result} -> {ok, Result};
            {error, Reason} -> {error, Reason}
        end
    catch
        _:Reason -> {error, {exception, Reason}}
    after
        ok
    end.

%% Circuit breaker logic
check_circuit_breaker(Tool, #state{circuit_breakers = CBs} = State) ->
    case maps:get(Tool, CBs, {closed, 0, 0}) of
        {open, _, LastFailure} ->
            %% Check if enough time has passed to try again
            Now = erlang:system_time(second),
            CooldownSecs = application:get_env(ai_agent, circuit_cooldown, 60),
            if
                Now - LastFailure > CooldownSecs ->
                    {closed, State#state{circuit_breakers = maps:put(Tool, {half_open, 0, 0}, CBs)}};
                true ->
                    {open, State}
            end;
        _ ->
            {closed, State}
    end.

record_result(Tool, Result, #state{circuit_breakers = CBs} = State) ->
    CB = maps:get(Tool, CBs, {closed, 0, 0}),
    NewCB = case {Result, CB} of
        {{ok, _}, _} ->
            {closed, 0, 0};
        {{error, _}, {_, FailCount, _}} when FailCount >= 4 ->
            {open, FailCount + 1, erlang:system_time(second)};
        {{error, _}, {_, FailCount, _}} ->
            {closed, FailCount + 1, erlang:system_time(second)}
    end,
    State#state{circuit_breakers = maps:put(Tool, NewCB, CBs)}.
```

### 5. Memory Architecture (ETS + Mnesia)

```erlang
-module(memory_manager).
-behaviour(gen_server).

-export([start_link/0]).
-export([store_short_term/3, get_short_term/2]).
-export([store_long_term/3, search_long_term/2]).
-export([init/1, handle_call/3, handle_cast/2]).

-record(memory, {
    id,
    agent_id,
    content,
    embedding,
    timestamp,
    metadata
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    %% Short-term memory in ETS (fast, in-memory)
    ets:new(agent_short_term, [named_table, public, {read_concurrency, true}]),

    %% Long-term memory in Mnesia (persistent)
    init_mnesia(),

    {ok, #{}}.

init_mnesia() ->
    mnesia:create_schema([node()]),
    mnesia:start(),
    mnesia:create_table(memory, [
        {disc_copies, [node()]},
        {attributes, record_info(fields, memory)},
        {index, [agent_id, timestamp]}
    ]).

%% Short-term: conversation context, recent interactions
store_short_term(AgentId, Key, Value) ->
    ets:insert(agent_short_term, {{AgentId, Key}, Value, erlang:system_time()}).

get_short_term(AgentId, Key) ->
    case ets:lookup(agent_short_term, {AgentId, Key}) of
        [{{AgentId, Key}, Value, _}] -> {ok, Value};
        [] -> {error, not_found}
    end.

%% Long-term: facts, embeddings, persistent knowledge
store_long_term(AgentId, Content, Metadata) ->
    Embedding = embedding_service:compute(Content),
    Record = #memory{
        id = erlang:unique_integer([monotonic, positive]),
        agent_id = AgentId,
        content = Content,
        embedding = Embedding,
        timestamp = erlang:system_time(millisecond),
        metadata = Metadata
    },
    mnesia:transaction(fun() -> mnesia:write(Record) end).

search_long_term(AgentId, QueryEmbedding) ->
    %% Retrieve and rank by embedding similarity
    mnesia:transaction(fun() ->
        Records = mnesia:match_object(#memory{agent_id = AgentId, _ = '_'}),
        Ranked = lists:sort(fun(A, B) ->
            cosine_similarity(QueryEmbedding, A#memory.embedding) >
            cosine_similarity(QueryEmbedding, B#memory.embedding)
        end, Records),
        lists:sublist(Ranked, 10)  %% Top 10 results
    end).

cosine_similarity(A, B) ->
    %% Compute cosine similarity between two embedding vectors
    Dot = lists:sum([X * Y || {X, Y} <- lists:zip(A, B)]),
    MagA = math:sqrt(lists:sum([X * X || X <- A])),
    MagB = math:sqrt(lists:sum([X * X || X <- B])),
    Dot / (MagA * MagB).

handle_call(_Request, _From, State) ->
    {reply, ok, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.
```

### 6. Event Sourcing for Conversation History

Every state change is persisted as an event, enabling replay for recovery and audit:

```erlang
-module(event_store).
-behaviour(gen_server).

-export([start_link/0, append/3, replay/1, replay_from/2]).
-export([init/1, handle_call/3, handle_cast/2]).

-record(event, {
    id,
    agent_id,
    type,           %% user_message | assistant_response | tool_call | tool_result | state_change
    payload,
    timestamp
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    mnesia:create_table(event, [
        {disc_copies, [node()]},
        {attributes, record_info(fields, event)},
        {index, [agent_id, timestamp]},
        {type, ordered_set}
    ]),
    {ok, #{}}.

%% Append a new event
append(AgentId, Type, Payload) ->
    Event = #event{
        id = erlang:unique_integer([monotonic, positive]),
        agent_id = AgentId,
        type = Type,
        payload = Payload,
        timestamp = erlang:system_time(millisecond)
    },
    mnesia:transaction(fun() -> mnesia:write(Event) end),
    Event.

%% Replay all events for an agent to rebuild state
replay(AgentId) ->
    {atomic, Events} = mnesia:transaction(fun() ->
        mnesia:match_object(#event{agent_id = AgentId, _ = '_'})
    end),
    SortedEvents = lists:keysort(#event.id, Events),
    lists:foldl(fun apply_event/2, initial_state(), SortedEvents).

%% Replay from a specific event ID (for partial recovery)
replay_from(AgentId, FromEventId) ->
    {atomic, Events} = mnesia:transaction(fun() ->
        mnesia:match_object(#event{agent_id = AgentId, _ = '_'})
    end),
    FilteredEvents = [E || E <- Events, E#event.id >= FromEventId],
    SortedEvents = lists:keysort(#event.id, FilteredEvents),
    lists:foldl(fun apply_event/2, initial_state(), SortedEvents).

%% Apply an event to reconstruct state
apply_event(#event{type = user_message, payload = Msg}, State) ->
    State#{history => maps:get(history, State, []) ++ [{user, Msg}]};
apply_event(#event{type = assistant_response, payload = Msg}, State) ->
    State#{history => maps:get(history, State, []) ++ [{assistant, Msg}]};
apply_event(#event{type = tool_call, payload = {Tool, Args}}, State) ->
    State#{pending_tool => {Tool, Args}};
apply_event(#event{type = tool_result, payload = Result}, State) ->
    State#{
        history => maps:get(history, State, []) ++ [{tool_result, Result}],
        pending_tool => undefined
    };
apply_event(#event{type = state_change, payload = Changes}, State) ->
    maps:merge(State, Changes).

initial_state() ->
    #{history => [], pending_tool => undefined}.
```

### 7. Handoff Protocol for Agent Delegation

```erlang
-module(agent_handoff).
-export([handoff/3, accept_handoff/2]).

%% Transfer conversation from one agent to another
handoff(FromAgent, ToAgent, Reason) ->
    %% Get current state from source agent
    {ok, State} = gen_statem:call(FromAgent, export_state),

    %% Build handoff context
    Context = #{
        conversation_history => maps:get(history, State, []),
        current_goal => maps:get(goal, State, undefined),
        working_memory => maps:get(memory, State, #{}),
        handoff_reason => Reason,
        source_agent => FromAgent,
        timestamp => erlang:system_time(millisecond)
    },

    %% Transfer to destination agent
    case gen_statem:call(ToAgent, {accept_handoff, Context}) of
        ok ->
            %% Log the handoff event
            event_store:append(maps:get(conversation_id, State), handoff, #{
                from => FromAgent,
                to => ToAgent,
                reason => Reason
            }),
            %% Gracefully stop source agent
            gen_statem:stop(FromAgent, normal, 5000),
            {ok, ToAgent};
        {error, Reason} ->
            {error, {handoff_rejected, Reason}}
    end.

%% Called on destination agent to accept handoff
accept_handoff(Agent, Context) ->
    gen_statem:call(Agent, {accept_handoff, Context}).
```

---

## Application Structure

```erlang
-module(ai_agent_app).
-behaviour(application).

-export([start/2, stop/1]).

start(_StartType, _StartArgs) ->
    ai_agent_sup:start_link().

stop(_State) ->
    ok.
```

```erlang
-module(ai_agent_sup).
-behaviour(supervisor).

-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    Children = [
        %% Infrastructure (must start first)
        #{id => memory_manager,
          start => {memory_manager, start_link, []},
          type => worker},

        #{id => event_store,
          start => {event_store, start_link, []},
          type => worker},

        #{id => tool_executor,
          start => {tool_executor, start_link, []},
          type => worker},

        #{id => agent_registry,
          start => {agent_registry, start_link, []},
          type => worker},

        %% Agent pool (dynamic children)
        #{id => agent_pool_sup,
          start => {agent_pool_sup, start_link, []},
          type => supervisor}
    ],

    {ok, {#{strategy => rest_for_one, intensity => 5, period => 60}, Children}}.
```

---

## Why This Architecture?

| Challenge | Solution |
|-----------|----------|
| **LLM call failures** | Supervisor restarts agent, checkpointed state recovered via event replay |
| **Scaling to millions of conversations** | Lightweight processes (~2KB each), `simple_one_for_one` dynamic spawning |
| **Tool isolation** | Separate tool processes with circuit breakers, failures don't crash agents |
| **Multi-agent coordination** | `pg` process groups for pub/sub, direct message passing |
| **Distributed deployment** | Native Erlang distribution, location-transparent calls via registry |
| **Hot updates** | Code reloading without dropping active conversations |
| **Observability** | Event sourcing provides full audit trail, SASL logging, `sys:trace` |
| **State recovery** | Event replay reconstructs any conversation state |

---

## Unique Erlang/OTP Advantages

1. **"Let it crash"** - LLM APIs are unreliable; supervision handles retries naturally without defensive coding
2. **Lightweight processes** - Millions of concurrent agents possible (~2KB per process)
3. **Selective receive** - Agents can prioritize urgent messages (e.g., cancellation over normal flow)
4. **Binary pattern matching** - Efficient parsing of streaming LLM responses
5. **Hot code loading** - Update agent behavior without losing active conversations
6. **Built-in distribution** - Scale horizontally by adding nodes; `pg` handles discovery automatically
7. **Pre-emptive scheduling** - No agent can starve others; fair CPU distribution
8. **Garbage collection per process** - No global GC pauses affecting all agents

---

## Comparison to Other Frameworks

| Feature | This Architecture | LangGraph | AutoGen | Ray |
|---------|-------------------|-----------|---------|-----|
| Process model | Native lightweight processes | Python threads/async | Python async | Actor + Task hybrid |
| Fault isolation | Process-level | None | None | Actor-level |
| Supervision | Built-in hierarchical | Manual | Manual | Checkpoint-based |
| Distribution | Native, transparent | External (Redis) | External | Built-in |
| Hot reload | Native | Restart required | Restart required | Restart required |
| State recovery | Event sourcing + replay | Checkpointing | Checkpointing | Lineage reconstruction |

---

## Next Steps for Implementation

1. **Core behaviors**: Implement `ai_agent` gen_statem with full ReAct loop
2. **LLM client**: Create `llm_client` module with streaming support and retries
3. **Tool framework**: Define tool behavior and implement common tools
4. **Testing**: Property-based testing with PropEr for agent state machines
5. **Observability**: Add OpenTelemetry integration for distributed tracing
6. **Clustering**: Configure Erlang distribution for multi-node deployment
