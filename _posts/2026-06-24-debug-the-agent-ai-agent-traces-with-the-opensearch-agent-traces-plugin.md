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
meta_description: Use the OpenSearch Agent Traces plugin to trace a multi-agent travel planner, find a failed tool call in the trace tree, and quantify the blast radius of a sub-agent failure with PPL.
---

Your multi-agent system is returning partial results. Users get trip recommendations without weather data, and you have no idea which sub-agent failed, what tool it tried to call, or how many requests were affected. In this post, we'll trace a failing multi-agent orchestration end-to-end using the Agent Traces plugin in OpenSearch Dashboards, drill into the exact tool execution error, and quantify the blast radius with a PPL aggregation query.

This post is part of our [Observability Stack series](https://opensearch.org/blog/technical-posts/2026/06/diving-into-services-with-opensearch-and-opentelemetry/). If you haven't set up the stack yet, check the first post for instructions.

## Setting up the demo

We use the [Observability Stack](https://github.com/aws-observability/observability-stack) with its built-in multi-agent travel planner example. The travel planner is an orchestrator agent that fans out to sub-agents (weather-agent, events-agent) and calls tools (flights, currency) via an MCP server. All services are instrumented with the [OpenSearch GenAI Observability SDK](https://github.com/opensearch-project/genai-observability-sdk-py) using OpenTelemetry GenAI semantic conventions.

```bash
git clone https://github.com/aws-observability/observability-stack.git
cd observability-stack
docker compose up -d
```

The stack starts: OpenSearch, Data Prepper, OTel Collector, Prometheus, OpenSearch Dashboards, and the travel planner agents (travel-planner, weather-agent, events-agent, mcp-server). A canary service generates continuous traffic with configurable fault injection.

### The scenario

We inject a `tool_error` fault into the weather agent. The MCP tool `get_current_weather` returns a 503, the weather-agent propagates a null response, and the orchestrator produces a partial result ("Weather info temporarily unavailable"). Our goal: find the failing tool call in the trace tree, inspect its error details, and measure how many requests were impacted.

```bash
curl -X POST http://localhost:8003/plan \
  -H "Content-Type: application/json" \
  -d '{"destination": "Tokyo", "origin": "Portland", "fault": {"weather": {"type": "tool_error"}}}'
```

## The multi-agent travel planner

Before diving into the failure, look at the service topology. Navigate to **Topology Map** in the Observability Stack workspace. The Application Map shows how the travel-planner fans out to weather-agent and events-agent, both of which call the mcp-server for their tools. The weather-agent node shows a red fault indicator (20% fault rate from our injected errors).

![Application Map showing the multi-agent travel planner service topology with travel-planner, weather-agent, events-agent, and mcp-server](/assets/media/blog-images/2026-06-24-debug-the-agent-ai-agent-traces/application-map.png){:class="img-centered"}

This gives you the 30,000-foot view: four services, a clear fan-out pattern, and one service with elevated faults. To understand what went wrong inside the agent logic, you need the Agent Traces plugin.

## Opening agent traces and finding the error

Navigate to **Agent Monitoring > Traces** in the left nav. The Agent Traces plugin shows root-level agent invocations with columns for Kind, Name, Status, Latency, Tokens, Input, and Output. Each row is a complete agent invocation. The metrics bar at the top shows aggregate stats: total traces, total spans, total tokens, and latency percentiles.

![Agent Traces table showing 10 root-level agent invocations with timestamps, status, latency, and token counts](/assets/media/blog-images/2026-06-24-debug-the-agent-ai-agent-traces/agent-traces-list.png){:class="img-centered"}

Scan the Output column. Two traces show "Weather info temporarily unavailable" in their response, confirming partial failures. Click the Tokyo trace row to open the detail flyout.

## Inspecting the trace tree and finding the failed tool call

The flyout opens with the trace header (Agent badge, status, trace ID, duration, span count, token count) and a left-panel **Trace Tree** showing the full span hierarchy. You can read the execution flow directly:

```
POST /plan (root, 1.41s, 2941 tokens)
└── invoke_agent Travel Planner
    ├── chat planning (LLM, 130ms, 1277 tokens)
    ├── invoke_agent weather-agent (1.01s)
    │   └── invoke_agent Weather Assistant
    │       └── execute_tool get_current_weather ← ERROR
    ├── invoke_agent events-agent (192ms)
    │   └── invoke_agent Events Agent
    │       ├── chat events-reasoning (LLM, 109ms)
    │       └── execute_tool fetch_events_api (72ms)
    └── chat summarize (LLM, 60ms, 831 tokens)
```

The right panel shows the selected span's details. Select `execute_tool get_current_weather` in the tree. The detail panel immediately shows "Error" with status code 2 and the exception event:

![Trace detail flyout showing the trace tree with execute_tool get_current_weather selected, displaying the ToolExecutionError and 503 status](/assets/media/blog-images/2026-06-24-debug-the-agent-ai-agent-traces/trace-tool-error.png){:class="img-centered"}

The raw span reveals everything you need for RCA:

- **status.code**: `2` (ERROR)
- **status.message**: `Tool 'get_current_weather' failed: External API returned 503`
- **exception.stacktrace**: Points to `/app/main.py`, line 574, `execute_tool`
- **gen_ai.tool.name**: `get_current_weather`
- **gen_ai.tool.call.arguments**: `{"location": "Tokyo"}`
- **serviceName**: `weather-agent`

You now know: the weather-agent's `get_current_weather` tool threw a `ToolExecutionError` because the upstream API returned 503. The orchestrator caught the error (status code 0 on the root span) and returned a degraded response. Total blast radius from this single tool failure: 1.01s of wasted latency on the weather-agent path, 175 tokens consumed before the error, and one user received incomplete results.

## Quantifying the blast radius

One trace tells you what failed. To understand how widespread the problem is, write a PPL query in the Agent Traces query bar:

```sql
source = otel-v1-apm-span-*
| where isnotnull(`attributes.gen_ai.operation.name`)
| where `status.code` = 2
| stats count() as error_count by serviceName, `attributes.gen_ai.operation.name`
```

This filters to GenAI spans with error status and aggregates by service and operation type. The Agent Traces plugin auto-switches to the **Visualization** tab when you run an aggregation query, rendering a bar chart:

![Agent Traces Visualization tab showing a bar chart of error counts by service, with weather-agent showing execute_tool and invoke_agent errors](/assets/media/blog-images/2026-06-24-debug-the-agent-ai-agent-traces/blast-radius-visualization.png){:class="img-centered"}

The chart confirms: weather-agent accounts for the majority of errors (both `execute_tool` and `invoke_agent` spans fail), events-agent has a single tool error from a separate fault injection, and travel-planner carries `invoke_agent` errors from orchestrating the failed sub-agents. This is your blast radius: a single MCP tool returning 503 cascades up through the weather-agent and into the orchestrator.

## Key takeaways

- *Trace tree as a DAG debugger* - The Agent Traces flyout lets you read the full execution graph of a multi-agent system and click directly to the failing span without scanning logs.
- *GenAI semantic conventions carry context* - `gen_ai.tool.name`, `gen_ai.tool.call.arguments`, and the exception event give you the tool name, input, and error in one place.
- *PPL aggregation for blast radius* - A `stats` query over error spans tells you how many invocations were impacted and which services bore the cost, turning a single-trace finding into a fleet-wide assessment.
- *Partial failures are silent without traces* - The orchestrator returned HTTP 200 with degraded content. Without the trace tree, you would only know the user got incomplete data, not which tool failed or why.

## What's next

- **Try it yourself**: Clone the [Observability Stack](https://github.com/aws-observability/observability-stack), start the stack, and inject faults via the control panel at http://localhost:8085.
- **Explore the Trace Map**: The Agent Traces flyout includes a DAG visualization (Trace Map tab) showing the execution flow as a directed graph with color-coded nodes.
- **Instrument your own agents**: Add the [OpenSearch GenAI Observability SDK](https://github.com/opensearch-project/genai-observability-sdk-py) to your Python agents for automatic span creation with GenAI semantic conventions.

Learn more in the [Agent Traces documentation](https://observability.opensearch.org/docs/ai-observability/agent-tracing/) and the [GenAI SDK reference](https://observability.opensearch.org/docs/send-data/ai-agents/).
