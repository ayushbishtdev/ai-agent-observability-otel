# AI Agent Observability: OTel & Debugging Cheatsheet

A dense reference for implementing OpenTelemetry GenAI conventions and identifying agent failures in trace graphs[cite: 4, 9].

## Critical OpenTelemetry GenAI Attributes
*(Note: As of June 2026, requires `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental` to emit[cite: 9])*

| Attribute Key | Description | Use Case |
| :--- | :--- | :--- |
| `gen_ai.system` | The base engine provider processing the request (e.g., `openai`, `anthropic`)[cite: 9]. | Vendor routing and performance comparison[cite: 9]. |
| `gen_ai.request.model` | The specific model version requested (e.g., `gpt-4o`)[cite: 9]. | Granular latency/quality tracking[cite: 9]. |
| `gen_ai.usage.input_tokens` | Absolute context length processed during the request phase[cite: 7]. | FinOps / identifying context truncation[cite: 4, 7]. |
| `gen_ai.usage.output_tokens` | Total completion length returned by the inference engine[cite: 7]. | FinOps cost calculation[cite: 7]. |

## Common Agent Failure Signatures in Traces

| Failure Mode | Visual Trace Signature | Root Cause / Fix |
| :--- | :--- | :--- |
| **Runaway Tool Loop** | Massive, repetitive staircase of identical child spans branching off a parent[cite: 4]. | Agent fails to interpret tool response and retries frantically. Fix: Implement circuit breakers[cite: 4]. |
| **Silent Context Truncation** | Sudden drop in `gen_ai.usage.input_tokens` between an orchestrator span and a sub-agent span[cite: 4]. | Payload size exceeded context limits; wrapper silently dropped tokens. Fix: Implement chunking[cite: 4]. |
| **Hallucination** | Inference span generating factual claims without any preceding retrieval/database tool span[cite: 4]. | Agent bypassed RAG sequence. Fix: Mandate RAG tool invocation prior to generation[cite: 4]. |
| **Wrong Tool Selection** | Root span prompt (e.g., "reset password") mismatches invoked tool span (e.g., `calculate_mortgage`)[cite: 4]. | Flawed system prompt or routing logic. Fix: Refine tool descriptions[cite: 4]. |

## Context Propagation Code Pattern (Python)
To prevent async agent steps from fracturing into disconnected orphan traces[cite: 5]:

```python
import asyncio
from opentelemetry.context import attach, detach

async def execute_async_subtask(shared_context, workload_data):
    # Explicitly map active tracking context to prevent orphan spans
    token = attach(shared_context)
    try:
        with tracer.start_as_current_span("agent.async_subtask"):
            await asyncio.sleep(0.5) 
    finally:
        detach(token)
