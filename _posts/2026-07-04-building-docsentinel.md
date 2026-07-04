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

<img src="/assets/posts/docsentinel-overview.svg" alt="DocSentinel system overview: documents and queries enter a LangGraph orchestrator of six phase-specialised agents, which draws on a hybrid retrieval layer (vector + graph) and a multi-LLM gateway, and is callable over MCP/A2A">

*Documents and queries flow into a six-agent orchestrator, which leans on a hybrid retrieval
layer and a multi-LLM gateway, and exposes itself over MCP/A2A. (Agent names here are
illustrative — see the note at the end.)*

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

<img src="/assets/posts/docsentinel-agent-graph.svg" alt="The agent graph: intake feeds threat-modelling, a risk router sends high-risk work through code review, controls and evidence before reporting while low-risk work reports directly, with a loop back to re-model on new findings">

*The graph makes control flow explicit: a risk router decides how deep to go, and new findings
loop back to re-model rather than pushing forward blindly. (Illustrative phase names.)*

## Retrieval: vector *and* graph

Security knowledge is relational — a control mitigates a threat against an asset. Pure semantic
search flattens that structure, so DocSentinel runs **hybrid retrieval**:

- **Vector search** (ChromaDB) for "what looks similar to this?"
- **Knowledge-graph retrieval** (LightRAG) for "what is this *connected* to?"

> Vector search finds the paragraph. The graph tells you which control it came from, what it
> mitigates, and what else that decision touches.

<img src="/assets/posts/docsentinel-hybrid-rag.svg" alt="Hybrid retrieval: a query fans out to vector search over ChromaDB and graph retrieval over LightRAG, whose results are fused and reranked into the context handed to the agent">

*A query fans out to both retrievers — semantic similarity and graph traversal — and the results
are fused and reranked into the context the agent actually sees.*

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

*DocSentinel is MIT-licensed and on [GitHub](https://github.com/arthurpanhku). The diagrams here
are explanatory, and the individual agent/phase names are representative rather than exact. More
write-ups on agentic AI and LLM security to come.*
