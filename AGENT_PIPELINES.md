# XAgent Workflow Architecture & Agent Pipelines

This document describes the complete workflow architecture of XAgent, detailing how agents interact, how data flows through the system, and how the orchestration logic coordinates the entire pipeline from user query to final answer.

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Main Entry Points](#main-entry-points)
- [Core Components](#core-components)
- [Agent Pipeline Flow](#agent-pipeline-flow)
- [Detailed Agent Interactions](#detailed-agent-interactions)
- [Data Flow & Data Structures](#data-flow--data-structures)
- [Search Algorithms](#search-algorithms)
- [Control Flow Summary](#control-flow-summary)

---

## Architecture Overview

XAgent uses a hierarchical, multi-agent architecture that breaks down complex tasks through iterative planning, execution, and refinement. The system follows a two-loop structure:

1. **Outer Loop** (`TaskHandler.outer_loop`): Manages the overall task execution, plan generation, and plan refinement
2. **Inner Loop** (`TaskHandler.inner_loop`): Executes individual subtasks using search algorithms (ReACT)

```
User Query
    ↓
CommandLine / XAgentServer
    ↓
XAgentCoreComponents (Initialization)
    ↓
TaskHandler.outer_loop()
    ↓
┌──────────────────────────────────────────┐
│  Outer Loop (Plan Management)            │
│  ┌────────────────────────────────────┐  │
│  │ 1. Plan Generation                 │  │
│  │ 2. Plan Iteration (Memory System)  │  │
│  │ 3. For each subtask:               │  │
│  │    ├─ Inner Loop (Execution)       │  │
│  │    ├─ Posterior Processing         │  │
│  │    └─ Plan Refinement (if needed)  │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
    ↓
Final Answer
```

---

## Main Entry Points

### 1. Command Line Entry (`run.py`)
```python
run.py → command.py → CommandLine.start() → XAgentServer.interact()
```

**Flow:**
- `run.py`: Parses command-line arguments (task, mode, config, etc.)
- `CommandLine.__init__()`: Initializes the client, database interaction, and logger
- `CommandLine.run()`: Creates interaction record in database
- `CommandLine.task_handler()`: Starts the XAgentServer

### 2. XAgentServer Entry (`XAgentServer/server.py`)
```python
XAgentServer.interact() → TaskHandler.outer_loop()
```

**Flow:**
- Loads configuration from `CONFIG`
- Builds `XAgentParam` with user query (task, role, plan)
- Initializes `XAgentCoreComponents`
- Creates `TaskHandler` instance
- Invokes `TaskHandler.outer_loop()` to start execution

---

## Core Components

### XAgentCoreComponents (`XAgent/core.py`)
Central registry for all system components:

```python
class XAgentCoreComponents:
    - logger: Logging system
    - recorder: Running recorder for persistence
    - toolserver_interface: Interface to tool execution server
    - function_handler: Handles tool calls and function execution
    - working_memory_agent: Memory system for cross-subtask communication
    - agent_dispatcher: Dispatches tasks to appropriate agents
    - interaction: User interaction handler
```

**Initialization Order:**
1. `register_interaction()`: Register interaction handler
2. `register_logger()`: Setup logging
3. `resister_recorder()`: Setup recording/persistence
4. `register_toolserver_interface()`: Connect to tool server
5. `register_function_handler()`: Setup function/tool handler
6. `register_working_memory_function()`: Setup memory system
7. `register_agent_dispatcher()`: Register all available agents
8. `register_vector_db_interface()`: Setup vector DB (optional)

### Agent Dispatcher (`XAgent/agent/dispatcher.py`)
Manages agent creation and routing based on required abilities:

```python
class XAgentDispatcher:
    - Registers agents by their abilities
    - Dispatches tasks to appropriate agents
    - Optionally refines prompts for specific agents
    
    Registered Agents:
    - PlanGenerateAgent (RequiredAbilities.plan_generation)
    - PlanRefineAgent (RequiredAbilities.plan_refinement)
    - ToolAgent (RequiredAbilities.tool_tree_search)
    - ReflectAgent (RequiredAbilities.reflection)
```

---

## Agent Pipeline Flow

### Complete Flow: User Query → Final Answer

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. INITIALIZATION PHASE                                         │
└─────────────────────────────────────────────────────────────────┘
    User provides query
    ↓
    XAgentCoreComponents initialized
    ↓
    TaskHandler created with query

┌─────────────────────────────────────────────────────────────────┐
│ 2. PLAN GENERATION PHASE (Outer Loop - Step 1)                 │
└─────────────────────────────────────────────────────────────────┘
    TaskHandler.outer_loop() starts
    ↓
    PlanAgent.initial_plan_generation()
    ↓
    AgentDispatcher.dispatch(RequiredAbilities.plan_generation)
    ↓
    PlanGenerateAgent created with prompts
    ↓
    PlanGenerateAgent.parse() → LLM call
    ↓
    Parse LLM response (subtask list)
    ↓
    Create Plan tree structure (root + subtasks as children)
    ↓
    plan_agent.latest_plan populated

┌─────────────────────────────────────────────────────────────────┐
│ 3. PLAN ITERATION PHASE (Outer Loop - Step 2)                  │
└─────────────────────────────────────────────────────────────────┘
    PlanAgent.plan_iterate_based_on_memory_system()
    ↓
    (Currently not implemented - placeholder for future memory integration)

┌─────────────────────────────────────────────────────────────────┐
│ 4. SUBTASK EXECUTION PHASE (Outer Loop - Step 3)               │
└─────────────────────────────────────────────────────────────────┘
    For each subtask in plan.children:
    
    now_dealing_task = plan.children[0]  # Get first TODO subtask
    ↓
    ┌────────────────────────────────────────────────────────────┐
    │ 4a. INNER LOOP - Tool Agent Execution                     │
    └────────────────────────────────────────────────────────────┘
        TaskHandler.inner_loop(now_dealing_task)
        ↓
        AgentDispatcher.dispatch(RequiredAbilities.tool_tree_search)
        ↓
        ToolAgent created with task description
        ↓
        ReACTChainSearch initialized
        ↓
        ReACTChainSearch.run()
        ↓
        ┌──────────────────────────────────────────────────────┐
        │ REACT CHAIN (Iterative Step-by-Step Execution)      │
        └──────────────────────────────────────────────────────┘
        while depth < max_subtask_chain_length:
            ↓
            Create message sequence (current state + history)
            ↓
            ToolAgent.parse() → LLM call with:
                - Thoughts (reasoning, plan, criticism)
                - Command (tool_name, arguments)
            ↓
            Create ToolNode from LLM response
            ↓
            FunctionHandler.handle_tool_call(node)
            ↓
            ┌──────────────────────────────────────────────┐
            │ Tool Execution Path                          │
            └──────────────────────────────────────────────┘
            if tool == "subtask_submit":
                Mark task as SUCCESS/FAILED
                Check if plan refinement needed
                BREAK
            elif tool == "ask_human_for_help":
                Request human input
                Continue
            else:
                ToolServerInterface.execute_command_client()
                Get tool result
                Continue to next step
        ↓
        Return finish_node (final ToolNode with complete process)
    
    ↓
    now_dealing_task.process_node = finish_node
    
    ┌────────────────────────────────────────────────────────────┐
    │ 4b. POSTERIOR PROCESSING - Reflection Agent                │
    └────────────────────────────────────────────────────────────┘
        TaskHandler.posterior_process(now_dealing_task)
        ↓
        get_posterior_knowledge()
        ↓
        AgentDispatcher.dispatch(RequiredAbilities.reflection)
        ↓
        ReflectAgent created with:
            - all_plan: Complete plan structure
            - terminal_plan: Current completed subtask
            - action_process: Sequence of actions taken
        ↓
        ReflectAgent.parse() → LLM call
        ↓
        Extract posterior knowledge:
            - summary: Condensed summary of actions
            - reflection_of_plan: Plan execution reflection
            - reflection_of_tool: Tool usage reflection
        ↓
        Update now_dealing_task.data with reflections
    
    ↓
    WorkingMemoryAgent.register_task(now_dealing_task)
    
    ┌────────────────────────────────────────────────────────────┐
    │ 4c. PLAN REFINEMENT (Conditional)                          │
    └────────────────────────────────────────────────────────────┘
    if search_method.need_for_plan_refine:
        ↓
        PlanAgent.plan_refine_mode(now_dealing_task)
        ↓
        AgentDispatcher.dispatch(RequiredAbilities.plan_refinement)
        ↓
        PlanRefineAgent created
        ↓
        Create PlanRefineChain (history of refinements)
        ↓
        while modify_steps < max_plan_refine_chain_length:
            ↓
            PlanRefineAgent.parse() → LLM call with:
                - Current plan state
                - Refine suggestions from subtask
                - Workspace file structure
            ↓
            Parse operation: split/add/delete/exit
            ↓
            ┌──────────────────────────────────────────────┐
            │ Plan Operations                              │
            └──────────────────────────────────────────────┘
            if operation == 'split':
                Split subtask into smaller subtasks
                Plan.make_relation(parent, children)
            elif operation == 'add':
                Add new subtask after specified task
            elif operation == 'delete':
                Remove subtask from plan tree
            elif operation == 'exit':
                Exit refinement mode
            ↓
            Register refinement in PlanRefineChain
            ↓
            if operation == 'exit' or 'success':
                BREAK
    
    ↓
    now_dealing_task = Plan.pop_next_subtask(now_dealing_task)
    ↓
    Repeat from Step 4a for next subtask

┌─────────────────────────────────────────────────────────────────┐
│ 5. COMPLETION PHASE                                             │
└─────────────────────────────────────────────────────────────────┘
    All subtasks completed
    ↓
    TaskHandler.outer_loop() returns
    ↓
    XAgentCoreComponents.close()
    ↓
    Download files, cleanup resources
    ↓
    Final answer returned to user
```

---

## Detailed Agent Pipelines with Prompts

Эта секция описывает детальные пайплайны для каждого агента, включая промты, входные и выходные данные.

### Pipeline 1: Plan Generation Pipeline

**Цель:** Преобразовать пользовательский запрос в иерархическое дерево подзадач.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PLAN GENERATION PIPELINE                         │
└─────────────────────────────────────────────────────────────────────┘

INPUT DATA:
┌────────────────────────────────────────────────────────────────┐
│ User Query (AutoGPTQuery)                                      │
│ ├─ task: string (пользовательский запрос)                     │
│ ├─ role: string (роль агента, опционально)                    │
│ └─ plan: list[string] (начальный план, опционально)           │
└────────────────────────────────────────────────────────────────┘
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 1: Initialize Plan Root Node                             │
└────────────────────────────────────────────────────────────────┘
        Plan.init()
        ├─ Create root node with query
        ├─ Set status = TODO
        └─ Initialize empty children list
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 2: Plan Generate Agent Dispatch                          │
└────────────────────────────────────────────────────────────────┘
        AgentDispatcher.dispatch(RequiredAbilities.plan_generation)
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 3: Prepare System Prompt                                 │
└────────────────────────────────────────────────────────────────┘
        SYSTEM_PROMPT (from plan_generate_agent/prompt.py):
        ┌──────────────────────────────────────────────────────┐
        │ You are an efficient plan-generation agent           │
        │ Task: Decompose query into subtasks (tree structure) │
        │                                                       │
        │ Background Information:                               │
        │ - Plan structure: 1, 1.1, 1.2, 1.2.1, etc.           │
        │ - Subtask JSON format:                                │
        │   {                                                   │
        │     "subtask name": string,                           │
        │     "goal.goal": string,                              │
        │     "goal.criticism": string,                         │
        │     "milestones": list[string]                        │
        │   }                                                   │
        │                                                       │
        │ Resources Available:                                  │
        │ - Internet access                                     │
        │ - FileSystemEnv                                       │
        │ - Python notebook                                     │
        │ - ShellEnv                                            │
        │                                                       │
        │ Important: Make feasible and efficient plans          │
        │ - Don't create duplicate subtasks                     │
        │ - Group similar goals                                 │
        │ - Minimize subtask count                              │
        └──────────────────────────────────────────────────────┘
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 4: Prepare User Prompt                                   │
└────────────────────────────────────────────────────────────────┘
        USER_PROMPT (from plan_generate_agent/prompt.py):
        ┌──────────────────────────────────────────────────────┐
        │ This is not the first time you are handling the task│
        │ so you should give an initial plan.                  │
        │                                                       │
        │ Here is the query:                                    │
        │ """                                                   │
        │ {{query}} ← FILLED WITH: user_query.task            │
        │ """                                                   │
        │                                                       │
        │ You will use operation SUBTASK_SPLIT to split the    │
        │ query into 2-4 subtasks and then commit.             │
        └──────────────────────────────────────────────────────┘
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 5: LLM Call with Function                                │
└────────────────────────────────────────────────────────────────┘
        PlanGenerateAgent.parse()
        ├─ Call LLM with system + user prompts
        ├─ Function: subtask_split_operation
        └─ Extract response
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 6: Parse LLM Response                                    │
└────────────────────────────────────────────────────────────────┘
        Response contains:
        {
          "operation": "split",
          "target_subtask_id": "root",
          "subtasks": [
            {
              "subtask name": "Subtask 1",
              "goal": {
                "goal": "What to achieve...",
                "criticism": "Potential problems..."
              },
              "milestones": ["Milestone 1", "Milestone 2", ...]
            },
            ... (2-4 subtasks total)
          ]
        }
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 7: Build Plan Tree                                       │
└────────────────────────────────────────────────────────────────┘
        For each subtask in response:
        ├─ Create Plan node
        ├─ Set subtask_id (1, 2, 3, ...)
        ├─ Set goal, milestones, criticism
        ├─ Link to parent (root)
        └─ Set status = TODO
        ↓
        Plan.make_relation(root_node, subtask_nodes)
        ↓
OUTPUT DATA:
┌────────────────────────────────────────────────────────────────┐
│ Plan Tree (hierarchical structure)                            │
│ root                                                           │
│  ├─ 1: Subtask 1 (TODO)                                       │
│  │    ├─ goal: {...}                                          │
│  │    └─ milestones: [...]                                    │
│  ├─ 2: Subtask 2 (TODO)                                       │
│  │    ├─ goal: {...}                                          │
│  │    └─ milestones: [...]                                    │
│  └─ 3: Subtask 3 (TODO)                                       │
│       ├─ goal: {...}                                          │
│       └─ milestones: [...]                                    │
└────────────────────────────────────────────────────────────────┘

DATA PASSED TO NEXT STAGE:
→ Plan tree with all subtasks
→ First subtask marked for execution (Plan.pop_next_subtask)
```

---

### Pipeline 2: Tool Agent Execution Pipeline

**Цель:** Выполнить конкретную подзадачу с использованием доступных инструментов через ReACT алгоритм.

```
┌─────────────────────────────────────────────────────────────────────┐
│                  TOOL AGENT EXECUTION PIPELINE                      │
│                       (Inner Loop - ReACT)                          │
└─────────────────────────────────────────────────────────────────────┘

INPUT DATA:
┌────────────────────────────────────────────────────────────────┐
│ Current Subtask (Plan node)                                    │
│ ├─ subtask_id: string (e.g., "1", "1.2", "2.3.1")            │
│ ├─ goal: {goal: string, criticism: string}                    │
│ ├─ milestones: list[string]                                   │
│ ├─ status: "DOING"                                            │
│ └─ process_node: None (will be filled)                        │
│                                                                │
│ Execution Context:                                             │
│ ├─ all_plan: Complete plan tree                               │
│ ├─ workspace_files: Current file structure                    │
│ ├─ available_tools: List of tool descriptions                 │
│ └─ max_length: Maximum steps allowed                          │
└────────────────────────────────────────────────────────────────┘
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 1: Initialize Search Tree                                │
└────────────────────────────────────────────────────────────────┘
        ReACTChainSearch.run()
        ├─ Create TaskSearchTree
        ├─ Create root ToolNode
        └─ Set process = "" (empty initially)
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 2: Prepare System Prompt                                 │
└────────────────────────────────────────────────────────────────┘
        SYSTEM_PROMPT (from tool_agent/prompt.py):
        ┌──────────────────────────────────────────────────────┐
        │ You are an experimental cutting-edge super capable    │
        │ autonomous agent specialized in learning from         │
        │ environmental feedback and following rules.           │
        │                                                       │
        │ Your Workflow:                                        │
        │ 1. Receive task + plan                                │
        │ 2. Handle subtask:                                    │
        │    - Decide action (use tools or do yourself)         │
        │    - Call functions to apply action                   │
        │ 3. Write task report to FileSystem                    │
        │ 4. Call subtask_submit                                │
        │                                                       │
        │ Resources:                                             │
        │ - Internet access                                     │
        │ - FileSystemEnv (read/write files)                    │
        │ - Python notebook                                     │
        │ - ShellEnv with root privilege                        │
        │ - ask_for_human_help                                  │
        │                                                       │
        │ Important Rules:                                       │
        │ - You can do actual things, not just write guides     │
        │ - For text tasks: do yourself, write to file          │
        │ - For other tasks: use tools                          │
        │ - Pass literal values to tools (no references)        │
        │ - Submit immediately when done                        │
        │                                                       │
        │ Plan Overview:                                         │
        │ {{all_plan}} ← FILLED WITH: Complete plan tree       │
        └──────────────────────────────────────────────────────┘
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 3: Iterative Execution Loop (ReACT)                      │
└────────────────────────────────────────────────────────────────┘
        step_num = 1
        while step_num <= max_length:
            ↓
        ┌──────────────────────────────────────────────────────┐
        │ STEP 3a: Prepare User Prompt                         │
        └──────────────────────────────────────────────────────┘
            USER_PROMPT (from tool_agent/prompt.py):
            ┌──────────────────────────────────────────────────┐
            │ Now, it's your turn give the next function call  │
            │                                                   │
            │ --- Status ---                                    │
            │ Current Subtask: {{subtask_id}}                  │
            │                  ← FILLED WITH: "1", "2.3", etc. │
            │ File System Structure: {{workspace_files}}       │
            │                       ← FILLED WITH: tree output │
            │                                                   │
            │ --- Available Operations ---                      │
            │ - Use tools to handle subtask                     │
            │ - Use "subtask_submit" when done or impossible    │
            │                                                   │
            │ Important Notice:                                 │
            │ - Max steps: {{max_length}} ← FILLED WITH: 50    │
            │ - Current step: {{step_num}} ← FILLED WITH: 1,2..│
            │ - Watch out the budget                            │
            │                                                   │
            │ {{human_help_prompt}} ← If human helped         │
            │                                                   │
            │ Now show your super capability!                   │
            └──────────────────────────────────────────────────┘
            ↓
        ┌──────────────────────────────────────────────────────┐
        │ STEP 3b: Add Context Messages                        │
        └──────────────────────────────────────────────────────┘
            make_message() from ReACT.py:
            ┌──────────────────────────────────────────────────┐
            │ Message 1: Current subtask info                  │
            │ """                                               │
            │ Now you will perform the following subtask:      │
            │ {{terminal_task_info}}                           │
            │    ← FILLED WITH: Subtask JSON or summary        │
            │ """                                               │
            │                                                   │
            │ Message 2: Action history                         │
            │ """                                               │
            │ The following steps have been performed:          │
            │ {{action_process}}                               │
            │    ← FILLED WITH: Previous actions or summary    │
            │ """                                               │
            └──────────────────────────────────────────────────┘
            ↓
        ┌──────────────────────────────────────────────────────┐
        │ STEP 3c: LLM Call with Function                      │
        └──────────────────────────────────────────────────────┘
            ToolAgent.parse()
            ├─ Call LLM with system + user + context prompts
            ├─ Function: subtask_handle
            └─ Extract response
            ↓
        ┌──────────────────────────────────────────────────────┐
        │ STEP 3d: Parse LLM Response                          │
        └──────────────────────────────────────────────────────┘
            Response format (from task_handle_functions.yml):
            {
              "plan": ["What to do next..."],      ← ~30 words
              "thought": "Internal reasoning...",  ← ~50 words
              "reasoning": "Why this thought...",  ← ~100 words
              "criticism": "Self-criticism...",
              "tool_call": {
                "tool_name": "ToolName",
                "tool_input": {...}  ← JSON parameters
              }
            }
            ↓
        ┌──────────────────────────────────────────────────────┐
        │ STEP 3e: Check Tool Type                             │
        └──────────────────────────────────────────────────────┘
            if tool_name == "subtask_submit":
                → Go to STEP 4 (Completion)
            elif tool_name == "ask_human_for_help":
                → Wait for human response
                → Continue loop with updated context
            else:
                → Execute tool via ToolServerInterface
                → Capture tool result
                → Update action_process
            ↓
        ┌──────────────────────────────────────────────────────┐
        │ STEP 3f: Update ToolNode                             │
        └──────────────────────────────────────────────────────┘
            Create new ToolNode:
            ├─ data: {plan, thought, reasoning, criticism}
            ├─ observation: tool result
            ├─ tool_call_status: SUCCESS/FAILED
            └─ process: action_process + new action
            ↓
            Append to ToolNode tree
            ↓
            step_num += 1
            Continue loop
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 4: Subtask Submission                                    │
└────────────────────────────────────────────────────────────────┘
        When tool_name == "subtask_submit":

        Response format:
        {
          "result": {
            "conclusion": "Summary of what done...",  ← ~400 words
            "milestones": ["file.txt written", ...],
            "success": true/false
          },
          "submit_type": "give_answer" | "give_up" | "max_tool_call",
          "suggestions_for_latter_subtasks_plan": {
            "need_for_plan_refine": true/false,
            "reason": "Detailed suggestions..."     ← ~50 words
          }
        }
        ↓
        Update search_method attributes:
        ├─ finish_node = current ToolNode
        ├─ status = SUCCESS/FAILED
        ├─ need_for_plan_refine = from response
        └─ suggestions = from response
        ↓
OUTPUT DATA:
┌────────────────────────────────────────────────────────────────┐
│ ToolNode Tree (execution trace)                               │
│ root                                                           │
│  ├─ Step 1:                                                    │
│  │    ├─ thought: "I need to..."                              │
│  │    ├─ reasoning: "Because..."                              │
│  │    ├─ tool_call: {name: "WebSearch", input: {...}}        │
│  │    └─ observation: "Search results..."                     │
│  ├─ Step 2:                                                    │
│  │    ├─ thought: "Now I should..."                           │
│  │    ├─ tool_call: {name: "FileSystemEnv.write", ...}       │
│  │    └─ observation: "File written..."                       │
│  └─ ...                                                        │
│  └─ Step N (final):                                            │
│       └─ tool_call: {name: "subtask_submit", ...}             │
│                                                                │
│ Search Method Result:                                          │
│ ├─ status: SUCCESS/FAILED                                     │
│ ├─ need_for_plan_refine: boolean                              │
│ └─ suggestions: string                                         │
└────────────────────────────────────────────────────────────────┘

DATA PASSED TO NEXT STAGE:
→ finish_node (complete ToolNode tree with all actions)
→ Subtask result (success/failed)
→ Plan refinement suggestions
```

---

### Pipeline 3: Reflect Agent Pipeline

**Цель:** Извлечь апостериорные знания из выполненной подзадачи для обучения и улучшения будущего планирования.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    REFLECT AGENT PIPELINE                           │
│                  (Posterior Knowledge Extraction)                   │
└─────────────────────────────────────────────────────────────────────┘

INPUT DATA:
┌────────────────────────────────────────────────────────────────┐
│ Completed Subtask Data                                         │
│ ├─ now_dealing_task: Plan node (just completed)               │
│ │   ├─ subtask_id: "1", "2.3", etc.                          │
│ │   ├─ goal: {goal, criticism}                                │
│ │   └─ process_node: ToolNode tree (execution trace)          │
│ │                                                              │
│ ├─ all_plan: Complete plan structure (для контекста)          │
│ └─ tool_functions: List of available tools                     │
└────────────────────────────────────────────────────────────────┘
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 1: Get Posterior Knowledge Trigger                       │
└────────────────────────────────────────────────────────────────┘
        TaskHandler.posterior_process(now_dealing_task)
        ├─ Called after subtask execution completes
        └─ Before plan refinement
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 2: Dispatch Reflect Agent                                │
└────────────────────────────────────────────────────────────────┘
        AgentDispatcher.dispatch(RequiredAbilities.reflection)
        ↓
        Create ReflectAgent with configuration
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 3: Prepare System Prompt                                 │
└────────────────────────────────────────────────────────────────┘
        SYSTEM_PROMPT (from reflect_agent/prompt.py):
        ┌──────────────────────────────────────────────────────┐
        │ You are a posterior_knowledge_obtainer.              │
        │ You have performed subtask with:                      │
        │ 1. Intermediate thoughts (reasoning path)             │
        │ 2. Tool calls (interact with physical world)          │
        │ 3. Workspace (minimal file system + code executer)    │
        │                                                       │
        │ You plan of the task is as follows:                   │
        │ --- Plan ---                                          │
        │ {{all_plan}}                                         │
        │    ← FILLED WITH: Complete plan tree JSON            │
        │                                                       │
        │ You have handled the following subtask:               │
        │ --- Handled Subtask ---                               │
        │ {{terminal_plan}}                                    │
        │    ← FILLED WITH: Completed subtask info             │
        │                                                       │
        │ the available tools are as follows:                   │
        │ --- Tools ---                                         │
        │ {{tool_functions_description_list}}                 │
        │    ← FILLED WITH: Tool descriptions                  │
        │                                                       │
        │ The following steps have been performed:              │
        │ --- Actions ---                                       │
        │ {{action_process}}                                   │
        │    ← FILLED WITH: ToolNode tree → actions list       │
        │                                                       │
        │ Now, learn posterior knowledge:                       │
        │                                                       │
        │ 1. Summary: Summarize tool calls and thoughts.        │
        │    This will carry to next subtasks.                  │
        │    If modified files, tell file_name and what done.   │
        │                                                       │
        │ 2. Reflection of SUBTASK_PLAN:                        │
        │    Knowledge for generating plan next time.           │
        │                                                       │
        │ 3. Reflection of tool calling:                        │
        │    What learned about tool usage?                     │
        │    (e.g., "tool xxx not available", "need field yyy") │
        └──────────────────────────────────────────────────────┘
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 4: Prepare User Prompt                                   │
└────────────────────────────────────────────────────────────────┘
        USER_PROMPT = "" (empty, uses only system prompt)
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 5: Extract Action Process                                │
└────────────────────────────────────────────────────────────────┘
        Convert ToolNode tree to action list:
        ├─ Traverse process_node (ToolNode tree)
        ├─ For each node: extract thought, tool_call, observation
        └─ Format as sequential list of actions

        Example action_process:
        """
        Step 1:
          Thought: I need to search for information
          Tool: WebSearch
          Input: {"query": "XAgent architecture"}
          Result: Found 5 relevant documents...

        Step 2:
          Thought: Now I should read the main document
          Tool: FileSystemEnv.read
          Input: {"path": "docs/architecture.md"}
          Result: File content...
        ...
        """
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 6: LLM Call with Function                                │
└────────────────────────────────────────────────────────────────┘
        ReflectAgent.parse()
        ├─ Call LLM with filled prompts
        ├─ Function: generate_posterior_knowledge
        └─ Extract response
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 7: Parse LLM Response                                    │
└────────────────────────────────────────────────────────────────┘
        Response format (from generate_posterior_knowledge.yml):
        {
          "summary": "Concise summary of subtask execution...",
          "reflection_of_plan": [
            "Plan should include X step earlier",
            "Subtask Y was unnecessary",
            "Could combine Z and W subtasks"
          ],
          "reflection_of_tool": [
            {
              "target_tool_name": "WebSearch",
              "reflection": [
                "Need to provide 'max_results' parameter",
                "Works better with specific queries"
              ]
            },
            {
              "target_tool_name": "FileSystemEnv.write",
              "reflection": [
                "Must create parent directory first",
                "Path should be absolute"
              ]
            }
          ]
        }
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 8: Update Task Data                                      │
└────────────────────────────────────────────────────────────────┘
        now_dealing_task.data.update({
            "posterior_knowledge": {
                "summary": response["summary"],
                "plan_reflection": response["reflection_of_plan"],
                "tool_reflection": response["reflection_of_tool"]
            }
        })
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 9: Register in Working Memory                            │
└────────────────────────────────────────────────────────────────┘
        WorkingMemoryAgent.register_task(now_dealing_task)
        ├─ Store subtask result
        ├─ Store posterior knowledge
        └─ Make available for future subtasks
        ↓
OUTPUT DATA:
┌────────────────────────────────────────────────────────────────┐
│ Posterior Knowledge (stored in task data)                     │
│ ├─ summary: Concise execution summary                         │
│ ├─ plan_reflection: List of planning insights                 │
│ └─ tool_reflection: List of tool usage insights               │
│                                                                │
│ Working Memory Entry:                                          │
│ ├─ task_id: subtask_id                                        │
│ ├─ status: SUCCESS/FAILED                                     │
│ ├─ data: Complete task data with reflections                  │
│ └─ Available for: chat_with_other_subtask function            │
└────────────────────────────────────────────────────────────────┘

DATA PASSED TO NEXT STAGE:
→ Posterior knowledge (summary + reflections)
→ Updated task in working memory
→ Insights for future planning and execution
```

---

### Pipeline 4: Plan Refine Pipeline

**Цель:** Уточнить план на основе результатов выполнения подзадачи и предложений агента.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     PLAN REFINE PIPELINE                            │
│                  (Iterative Plan Modification)                      │
└─────────────────────────────────────────────────────────────────────┘

INPUT DATA:
┌────────────────────────────────────────────────────────────────┐
│ Plan Refinement Trigger                                        │
│ ├─ now_dealing_task: Just completed subtask                   │
│ │   ├─ suggestions_for_latter_subtasks_plan:                  │
│ │   │   ├─ need_for_plan_refine: true                        │
│ │   │   └─ reason: "Detailed suggestions..."                 │
│ │   └─ status: SUCCESS/FAILED                                 │
│ │                                                              │
│ ├─ current_plan: Complete plan tree                           │
│ ├─ workspace_files: Current workspace structure                │
│ └─ config:                                                     │
│     ├─ max_plan_refine_chain_length: 4 (max iterations)       │
│     ├─ max_plan_tree_width: 5 (max children per node)         │
│     └─ max_plan_tree_depth: 3 (max tree depth)                │
└────────────────────────────────────────────────────────────────┘
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 1: Check Refinement Need                                 │
└────────────────────────────────────────────────────────────────┘
        if search_method.need_for_plan_refine == false:
            → Skip refinement, go to next subtask
        else:
            → Continue to refinement
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 2: Initialize Plan Refine Mode                           │
└────────────────────────────────────────────────────────────────┘
        PlanAgent.plan_refine_mode(now_dealing_task)
        ├─ Create PlanRefineChain (history)
        └─ Set modify_steps = 0
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 3: Dispatch Plan Refine Agent                            │
└────────────────────────────────────────────────────────────────┘
        AgentDispatcher.dispatch(RequiredAbilities.plan_refinement)
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 4: Iterative Refinement Loop                             │
└────────────────────────────────────────────────────────────────┘
        while modify_steps < max_plan_refine_chain_length:
            ↓
        ┌──────────────────────────────────────────────────────┐
        │ STEP 4a: Prepare System Prompt                       │
        └──────────────────────────────────────────────────────┘
            SYSTEM_PROMPT (from plan_refine_agent/prompt.py):
            ┌──────────────────────────────────────────────────┐
            │ You are plan-rectify agent                        │
            │ Task: Iteratively rectify plan of a query         │
            │                                                   │
            │ Background Information:                            │
            │ PLAN STRUCTURE:                                    │
            │ - Tree: 1, 1.1, 1.2, 1.2.1, etc.                 │
            │ - Max width: {{max_plan_tree_width}} (e.g., 4)  │
            │    ← FILLED WITH: config value                   │
            │ - Max depth: {{max_plan_tree_depth}} (e.g., 3)  │
            │    ← FILLED WITH: config value                   │
            │                                                   │
            │ SUBTASK FORMAT:                                    │
            │ {                                                 │
            │   "subtask name": string,                         │
            │   "goal": {goal, criticism},                      │
            │   "milestones": list[string]                      │
            │ }                                                 │
            │                                                   │
            │ RESOURCES: Internet, FileSystem, Python, Shell    │
            │                                                   │
            │ PLAN_REFINE_MODE OPERATIONS:                      │
            │                                                   │
            │ - split: Split failed leaf subtask into 2-4 new   │
            │   Example: split 1.2 → creates 1.2.1, 1.2.2      │
            │                                                   │
            │ - add: Add brother nodes to target subtask        │
            │   Example: add after 1.1 → creates 1.2, 1.3      │
            │                                                   │
            │ - delete: Remove future/TODO subtask              │
            │   Example: delete 1.2.1                           │
            │                                                   │
            │ - exit: Exit PLAN_REFINE_MODE                     │
            │                                                   │
            │ Important Notice:                                  │
            │ - Never change subtasks before handling position  │
            │ - Never create duplicate subtasks                 │
            │ - Group similar goals in one subtask              │
            │ - Maintain hierarchy structure                    │
            │ - Max 4 operations before forced exit             │
            │ - Use LLM capabilities, minimize complexity        │
            └──────────────────────────────────────────────────┘
            ↓
        ┌──────────────────────────────────────────────────────┐
        │ STEP 4b: Prepare User Prompt                         │
        └──────────────────────────────────────────────────────┘
            USER_PROMPT (from plan_refine_agent/prompt.py):
            ┌──────────────────────────────────────────────────┐
            │ Your task: choose one SUBTASK OPERATION           │
            │                                                   │
            │ Constraints:                                       │
            │ 1. Only modify subtask_id > {{subtask_id}}       │
            │    ← FILLED WITH: now_dealing_task.subtask_id    │
            │                                                   │
            │ 2. If plan is good enough, use REFINE_SUBMIT      │
            │                                                   │
            │ 3. Budget: {{max_step}} operations max,          │
            │    already made {{modify_steps}} steps           │
            │    ← FILLED WITH: 4 and current count            │
            │                                                   │
            │ 4. Max depth: {{max_plan_tree_depth}}            │
            │    Be careful when using SUBTASK_SPLIT            │
            │                                                   │
            │ 5. Please use function call to respond!!!         │
            │                                                   │
            │ --- Status ---                                    │
            │ File System Structure: {{workspace_files}}       │
            │    ← FILLED WITH: workspace tree                 │
            │                                                   │
            │ Refine Node Message: {{refine_node_message}}     │
            │    ← FILLED WITH: suggestions from subtask       │
            └──────────────────────────────────────────────────┘
            ↓
        ┌──────────────────────────────────────────────────────┐
        │ STEP 4c: LLM Call with Function                      │
        └──────────────────────────────────────────────────────┘
            PlanRefineAgent.parse()
            ├─ Call LLM with filled prompts
            ├─ Function: subtask_operations
            └─ Extract response
            ↓
        ┌──────────────────────────────────────────────────────┐
        │ STEP 4d: Parse Operation                             │
        └──────────────────────────────────────────────────────┘
            Response format (from task_manage_functions.yml):
            {
              "operation": "split" | "add" | "delete" | "exit",
              "target_subtask_id": "1.2",
              "subtasks": [
                {
                  "subtask name": "New Subtask 1",
                  "goal": {
                    "goal": "What to achieve...",
                    "criticism": "Potential problems..."
                  },
                  "milestones": ["Milestone 1", ...]
                },
                ... (if operation is split or add)
              ]
            }
            ↓
        ┌──────────────────────────────────────────────────────┐
        │ STEP 4e: Execute Operation                           │
        └──────────────────────────────────────────────────────┘
            ┌──────────────────────────────────────────────────┐
            │ if operation == "split":                          │
            └──────────────────────────────────────────────────┘
                Validate:
                ├─ target_subtask exists
                ├─ target is leaf node (no children)
                ├─ target is failed task
                └─ new depth < max_plan_tree_depth
                ↓
                Execute:
                ├─ Get target node from plan tree
                ├─ For each new subtask:
                │   ├─ Create Plan node
                │   ├─ Set subtask_id (target_id.1, .2, .3)
                │   ├─ Set goal, milestones, criticism
                │   └─ Set status = TODO
                └─ Plan.make_relation(target, new_subtasks)

            ┌──────────────────────────────────────────────────┐
            │ elif operation == "add":                          │
            └──────────────────────────────────────────────────┘
                Validate:
                ├─ target_subtask exists
                ├─ target is current or future subtask
                └─ new width <= max_plan_tree_width
                ↓
                Execute:
                ├─ Get parent of target
                ├─ Get current last sibling index
                ├─ For each new subtask:
                │   ├─ Create Plan node
                │   ├─ Set subtask_id (parent_id.next_index)
                │   ├─ Set goal, milestones, criticism
                │   └─ Set status = TODO
                └─ Plan.make_relation(parent, new_subtasks)

            ┌──────────────────────────────────────────────────┐
            │ elif operation == "delete":                       │
            └──────────────────────────────────────────────────┘
                Validate:
                ├─ target_subtask exists
                └─ target is future/TODO subtask (not done/doing)
                ↓
                Execute:
                └─ Plan.remove_subtask(target_subtask_id)

            ┌──────────────────────────────────────────────────┐
            │ elif operation == "exit":                         │
            └──────────────────────────────────────────────────┘
                Exit refinement mode
                Break loop
            ↓
        ┌──────────────────────────────────────────────────────┐
        │ STEP 4f: Record Operation                            │
        └──────────────────────────────────────────────────────┘
            PlanRefineChain.add_refinement({
                "operation": operation,
                "target": target_subtask_id,
                "changes": subtasks,
                "step": modify_steps
            })
            ↓
            modify_steps += 1
            ↓
            if operation == "exit" or modify_steps >= max:
                Break loop
            else:
                Continue to next iteration
        ↓
┌────────────────────────────────────────────────────────────────┐
│ STEP 5: Finalize Refinement                                   │
└────────────────────────────────────────────────────────────────┘
        Updated plan tree with:
        ├─ New subtasks (from split/add)
        ├─ Removed subtasks (from delete)
        └─ Maintained hierarchy and constraints
        ↓
OUTPUT DATA:
┌────────────────────────────────────────────────────────────────┐
│ Modified Plan Tree                                             │
│ Example after split operation on failed subtask 2:            │
│ root                                                           │
│  ├─ 1: Subtask 1 (DONE)                                       │
│  ├─ 2: Subtask 2 (FAILED) ← Split into:                       │
│  │   ├─ 2.1: Sub-subtask 1 (TODO)                            │
│  │   ├─ 2.2: Sub-subtask 2 (TODO)                            │
│  │   └─ 2.3: Sub-subtask 3 (TODO)                            │
│  └─ 3: Subtask 3 (TODO)                                       │
│                                                                │
│ PlanRefineChain History:                                       │
│ ├─ Refinement 1: split 2 into 2.1, 2.2, 2.3                  │
│ └─ (up to 4 refinements)                                      │
└────────────────────────────────────────────────────────────────┘

DATA PASSED TO NEXT STAGE:
→ Updated plan tree with refined subtasks
→ Next subtask to execute (Plan.pop_next_subtask)
→ Refinement history
```

---

## Detailed Agent Interactions

### 1. Plan Generate Agent

**Purpose:** Generate initial hierarchical plan from user query

**Input:**
- User query (task description)
- Available tool names
- Role name

**Process:**
```python
# Dispatch to Plan Generation Agent
agent = agent_dispatcher.dispatch(
    RequiredAbilities.plan_generation,
    target_task=query.task
)

# Generate plan with LLM
new_message = agent.parse(
    placeholders={
        "system": {"avaliable_tool_names": tool_names},
        "user": {"query": task_description}
    },
    arguments=simple_thought_schema,
    functions=[subtask_split_operation_schema]
)

# Parse subtasks from LLM response
subtasks = json5.loads(new_message["function_call"]["arguments"])

# Build plan tree
for subtask_item in subtasks["subtasks"]:
    subplan = plan_function_output_parser(subtask_item)
    Plan.make_relation(root_plan, subplan)
```

**Output:**
- `Plan` tree structure with root and child subtasks
- Each subtask has: name, goal, milestones, expected tools

### 2. Tool Agent (Inner Loop Executor)

**Purpose:** Execute individual subtasks step-by-step using ReACT

**Input:**
- Current subtask (Plan node)
- Available tools/functions
- Workspace state

**Process:**
```python
# Create ReACT search algorithm
search_method = ReACTChainSearch(xagent_core_components)

# Iterative execution loop
while depth < max_subtask_chain_length:
    # Build context messages
    messages = make_message(
        now_node=current_node,
        now_dealing_task=subtask,
        all_plan=plan_agent.latest_plan
    )
    
    # Get next action from LLM
    new_message = agent.parse(
        placeholders={
            "system": {"all_plan": serialized_plan},
            "user": {
                "workspace_files": current_file_structure,
                "subtask_id": task_id,
                "step_num": current_step
            }
        },
        arguments=action_reasoning_schema,
        functions=available_tools,
        additional_messages=messages
    )
    
    # Convert to ToolNode
    new_node = agent.message_to_tool_node(new_message)
    
    # Execute tool
    tool_output, status_code, need_refine = function_handler.handle_tool_call(new_node)
    
    # Add to tree
    tree.make_father_relation(current_node, new_node)
    current_node = new_node
    
    # Check termination
    if status_code in [SUBMIT_AS_SUCCESS, SUBMIT_AS_FAILED]:
        break
```

**Output:**
- `ToolNode` tree representing execution trace
- Final node contains submission result
- Flag indicating if plan refinement needed

**Data Passed to Next Agent:**
- Complete execution trace (`finish_node.process`)
- Tool outputs and status codes
- Suggestions for plan refinement

### 3. Reflect Agent

**Purpose:** Analyze completed subtask and generate insights

**Input:**
- All plan (complete plan structure)
- Terminal plan (completed subtask)
- Action process (sequence of tool calls)
- Tool descriptions

**Process:**
```python
# Dispatch to Reflection Agent
agent = agent_dispatcher.dispatch(
    RequiredAbilities.reflection,
    target_task="Reflect on previous actions"
)

# Summarize action history
if config.enable_summary:
    terminal_plan_summary = summarize_plan(terminal_plan)
    action_summary = summarize_action(finish_node.process)
else:
    # Use raw JSON
    ...

# Generate reflection with LLM
new_message = agent.parse(
    placeholders={
        "system": {
            "all_plan": all_plan_json,
            "terminal_plan": terminal_plan_json,
            "tool_functions_description_list": tool_descriptions,
            "action_process": action_summary
        }
    },
    arguments=generate_posterior_knowledge_schema
)

# Extract insights
posterior_knowledge = json5.loads(new_message["arguments"])
# Contains: summary, reflection_of_plan, reflection_of_tool
```

**Output:**
- `summary`: Condensed description of what was accomplished
- `reflection_of_plan`: Analysis of plan execution effectiveness
- `reflection_of_tool`: Insights about tool usage

**Data Passed to Next Agent:**
- Updated `TaskSaveItem` with reflections
- Summary for working memory registration

### 4. Plan Refine Agent

**Purpose:** Modify plan structure based on execution feedback

**Input:**
- Current plan state
- Completed subtask with refinement suggestions
- Workspace file structure
- Plan operation history

**Process:**
```python
# Create refinement chain to track changes
refine_chain = PlanRefineChain(current_plan)

# Iterative refinement loop
while modify_steps < max_plan_refine_chain_length:
    # Build context with refinement history
    additional_messages = refine_chain.parse_to_message_list(flag_changed)
    
    # Get refinement operation from LLM
    new_message = agent.parse(
        placeholders={
            "system": {
                "avaliable_tool_names": tool_names,
                "max_plan_tree_width": max_width,
                "max_plan_tree_depth": max_depth
            },
            "user": {
                "subtask_id": current_subtask_id,
                "modify_steps": current_step,
                "workspace_files": file_structure,
                "refine_node_message": suggestions_from_subtask
            }
        },
        arguments=simple_thought_schema,
        functions=[subtask_operations_schema],
        additional_messages=additional_messages,
        additional_insert_index=-1  # Insert before last message
    )
    
    # Parse operation
    operation = function_input['operation']  # split/add/delete/exit
    
    # Execute operation
    if operation == 'split':
        function_output, status = deal_subtask_split(function_input)
    elif operation == 'add':
        function_output, status = deal_subtask_add(function_input)
    elif operation == 'delete':
        function_output, status = deal_subtask_delete(function_input)
    elif operation == 'exit':
        status = PLAN_REFINE_EXIT
        break
    
    # Register change
    refine_chain.register(
        function_name, function_input, 
        function_output, new_plan
    )
    
    if status in [MODIFY_SUCCESS, PLAN_REFINE_EXIT]:
        break
```

**Plan Operations:**

1. **Split Operation:**
   - Target: Existing subtask
   - Action: Replace subtask with multiple smaller subtasks
   - Updates: Parent's children list, subtask marked as SPLIT

2. **Add Operation:**
   - Target: Position after existing subtask
   - Action: Insert new subtasks at specified position
   - Updates: Parent's children list at insertion point

3. **Delete Operation:**
   - Target: TODO subtask (not yet executed)
   - Action: Remove subtask from parent's children
   - Updates: Parent's children list, orphan the subtask

**Output:**
- Modified `Plan` tree structure
- Refinement history in `PlanRefineChain`
- Updated subtask list for continued execution

---

## Data Flow & Data Structures

### Key Data Structures

#### 1. Plan (Hierarchical Task Tree)
```python
class Plan:
    father: Optional[Plan]          # Parent task
    children: List[Plan]            # Child subtasks
    data: TaskSaveItem              # Task metadata
    process_node: ToolNode          # Execution trace
    
    Methods:
    - get_subtask_id(): Returns hierarchical ID (e.g., "1.2.3")
    - to_json(): Serializes plan tree
    - get_inorder_travel(): Returns all subtasks in order
    - pop_next_subtask(): Gets next TODO subtask
```

**Example Plan Tree:**
```
Plan (1: "Research and write report")
├── Plan (1.1: "Research topic A")
│   └── process_node: ToolNode (execution trace)
├── Plan (1.2: "Research topic B")
│   └── process_node: ToolNode (execution trace)
└── Plan (1.3: "Write report")
    └── process_node: ToolNode (execution trace)
```

#### 2. ToolNode (Execution Trace Tree)
```python
class ToolNode:
    father: ToolNode                # Parent step
    children: List[ToolNode]        # Child steps
    data: dict                      # Step data
        - content: LLM raw content
        - thoughts: {
            thought: reasoning text
            reasoning: detailed reasoning
            plan: step-by-step plan
            criticism: self-critique
          }
        - command: {
            name: tool/function name
            args: tool arguments
          }
        - tool_output: execution result
        - tool_status_code: success/failure
    history: MessageHistory         # Message log
    
    Properties:
    - process: Returns execution trace from root to current node
```

**Example ToolNode Tree (for one subtask):**
```
ToolNode (root - empty)
└── ToolNode (Step 1: "Search for information")
    └── thoughts: "Need to search topic A"
    └── command: {name: "WebEnv_search", args: {query: "topic A"}}
    └── tool_output: "Found 5 results..."
    └── ToolNode (Step 2: "Read article")
        └── command: {name: "WebEnv_browse_website", args: {url: "..."}}
        └── tool_output: "Article content..."
        └── ToolNode (Step 3: "Submit results")
            └── command: {name: "subtask_submit", args: {success: true}}
```

#### 3. TaskSaveItem (Subtask Metadata)
```python
class TaskSaveItem:
    name: str                               # Subtask name
    goal: str                               # Subtask objective
    milestones: List[str]                   # Expected milestones
    prior_plan_criticism: str               # Initial critique
    posterior_plan_reflection: List[str]    # Post-execution reflection
    tool_reflection: List[dict]             # Tool usage insights
    action_list_summary: str                # Execution summary
    status: TaskStatusCode                  # TODO/DOING/SUCCESS/FAIL/SPLIT
```

#### 4. BaseQuery (User Input)
```python
class AutoGPTQuery(BaseQuery):
    role_name: str      # e.g., "Assistant"
    task: str           # User's task description
    plan: List[str]     # Initial high-level plan (optional)
```

### Data Flow Between Agents

```
User Query (AutoGPTQuery)
    ↓
[Plan Generate Agent]
    Input: AutoGPTQuery {task, role_name, plan}
    Output: Plan tree with TaskSaveItem nodes
    ↓
Plan tree (hierarchical)
    ↓
For each subtask:
    ↓
[Tool Agent via ReACT]
    Input: Plan node (current subtask)
           Available tools
           Plan context (all_plan)
    Processing: Creates ToolNode tree (step-by-step execution)
    Output: ToolNode with complete process
            Flag: need_for_plan_refine
    ↓
ToolNode (execution trace)
    ↓
[Reflect Agent]
    Input: all_plan (Plan tree)
           terminal_plan (completed Plan node)
           finish_node (ToolNode with process)
           tool_descriptions
    Output: Posterior knowledge {
              summary: str
              reflection_of_plan: List[str]
              reflection_of_tool: List[dict]
            }
    ↓
Updated Plan node (with reflections)
    ↓
[Working Memory Agent]
    Input: terminal_plan (Plan node)
    Action: Register in memory {plan, task_id, qa_sequence}
    ↓
    (Conditional) if need_for_plan_refine:
        ↓
    [Plan Refine Agent]
        Input: Current Plan tree
               Completed subtask with suggestions
               Workspace file structure
        Processing: PlanRefineChain (tracks modifications)
        Operations: split/add/delete subtasks
        Output: Modified Plan tree
        ↓
Updated Plan tree
    ↓
Continue to next subtask...
```

### Data Flow Example (Concrete)

**User Input:**
```json
{
  "task": "Research machine learning and write a summary",
  "role_name": "Assistant",
  "plan": []
}
```

**After Plan Generate Agent:**
```python
Plan {
  data: TaskSaveItem {
    name: "Research and write ML summary",
    goal: "Research machine learning and write a summary",
    status: TODO
  },
  children: [
    Plan {
      data: TaskSaveItem {
        name: "Research ML basics",
        goal: "Find information about machine learning fundamentals",
        milestones: ["Search for ML articles", "Read top 3 articles"],
        status: TODO
      }
    },
    Plan {
      data: TaskSaveItem {
        name: "Write summary",
        goal: "Write a comprehensive summary of ML",
        milestones: ["Draft introduction", "Write key concepts", "Add conclusion"],
        status: TODO
      }
    }
  ]
}
```

**During Tool Agent Execution (Subtask 1):**
```python
# now_dealing_task = plan.children[0] (Research ML basics)

ToolNode tree:
  ToolNode {
    data: {
      thoughts: {thought: "Need to search for ML information"},
      command: {name: "WebEnv_search", args: {query: "machine learning basics"}},
      tool_output: ["https://ml-article1.com", "https://ml-article2.com"],
      tool_status_code: TOOL_CALL_SUCCESS
    },
    children: [
      ToolNode {
        data: {
          thoughts: {thought: "Read first article"},
          command: {name: "WebEnv_browse_website", args: {url: "https://ml-article1.com"}},
          tool_output: "Machine learning is...",
          tool_status_code: TOOL_CALL_SUCCESS
        },
        children: [
          ToolNode {
            data: {
              thoughts: {thought: "Have enough information"},
              command: {name: "subtask_submit", args: {
                result: {success: true, conclusion: "Found ML information"},
                suggestions_for_latter_subtasks_plan: {
                  need_for_plan_refine: false
                }
              }},
              tool_status_code: SUBMIT_AS_SUCCESS
            }
          }
        ]
      }
    ]
  }
```

**After Reflect Agent:**
```python
plan.children[0].data updated:
  TaskSaveItem {
    name: "Research ML basics",
    goal: "Find information about machine learning fundamentals",
    status: SUCCESS,
    action_list_summary: "Searched for ML articles, read top article, found comprehensive information",
    posterior_plan_reflection: [
      "Search was effective in finding relevant sources",
      "Reading one article was sufficient for basic understanding"
    ],
    tool_reflection: [
      {
        target_tool_name: "WebEnv_search",
        reflection: "Provided high-quality results quickly"
      }
    ]
  }
```

**If Plan Refinement Triggered:**
```python
# Suppose Tool Agent suggested splitting "Write summary" into smaller tasks

Before refinement:
  Plan.children = [
    Plan (1.1: Research ML basics - SUCCESS),
    Plan (1.2: Write summary - TODO)
  ]

After PlanRefineAgent (split operation):
  Plan.children = [
    Plan (1.1: Research ML basics - SUCCESS),
    Plan (1.2: Write summary - SPLIT)  # Marked as SPLIT
      children = [
        Plan (1.2.1: Write introduction - TODO),
        Plan (1.2.2: Write key concepts - TODO),
        Plan (1.2.3: Write conclusion - TODO)
      ]
  ]
```

---

## Search Algorithms

### ReACT (Reasoning + Acting) Chain Search

**Purpose:** Execute subtasks through iterative reasoning and action cycles

**Algorithm:** `XAgent/inner_loop_search_algorithms/ReACT.py`

**Key Concepts:**

1. **Chain-based Search:** Linear progression through steps (depth-first)
2. **Reasoning Before Acting:** Each step requires explicit thought process
3. **Context Accumulation:** Each step builds on previous steps' results

**Algorithm Structure:**

```python
class ReACTChainSearch(BaseSearchMethod):
    def run(config, agent, functions, task_id, now_dealing_task, plan_agent):
        # Create search tree
        tree = TaskSearchTree()
        current_node = tree.root
        
        # Iterative search up to max depth
        while current_node.get_depth() < max_subtask_chain_length:
            # 1. Build context from execution history
            messages = make_message(
                now_node=current_node,
                now_dealing_task=task,
                all_plan=plan_agent.latest_plan
            )
            
            # 2. Get next action from agent
            new_message = agent.parse(
                placeholders={...},
                arguments=action_reasoning_schema,
                functions=available_tools,
                additional_messages=messages
            )
            
            # 3. Convert to ToolNode
            new_node = agent.message_to_tool_node(new_message)
            
            # 4. Execute tool
            tool_output, status_code, need_refine = \
                function_handler.handle_tool_call(new_node)
            
            # 5. Add to tree
            tree.make_father_relation(current_node, new_node)
            current_node = new_node
            
            # 6. Check termination
            if status_code in [SUBMIT_AS_SUCCESS, SUBMIT_AS_FAILED]:
                break
        
        return current_node  # finish_node
```

**Message Building (Context Construction):**

```python
def make_message(now_node, now_dealing_task, config):
    messages = []
    
    # 1. Current subtask description
    messages.append(Message("user", f"""
        Now you will perform the following subtask:
        {subtask_description}
    """))
    
    # 2. Execution history (actions taken so far)
    action_process = now_node.process  # All steps from root to current
    if config.enable_summary:
        action_process = summarize_action(action_process)
    
    messages.append(Message("user", f"""
        The following steps have been performed:
        {action_process}
    """))
    
    return messages
```

**Step Example:**

```
Iteration 1:
  Context: "Perform subtask: Research ML basics"
  LLM Output: {
    thoughts: "I need to search for ML articles",
    command: {name: "WebEnv_search", args: {query: "machine learning"}}
  }
  Tool Execution: Returns ["url1", "url2", "url3"]

Iteration 2:
  Context: "Perform subtask: Research ML basics"
           "Steps performed: [Step 1: Searched ML, found 3 URLs]"
  LLM Output: {
    thoughts: "I should read the first article",
    command: {name: "WebEnv_browse_website", args: {url: "url1"}}
  }
  Tool Execution: Returns article content

Iteration 3:
  Context: "Perform subtask: Research ML basics"
           "Steps performed: [Step 1: Searched, Step 2: Read article]"
  LLM Output: {
    thoughts: "I have sufficient information",
    command: {name: "subtask_submit", args: {success: true}}
  }
  Status: SUBMIT_AS_SUCCESS → BREAK
```

**Why ReACT?**

1. **Explicit Reasoning:** Forces model to articulate its thought process
2. **Error Recovery:** Can adjust actions based on previous results
3. **Transparency:** Clear trace of decision-making
4. **Flexibility:** Can adapt plan within subtask based on intermediate results

**Comparison with Other Search Methods:**

| Aspect | ReACT Chain | Tree Search | Monte Carlo |
|--------|-------------|-------------|-------------|
| Structure | Linear chain | Tree with branches | Tree with sampling |
| Exploration | Single path | Multiple paths | Statistical sampling |
| Memory | Full history | Selective pruning | Aggregate statistics |
| Best for | Sequential tasks | Exploration tasks | Optimization tasks |

**Current Implementation:** XAgent uses **ReACT Chain Search** exclusively for the inner loop.

---

## Control Flow Summary

### Execution Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         ENTRY POINT                             │
│  run.py / XAgentServer                                          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    INITIALIZATION                               │
│  XAgentCoreComponents.build()                                   │
│  - Logger, Recorder, ToolServer, FunctionHandler               │
│  - WorkingMemory, AgentDispatcher                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                 OUTER LOOP: Plan Management                     │
│  TaskHandler.outer_loop()                                       │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Phase 1: Plan Generation                                │   │
│  │   PlanAgent.initial_plan_generation()                   │   │
│  │   → PlanGenerateAgent (LLM)                             │   │
│  │   → Output: Plan tree with subtasks                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                         │                                       │
│                         ▼                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Phase 2: Plan Iteration (Memory System)                 │   │
│  │   PlanAgent.plan_iterate_based_on_memory_system()       │   │
│  │   → (Currently not implemented)                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                         │                                       │
│                         ▼                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Phase 3: Subtask Execution Loop                         │   │
│  │   for each subtask in plan:                             │   │
│  │                                                          │   │
│  │   ┌──────────────────────────────────────────────────┐  │   │
│  │   │ 3a. Inner Loop: Tool Agent Execution            │  │   │
│  │   │   TaskHandler.inner_loop(subtask)                │  │   │
│  │   │   → ToolAgent via ReACTChainSearch               │  │   │
│  │   │   → Iterative step-by-step execution             │  │   │
│  │   │   → Output: ToolNode tree (execution trace)      │  │   │
│  │   └──────────────────────────────────────────────────┘  │   │
│  │                         │                                │   │
│  │                         ▼                                │   │
│  │   ┌──────────────────────────────────────────────────┐  │   │
│  │   │ 3b. Posterior Processing: Reflection            │  │   │
│  │   │   TaskHandler.posterior_process(subtask)         │  │   │
│  │   │   → ReflectAgent (LLM)                           │  │   │
│  │   │   → Output: Summary + Reflections                │  │   │
│  │   └──────────────────────────────────────────────────┘  │   │
│  │                         │                                │   │
│  │                         ▼                                │   │
│  │   ┌──────────────────────────────────────────────────┐  │   │
│  │   │ 3c. Register in Working Memory                   │  │   │
│  │   │   WorkingMemoryAgent.register_task(subtask)      │  │   │
│  │   └──────────────────────────────────────────────────┘  │   │
│  │                         │                                │   │
│  │                         ▼                                │   │
│  │   ┌──────────────────────────────────────────────────┐  │   │
│  │   │ 3d. Plan Refinement (Conditional)                │  │   │
│  │   │   if need_for_plan_refine:                       │  │   │
│  │   │     PlanAgent.plan_refine_mode(subtask)          │  │   │
│  │   │     → PlanRefineAgent (LLM)                      │  │   │
│  │   │     → Operations: split/add/delete/exit          │  │   │
│  │   │     → Output: Modified plan tree                 │  │   │
│  │   └──────────────────────────────────────────────────┘  │   │
│  │                         │                                │   │
│  │                         ▼                                │   │
│  │   next_subtask = Plan.pop_next_subtask()                │   │
│  │   if next_subtask: continue loop                        │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                       COMPLETION                                │
│  XAgentCoreComponents.close()                                   │
│  - Download files, cleanup resources                            │
│  - Return final results                                         │
└─────────────────────────────────────────────────────────────────┘
```

### Key Control Mechanisms

#### 1. Subtask Ordering
```python
# Inorder traversal of plan tree ensures proper execution order
def pop_next_subtask(now_plan):
    all_plans = Plan.get_inorder_travel(root)
    for subtask in all_plans[current_index+1:]:
        if subtask.data.status == TaskStatusCode.TODO:
            return subtask
    return None  # All done
```

#### 2. Termination Conditions

**Inner Loop (ReACT) Termination:**
- `depth >= max_subtask_chain_length`: Maximum steps reached
- `tool_status_code == SUBMIT_AS_SUCCESS`: Successful completion
- `tool_status_code == SUBMIT_AS_FAILED`: Failed completion

**Plan Refinement Termination:**
- `operation == 'exit'`: LLM decides no more changes needed
- `status == MODIFY_SUCCESS`: Successful modification completed
- `modify_steps >= max_plan_refine_chain_length`: Maximum refinements reached

**Outer Loop Termination:**
- `Plan.pop_next_subtask() == None`: No more TODO subtasks

#### 3. Error Handling

**Tool Execution Errors:**
```python
# Retry mechanism for timeout errors
MAX_RETRY = 10
while retry_time < MAX_RETRY and status == TIMEOUT_ERROR:
    time.sleep(3)
    result, status = toolserver_interface.execute_command_client(...)
    retry_time += 1
```

**LLM Parsing Errors:**
```python
# JSON validation with retry
try:
    validate_tool_arguments(tool_args, tool_schema)
except Exception as e:
    # Use LLM to fix broken JSON
    fixed_args = objgenerator.dynamic_json_fixes(
        broken_json=tool_args,
        function_schema=tool_schema,
        error_message=str(e)
    )
```

#### 4. Human-in-the-Loop

**Interaction Points:**
- **Before subtask execution:** User can modify subtask goal
- **Before tool execution:** User can modify tool arguments (if interrupt mode)
- **Ask for help:** Agent can explicitly request human assistance

```python
if interaction.interrupt:
    user_input = interaction.receive(current_data)
    updated_data = rewrite_input_func(old_data, user_input)
```

### Agent Coordination Summary

```
┌──────────────────┐     generates      ┌──────────────────┐
│ PlanGenerate     │──────────────────→ │  Plan Tree       │
│ Agent            │                     │                  │
└──────────────────┘                     └────────┬─────────┘
                                                  │
                                    provides      │
                                    subtask       │
                                                  ▼
                                         ┌──────────────────┐
                              executes   │  Tool Agent      │   produces
                          ┌──────────────│  (via ReACT)     │────────────┐
                          │              └──────────────────┘            │
                          │                                              │
                          ▼                                              ▼
                 ┌──────────────────┐                          ┌──────────────────┐
                 │ ToolServer       │                          │ ToolNode Tree    │
                 │ Interface        │                          │ (exec trace)     │
                 └──────────────────┘                          └────────┬─────────┘
                                                                        │
                                                          analyzes      │
                                                                        ▼
                                                               ┌──────────────────┐
                                                               │ Reflect Agent    │
                                                               │                  │
                                                               └────────┬─────────┘
                                                                        │
                                                         produces       │
                                                         insights       │
                                                                        ▼
                                                               ┌──────────────────┐
                                                               │ Posterior        │
                                                               │ Knowledge        │
                                                               └────────┬─────────┘
                                                                        │
                                                                        │
                                         ┌──────────────────────────────┘
                                         │
                                         ▼
                                ┌──────────────────┐
                                │ PlanRefine       │    modifies
                                │ Agent            │────────────┐
                                └──────────────────┘            │
                                                                │
                                                                ▼
                                                       ┌──────────────────┐
                                                       │ Updated Plan     │
                                                       │ Tree             │
                                                       └──────────────────┘
```

---

## Conclusion

The XAgent workflow architecture is a sophisticated multi-agent system that combines:

1. **Hierarchical Planning:** Break complex tasks into manageable subtasks
2. **Iterative Execution:** Step-by-step tool usage with ReACT algorithm
3. **Reflection & Learning:** Analyze execution to improve future performance
4. **Dynamic Adaptation:** Refine plans based on execution feedback
5. **Memory System:** Share insights across subtasks

**Key Strengths:**
- Clear separation of concerns (planning vs. execution vs. reflection)
- Transparent decision-making through explicit reasoning
- Adaptive planning through refinement loops
- Robust error handling and retry mechanisms

**Data Flow:** User Query → Plan Tree → Execution Traces → Reflections → Updated Plan → Final Answer

**Agent Collaboration:** Agents work in sequence, each adding value:
- **PlanGenerate:** Structure the problem
- **Tool:** Execute the solution
- **Reflect:** Understand what happened
- **PlanRefine:** Adjust the approach

This architecture enables XAgent to handle complex, multi-step tasks that require both strategic planning and tactical execution.
