---
layout: post
title: "Building DocSentinel: six agents instead of one big prompt"
date: 2026-07-04
tags: [agentic-ai, security, rag]
description: "Why I built DocSentinel as six specialised agents coordinated by a graph, and what hybrid vector + knowledge-graph retrieval actually buys a security review."
---

[DocSentinel](https://github.com/arthurpanhku) is an open-source platform that automates
security assessment across the software lifecycle. The interesting decision wasn't the model —
it was refusing to make it *one* model call. Instead it's **six specialised agents** coordinated
by a graph. This post is about why.

## The problem with one big prompt

The tempting first version of any LLM tool is a single prompt: dump the context in, ask for the
answer, ship it. For a security review that falls apart quickly:

- **Context dilution.** A threat-modelling instruction and a report-writing instruction pull the
  model in different directions when they share one prompt.
- **No separation of concerns.** You can't test, swap, or harden a "phase" that doesn't exist as
  its own unit.
- **No place to put guardrails.** Injection defence and output validation want clear boundaries —
  a monolith has none.

A security assessment is not one task. It's intake, threat modelling, control review, evidence
checking, and reporting — each a *different* job with different inputs and different failure modes.
So each becomes an agent.

## Six agents on a graph

The agents are orchestrated with **LangGraph**, which lets the workflow be an explicit state
machine rather than an implicit chain of hope. Each node owns one phase, has its own prompt and
tools, and hands typed state to the next:

```python
graph.add_node("intake", intake_agent)
graph.add_node("threat_model", threat_agent)
graph.add_node("control_review", control_agent)
graph.add_edge("intake", "threat_model")
graph.add_conditional_edges(
    "threat_model",
    route_by_risk,          # high-risk findings loop back for a deeper pass
    {"deep": "control_review", "done": "report"},
)
```

The payoff: the parts you *can* make deterministic (routing, gating, retries) stay deterministic,
and only the genuinely fuzzy work is left to the model.

## Retrieval: vector *and* graph

Security knowledge is relational — a control mitigates a threat against an asset. Pure semantic
search flattens that structure, so DocSentinel runs **hybrid retrieval**:

- **Vector search** (ChromaDB) for "what looks similar to this?"
- **Knowledge-graph retrieval** (LightRAG) for "what is this *connected* to?"

> Vector search finds the paragraph. The graph tells you which control it came from, what it
> mitigates, and what else that decision touches.

For a reviewer, the second question is the one that matters — and it's the one a flat index can't
answer.

## Making it callable: MCP and A2A

DocSentinel exposes itself over the **Model Context Protocol (MCP)** so other agents and IDEs can
call it as a tool, and speaks **A2A** for agent-to-agent hand-off. The lesson that generalises:
build the capability as a *protocol endpoint*, not a UI feature, and it composes into workflows you
didn't anticipate.

## What I'd tell past me

- **Guardrails are architecture, not a filter you bolt on later.** Design the boundaries first.
- **Prefer a state machine to a prompt chain** wherever the control flow is knowable.
- **Model the relationships, not just the text.** In security, the edges *are* the content.

---

*DocSentinel is MIT-licensed and on [GitHub](https://github.com/arthurpanhku). This is the first
post here — more write-ups on agentic AI and LLM security to come.*
