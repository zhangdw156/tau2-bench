# Environment Contract

This document explains the environment contract of `tau2-bench` from the perspective of an external reader. You do not need to read the source code to understand it.

In `tau2-bench`, an evaluation case is not just a prompt-response exchange. It is a structured interaction between:

- a task definition
- a domain environment with world state and tools
- an agent
- a user simulator
- an orchestrator that runs the interaction
- an evaluator that scores the resulting trajectory

The environment contract is the set of rules that make these pieces work together in a predictable way.

## What The Environment Contract Is

At a high level, the contract says:

- a task defines what the case is about and how success should be judged
- a domain environment defines the world the case runs inside
- the agent and user can only interact with that world through messages and tools
- the orchestrator records the full interaction as a trajectory
- the evaluator scores the case by comparing the trajectory against the task's evaluation criteria

This means `tau2-bench` evaluates not only what the agent says, but also what the agent actually does to the environment.

## Core Actors

Every text-based case involves five conceptual actors.

### Task

A task is the unit of evaluation. It specifies:

- the user scenario
- optional initial state
- evaluation criteria

The task tells the benchmark what the user wants, what information the user knows, and what counts as a correct resolution.

### Environment

The environment is the domain-specific world in which the case runs. It contains:

- the domain policy
- the domain database or state
- the tools available to the agent
- optionally, tools available to the user simulator

Examples of domain data include products, users, accounts, orders, flights, tasks, or knowledge documents, depending on the domain.

The environment is the source of truth for stateful actions. If an agent changes an order, cancels a flight, updates an address, or creates a task, that change happens in the environment.

### Agent

The agent is the system being evaluated. In the standard setup, it is an LLM-based conversational participant that:

- receives the domain policy
- receives the tool schemas
- sees the conversation history
- chooses either to send a message or make a tool call on each turn

The agent cannot directly mutate the world. It must do so through tools exposed by the environment.

### User Simulator

The user simulator is another model-driven participant. It is not a static script. It is given:

- global user simulation instructions
- the task-specific user scenario
- the conversation history
- optionally, user-side tools

Its job is to behave like a realistic user within the scope of the task.

### Orchestrator

The orchestrator is the runtime controller. It:

- initializes the environment, agent, and user
- decides who speaks next
- routes tool calls to the environment
- appends messages and tool results to the trajectory
- detects stop conditions

The orchestrator does not decide whether the agent is correct. It only runs the interaction.

## World State

Each domain has a world state that acts like a domain database. For example, in the retail domain, the world state contains:

- products
- users
- orders

This state is loaded from domain data files before a simulation begins. During a case, tool calls may read from or mutate this state.

This is one of the key differences between `tau2-bench` and a pure prompt benchmark:

- a pure prompt benchmark often evaluates text only
- `tau2-bench` evaluates text plus stateful behavior in an explicit environment

## Policy And Tools

The domain policy describes what the agent is allowed or expected to do. It is part of the agent's runtime context.

The tools define the environment's action surface. Tools fall into a few common categories:

- read tools, which retrieve information
- write tools, which modify state
- generic tools, such as transfer or calculation helpers

The environment contract requires that state-changing operations go through tools. An agent does not directly edit the environment. It requests actions through tool calls, and the environment executes them.

This gives the benchmark a clean separation between:

- what the agent intends to do
- what the environment actually does

## Messages

The environment contract is built around a small set of message types.

### System Messages

These define runtime instructions for a participant. For the agent, this includes the domain policy and agent operating instructions. For the user simulator, this includes user simulation guidelines and the task scenario.

### User Messages

These are messages from the simulated user to the agent.

### Assistant Messages

These are messages from the agent to the user, or tool-calling messages from the agent to the environment.

### Tool Messages

These are environment responses to tool calls.

Together, these messages form the full history of the case.

## Tool Calls And Tool Results

On any given turn, the agent must choose one of two actions:

- send a message to the user
- make a tool call

It cannot do both at the same time.

When a tool call is made:

1. the orchestrator sends the call to the environment
2. the environment executes the tool
3. the environment returns a tool result
4. the tool result is inserted into the trajectory
5. the caller receives the result on the next step

This means tool use is explicit, observable, and replayable.

## Conversation Lifecycle

In standard text mode, a case runs as a turn-based interaction.

The typical lifecycle looks like this:

1. the environment, agent, and user are created
2. the orchestrator initializes the case
3. a default opening assistant message may start the conversation if the task does not provide prior history
4. the user responds
5. the agent responds or calls a tool
6. if a tool is called, the environment returns a tool result
7. the process repeats until the conversation ends
8. the full interaction is saved as a trajectory
9. the evaluator scores the trajectory

