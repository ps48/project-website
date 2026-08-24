---
layout: post
title: "Debug the agent: AI agent traces with the OpenSearch Agent Traces plugin"
category: blog
tags: [observability, ai-observability, agent-traces, genai-sdk, opensearch]
authors:
    - pshenoy
    - reddyvam
date: 2026-06-24
categories:
  - technical-posts
meta_keywords: agent traces, AI observability, OpenSearch, OpenTelemetry, GenAI SDK, multi-agent, trace tree, tool failure, root cause analysis, LLM debugging
meta_description: Use the OpenSearch Agent Traces plugin to trace a multi-agent travel planner, find a failed tool call in the trace tree, and measure how many requests a sub-agent failure affected using PPL.
---

Your multi-agent system is returning partial results. Users get trip recommendations without weather data, and you cannot tell which sub-agent failed, which tool it called, or how many requests were affected. In this post, we'll trace a failing multi-agent orchestration end-to-end using the Agent Traces plugin in OpenSearch Dashboards, drill into the exact tool execution error, and measure how many requests it affected with a PPL aggregation query.

This post is part of our [Observability Stack series](https://opensearch.org/blog/single-pane-of-glass-for-all-your-telemetry-the-opensearch-observability-stack/). If you haven't set up the stack yet, check the first post for instructions.

## Setting up the demo

This example uses the [Observability Stack](https://github.com/opensearch-project/observability-stack) with its built-in multi-agent travel planner. The travel planner is an orchestrator agent that delegates to sub-agents (weather-agent, events-agent) and calls tools (flights, currency) through an MCP server. All services are instrumented with the [OpenSearch GenAI Observability SDK](https://github.com/opensearch-project/genai-observability-sdk-py) using OpenTelemetry GenAI semantic conventions.

Clone the repository and start the stack:

```bash
git clone https://github.com/opensearch-project/observability-stack.git
cd observability-stack
docker compose up -d
```

The stack starts: OpenSearch, Data Prepper, OTel Collector, Prometheus, OpenSearch Dashboards, and the travel planner agents (travel-planner, weather-agent, events-agent, mcp-server). A canary service generates continuous traffic with configurable fault injection.

### The scenario

You inject a `tool_error` fault into the weather-agent. The MCP tool `get_current_weather` returns a 503, the weather-agent propagates a null response, and the orchestrator produces a partial result ("Weather info temporarily unavailable"). Your goal is to find the failing tool call in the trace tree, inspect its error details, and measure how many requests it affected.

Send a planning request with a fault directive to reproduce the failure:

```bash
curl -X POST http://localhost:8003/plan \
  -H "Content-Type: application/json" \
  -d '{"destination": "Tokyo", "origin": "Portland", "fault": {"weather": {"type": "tool_error"}}}'
```

## The multi-agent travel planner

Before looking at the failure, start with the service topology. Open **Application Map** from the left navigation. The map shows the travel-planner calling the weather-agent and the events-agent, both of which call the mcp-server for their tools. The weather-agent node shows a red segment on its request ring, which marks the injected faults, as shown in the following image.

![Application Map showing the multi-agent travel planner service topology with travel-planner, weather-agent, events-agent, and mcp-server](/assets/media/blog-images/2026-06-24-debug-the-agent-ai-agent-traces/application-map.png){:class="img-centered"}

This gives you a high-level summary: four services, a clear call pattern, and one service with elevated faults. To understand what went wrong inside the agent logic, you need the Agent Traces plugin.

## Opening agent traces and finding the error

Open the **Agent Traces** plugin from the left navigation. The **Traces** tab lists each root-level agent invocation, one row per `POST /plan` request, with its status, latency, and token count. The metrics bar summarizes the run: total traces, total spans with an error count, total tokens, and P50/P99 latency, as shown in the following image.

![Agent Traces table listing root-level agent invocations with time, status, latency, and token counts, and a metrics bar reporting 10 traces, 171 spans with 12 errors, 26K tokens, and P50/P99 latency](/assets/media/blog-images/2026-06-24-debug-the-agent-ai-agent-traces/agent-traces-list.png){:class="img-centered"}

Every row reports a **Success** status, even though the metrics bar counts 12 span errors. That gap is the point: the orchestrator returns HTTP 200 with a degraded response, so the trace status stays green while the errors sit lower in the tree. Open a trace to inspect its spans. The trace with 1.41s latency and 2,941 tokens is the one analyzed below.

## Inspecting the trace tree and finding the failed tool call

The flyout opens with the trace header (Agent badge, Success status, trace ID, duration 1.41s, 30 spans, 2,941 tokens) and three tabs: **Trace Tree**, **Trace Map**, and **Timeline**. The **Trace Tree** shows the full span hierarchy, so you can read the execution flow directly:

```
POST /plan  (root agent trace, 1.41s, 30 spans, 2,941 tokens)
└── invoke_agent Travel Planner  (1.41s)
    ├── chat planning  (LLM, 130ms, 1,277 tokens)
    ├── invoke_agent weather-agent  (1.01s)  [warning]
    │   └── invoke_agent Weather Assistant  (1.00s, 175 tokens)  [warning]
    │       └── execute_tool get_current_weather  (1.73ms)  <- ERROR
    └── invoke_agent events-agent  (192ms)
        └── invoke_agent Events Agent  (182ms)
            ├── chat events-reasoning  (LLM, 109ms, 658 tokens)
            └── execute_tool fetch_events_api  (72ms)
```

Select `execute_tool get_current_weather` in the trace tree. The right panel then shows the span's details: an **Error** status, the operation `execute_tool`, a 1.73ms duration, and an `exception` event in the Raw Span panel, as shown in the following image.

![Trace detail flyout with execute_tool get_current_weather selected in the trace tree, the right panel showing an Error status and an exception event in the raw span](/assets/media/blog-images/2026-06-24-debug-the-agent-ai-agent-traces/trace-tool-error.png){:class="img-centered"}

The raw span includes the fields you need for root cause analysis:

- **status.code**: `2` (ERROR), while the root `POST /plan` span stays Success.
- **exception event**: an `exception` entry on the span with `escaped: False` and a stack trace, visible in the Raw Span panel.
- **operation**: `execute_tool`, span name `get_current_weather`, duration 1.73ms.
- **serviceName**: `weather-agent`.

The exception event and the scenario line up: the `get_current_weather` tool raised an error after the upstream weather API returned 503. The orchestrator caught the failure (the root span stays Success) and returned a degraded response. The cost of this single tool failure is visible in the same trace: 1.01s spent on the weather-agent branch, 175 tokens consumed by the Weather Assistant before it failed, and one user receiving an incomplete plan. Nothing at the top level signals the problem.

## Measuring how many requests were affected

One trace tells you what failed. To see how many invocations were affected, run a PPL query in the Agent Traces query bar:

```sql
source = otel-v1-apm-span-*
| where isnotnull(`attributes.gen_ai.operation.name`)
| where `status.code` = 2
| stats count() as error_count by serviceName, `attributes.gen_ai.operation.name`
```

This filters to GenAI spans with an error status and aggregates by service and operation type. The Agent Traces plugin switches to the **Visualization** tab when you run an aggregation query, rendering a bar chart, as shown in the following image.

![Agent Traces Visualization tab showing a bar chart of error counts by service, with events-agent at 1, travel-planner at 5, and weather-agent at 6 split into invoke_agent and execute_tool spans](/assets/media/blog-images/2026-06-24-debug-the-agent-ai-agent-traces/blast-radius-visualization.png){:class="img-centered"}

The chart accounts for all 12 span errors, split by service and colored by operation type. The weather-agent has 6 errors (4 `invoke_agent` spans and 2 `execute_tool` spans), the travel-planner has 5 `invoke_agent` spans, and the events-agent has 1 `invoke_agent` span. The two `execute_tool` errors on the weather-agent are the MCP tool returning 503. The `invoke_agent` errors on the weather-agent and the travel-planner are those services recording that the call below them failed. One tool fault therefore produces errors at three levels: the tool itself, the agent that called it, and the orchestrator that called that agent.

## Key takeaways

- *Trace tree for multi-agent debugging* - The Agent Traces flyout shows the full execution graph of a multi-agent request and lets you select the failing span directly, without scanning logs.
- *GenAI semantic conventions carry context* - The span name, operation type, and exception event give you the tool, the operation, and the error in one place.
- *PPL aggregation for impact* - A `stats` query over error spans shows how many invocations were affected and which services carried the errors, turning a single-trace finding into a fleet-wide count.
- *Partial failures are silent without traces* - The orchestrator returned HTTP 200 with degraded content. Without the trace tree, you would know only that the user got incomplete data, not which tool failed or why.

## What's next

- **Try it yourself**: Clone the [Observability Stack](https://github.com/opensearch-project/observability-stack), start the stack, and inject faults through the control panel at http://localhost:8085.
- **Explore the Trace Map**: The Agent Traces flyout includes a **Trace Map** tab that renders the execution as a directed graph with color-coded nodes.
- **Instrument your own agents**: Add the [OpenSearch GenAI Observability SDK](https://github.com/opensearch-project/genai-observability-sdk-py) to your Python agents for automatic span creation with GenAI semantic conventions.

Learn more in the [Agent Traces documentation](https://observability.opensearch.org/docs/ai-observability/agent-tracing/) and the [GenAI SDK reference](https://observability.opensearch.org/docs/send-data/ai-agents/).
