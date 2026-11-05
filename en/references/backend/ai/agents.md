# Agent engineering standards

This document sets rules for layering, failure handling, tools, context, cost, and security in agent services. For general error principles, see [Error handling principles](../../general/error-handling.md). For general retries and timeouts, see [Resilience](../reliability/resilience.md). For retrieval augmentation, see [RAG](rag.md).

## Layering

### Start with the simplest approach [Recommended]

If a single model call plus retrieval can solve it, do not build an agent. Orchestrate tasks with fixed steps in code, and let the model decide the flow only when the number of steps cannot be known in advance. Use multiple agents only for tasks that can run in parallel and depend little on each other, and have subagents return only concise summaries.

**Why:** Agents trade latency and cost for results. Multi-agent systems consume more tokens, and errors also accumulate step by step.

**Source:** Anthropic Building Effective Agents, Anthropic: Multi-Agent Systems

### Layer by responsibility [Recommended]

| Layer | Responsibility | Does not |
|---|---|---|
| Orchestration | Drives the loop, decides the next step, determines when to stop | Implement tools |
| Model calls | Assembles requests, parses responses, isolates the SDK or framework | Make business decisions |
| Tools | Validates parameters, executes, returns results | Know about loop state |
| Storage | Persists run records | Contain decision logic |

When using a framework, know what it actually sends to the model. Framework types do not cross the model call layer.

**Source:** Anthropic Building Effective Agents

## Tool call failure handling

### Tool failures must not bring down the process [Required]

Catch all tool execution failures in the tool layer and convert them into tool results handed back to the model, so the process and current task keep running. The result states what failed, why, and what can be done next. Do not return stack traces, internal identifiers, or secrets. Also log the failure. Program defects must still be surfaced and fixed.

Invalid parameter values, downstream API failures, and unmet business rules all go in the tool result. Problems the model can hardly fix on its own, such as a nonexistent tool or an invalid request structure, are returned by the tool server as request errors of the protocol in use. When the agent loop receives a protocol error, or finds the model called a nonexistent tool, it must still synthesize an error tool result and write it back, so that the call is closed.

**Why:** A process exit interrupts all in-progress tasks, and vague errors leave the model only guessing.

**Example:**

```json
{"tool_call_id": "call_7", "status": "error", "error_type": "invalid_argument", "content": "Departure date must be after today. Today is 2026-11-05. Correct departure_date and call again."}
```

**Source:** Team, MCP Tools Specification, Anthropic: Handling Tool Calls, Anthropic: Tool Design

### Close every tool call [Required]

Set a timeout on every tool call, and write back a matching result on timeout, cancellation, failure, and nonexistent tool too. When a tool with side effects times out, the result says "status unknown", not "failed", to keep the model from repeating the action.

**Why:** Calls with no result corrupt the conversation history, and later model calls may fail outright.

**Source:** MCP Tools Specification, Anthropic: Handling Tool Calls

## Run control

### Do not treat incomplete results as success [Required]

When a model call fails or the response is truncated, handle it as a failure and record the error type. Do not treat an incomplete response as a successful result.

### Loops must have stop conditions [Required]

Limit maximum steps, total duration, and token budget on every run. When a limit is exceeded, end as a failure and state which limit was hit.

**Source:** Anthropic Building Effective Agents, OWASP LLM Top 10

## Tool design

### Keep a small set of tools with clear boundaries [Recommended]

Design tools around tasks. Do not wrap APIs one by one. Multi-step operations can merge into one tool. Functions do not overlap: if engineers cannot tell which one to use, the model cannot either. Across services, prefix with the service name and resource name.

**Source:** Anthropic: Tool Design, Anthropic: Context Engineering, Anthropic: Tool Definitions

### Leave no ambiguity in descriptions and parameters [Recommended]

Descriptions state the purpose, when to use and not use the tool, and parameter meanings and limits. Parameters use clear names and strict schemas, such as `user_id` rather than `user`. Change inputs that are easy to get wrong, for example by requiring absolute paths. Evaluate with multi-step tasks whose results can be verified. Repeated parameter errors usually mean the description is unclear.

**Why:** The description is the top factor in tool effectiveness, and small wording changes can noticeably change results.

**Source:** Anthropic: Tool Definitions, Anthropic: Tool Design, Anthropic Building Effective Agents

### Return bounded, high-signal results [Recommended]

Return only the fields the model needs for its next step, and prefer meaningful stable identifiers. Paginate, filter, or truncate large results by default, and when truncating, say how to narrow the query.

**Source:** Anthropic: Tool Design, Anthropic: Tool Definitions

## Context management

### Load on demand and compact proactively [Recommended]

Keep lightweight references such as paths, links, and query conditions in context, and read the content with tools when needed. In long tasks, first clear raw tool results that are no longer needed. When nearing the limit, compact into a summary, with priority on not losing key information. Write progress that spans context windows to external notes.

**Source:** Anthropic: Context Engineering

## Cost and observability

### Instrument with GenAI semantic conventions [Recommended]

A run, a model call, and a tool call each produce a span, named `invoke_agent {gen_ai.agent.name}`, `{gen_ai.operation.name} {gen_ai.request.model}`, and `execute_tool {gen_ai.tool.name}` respectively. Record at least `gen_ai.conversation.id`, `gen_ai.request.model`, `gen_ai.response.finish_reasons`, and `error.type`. For other requirements, see [Distributed tracing](../observability/tracing.md).

**Source:** OTel GenAI Semantic Conventions

### Keep prompts and responses out of telemetry by default [Required]

Message content, tool parameters, and results can contain personal data, so do not record them by default. Enable recording explicitly when troubleshooting requires it, and prefer writing to controlled external storage. For other requirements, see [Privacy](../security/privacy.md).

**Source:** OTel GenAI Semantic Conventions, Anthropic: Multi-Agent Systems

### Track and limit usage [Required]

**Applies to:** production service

Record `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, and cache tokens, aggregated by user and session. Set limits on request rate, quota, input size, and queue length, and alert on abnormal consumption.

**Why:** Unlimited calls lead to runaway costs and service unavailability.

**Source:** OWASP LLM Top 10, OTel GenAI Semantic Conventions

## Prompt injection protection

Prompt injection cannot be fully prevented. The goal of protection is to limit the impact when an injection succeeds. For general injection protection, see [Input and injection](../security/input-and-injection.md).

### Treat all external content as data [Required]

Web pages, emails, uploaded files, third-party APIs, and retrieved content are all untrusted. Put them in tool results and label them clearly. Do not splice them into the system prompt, and treat them as untrusted when reused after persistence too. Run adversarial tests with direct and indirect injection regularly.

**Source:** OWASP LLM Top 10, Anthropic: Handling Tool Calls

### Enforce permissions in code [Required]

Provide only the tools and operations the task needs. Do not provide open tools such as arbitrary commands or arbitrary URLs. Tools run as the current user, downstream systems decide authorization, and secrets stay out of prompts. For operations such as deleting, sending externally, and paying, show the parameters before execution and have the user confirm. The model cannot skip this. For other requirements, see [Authorization](../security/authorization.md).

**Source:** OWASP LLM Top 10, MCP Tools Specification

### Treat model output as untrusted input [Required]

Before model output goes downstream, validate its format with deterministic code, encode it for where it is used, and use parameterized queries for database operations.

**Source:** OWASP LLM Top 10