The conversation ends when one of the stop conditions is reached. Common examples are:

- the user simulator emits a stop token because the task is complete
- the agent explicitly stops
- the run hits a guardrail such as max steps, too many errors, or timeout

## Message History As Runtime Memory

Both the agent and the user simulator receive accumulated message history as part of their state.

For the agent, the effective runtime context includes:

- system instructions
- past user messages
- past assistant messages
- tool calls previously made by the agent
- tool results returned by the environment

For the user simulator, the runtime context includes:

- user simulation instructions
- the task scenario
- prior conversation history
- relevant tool results when applicable

This means both participants operate over a live conversation state, not isolated single-turn prompts.

## What A Trajectory Is

A trajectory is the full record of what happened during a case.

In text mode, it is a linear sequence of messages such as:

- assistant opening message
- user request
- assistant clarification
- assistant tool call
- tool result
- assistant follow-up
- user confirmation
- assistant final update
- user stop

The trajectory is the canonical artifact used for evaluation.

It captures both:

- conversational behavior
- environment interaction

## Evaluation Contract

The evaluator does not score a case based on an informal impression. It uses the task's evaluation criteria.

These criteria can include:

- expected actions
- expected environment assertions
- expected communicated information
- natural-language assertions
- reward basis

The reward basis determines which of the available criteria actually count toward the final score.

### Expected Actions

These define the actions a correct solution is expected to perform. They are usually expressed as tool calls with expected arguments.

### Environment Assertions

These define checks over the environment after the case completes, such as whether a task exists, an order status changed, or a record has a specific value.

### Communicated Information

These specify information the agent should tell the user.

### Natural-Language Assertions

These are higher-level semantic requirements, such as:

- the agent explained the key finding correctly
- the agent confirmed success clearly
- the agent followed the expected conversational behavior

These may be judged with an LLM-based evaluator.

## Programmatic Scoring Versus LLM Judging

Not every part of evaluation uses an LLM judge.

Some checks are programmatic:

- whether the correct tool call occurred
- whether the final environment state matches the expected state
- whether assertions over the environment pass
- whether certain text snippets were communicated

Some checks are LLM-judged:

- whether a natural-language assertion is semantically satisfied by the trajectory

So the evaluator is best understood as a scoring coordinator. It combines deterministic checks with optional judge-model checks, depending on the task.

## How Pass Or Fail Is Determined

For most benchmark usage, a case is considered successful when its final reward is effectively `1.0`.

If the case fails an essential requirement, the reward usually becomes `0.0`.

This commonly happens when:

- the agent did not complete the required world-state change
- the agent did not satisfy a required natural-language assertion
- the run ended prematurely

In practice, many tasks behave like strict pass/fail tasks even if the scoring is internally composed from multiple parts.

## Replay And State Comparison

One of the most important parts of the environment contract is that final correctness can be checked by replay.

When environment-based scoring is used, the evaluator reconstructs outcome in two independent ways:

- a predicted environment is built by replaying the actual trajectory
- a gold environment is built by applying the task's expected actions

The evaluator then compares the resulting states.

This is what makes it possible to say:

- the agent really changed the order correctly
- the agent really created the record
- the agent really updated the environment to the expected final state

instead of merely saying:

- the agent seemed to say the right thing

## Environment Isolation

`tau2-bench` treats each simulation as an isolated environment instance.

This has two major consequences:

- one case does not contaminate another case's world state
- evaluation replay does not contaminate the original run state

Conceptually, the benchmark uses fresh environment instances for:

- the actual simulation run
- replaying the observed trajectory
- building the gold reference state for comparison

This means world-state mutations are local to the instance being used for that purpose.

The original domain data acts as a starting template, not as a shared mutable global database for all runs.

## Practical Mental Model

If you want a compact way to think about the contract, use this:

- a task says what should happen
- an environment says what can happen
- the agent and user interact through messages and tools
- the orchestrator records what actually happened
- the evaluator decides whether what happened matches what should have happened

That is the core environment contract of `tau2-bench`.

## Summary

The environment contract exists so that benchmark cases can be:

- stateful rather than text-only
- observable rather than implicit
- replayable rather than subjective
- isolated rather than globally mutated
- scoreable by explicit task-defined criteria

As a result, `tau2-bench` can evaluate not only whether an agent sounds helpful, but whether it correctly operates in a constrained domain world and produces the intended final outcome.
