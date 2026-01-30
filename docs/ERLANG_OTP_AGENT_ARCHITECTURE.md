# Erlang/OTP Agent Architecture Design

> A synthesis layer bringing agentic AI capabilities to the battle-tested Erlang/OTP platform.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Core Behaviours](#core-behaviours)
   - [agent Behaviour](#1-agent-behaviour)
   - [agent_statem Behaviour](#2-agent_statem-behaviour)
   - [tool Behaviour](#3-tool-behaviour)
3. [Core Modules](#core-modules)
   - [Agent Registry](#agent-registry)
   - [Tool Registry](#tool-registry)
   - [Memory Manager](#memory-manager)
   - [LLM Client](#llm-client)
   - [Checkpoint Manager](#checkpoint-manager)
4. [Message Protocol](#message-protocol)
5. [Supervision Tree Design](#supervision-tree-design)
6. [Orchestration Patterns](#orchestration-patterns)
7. [Example Implementations](#example-implementations)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              APPLICATION LAYER                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                         User Applications                                │    │
│  │              (Chatbots, Autonomous Agents, Workflows)                    │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                            ORCHESTRATION LAYER                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │  Supervisor  │  │   Handoff    │  │   Pub/Sub    │  │   Pipeline   │        │
│  │   Pattern    │  │   Pattern    │  │   (pg)       │  │   Pattern    │        │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                              AGENT LAYER                                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                                                                         │    │
│  │  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐ │    │
│  │  │   agent     │   │agent_statem │   │    tool     │   │  guardrail  │ │    │
│  │  │ (behaviour) │   │ (behaviour) │   │ (behaviour) │   │ (behaviour) │ │    │
│  │  └─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘ │    │
│  │                                                                         │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                            INFRASTRUCTURE LAYER                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │
│  │   Agent    │  │    Tool    │  │   Memory   │  │    LLM     │  │Checkpoint│  │
│  │  Registry  │  │  Registry  │  │  Manager   │  │   Client   │  │ Manager  │  │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘  └──────────┘  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                               OTP FOUNDATION                                     │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │
│  │gen_server  │  │ gen_statem │  │ supervisor │  │    pg      │  │  mnesia  │  │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘  └──────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Core Behaviours

### 1. agent Behaviour

The `agent` behaviour extends `gen_server` with agent-specific callbacks for LLM interaction, tool execution, and memory management.

#### Callback Specification

```erlang
%%%-------------------------------------------------------------------
%%% @doc Agent behaviour for LLM-powered agents.
%%%
%%% This behaviour provides a standard interface for implementing
%%% agents that interact with LLMs, execute tools, and manage
%%% conversational state.
%%%-------------------------------------------------------------------
-module(agent).
-behaviour(gen_server).

%% Behaviour callbacks that implementing modules must export
-callback init(Args :: term()) ->
    {ok, State :: term()} |
    {ok, State :: term(), Opts :: [agent_opt()]} |
    {stop, Reason :: term()}.

-callback handle_message(Message :: user_message(), State :: term()) ->
    {reply, Response :: agent_response(), NewState :: term()} |
    {tool_call, ToolCall :: tool_request(), NewState :: term()} |
    {handoff, Target :: agent_ref(), Context :: term(), NewState :: term()} |
    {stop, Reason :: term(), Response :: agent_response(), NewState :: term()}.

-callback handle_tool_result(ToolName :: atom(), Result :: term(), State :: term()) ->
    {continue, NewState :: term()} |
    {reply, Response :: agent_response(), NewState :: term()} |
    {tool_call, ToolCall :: tool_request(), NewState :: term()}.

-callback system_prompt(State :: term()) ->
    binary().

-callback available_tools(State :: term()) ->
    [tool_spec()].

%% Optional callbacks with defaults
-callback handle_handoff(FromAgent :: agent_ref(), Context :: term(), State :: term()) ->
    {ok, NewState :: term()} |
    {reject, Reason :: term()}.

-callback format_error(Reason :: term(), State :: term()) ->
    binary().

-callback terminate(Reason :: term(), State :: term()) ->
    ok.

-optional_callbacks([handle_handoff/3, format_error/2, terminate/2]).

%% Type definitions
-type agent_opt() ::
    {llm_client, module()} |
    {memory, memory_ref()} |
    {checkpoint_interval, pos_integer()} |
    {max_turns, pos_integer()} |
    {timeout, timeout()}.

-type user_message() :: #{
    role := user,
    content := binary(),
    metadata => map()
}.

-type agent_response() :: #{
    role := assistant,
    content := binary(),
    tool_calls => [tool_call()],
    metadata => map()
}.

-type tool_request() :: #{
    name := atom(),
    arguments := map(),
    call_id := binary()
}.

-type tool_spec() :: #{
    name := atom(),
    description := binary(),
    parameters := json_schema()
}.

-type agent_ref() :: pid() | atom() | {atom(), node()}.

-export_type([agent_opt/0, user_message/0, agent_response/0,
              tool_request/0, tool_spec/0, agent_ref/0]).
```

#### Agent Behaviour Implementation

```erlang
-module(agent).
-behaviour(gen_server).

%% API
-export([start_link/2, start_link/3]).
-export([send_message/2, send_message/3]).
-export([get_conversation/1]).
-export([handoff_to/3]).
-export([stop/1, stop/2]).

%% gen_server callbacks
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-record(agent_state, {
    module          :: module(),
    mod_state       :: term(),
    conversation    :: [message()],
    llm_client      :: module(),
    memory          :: memory_ref() | undefined,
    checkpoint_ref  :: reference() | undefined,
    turn_count      :: non_neg_integer(),
    max_turns       :: pos_integer() | infinity,
    opts            :: [agent_opt()]
}).

%%%===================================================================
%%% API
%%%===================================================================

-spec start_link(Module :: module(), Args :: term()) ->
    {ok, pid()} | {error, term()}.
start_link(Module, Args) ->
    start_link(Module, Args, []).

-spec start_link(Module :: module(), Args :: term(), Opts :: [agent_opt()]) ->
    {ok, pid()} | {error, term()}.
start_link(Module, Args, Opts) ->
    gen_server:start_link(?MODULE, {Module, Args, Opts}, []).

-spec send_message(Agent :: agent_ref(), Message :: binary()) ->
    {ok, agent_response()} | {error, term()}.
send_message(Agent, Content) ->
    send_message(Agent, Content, #{}).

-spec send_message(Agent :: agent_ref(), Content :: binary(), Metadata :: map()) ->
    {ok, agent_response()} | {error, term()}.
send_message(Agent, Content, Metadata) ->
    Message = #{role => user, content => Content, metadata => Metadata},
    gen_server:call(Agent, {message, Message}, infinity).

-spec get_conversation(Agent :: agent_ref()) -> [message()].
get_conversation(Agent) ->
    gen_server:call(Agent, get_conversation).

-spec handoff_to(FromAgent :: agent_ref(), ToAgent :: agent_ref(), Context :: term()) ->
    ok | {error, rejected}.
handoff_to(FromAgent, ToAgent, Context) ->
    gen_server:call(FromAgent, {initiate_handoff, ToAgent, Context}).

-spec stop(Agent :: agent_ref()) -> ok.
stop(Agent) ->
    stop(Agent, normal).

-spec stop(Agent :: agent_ref(), Reason :: term()) -> ok.
stop(Agent, Reason) ->
    gen_server:stop(Agent, Reason, 5000).

%%%===================================================================
%%% gen_server callbacks
%%%===================================================================

init({Module, Args, Opts}) ->
    process_flag(trap_exit, true),

    case Module:init(Args) of
        {ok, ModState} ->
            {ok, init_agent_state(Module, ModState, Opts)};
        {ok, ModState, ModOpts} ->
            {ok, init_agent_state(Module, ModState, Opts ++ ModOpts)};
        {stop, Reason} ->
            {stop, Reason}
    end.

handle_call({message, Message}, From, State) ->
    #agent_state{
        module = Module,
        mod_state = ModState,
        conversation = Conv,
        turn_count = Turns,
        max_turns = MaxTurns
    } = State,

    %% Check turn limit
    case Turns >= MaxTurns of
        true ->
            {reply, {error, max_turns_exceeded}, State};
        false ->
            %% Add user message to conversation
            NewConv = Conv ++ [Message],

            %% Get LLM response
            SystemPrompt = Module:system_prompt(ModState),
            Tools = Module:available_tools(ModState),

            case invoke_llm(SystemPrompt, NewConv, Tools, State) of
                {ok, LLMResponse} ->
                    handle_llm_response(LLMResponse, Message, From,
                                       State#agent_state{conversation = NewConv});
                {error, Reason} ->
                    {reply, {error, Reason}, State}
            end
    end;

handle_call(get_conversation, _From, #agent_state{conversation = Conv} = State) ->
    {reply, Conv, State};

handle_call({initiate_handoff, ToAgent, Context}, _From, State) ->
    #agent_state{module = Module, mod_state = ModState, conversation = Conv} = State,

    HandoffData = #{
        conversation => Conv,
        context => Context,
        from_agent => self()
    },

    case gen_server:call(ToAgent, {receive_handoff, HandoffData}) of
        ok ->
            {reply, ok, State};
        {error, Reason} ->
            {reply, {error, Reason}, State}
    end;

handle_call({receive_handoff, HandoffData}, _From, State) ->
    #agent_state{module = Module, mod_state = ModState} = State,
    #{conversation := Conv, context := Context, from_agent := FromAgent} = HandoffData,

    case erlang:function_exported(Module, handle_handoff, 3) of
        true ->
            case Module:handle_handoff(FromAgent, Context, ModState) of
                {ok, NewModState} ->
                    NewState = State#agent_state{
                        mod_state = NewModState,
                        conversation = Conv
                    },
                    {reply, ok, NewState};
                {reject, Reason} ->
                    {reply, {error, Reason}, State}
            end;
        false ->
            %% Default: accept handoff
            NewState = State#agent_state{conversation = Conv},
            {reply, ok, NewState}
    end;

handle_call(_Request, _From, State) ->
    {reply, {error, unknown_request}, State}.

handle_cast({tool_result, CallId, Result}, State) ->
    #agent_state{module = Module, mod_state = ModState} = State,

    %% Find the tool call and process result
    case Module:handle_tool_result(CallId, Result, ModState) of
        {continue, NewModState} ->
            %% Continue the agent loop
            {noreply, State#agent_state{mod_state = NewModState}};
        {reply, Response, NewModState} ->
            %% We'd need to track the original caller - simplified here
            {noreply, State#agent_state{mod_state = NewModState}};
        {tool_call, ToolCall, NewModState} ->
            execute_tool(ToolCall, State#agent_state{mod_state = NewModState}),
            {noreply, State#agent_state{mod_state = NewModState}}
    end;

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info({checkpoint, Ref}, #agent_state{checkpoint_ref = Ref} = State) ->
    %% Periodic checkpoint
    ok = save_checkpoint(State),
    NewRef = schedule_checkpoint(State),
    {noreply, State#agent_state{checkpoint_ref = NewRef}};

handle_info(_Info, State) ->
    {noreply, State}.

terminate(Reason, #agent_state{module = Module, mod_state = ModState} = State) ->
    %% Save final checkpoint
    ok = save_checkpoint(State),

    %% Call module terminate if exported
    case erlang:function_exported(Module, terminate, 2) of
        true -> Module:terminate(Reason, ModState);
        false -> ok
    end.

%%%===================================================================
%%% Internal functions
%%%===================================================================

init_agent_state(Module, ModState, Opts) ->
    LLMClient = proplists:get_value(llm_client, Opts, llm_client_openai),
    Memory = proplists:get_value(memory, Opts, undefined),
    MaxTurns = proplists:get_value(max_turns, Opts, infinity),

    State = #agent_state{
        module = Module,
        mod_state = ModState,
        conversation = [],
        llm_client = LLMClient,
        memory = Memory,
        turn_count = 0,
        max_turns = MaxTurns,
        opts = Opts
    },

    %% Schedule first checkpoint if interval specified
    CheckpointRef = schedule_checkpoint(State),
    State#agent_state{checkpoint_ref = CheckpointRef}.

invoke_llm(SystemPrompt, Conversation, Tools, #agent_state{llm_client = Client}) ->
    Client:chat_completion(#{
        system => SystemPrompt,
        messages => Conversation,
        tools => Tools
    }).

handle_llm_response(LLMResponse, OriginalMessage, From, State) ->
    #agent_state{module = Module, mod_state = ModState, conversation = Conv} = State,

    case Module:handle_message(OriginalMessage, ModState) of
        {reply, Response, NewModState} ->
            NewConv = Conv ++ [Response],
            NewState = State#agent_state{
                mod_state = NewModState,
                conversation = NewConv,
                turn_count = State#agent_state.turn_count + 1
            },
            {reply, {ok, Response}, NewState};

        {tool_call, ToolCall, NewModState} ->
            %% Execute tool asynchronously
            execute_tool(ToolCall, State#agent_state{mod_state = NewModState}),
            %% This is simplified - real impl would track pending calls
            {noreply, State#agent_state{mod_state = NewModState}};

        {handoff, Target, Context, NewModState} ->
            case handoff_to(self(), Target, Context) of
                ok ->
                    Response = #{role => assistant,
                                content => <<"Transferred to specialist agent.">>},
                    {reply, {ok, Response}, State#agent_state{mod_state = NewModState}};
                {error, Reason} ->
                    {reply, {error, {handoff_failed, Reason}}, State}
            end;

        {stop, Reason, Response, NewModState} ->
            NewConv = Conv ++ [Response],
            {stop, Reason, {ok, Response},
             State#agent_state{mod_state = NewModState, conversation = NewConv}}
    end.

execute_tool(#{name := ToolName, arguments := Args, call_id := CallId}, State) ->
    %% Execute tool via tool_registry
    Self = self(),
    spawn_link(fun() ->
        Result = tool_registry:execute(ToolName, Args),
        gen_server:cast(Self, {tool_result, CallId, Result})
    end).

save_checkpoint(#agent_state{conversation = Conv, mod_state = ModState}) ->
    checkpoint_manager:save(self(), #{
        conversation => Conv,
        mod_state => ModState,
        timestamp => erlang:system_time(millisecond)
    }).

schedule_checkpoint(#agent_state{opts = Opts}) ->
    case proplists:get_value(checkpoint_interval, Opts) of
        undefined -> undefined;
        Interval ->
            Ref = make_ref(),
            erlang:send_after(Interval, self(), {checkpoint, Ref}),
            Ref
    end.
```

---

### 2. agent_statem Behaviour

For agents with complex workflows (ReAct pattern), extending `gen_statem`.

```erlang
%%%-------------------------------------------------------------------
%%% @doc State machine agent behaviour for workflow-driven agents.
%%%
%%% Implements the ReAct (Reasoning + Acting) pattern with explicit
%%% states: idle -> thinking -> tool_calling -> observing -> thinking...
%%%-------------------------------------------------------------------
-module(agent_statem).
-behaviour(gen_statem).

%% Behaviour callbacks
-callback init(Args :: term()) ->
    {ok, InitialData :: term()} |
    {ok, InitialData :: term(), Opts :: [agent_opt()]} |
    {stop, Reason :: term()}.

-callback system_prompt(Data :: term()) ->
    binary().

-callback available_tools(Data :: term()) ->
    [tool_spec()].

-callback handle_thinking_result(Response :: llm_response(), Data :: term()) ->
    {final_answer, Answer :: binary(), NewData :: term()} |
    {tool_call, ToolCall :: tool_request(), NewData :: term()} |
    {continue_thinking, NewData :: term()} |
    {error, Reason :: term(), NewData :: term()}.

-callback handle_observation(ToolName :: atom(), Result :: term(), Data :: term()) ->
    {continue, NewData :: term()} |
    {error, Reason :: term(), NewData :: term()}.

%% Optional callbacks
-callback max_iterations(Data :: term()) -> pos_integer().
-callback on_state_enter(State :: atom(), Data :: term()) -> ok.

-optional_callbacks([max_iterations/1, on_state_enter/2]).

%% State definitions
-type state() :: idle | thinking | tool_calling | observing | finished | error.

%% API
-export([start_link/2, start_link/3]).
-export([submit_task/2, submit_task/3]).
-export([get_status/1]).
-export([cancel/1]).

%% gen_statem callbacks
-export([callback_mode/0, init/1, terminate/3]).
-export([idle/3, thinking/3, tool_calling/3, observing/3, finished/3, error/3]).

-record(data, {
    module          :: module(),
    mod_data        :: term(),
    task            :: binary() | undefined,
    conversation    :: [message()],
    pending_tool    :: tool_request() | undefined,
    iteration       :: non_neg_integer(),
    max_iterations  :: pos_integer(),
    caller          :: {pid(), reference()} | undefined,
    llm_client      :: module(),
    opts            :: [agent_opt()]
}).

%%%===================================================================
%%% API
%%%===================================================================

start_link(Module, Args) ->
    start_link(Module, Args, []).

start_link(Module, Args, Opts) ->
    gen_statem:start_link(?MODULE, {Module, Args, Opts}, []).

-spec submit_task(Agent :: pid(), Task :: binary()) ->
    {ok, Result :: binary()} | {error, term()}.
submit_task(Agent, Task) ->
    submit_task(Agent, Task, infinity).

-spec submit_task(Agent :: pid(), Task :: binary(), Timeout :: timeout()) ->
    {ok, Result :: binary()} | {error, term()}.
submit_task(Agent, Task, Timeout) ->
    gen_statem:call(Agent, {submit_task, Task}, Timeout).

-spec get_status(Agent :: pid()) -> {state(), map()}.
get_status(Agent) ->
    gen_statem:call(Agent, get_status).

-spec cancel(Agent :: pid()) -> ok.
cancel(Agent) ->
    gen_statem:cast(Agent, cancel).

%%%===================================================================
%%% gen_statem callbacks
%%%===================================================================

callback_mode() -> [state_functions, state_enter].

init({Module, Args, Opts}) ->
    case Module:init(Args) of
        {ok, ModData} ->
            {ok, idle, init_data(Module, ModData, Opts)};
        {ok, ModData, ModOpts} ->
            {ok, idle, init_data(Module, ModData, Opts ++ ModOpts)};
        {stop, Reason} ->
            {stop, Reason}
    end.

%%% State: idle - Waiting for a task
idle(enter, _OldState, Data) ->
    maybe_notify_state_enter(idle, Data),
    {keep_state, Data#data{task = undefined, iteration = 0}};

idle({call, From}, {submit_task, Task}, Data) ->
    NewData = Data#data{
        task = Task,
        caller = From,
        conversation = [#{role => user, content => Task}]
    },
    {next_state, thinking, NewData};

idle({call, From}, get_status, Data) ->
    {keep_state, Data, [{reply, From, {idle, #{}}}]};

idle(cast, cancel, Data) ->
    {keep_state, Data}.

%%% State: thinking - LLM is reasoning
thinking(enter, _OldState, #data{iteration = Iter, max_iterations = Max} = Data)
  when Iter >= Max ->
    %% Exceeded max iterations
    {next_state, error, Data#data{mod_data = {error, max_iterations_exceeded}}};

thinking(enter, _OldState, Data) ->
    maybe_notify_state_enter(thinking, Data),
    %% Trigger async LLM call
    Self = self(),
    #data{module = Module, mod_data = ModData, conversation = Conv, llm_client = Client} = Data,

    spawn_link(fun() ->
        SystemPrompt = Module:system_prompt(ModData),
        Tools = Module:available_tools(ModData),
        Result = Client:chat_completion(#{
            system => SystemPrompt,
            messages => Conv,
            tools => Tools
        }),
        gen_statem:cast(Self, {llm_response, Result})
    end),

    {keep_state, Data#data{iteration = Data#data.iteration + 1},
     [{state_timeout, 60000, llm_timeout}]};

thinking(cast, {llm_response, {ok, Response}}, Data) ->
    #data{module = Module, mod_data = ModData} = Data,

    case Module:handle_thinking_result(Response, ModData) of
        {final_answer, Answer, NewModData} ->
            NewConv = Data#data.conversation ++ [#{role => assistant, content => Answer}],
            {next_state, finished, Data#data{
                mod_data = NewModData,
                conversation = NewConv
            }};

        {tool_call, ToolCall, NewModData} ->
            {next_state, tool_calling, Data#data{
                mod_data = NewModData,
                pending_tool = ToolCall
            }};

        {continue_thinking, NewModData} ->
            %% Re-enter thinking state
            {repeat_state, Data#data{mod_data = NewModData}};

        {error, Reason, NewModData} ->
            {next_state, error, Data#data{mod_data = {error, Reason}}}
    end;

thinking(cast, {llm_response, {error, Reason}}, Data) ->
    {next_state, error, Data#data{mod_data = {error, {llm_error, Reason}}}};

thinking(state_timeout, llm_timeout, Data) ->
    {next_state, error, Data#data{mod_data = {error, llm_timeout}}};

thinking({call, From}, get_status, Data) ->
    {keep_state, Data, [{reply, From, {thinking, #{iteration => Data#data.iteration}}}]};

thinking(cast, cancel, Data) ->
    {next_state, idle, Data}.

%%% State: tool_calling - Executing a tool
tool_calling(enter, _OldState, #data{pending_tool = ToolCall} = Data) ->
    maybe_notify_state_enter(tool_calling, Data),

    Self = self(),
    #{name := ToolName, arguments := Args} = ToolCall,

    spawn_link(fun() ->
        Result = tool_registry:execute(ToolName, Args),
        gen_statem:cast(Self, {tool_result, ToolName, Result})
    end),

    {keep_state, Data, [{state_timeout, 30000, tool_timeout}]};

tool_calling(cast, {tool_result, ToolName, Result}, Data) ->
    {next_state, observing, Data#data{pending_tool = {ToolName, Result}}};

tool_calling(state_timeout, tool_timeout, Data) ->
    {next_state, error, Data#data{mod_data = {error, tool_timeout}}};

tool_calling({call, From}, get_status, #data{pending_tool = Tool} = Data) ->
    {keep_state, Data, [{reply, From, {tool_calling, #{tool => Tool}}}]};

tool_calling(cast, cancel, Data) ->
    {next_state, idle, Data}.

%%% State: observing - Processing tool result
observing(enter, _OldState, #data{pending_tool = {ToolName, Result}} = Data) ->
    maybe_notify_state_enter(observing, Data),

    #data{module = Module, mod_data = ModData, conversation = Conv} = Data,

    %% Add tool result to conversation
    Observation = #{
        role => tool,
        name => ToolName,
        content => format_tool_result(Result)
    },
    NewConv = Conv ++ [Observation],

    case Module:handle_observation(ToolName, Result, ModData) of
        {continue, NewModData} ->
            {next_state, thinking, Data#data{
                mod_data = NewModData,
                conversation = NewConv,
                pending_tool = undefined
            }};

        {error, Reason, NewModData} ->
            {next_state, error, Data#data{mod_data = {error, Reason}}}
    end;

observing({call, From}, get_status, Data) ->
    {keep_state, Data, [{reply, From, {observing, #{}}}]}.

%%% State: finished - Task completed successfully
finished(enter, _OldState, #data{caller = Caller, conversation = Conv} = Data) ->
    maybe_notify_state_enter(finished, Data),

    %% Extract final answer from last assistant message
    FinalAnswer = get_last_assistant_message(Conv),

    case Caller of
        undefined -> ok;
        {Pid, _} = From ->
            gen_statem:reply(From, {ok, FinalAnswer})
    end,

    {keep_state, Data#data{caller = undefined}};

finished({call, From}, get_status, Data) ->
    {keep_state, Data, [{reply, From, {finished, #{result => get_last_assistant_message(Data#data.conversation)}}}]};

finished({call, From}, {submit_task, Task}, Data) ->
    %% Allow reuse - transition back to idle then thinking
    NewData = Data#data{
        task = Task,
        caller = From,
        conversation = [#{role => user, content => Task}],
        iteration = 0
    },
    {next_state, thinking, NewData}.

%%% State: error - Something went wrong
error(enter, _OldState, #data{caller = Caller, mod_data = {error, Reason}} = Data) ->
    maybe_notify_state_enter(error, Data),

    case Caller of
        undefined -> ok;
        {Pid, _} = From ->
            gen_statem:reply(From, {error, Reason})
    end,

    {keep_state, Data#data{caller = undefined}};

error({call, From}, get_status, #data{mod_data = {error, Reason}} = Data) ->
    {keep_state, Data, [{reply, From, {error, #{reason => Reason}}}]};

error({call, From}, {submit_task, Task}, Data) ->
    %% Allow retry from error state
    NewData = Data#data{
        task = Task,
        caller = From,
        conversation = [#{role => user, content => Task}],
        iteration = 0,
        mod_data = reset_mod_data(Data)
    },
    {next_state, thinking, NewData}.

terminate(_Reason, _State, _Data) ->
    ok.

%%%===================================================================
%%% Internal functions
%%%===================================================================

init_data(Module, ModData, Opts) ->
    MaxIter = case erlang:function_exported(Module, max_iterations, 1) of
        true -> Module:max_iterations(ModData);
        false -> proplists:get_value(max_iterations, Opts, 10)
    end,

    #data{
        module = Module,
        mod_data = ModData,
        conversation = [],
        iteration = 0,
        max_iterations = MaxIter,
        llm_client = proplists:get_value(llm_client, Opts, llm_client_openai),
        opts = Opts
    }.

maybe_notify_state_enter(State, #data{module = Module, mod_data = ModData}) ->
    case erlang:function_exported(Module, on_state_enter, 2) of
        true -> Module:on_state_enter(State, ModData);
        false -> ok
    end.

format_tool_result(Result) when is_binary(Result) -> Result;
format_tool_result(Result) -> iolist_to_binary(io_lib:format("~p", [Result])).

get_last_assistant_message([]) -> <<>>;
get_last_assistant_message(Conv) ->
    case lists:last(Conv) of
        #{role := assistant, content := Content} -> Content;
        _ -> <<>>
    end.

reset_mod_data(#data{module = Module, mod_data = {error, _}}) ->
    %% Would need to call init again for proper reset
    undefined.
```

---

### 3. tool Behaviour

Standard interface for implementing tools that agents can invoke.

```erlang
%%%-------------------------------------------------------------------
%%% @doc Tool behaviour for agent-callable tools.
%%%
%%% Tools are synchronous operations that agents can invoke.
%%% Each tool has a schema for validation and a execute function.
%%%-------------------------------------------------------------------
-module(tool).

%% Behaviour callbacks
-callback name() -> atom().

-callback description() -> binary().

-callback parameters_schema() -> json_schema().

-callback execute(Args :: map()) ->
    {ok, Result :: term()} |
    {error, Reason :: term()}.

%% Optional callbacks
-callback validate(Args :: map()) -> ok | {error, term()}.
-callback timeout() -> timeout().

-optional_callbacks([validate/1, timeout/0]).

%% Type definitions
-type json_schema() :: #{
    type := object,
    properties := #{atom() => property_schema()},
    required => [atom()]
}.

-type property_schema() :: #{
    type := string | number | integer | boolean | array | object,
    description => binary(),
    enum => [term()],
    items => property_schema(),
    properties => #{atom() => property_schema()}
}.

-export_type([json_schema/0, property_schema/0]).
```

#### Example Tool Implementation

```erlang
%%%-------------------------------------------------------------------
%%% @doc Web search tool implementation.
%%%-------------------------------------------------------------------
-module(tool_web_search).
-behaviour(tool).

-export([name/0, description/0, parameters_schema/0, execute/1]).

name() -> web_search.

description() ->
    <<"Search the web for current information. Use this when you need "
      "up-to-date information that may not be in your training data.">>.

parameters_schema() ->
    #{
        type => object,
        properties => #{
            query => #{
                type => string,
                description => <<"The search query">>
            },
            num_results => #{
                type => integer,
                description => <<"Number of results to return (default 5)">>
            }
        },
        required => [query]
    }.

execute(#{query := Query} = Args) ->
    NumResults = maps:get(num_results, Args, 5),

    case httpc:request(get, {build_search_url(Query, NumResults), []}, [], []) of
        {ok, {{_, 200, _}, _, Body}} ->
            Results = parse_search_results(Body),
            {ok, Results};
        {ok, {{_, StatusCode, _}, _, _}} ->
            {error, {http_error, StatusCode}};
        {error, Reason} ->
            {error, Reason}
    end.

build_search_url(Query, NumResults) ->
    %% Implementation details...
    "https://api.search.example.com/search?q=" ++
    http_uri:encode(binary_to_list(Query)) ++
    "&n=" ++ integer_to_list(NumResults).

parse_search_results(Body) ->
    %% Parse JSON response...
    jsx:decode(Body, [return_maps]).
```

---

## Core Modules

### Agent Registry

Manages agent registration, discovery, and metadata.

```erlang
%%%-------------------------------------------------------------------
%%% @doc Agent registry for discovery and management.
%%%-------------------------------------------------------------------
-module(agent_registry).
-behaviour(gen_server).

-export([start_link/0]).
-export([register/2, register/3]).
-export([unregister/1]).
-export([lookup/1]).
-export([find_by_capability/1]).
-export([list_all/0]).

%% gen_server callbacks
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-record(state, {
    agents :: ets:tid(),      %% ETS table: {Name, Pid, Metadata}
    monitors :: #{reference() => atom()}  %% Monitor ref -> Name
}).

-record(agent_entry, {
    name        :: atom(),
    pid         :: pid(),
    module      :: module(),
    capabilities :: [atom()],
    metadata    :: map(),
    registered  :: integer()  %% Timestamp
}).

%%%===================================================================
%%% API
%%%===================================================================

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

-spec register(Name :: atom(), Pid :: pid()) -> ok | {error, already_registered}.
register(Name, Pid) ->
    register(Name, Pid, #{}).

-spec register(Name :: atom(), Pid :: pid(), Metadata :: map()) ->
    ok | {error, already_registered}.
register(Name, Pid, Metadata) ->
    gen_server:call(?MODULE, {register, Name, Pid, Metadata}).

-spec unregister(Name :: atom()) -> ok.
unregister(Name) ->
    gen_server:call(?MODULE, {unregister, Name}).

-spec lookup(Name :: atom()) -> {ok, pid(), map()} | {error, not_found}.
lookup(Name) ->
    case ets:lookup(agent_registry_tab, Name) of
        [#agent_entry{pid = Pid, metadata = Meta}] ->
            {ok, Pid, Meta};
        [] ->
            {error, not_found}
    end.

-spec find_by_capability(Capability :: atom()) -> [{atom(), pid()}].
find_by_capability(Capability) ->
    ets:foldl(
        fun(#agent_entry{name = Name, pid = Pid, capabilities = Caps}, Acc) ->
            case lists:member(Capability, Caps) of
                true -> [{Name, Pid} | Acc];
                false -> Acc
            end
        end,
        [],
        agent_registry_tab
    ).

-spec list_all() -> [#agent_entry{}].
list_all() ->
    ets:tab2list(agent_registry_tab).

%%%===================================================================
%%% gen_server callbacks
%%%===================================================================

init([]) ->
    Tab = ets:new(agent_registry_tab, [
        named_table,
        {keypos, #agent_entry.name},
        {read_concurrency, true}
    ]),

    %% Join pg group for distributed discovery
    ok = pg:join(agent_registries, self()),

    {ok, #state{agents = Tab, monitors = #{}}}.

handle_call({register, Name, Pid, Metadata}, _From, State) ->
    case ets:lookup(agent_registry_tab, Name) of
        [] ->
            %% Monitor the agent process
            MonRef = monitor(process, Pid),

            Entry = #agent_entry{
                name = Name,
                pid = Pid,
                module = maps:get(module, Metadata, unknown),
                capabilities = maps:get(capabilities, Metadata, []),
                metadata = Metadata,
                registered = erlang:system_time(millisecond)
            },

            true = ets:insert(agent_registry_tab, Entry),

            %% Join capability-based pg groups
            lists:foreach(
                fun(Cap) -> pg:join({capability, Cap}, Pid) end,
                Entry#agent_entry.capabilities
            ),

            NewMonitors = maps:put(MonRef, Name, State#state.monitors),
            {reply, ok, State#state{monitors = NewMonitors}};

        [_Existing] ->
            {reply, {error, already_registered}, State}
    end;

handle_call({unregister, Name}, _From, State) ->
    case ets:lookup(agent_registry_tab, Name) of
        [#agent_entry{pid = Pid, capabilities = Caps}] ->
            true = ets:delete(agent_registry_tab, Name),

            %% Leave pg groups
            lists:foreach(
                fun(Cap) -> pg:leave({capability, Cap}, Pid) end,
                Caps
            ),

            {reply, ok, State};
        [] ->
            {reply, ok, State}
    end;

handle_call(_Request, _From, State) ->
    {reply, {error, unknown_request}, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info({'DOWN', MonRef, process, _Pid, _Reason}, State) ->
    case maps:get(MonRef, State#state.monitors, undefined) of
        undefined ->
            {noreply, State};
        Name ->
            true = ets:delete(agent_registry_tab, Name),
            NewMonitors = maps:remove(MonRef, State#state.monitors),
            {noreply, State#state{monitors = NewMonitors}}
    end;

handle_info(_Info, State) ->
    {noreply, State}.

terminate(_Reason, _State) ->
    ok.
```

### Tool Registry

```erlang
%%%-------------------------------------------------------------------
%%% @doc Tool registry for dynamic tool registration and execution.
%%%-------------------------------------------------------------------
-module(tool_registry).
-behaviour(gen_server).

-export([start_link/0]).
-export([register/1, register/2]).
-export([unregister/1]).
-export([execute/2, execute/3]).
-export([list_tools/0]).
-export([get_tool_specs/0]).

%% gen_server callbacks
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(state, {
    tools :: #{atom() => tool_entry()}
}).

-record(tool_entry, {
    module   :: module(),
    name     :: atom(),
    spec     :: tool:json_schema(),
    timeout  :: timeout()
}).

%%%===================================================================
%%% API
%%%===================================================================

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

-spec register(Module :: module()) -> ok | {error, term()}.
register(Module) ->
    register(Module, #{}).

-spec register(Module :: module(), Opts :: map()) -> ok | {error, term()}.
register(Module, Opts) ->
    gen_server:call(?MODULE, {register, Module, Opts}).

-spec unregister(Name :: atom()) -> ok.
unregister(Name) ->
    gen_server:call(?MODULE, {unregister, Name}).

-spec execute(Name :: atom(), Args :: map()) -> {ok, term()} | {error, term()}.
execute(Name, Args) ->
    execute(Name, Args, 30000).

-spec execute(Name :: atom(), Args :: map(), Timeout :: timeout()) ->
    {ok, term()} | {error, term()}.
execute(Name, Args, Timeout) ->
    case gen_server:call(?MODULE, {get_tool, Name}) of
        {ok, #tool_entry{module = Module} = Entry} ->
            ActualTimeout = case Entry#tool_entry.timeout of
                undefined -> Timeout;
                T -> T
            end,
            execute_with_timeout(Module, Args, ActualTimeout);
        {error, not_found} ->
            {error, {tool_not_found, Name}}
    end.

-spec list_tools() -> [atom()].
list_tools() ->
    gen_server:call(?MODULE, list_tools).

-spec get_tool_specs() -> [map()].
get_tool_specs() ->
    gen_server:call(?MODULE, get_tool_specs).

%%%===================================================================
%%% gen_server callbacks
%%%===================================================================

init([]) ->
    {ok, #state{tools = #{}}}.

handle_call({register, Module, _Opts}, _From, #state{tools = Tools} = State) ->
    try
        Name = Module:name(),
        Description = Module:description(),
        Schema = Module:parameters_schema(),

        Timeout = case erlang:function_exported(Module, timeout, 0) of
            true -> Module:timeout();
            false -> undefined
        end,

        Entry = #tool_entry{
            module = Module,
            name = Name,
            spec = #{
                name => Name,
                description => Description,
                parameters => Schema
            },
            timeout = Timeout
        },

        NewTools = maps:put(Name, Entry, Tools),
        {reply, ok, State#state{tools = NewTools}}
    catch
        _:Reason ->
            {reply, {error, {invalid_tool, Reason}}, State}
    end;

handle_call({unregister, Name}, _From, #state{tools = Tools} = State) ->
    NewTools = maps:remove(Name, Tools),
    {reply, ok, State#state{tools = NewTools}};

handle_call({get_tool, Name}, _From, #state{tools = Tools} = State) ->
    case maps:get(Name, Tools, undefined) of
        undefined ->
            {reply, {error, not_found}, State};
        Entry ->
            {reply, {ok, Entry}, State}
    end;

handle_call(list_tools, _From, #state{tools = Tools} = State) ->
    {reply, maps:keys(Tools), State};

handle_call(get_tool_specs, _From, #state{tools = Tools} = State) ->
    Specs = [Entry#tool_entry.spec || Entry <- maps:values(Tools)],
    {reply, Specs, State};

handle_call(_Request, _From, State) ->
    {reply, {error, unknown_request}, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info(_Info, State) ->
    {noreply, State}.

%%%===================================================================
%%% Internal functions
%%%===================================================================

execute_with_timeout(Module, Args, Timeout) ->
    %% Validate if module supports it
    ValidatedArgs = case erlang:function_exported(Module, validate, 1) of
        true ->
            case Module:validate(Args) of
                ok -> Args;
                {error, _} = Err -> throw(Err)
            end;
        false ->
            Args
    end,

    %% Execute with timeout
    Parent = self(),
    Ref = make_ref(),

    Pid = spawn(fun() ->
        Result = try
            Module:execute(ValidatedArgs)
        catch
            Class:Reason:Stack ->
                {error, {Class, Reason, Stack}}
        end,
        Parent ! {Ref, Result}
    end),

    receive
        {Ref, Result} ->
            Result
    after Timeout ->
        exit(Pid, kill),
        {error, timeout}
    end.
```

### Memory Manager

```erlang
%%%-------------------------------------------------------------------
%%% @doc Memory manager with short-term (ETS) and long-term (Mnesia) storage.
%%%-------------------------------------------------------------------
-module(memory_manager).
-behaviour(gen_server).

-export([start_link/0]).
-export([store_short_term/3, get_short_term/2, delete_short_term/2]).
-export([store_long_term/3, get_long_term/2, query_long_term/2]).
-export([get_conversation_history/1, append_to_conversation/2]).

%% gen_server callbacks
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

%% Mnesia table records
-record(agent_memory, {
    key         :: {agent_id(), memory_key()},
    value       :: term(),
    timestamp   :: integer(),
    metadata    :: map()
}).

-record(conversation, {
    agent_id    :: agent_id(),
    messages    :: [map()],
    created     :: integer(),
    updated     :: integer()
}).

-type agent_id() :: atom() | binary().
-type memory_key() :: atom() | binary().

%%%===================================================================
%%% API
%%%===================================================================

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%% Short-term memory (ETS - fast, volatile)

-spec store_short_term(AgentId :: agent_id(), Key :: memory_key(), Value :: term()) -> ok.
store_short_term(AgentId, Key, Value) ->
    true = ets:insert(short_term_memory, {{AgentId, Key}, Value, erlang:system_time(millisecond)}),
    ok.

-spec get_short_term(AgentId :: agent_id(), Key :: memory_key()) ->
    {ok, term()} | {error, not_found}.
get_short_term(AgentId, Key) ->
    case ets:lookup(short_term_memory, {AgentId, Key}) of
        [{_, Value, _}] -> {ok, Value};
        [] -> {error, not_found}
    end.

-spec delete_short_term(AgentId :: agent_id(), Key :: memory_key()) -> ok.
delete_short_term(AgentId, Key) ->
    true = ets:delete(short_term_memory, {AgentId, Key}),
    ok.

%% Long-term memory (Mnesia - durable, distributed)

-spec store_long_term(AgentId :: agent_id(), Key :: memory_key(), Value :: term()) ->
    ok | {error, term()}.
store_long_term(AgentId, Key, Value) ->
    Record = #agent_memory{
        key = {AgentId, Key},
        value = Value,
        timestamp = erlang:system_time(millisecond),
        metadata = #{}
    },
    case mnesia:transaction(fun() -> mnesia:write(Record) end) of
        {atomic, ok} -> ok;
        {aborted, Reason} -> {error, Reason}
    end.

-spec get_long_term(AgentId :: agent_id(), Key :: memory_key()) ->
    {ok, term()} | {error, not_found | term()}.
get_long_term(AgentId, Key) ->
    case mnesia:transaction(fun() -> mnesia:read(agent_memory, {AgentId, Key}) end) of
        {atomic, [#agent_memory{value = Value}]} -> {ok, Value};
        {atomic, []} -> {error, not_found};
        {aborted, Reason} -> {error, Reason}
    end.

-spec query_long_term(AgentId :: agent_id(), Pattern :: term()) -> {ok, [term()]}.
query_long_term(AgentId, Pattern) ->
    MatchSpec = [{
        #agent_memory{key = {AgentId, '_'}, value = Pattern, _ = '_'},
        [],
        ['$_']
    }],
    case mnesia:transaction(fun() -> mnesia:select(agent_memory, MatchSpec) end) of
        {atomic, Results} ->
            {ok, [R#agent_memory.value || R <- Results]};
        {aborted, Reason} ->
            {error, Reason}
    end.

%% Conversation history

-spec get_conversation_history(AgentId :: agent_id()) -> {ok, [map()]} | {error, term()}.
get_conversation_history(AgentId) ->
    case mnesia:transaction(fun() -> mnesia:read(conversation, AgentId) end) of
        {atomic, [#conversation{messages = Messages}]} -> {ok, Messages};
        {atomic, []} -> {ok, []};
        {aborted, Reason} -> {error, Reason}
    end.

-spec append_to_conversation(AgentId :: agent_id(), Message :: map()) -> ok | {error, term()}.
append_to_conversation(AgentId, Message) ->
    Now = erlang:system_time(millisecond),

    F = fun() ->
        case mnesia:read(conversation, AgentId) of
            [Conv] ->
                NewConv = Conv#conversation{
                    messages = Conv#conversation.messages ++ [Message],
                    updated = Now
                },
                mnesia:write(NewConv);
            [] ->
                NewConv = #conversation{
                    agent_id = AgentId,
                    messages = [Message],
                    created = Now,
                    updated = Now
                },
                mnesia:write(NewConv)
        end
    end,

    case mnesia:transaction(F) of
        {atomic, ok} -> ok;
        {aborted, Reason} -> {error, Reason}
    end.

%%%===================================================================
%%% gen_server callbacks
%%%===================================================================

init([]) ->
    %% Create ETS table for short-term memory
    ets:new(short_term_memory, [
        named_table,
        public,
        {read_concurrency, true},
        {write_concurrency, true}
    ]),

    %% Initialize Mnesia tables
    ok = init_mnesia_tables(),

    %% Start cleanup timer for expired short-term memory
    erlang:send_after(60000, self(), cleanup_expired),

    {ok, #{}}.

handle_call(_Request, _From, State) ->
    {reply, {error, unknown_request}, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info(cleanup_expired, State) ->
    %% Clean up short-term memory older than 1 hour
    Cutoff = erlang:system_time(millisecond) - 3600000,
    ets:select_delete(short_term_memory, [{{'_', '_', '$1'}, [{'<', '$1', Cutoff}], [true]}]),

    erlang:send_after(60000, self(), cleanup_expired),
    {noreply, State};

handle_info(_Info, State) ->
    {noreply, State}.

%%%===================================================================
%%% Internal functions
%%%===================================================================

init_mnesia_tables() ->
    %% Create schema if not exists
    case mnesia:create_schema([node()]) of
        ok -> ok;
        {error, {_, {already_exists, _}}} -> ok
    end,

    ok = mnesia:start(),

    %% Create agent_memory table
    case mnesia:create_table(agent_memory, [
        {attributes, record_info(fields, agent_memory)},
        {disc_copies, [node()]},
        {type, set}
    ]) of
        {atomic, ok} -> ok;
        {aborted, {already_exists, agent_memory}} -> ok
    end,

    %% Create conversation table
    case mnesia:create_table(conversation, [
        {attributes, record_info(fields, conversation)},
        {disc_copies, [node()]},
        {type, set}
    ]) of
        {atomic, ok} -> ok;
        {aborted, {already_exists, conversation}} -> ok
    end,

    ok = mnesia:wait_for_tables([agent_memory, conversation], 30000),
    ok.
```

### LLM Client (Abstraction Layer)

```erlang
%%%-------------------------------------------------------------------
%%% @doc LLM client behaviour and OpenAI implementation.
%%%-------------------------------------------------------------------
-module(llm_client).

%% Behaviour callbacks
-callback chat_completion(Request :: map()) ->
    {ok, Response :: map()} | {error, term()}.

-callback models() -> [binary()].

-callback default_model() -> binary().

%% Optional callbacks
-callback stream_chat_completion(Request :: map(), Callback :: fun()) ->
    {ok, FinalResponse :: map()} | {error, term()}.

-optional_callbacks([stream_chat_completion/2]).

%% Common types
-type message() :: #{
    role := user | assistant | system | tool,
    content := binary(),
    name => binary(),
    tool_calls => [tool_call()],
    tool_call_id => binary()
}.

-type tool_call() :: #{
    id := binary(),
    type := function,
    function := #{
        name := binary(),
        arguments := binary()  %% JSON string
    }
}.

-type tool_spec() :: #{
    type := function,
    function := #{
        name := binary(),
        description := binary(),
        parameters := map()
    }
}.

-export_type([message/0, tool_call/0, tool_spec/0]).


%%%-------------------------------------------------------------------
%%% OpenAI Implementation
%%%-------------------------------------------------------------------
-module(llm_client_openai).
-behaviour(llm_client).

-export([chat_completion/1, models/0, default_model/0]).
-export([stream_chat_completion/2]).

-define(API_BASE, "https://api.openai.com/v1").
-define(DEFAULT_MODEL, <<"gpt-4-turbo">>).

models() ->
    [<<"gpt-4-turbo">>, <<"gpt-4">>, <<"gpt-3.5-turbo">>].

default_model() ->
    ?DEFAULT_MODEL.

-spec chat_completion(Request :: map()) -> {ok, map()} | {error, term()}.
chat_completion(Request) ->
    #{
        system := SystemPrompt,
        messages := Messages
    } = Request,

    Model = maps:get(model, Request, ?DEFAULT_MODEL),
    Tools = maps:get(tools, Request, []),

    %% Build request body
    Body = #{
        model => Model,
        messages => build_messages(SystemPrompt, Messages),
        tools => format_tools(Tools)
    },

    %% Get API key from environment
    ApiKey = os:getenv("OPENAI_API_KEY"),

    Headers = [
        {"Authorization", "Bearer " ++ ApiKey},
        {"Content-Type", "application/json"}
    ],

    JsonBody = jsx:encode(Body),

    case httpc:request(
        post,
        {?API_BASE ++ "/chat/completions", Headers, "application/json", JsonBody},
        [{timeout, 60000}],
        [{body_format, binary}]
    ) of
        {ok, {{_, 200, _}, _, RespBody}} ->
            parse_response(jsx:decode(RespBody, [return_maps]));
        {ok, {{_, StatusCode, _}, _, RespBody}} ->
            {error, {http_error, StatusCode, RespBody}};
        {error, Reason} ->
            {error, Reason}
    end.

stream_chat_completion(Request, Callback) ->
    %% Streaming implementation would use SSE
    %% Simplified here - would need proper SSE client
    case chat_completion(Request) of
        {ok, Response} ->
            Callback(Response),
            {ok, Response};
        Error ->
            Error
    end.

%%%===================================================================
%%% Internal functions
%%%===================================================================

build_messages(SystemPrompt, Messages) ->
    SystemMsg = #{role => <<"system">>, content => SystemPrompt},
    [SystemMsg | [format_message(M) || M <- Messages]].

format_message(#{role := Role, content := Content} = Msg) ->
    Base = #{role => atom_to_binary(Role), content => Content},

    %% Add tool_calls if present
    case maps:get(tool_calls, Msg, undefined) of
        undefined -> Base;
        ToolCalls -> Base#{tool_calls => ToolCalls}
    end.

format_tools([]) ->
    undefined;
format_tools(Tools) ->
    [#{
        type => <<"function">>,
        function => #{
            name => atom_to_binary(maps:get(name, T)),
            description => maps:get(description, T),
            parameters => maps:get(parameters, T)
        }
    } || T <- Tools].

parse_response(#{<<"choices">> := [Choice | _]}) ->
    #{<<"message">> := Message} = Choice,

    Response = #{
        role => assistant,
        content => maps:get(<<"content">>, Message, <<>>)
    },

    %% Parse tool calls if present
    FinalResponse = case maps:get(<<"tool_calls">>, Message, undefined) of
        undefined ->
            Response;
        ToolCalls ->
            Response#{tool_calls => [parse_tool_call(TC) || TC <- ToolCalls]}
    end,

    {ok, FinalResponse};

parse_response(Other) ->
    {error, {unexpected_response, Other}}.

parse_tool_call(#{<<"id">> := Id, <<"function">> := Func}) ->
    #{<<"name">> := Name, <<"arguments">> := Args} = Func,
    #{
        id => Id,
        name => binary_to_atom(Name),
        arguments => jsx:decode(Args, [return_maps])
    }.
```

### Checkpoint Manager

```erlang
%%%-------------------------------------------------------------------
%%% @doc Checkpoint manager for agent state persistence and recovery.
%%%-------------------------------------------------------------------
-module(checkpoint_manager).
-behaviour(gen_server).

-export([start_link/0]).
-export([save/2, load/1, list_checkpoints/1, delete/2]).
-export([enable_auto_checkpoint/2, disable_auto_checkpoint/1]).

%% gen_server callbacks
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(checkpoint, {
    id              :: {agent_id(), timestamp()},
    agent_id        :: agent_id(),
    timestamp       :: timestamp(),
    state           :: term(),
    metadata        :: map()
}).

-type agent_id() :: pid() | atom().
-type timestamp() :: integer().

-record(state, {
    auto_checkpoints :: #{pid() => {interval(), timer_ref()}}
}).

%%%===================================================================
%%% API
%%%===================================================================

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

-spec save(AgentId :: agent_id(), State :: map()) -> {ok, timestamp()} | {error, term()}.
save(AgentId, AgentState) ->
    Timestamp = erlang:system_time(millisecond),

    Checkpoint = #checkpoint{
        id = {AgentId, Timestamp},
        agent_id = AgentId,
        timestamp = Timestamp,
        state = AgentState,
        metadata = #{
            node => node(),
            saved_at => calendar:universal_time()
        }
    },

    case mnesia:transaction(fun() -> mnesia:write(Checkpoint) end) of
        {atomic, ok} ->
            %% Clean up old checkpoints (keep last 10)
            spawn(fun() -> cleanup_old_checkpoints(AgentId, 10) end),
            {ok, Timestamp};
        {aborted, Reason} ->
            {error, Reason}
    end.

-spec load(AgentId :: agent_id()) -> {ok, map()} | {error, not_found | term()}.
load(AgentId) ->
    %% Load most recent checkpoint
    MatchSpec = [{
        #checkpoint{agent_id = AgentId, _ = '_'},
        [],
        ['$_']
    }],

    case mnesia:transaction(fun() -> mnesia:select(checkpoint, MatchSpec) end) of
        {atomic, []} ->
            {error, not_found};
        {atomic, Checkpoints} ->
            %% Get most recent
            Sorted = lists:sort(
                fun(A, B) -> A#checkpoint.timestamp > B#checkpoint.timestamp end,
                Checkpoints
            ),
            [Latest | _] = Sorted,
            {ok, Latest#checkpoint.state};
        {aborted, Reason} ->
            {error, Reason}
    end.

-spec list_checkpoints(AgentId :: agent_id()) -> {ok, [map()]}.
list_checkpoints(AgentId) ->
    MatchSpec = [{
        #checkpoint{agent_id = AgentId, _ = '_'},
        [],
        ['$_']
    }],

    case mnesia:transaction(fun() -> mnesia:select(checkpoint, MatchSpec) end) of
        {atomic, Checkpoints} ->
            Info = [#{
                timestamp => C#checkpoint.timestamp,
                metadata => C#checkpoint.metadata
            } || C <- Checkpoints],
            {ok, Info};
        {aborted, Reason} ->
            {error, Reason}
    end.

-spec delete(AgentId :: agent_id(), Timestamp :: timestamp()) -> ok | {error, term()}.
delete(AgentId, Timestamp) ->
    case mnesia:transaction(fun() -> mnesia:delete(checkpoint, {AgentId, Timestamp}) end) of
        {atomic, ok} -> ok;
        {aborted, Reason} -> {error, Reason}
    end.

-spec enable_auto_checkpoint(AgentPid :: pid(), Interval :: pos_integer()) -> ok.
enable_auto_checkpoint(AgentPid, Interval) ->
    gen_server:call(?MODULE, {enable_auto, AgentPid, Interval}).

-spec disable_auto_checkpoint(AgentPid :: pid()) -> ok.
disable_auto_checkpoint(AgentPid) ->
    gen_server:call(?MODULE, {disable_auto, AgentPid}).

%%%===================================================================
%%% gen_server callbacks
%%%===================================================================

init([]) ->
    %% Initialize Mnesia table
    ok = init_checkpoint_table(),
    {ok, #state{auto_checkpoints = #{}}}.

handle_call({enable_auto, Pid, Interval}, _From, #state{auto_checkpoints = Auto} = State) ->
    %% Cancel existing timer if any
    case maps:get(Pid, Auto, undefined) of
        undefined -> ok;
        {_, OldRef} -> erlang:cancel_timer(OldRef)
    end,

    %% Monitor the agent
    monitor(process, Pid),

    %% Start new timer
    Ref = erlang:send_after(Interval, self(), {auto_checkpoint, Pid, Interval}),
    NewAuto = maps:put(Pid, {Interval, Ref}, Auto),

    {reply, ok, State#state{auto_checkpoints = NewAuto}};

handle_call({disable_auto, Pid}, _From, #state{auto_checkpoints = Auto} = State) ->
    case maps:get(Pid, Auto, undefined) of
        undefined ->
            {reply, ok, State};
        {_, Ref} ->
            erlang:cancel_timer(Ref),
            NewAuto = maps:remove(Pid, Auto),
            {reply, ok, State#state{auto_checkpoints = NewAuto}}
    end;

handle_call(_Request, _From, State) ->
    {reply, {error, unknown_request}, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info({auto_checkpoint, Pid, Interval}, #state{auto_checkpoints = Auto} = State) ->
    case is_process_alive(Pid) of
        true ->
            %% Request state from agent and save checkpoint
            case catch gen_server:call(Pid, get_state_for_checkpoint, 5000) of
                {'EXIT', _} ->
                    ok;
                AgentState ->
                    save(Pid, AgentState)
            end,

            %% Schedule next checkpoint
            Ref = erlang:send_after(Interval, self(), {auto_checkpoint, Pid, Interval}),
            NewAuto = maps:put(Pid, {Interval, Ref}, Auto),
            {noreply, State#state{auto_checkpoints = NewAuto}};
        false ->
            NewAuto = maps:remove(Pid, Auto),
            {noreply, State#state{auto_checkpoints = NewAuto}}
    end;

handle_info({'DOWN', _Ref, process, Pid, _Reason}, #state{auto_checkpoints = Auto} = State) ->
    case maps:get(Pid, Auto, undefined) of
        undefined ->
            {noreply, State};
        {_, TimerRef} ->
            erlang:cancel_timer(TimerRef),
            NewAuto = maps:remove(Pid, Auto),
            {noreply, State#state{auto_checkpoints = NewAuto}}
    end;

handle_info(_Info, State) ->
    {noreply, State}.

%%%===================================================================
%%% Internal functions
%%%===================================================================

init_checkpoint_table() ->
    case mnesia:create_table(checkpoint, [
        {attributes, record_info(fields, checkpoint)},
        {disc_copies, [node()]},
        {type, set},
        {index, [agent_id, timestamp]}
    ]) of
        {atomic, ok} -> ok;
        {aborted, {already_exists, checkpoint}} -> ok
    end,

    ok = mnesia:wait_for_tables([checkpoint], 30000),
    ok.

cleanup_old_checkpoints(AgentId, Keep) ->
    case list_checkpoints(AgentId) of
        {ok, Checkpoints} when length(Checkpoints) > Keep ->
            Sorted = lists:sort(
                fun(A, B) ->
                    maps:get(timestamp, A) > maps:get(timestamp, B)
                end,
                Checkpoints
            ),
            ToDelete = lists:nthtail(Keep, Sorted),
            lists:foreach(
                fun(#{timestamp := T}) -> delete(AgentId, T) end,
                ToDelete
            );
        _ ->
            ok
    end.
```

---

## Message Protocol

### Message Types

```erlang
%%%-------------------------------------------------------------------
%%% @doc Message types for inter-agent communication.
%%%-------------------------------------------------------------------
-module(agent_messages).

%% Message type definitions as records for pattern matching efficiency

%% User input message
-record(user_message, {
    content     :: binary(),
    metadata    :: map(),
    timestamp   :: integer()
}).

%% Agent response message
-record(agent_response, {
    content     :: binary(),
    tool_calls  :: [#tool_call{}] | undefined,
    metadata    :: map(),
    timestamp   :: integer()
}).

%% Tool execution request
-record(tool_call, {
    id          :: binary(),
    name        :: atom(),
    arguments   :: map()
}).

%% Tool execution result
-record(tool_result, {
    call_id     :: binary(),
    name        :: atom(),
    result      :: term(),
    error       :: term() | undefined
}).

%% Handoff request between agents
-record(handoff, {
    from_agent  :: agent_ref(),
    to_agent    :: agent_ref(),
    context     :: term(),
    conversation :: [message()],
    reason      :: binary()
}).

%% System control messages
-record(agent_control, {
    command     :: start | stop | pause | resume | checkpoint,
    params      :: map()
}).

%% Event notification (for pub/sub)
-record(agent_event, {
    type        :: atom(),
    source      :: agent_ref(),
    data        :: term(),
    timestamp   :: integer()
}).

%% Export types
-export_type([
    user_message/0,
    agent_response/0,
    tool_call/0,
    tool_result/0,
    handoff/0,
    agent_control/0,
    agent_event/0
]).

%% Helper functions for creating messages
-export([
    new_user_message/1, new_user_message/2,
    new_tool_call/3,
    new_handoff/4
]).

new_user_message(Content) ->
    new_user_message(Content, #{}).

new_user_message(Content, Metadata) ->
    #user_message{
        content = Content,
        metadata = Metadata,
        timestamp = erlang:system_time(millisecond)
    }.

new_tool_call(Name, Arguments, Id) ->
    #tool_call{
        id = Id,
        name = Name,
        arguments = Arguments
    }.

new_handoff(FromAgent, ToAgent, Context, Reason) ->
    #handoff{
        from_agent = FromAgent,
        to_agent = ToAgent,
        context = Context,
        conversation = [],  %% Filled by sender
        reason = Reason
    }.
```

---

## Supervision Tree Design

```
                         ┌─────────────────────────────┐
                         │     agent_framework_sup     │
                         │      (one_for_one)          │
                         └──────────────┬──────────────┘
                                        │
        ┌───────────────┬───────────────┼───────────────┬───────────────┐
        │               │               │               │               │
        ▼               ▼               ▼               ▼               ▼
┌───────────────┐┌───────────────┐┌───────────────┐┌───────────────┐┌───────────────┐
│agent_registry ││tool_registry  ││memory_manager ││checkpoint_mgr ││ agent_pool_sup│
│  (worker)     ││  (worker)     ││  (worker)     ││  (worker)     ││(simple_1_for_1)│
└───────────────┘└───────────────┘└───────────────┘└───────────────┘└───────┬───────┘
                                                                            │
                                                                    ┌───────┴───────┐
                                                                    │               │
                                                              ┌─────┴─────┐   ┌─────┴─────┐
                                                              │  Agent 1  │   │  Agent N  │
                                                              │(gen_server│   │(gen_server│
                                                              │    or     │   │    or     │
                                                              │gen_statem)│   │gen_statem)│
                                                              └───────────┘   └───────────┘
```

### Application Module

```erlang
%%%-------------------------------------------------------------------
%%% @doc Agent framework application.
%%%-------------------------------------------------------------------
-module(agent_framework_app).
-behaviour(application).

-export([start/2, stop/1]).

start(_StartType, _StartArgs) ->
    agent_framework_sup:start_link().

stop(_State) ->
    ok.


%%%-------------------------------------------------------------------
%%% @doc Top-level supervisor.
%%%-------------------------------------------------------------------
-module(agent_framework_sup).
-behaviour(supervisor).

-export([start_link/0]).
-export([init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    SupFlags = #{
        strategy => one_for_one,
        intensity => 10,
        period => 60
    },

    Children = [
        %% Agent Registry
        #{
            id => agent_registry,
            start => {agent_registry, start_link, []},
            restart => permanent,
            type => worker
        },

        %% Tool Registry
        #{
            id => tool_registry,
            start => {tool_registry, start_link, []},
            restart => permanent,
            type => worker
        },

        %% Memory Manager
        #{
            id => memory_manager,
            start => {memory_manager, start_link, []},
            restart => permanent,
            type => worker
        },

        %% Checkpoint Manager
        #{
            id => checkpoint_manager,
            start => {checkpoint_manager, start_link, []},
            restart => permanent,
            type => worker
        },

        %% Agent Pool Supervisor (dynamic children)
        #{
            id => agent_pool_sup,
            start => {agent_pool_sup, start_link, []},
            restart => permanent,
            type => supervisor
        }
    ],

    {ok, {SupFlags, Children}}.


%%%-------------------------------------------------------------------
%%% @doc Agent pool supervisor for dynamic agent creation.
%%%-------------------------------------------------------------------
-module(agent_pool_sup).
-behaviour(supervisor).

-export([start_link/0]).
-export([start_agent/2, start_agent/3]).
-export([stop_agent/1]).
-export([init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

-spec start_agent(Module :: module(), Args :: term()) -> {ok, pid()} | {error, term()}.
start_agent(Module, Args) ->
    start_agent(Module, Args, []).

-spec start_agent(Module :: module(), Args :: term(), Opts :: list()) ->
    {ok, pid()} | {error, term()}.
start_agent(Module, Args, Opts) ->
    supervisor:start_child(?MODULE, [Module, Args, Opts]).

-spec stop_agent(Pid :: pid()) -> ok.
stop_agent(Pid) ->
    supervisor:terminate_child(?MODULE, Pid).

init([]) ->
    SupFlags = #{
        strategy => simple_one_for_one,
        intensity => 100,
        period => 60
    },

    %% Child spec template for agents
    ChildSpec = #{
        id => agent,
        start => {agent, start_link, []},  %% Args appended by start_child
        restart => transient,
        shutdown => 5000,
        type => worker
    },

    {ok, {SupFlags, [ChildSpec]}}.
```

---

## Orchestration Patterns

### 1. Supervisor Pattern

```erlang
%%%-------------------------------------------------------------------
%%% @doc Supervisor agent that delegates to specialist agents.
%%%-------------------------------------------------------------------
-module(supervisor_agent).
-behaviour(agent).

-export([init/1, handle_message/2, handle_tool_result/3]).
-export([system_prompt/1, available_tools/1]).

-record(state, {
    specialists :: #{atom() => pid()},
    pending_delegation :: map() | undefined
}).

init(#{specialists := Specialists}) ->
    {ok, #state{specialists = Specialists}}.

system_prompt(_State) ->
    <<"You are a supervisor agent. Analyze user requests and delegate to the "
      "appropriate specialist agent. Available specialists: coding, research, writing. "
      "Use the delegate_to tool to hand off tasks.">>.

available_tools(#state{specialists = Specialists}) ->
    SpecialistNames = maps:keys(Specialists),
    [#{
        name => delegate_to,
        description => <<"Delegate a task to a specialist agent">>,
        parameters => #{
            type => object,
            properties => #{
                specialist => #{
                    type => string,
                    enum => [atom_to_binary(S) || S <- SpecialistNames],
                    description => <<"The specialist to delegate to">>
                },
                task => #{
                    type => string,
                    description => <<"The task description for the specialist">>
                }
            },
            required => [specialist, task]
        }
    }].

handle_message(_Message, State) ->
    %% LLM will decide whether to delegate or respond directly
    %% This is handled by the agent behaviour's LLM invocation
    {continue, State}.

handle_tool_result(delegate_to, #{specialist := SpecName, task := Task}, State) ->
    #state{specialists = Specialists} = State,
    Specialist = binary_to_atom(SpecName),

    case maps:get(Specialist, Specialists, undefined) of
        undefined ->
            {reply, #{role => assistant,
                     content => <<"Unknown specialist: ", SpecName/binary>>}, State};
        Pid ->
            %% Delegate to specialist
            case agent:send_message(Pid, Task) of
                {ok, Response} ->
                    {reply, Response, State};
                {error, Reason} ->
                    {reply, #{role => assistant,
                             content => <<"Specialist error: ",
                                         (iolist_to_binary(io_lib:format("~p", [Reason])))/binary>>},
                     State}
            end
    end.
```

### 2. Handoff Pattern (Swarm-style)

```erlang
%%%-------------------------------------------------------------------
%%% @doc Triage agent that hands off to specialists.
%%%-------------------------------------------------------------------
-module(triage_agent).
-behaviour(agent).

-export([init/1, handle_message/2, handle_tool_result/3, handle_handoff/3]).
-export([system_prompt/1, available_tools/1]).

-record(state, {
    specialists :: #{atom() => pid()}
}).

init(#{specialists := Specialists}) ->
    {ok, #state{specialists = Specialists}}.

system_prompt(_State) ->
    <<"You are a triage agent. Determine the user's needs and hand off to the "
      "appropriate specialist. Use handoff_to when the user needs specialized help. "
      "You can handle simple greetings and clarifying questions yourself.">>.

available_tools(#state{specialists = Specialists}) ->
    [#{
        name => handoff_to,
        description => <<"Transfer the conversation to a specialist">>,
        parameters => #{
            type => object,
            properties => #{
                specialist => #{
                    type => string,
                    enum => [atom_to_binary(S) || S <- maps:keys(Specialists)],
                    description => <<"The specialist to hand off to">>
                },
                context => #{
                    type => string,
                    description => <<"Context/summary for the specialist">>
                }
            },
            required => [specialist]
        }
    }].

handle_message(_Message, State) ->
    {continue, State}.

handle_tool_result(handoff_to, #{specialist := SpecName} = Args, State) ->
    #state{specialists = Specialists} = State,
    Specialist = binary_to_atom(SpecName),
    Context = maps:get(context, Args, <<>>),

    case maps:get(Specialist, Specialists, undefined) of
        undefined ->
            {reply, #{role => assistant,
                     content => <<"Unknown specialist">>}, State};
        TargetPid ->
            %% Perform handoff - transfers full conversation
            {handoff, TargetPid, #{summary => Context}, State}
    end.

%% Called when this agent receives a handoff
handle_handoff(FromAgent, Context, State) ->
    %% Accept the handoff
    {ok, State}.
```

### 3. Pipeline Pattern

```erlang
%%%-------------------------------------------------------------------
%%% @doc Pipeline orchestrator for sequential agent processing.
%%%-------------------------------------------------------------------
-module(pipeline_orchestrator).
-behaviour(gen_server).

-export([start_link/1]).
-export([execute/2]).

%% gen_server callbacks
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(state, {
    stages :: [stage()],
    current_execution :: map() | undefined
}).

-type stage() :: #{
    name := atom(),
    agent := pid(),
    transform := fun((term()) -> term()) | undefined
}.

start_link(Stages) ->
    gen_server:start_link(?MODULE, Stages, []).

-spec execute(Pid :: pid(), Input :: binary()) -> {ok, term()} | {error, term()}.
execute(Pid, Input) ->
    gen_server:call(Pid, {execute, Input}, infinity).

init(Stages) ->
    {ok, #state{stages = Stages}}.

handle_call({execute, Input}, From, #state{stages = Stages} = State) ->
    %% Execute pipeline sequentially
    Self = self(),
    spawn_link(fun() ->
        Result = execute_stages(Stages, Input),
        gen_server:reply(From, Result)
    end),
    {noreply, State};

handle_call(_Request, _From, State) ->
    {reply, {error, unknown_request}, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info(_Info, State) ->
    {noreply, State}.

%%%===================================================================
%%% Internal functions
%%%===================================================================

execute_stages([], Result) ->
    {ok, Result};
execute_stages([Stage | Rest], Input) ->
    #{name := Name, agent := Agent} = Stage,
    Transform = maps:get(transform, Stage, fun(X) -> X end),

    %% Transform input for this stage
    StageInput = Transform(Input),

    %% Execute stage
    case agent:send_message(Agent, format_input(StageInput)) of
        {ok, #{content := Output}} ->
            execute_stages(Rest, Output);
        {error, Reason} ->
            {error, {stage_failed, Name, Reason}}
    end.

format_input(Input) when is_binary(Input) -> Input;
format_input(Input) -> iolist_to_binary(io_lib:format("~p", [Input])).
```

### 4. Pub/Sub Pattern (using pg)

```erlang
%%%-------------------------------------------------------------------
%%% @doc Event-driven agent coordination using process groups.
%%%-------------------------------------------------------------------
-module(agent_pubsub).

-export([subscribe/2, unsubscribe/2]).
-export([publish/2, publish/3]).
-export([broadcast_to_capability/2]).

%% Subscribe an agent to a topic
-spec subscribe(Topic :: atom(), Agent :: pid()) -> ok.
subscribe(Topic, Agent) ->
    pg:join({agent_topic, Topic}, Agent).

%% Unsubscribe an agent from a topic
-spec unsubscribe(Topic :: atom(), Agent :: pid()) -> ok.
unsubscribe(Topic, Agent) ->
    pg:leave({agent_topic, Topic}, Agent).

%% Publish an event to a topic
-spec publish(Topic :: atom(), Event :: term()) -> ok.
publish(Topic, Event) ->
    publish(Topic, Event, #{}).

-spec publish(Topic :: atom(), Event :: term(), Opts :: map()) -> ok.
publish(Topic, Event, Opts) ->
    Subscribers = pg:get_members({agent_topic, Topic}),

    Message = #agent_event{
        type = Topic,
        source = self(),
        data = Event,
        timestamp = erlang:system_time(millisecond)
    },

    %% Determine delivery mode
    case maps:get(mode, Opts, async) of
        async ->
            [gen_server:cast(Pid, {event, Message}) || Pid <- Subscribers];
        sync ->
            [gen_server:call(Pid, {event, Message}, 5000) || Pid <- Subscribers]
    end,

    ok.

%% Broadcast to all agents with a specific capability
-spec broadcast_to_capability(Capability :: atom(), Message :: term()) -> ok.
broadcast_to_capability(Capability, Message) ->
    Agents = pg:get_members({capability, Capability}),
    [gen_server:cast(Pid, Message) || Pid <- Agents],
    ok.
```

---

## Example Implementations

### Simple Conversational Agent

```erlang
%%%-------------------------------------------------------------------
%%% @doc Simple conversational agent example.
%%%-------------------------------------------------------------------
-module(chat_agent).
-behaviour(agent).

-export([init/1, handle_message/2, handle_tool_result/3]).
-export([system_prompt/1, available_tools/1]).

-record(state, {
    name :: binary(),
    personality :: binary()
}).

init(#{name := Name, personality := Personality}) ->
    {ok, #state{name = Name, personality = Personality}}.

system_prompt(#state{name = Name, personality = Personality}) ->
    <<"Your name is ", Name/binary, ". ", Personality/binary>>.

available_tools(_State) ->
    %% No tools for simple chat
    [].

handle_message(_Message, State) ->
    %% Just let the LLM respond
    {continue, State}.

handle_tool_result(_Tool, _Result, State) ->
    {continue, State}.
```

### ReAct Agent (State Machine)

```erlang
%%%-------------------------------------------------------------------
%%% @doc Research agent using ReAct pattern.
%%%-------------------------------------------------------------------
-module(research_agent).
-behaviour(agent_statem).

-export([init/1, system_prompt/1, available_tools/1]).
-export([handle_thinking_result/2, handle_observation/3]).

-record(data, {
    topic :: binary(),
    findings :: [binary()],
    sources :: [binary()]
}).

init(#{topic := Topic}) ->
    {ok, #data{topic = Topic, findings = [], sources = []}}.

system_prompt(#data{topic = Topic}) ->
    <<"You are a research agent investigating: ", Topic/binary, "\n\n"
      "Use the web_search tool to find information. "
      "Analyze results carefully and search for multiple perspectives. "
      "When you have gathered enough information, provide a comprehensive summary.">>.

available_tools(_Data) ->
    [#{
        name => web_search,
        description => <<"Search the web for information">>,
        parameters => #{
            type => object,
            properties => #{
                query => #{type => string, description => <<"Search query">>}
            },
            required => [query]
        }
    }].

handle_thinking_result(Response, Data) ->
    case maps:get(tool_calls, Response, []) of
        [] ->
            %% No tool calls - this is the final answer
            Content = maps:get(content, Response, <<>>),
            {final_answer, Content, Data};

        [ToolCall | _] ->
            %% Tool call requested
            {tool_call, ToolCall, Data}
    end.

handle_observation(web_search, Result, #data{findings = Findings, sources = Sources} = Data) ->
    %% Add findings from search results
    NewFindings = case Result of
        {ok, Results} ->
            [maps:get(snippet, R, <<>>) || R <- Results] ++ Findings;
        {error, _} ->
            Findings
    end,

    NewSources = case Result of
        {ok, Results} ->
            [maps:get(url, R, <<>>) || R <- Results] ++ Sources;
        {error, _} ->
            Sources
    end,

    {continue, Data#data{findings = NewFindings, sources = NewSources}}.
```

### Usage Example

```erlang
%% Start the framework
application:start(agent_framework).

%% Register tools
tool_registry:register(tool_web_search),
tool_registry:register(tool_calculator),

%% Create a simple chat agent
{ok, ChatAgent} = agent_pool_sup:start_agent(chat_agent, #{
    name => <<"Assistant">>,
    personality => <<"You are helpful and friendly.">>
}).

%% Register it
agent_registry:register(assistant, ChatAgent, #{
    capabilities => [chat, general]
}).

%% Send a message
{ok, Response} = agent:send_message(ChatAgent, <<"Hello! How are you?">>).

%% Create a ReAct research agent
{ok, Researcher} = agent_pool_sup:start_agent(research_agent, #{
    topic => <<"climate change solutions">>
}, [{llm_client, llm_client_openai}]).

%% Submit a research task
{ok, Report} = agent_statem:submit_task(Researcher,
    <<"Research the latest developments in carbon capture technology">>).

%% Create a supervisor with specialists
{ok, CodingAgent} = agent_pool_sup:start_agent(coding_agent, #{}),
{ok, WritingAgent} = agent_pool_sup:start_agent(writing_agent, #{}),

{ok, Supervisor} = agent_pool_sup:start_agent(supervisor_agent, #{
    specialists => #{
        coding => CodingAgent,
        writing => WritingAgent
    }
}).

%% The supervisor will delegate appropriately
{ok, _} = agent:send_message(Supervisor, <<"Write a Python function to sort a list">>).
```

---

## Summary

This architecture provides:

1. **Behaviours**: `agent`, `agent_statem`, `tool` - standard interfaces
2. **Infrastructure**: Registry, memory, checkpoints, LLM abstraction
3. **Patterns**: Supervisor, handoff, pipeline, pub/sub
4. **OTP Integration**: Supervision trees, gen_server/gen_statem, pg, Mnesia

The design leverages Erlang/OTP's strengths (fault tolerance, distribution, lightweight processes) while providing the abstractions needed for modern AI agent development (LLM integration, tool execution, memory management).
