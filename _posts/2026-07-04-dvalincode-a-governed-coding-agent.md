---
layout: post
title: "DvalinCode: a coding agent you can actually let into a regulated codebase"
date: 2026-07-04 09:00:00 +0800
tags: [agentic-ai, security, developer-tools]
description: "An AI coding agent built for regulated environments — org-policy enforcement over tools, models, shell and file paths, sandboxed execution, and a hash-chained tamper-evident audit log."
---

Most coding agents assume trust: give them a shell, a model key, and your repo, and let them
go. That assumption is fine on a hobby project and a non-starter in a bank. [DvalinCode](https://github.com/arthurpanhku)
is my answer to a narrower question — **what does a coding agent look like if it has to satisfy an
auditor?**

It's an AI coding agent in TypeScript, MIT-licensed, with 183 passing tests, that runs as terminal,
web, and desktop from a single binary. The interesting parts aren't the frontends, though — they're
the constraints.

## The shape of it

<img src="/assets/posts/dvalincode-architecture.svg" alt="DvalinCode architecture: three frontends from one binary feed an agent core, whose actions pass through a policy gate into a sandbox, with every step appended to a hash-chained audit log">

*One binary serves three frontends. Every action the agent takes flows through a policy gate,
executes in a sandbox, and is appended to a tamper-evident log.*

Three ideas do the work: a **bounded execution loop**, a **policy gate** in front of every
side-effect, and an **audit log you can't quietly edit**.

## A loop, not a free-for-all

The agent runs a fixed execution loop rather than an open-ended "keep calling tools until done".
Making the control flow explicit is what lets you insert a policy check *before* any action with
side effects, and an audit append *after* every one.

<img src="/assets/posts/dvalincode-loop.svg" alt="A conceptual eight-step agent loop: receive task, plan, policy check, select tool, execute in sandbox, observe, append audit, reflect, then loop back or finish">

*The execution loop, conceptually. The point isn't the exact stages — it's that policy and audit
sit on fixed edges of the cycle, so nothing side-effecting slips past them.*

Because the loop is a state machine, the deterministic parts stay deterministic: routing, retries,
and gating don't depend on the model behaving.

## Policy over tools, models, shell, and file paths

The policy gate is the heart of it. Before the agent runs a tool, calls a model, executes a shell
command, or touches a file, the request is checked against an org policy:

- **Tools** — only an allow-listed set is callable.
- **Models** — which providers/models are permitted (keeping code off unapproved endpoints).
- **Shell** — commands are constrained, not handed a blank prompt.
- **File paths** — reads and writes are scoped; the agent can't wander outside its lane.

Denied actions never reach the sandbox. In a regulated shop this is the difference between "an AI
touched our code" and "an AI operated inside a documented, enforced boundary."

## An audit log an auditor trusts

Every action is appended to a **hash-chained** log: each entry stores the hash of the previous one,
so the entries form a chain.

<img src="/assets/posts/dvalincode-audit-chain.svg" alt="Three chained audit entries, each hashing the previous entry's hash together with its own data; altering any past entry breaks every later hash">

*Each entry hashes the previous entry's hash together with its own data. Change one line of history
and every downstream hash stops matching — so tampering is detectable, not silent.*

This is the same tamper-evidence idea a blockchain uses, minus the distributed consensus you don't
need for a local audit trail. You don't get "nobody can ever change it"; you get something better
for compliance: **if anyone does, it shows.**

## Why one binary, three frontends

The same core ships as a terminal app, a web UI, and a desktop app. Beyond convenience, it means
the policy and audit guarantees are identical no matter how a developer drives the agent — there's
no "relaxed" mode hiding behind a different entry point.

## Takeaways

- **Constrain the loop before you enrich it.** A fixed cycle is what gives policy and audit a place
  to live.
- **Put enforcement in front of side-effects, not in the prompt.** Prompts advise; gates enforce.
- **Make history tamper-evident, not just stored.** For anything auditable, detectability beats
  good intentions.

---

*DvalinCode is MIT-licensed and on [GitHub](https://github.com/arthurpanhku). The diagrams here are
explanatory, not screenshots. Next up: notes on the policy language itself.*
