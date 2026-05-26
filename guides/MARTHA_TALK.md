---
title: "Meet Martha: Your AI-Powered Software Development Guide"
layer: guide
audience: [agent, human]
stage: stable
---

# Meet Martha: Your AI-Powered Software Development Guide

_A talk about how Martha transforms the way you build software._

---

![Martha Hero](assets/martha-talk-hero.svg)

---

## The Problem

You have an idea. A vision for software that solves a real problem.

But between the idea and the running system, there's a vast wasteland of decisions:

- Where do the boundaries go?
- What are the domain events?
- How do the pieces connect?
- What do you build first?
- When is it ready to ship?

Most developers stumble through this blind. They start coding too early, discover architectural mistakes too late, and spend months refactoring what should have been designed right from the start.

**What if there was a guide?**

---

## Martha: The Application Lifecycle Tool

Martha is a Hecate plugin that implements the **Application Lifecycle (ALC)** -- a structured, AI-assisted process for building software from vision to production.

Think of it this way:

> **Martha is to the Application Lifecycle what QuickBooks is to accounting.**

QuickBooks doesn't do your accounting for you. It implements the *discipline* -- double-entry bookkeeping, chart of accounts, reconciliation. You make the decisions; QuickBooks ensures they're structured, recorded, and consistent.

Martha does the same for software development. She doesn't write your code (though she helps generate it). She implements the *discipline* -- domain discovery, event storming, vertical slicing, process-centric architecture. You make the decisions; Martha ensures they compound correctly.

---

## How It Works: Three Phases

### Phase 1: Envision

You start with a conversation. Martha's **Oracle** -- powered by your local LLM (Ollama, fully private) -- asks you about your vision. Not technical questions. Business questions:

- What problem does this solve?
- Who are the users?
- What are the key workflows?

From this conversation, Martha produces a **domain brief** -- the seed from which everything grows.

### Phase 2: Discover

This is where it gets interesting.

Martha guides you through **Event Storming** -- the domain discovery technique pioneered by Alberto Brandolini. But instead of a room full of people and physical sticky notes, Martha provides a digital board with AI assistance.

![Event Storming Flow](assets/martha-talk-event-storming.svg)

The process has seven phases:

1. **Storm** -- Chaotic, timeboxed brainstorming. Post every domain event you can think of as a sticky note. No judgment, no organization. Pure creative energy.

2. **Stack** -- Group duplicate and related stickies together.

3. **Groom** -- Pick the canonical sticky from each stack. Absorb duplicates.

4. **Cluster** -- Group stickies into bounded context clusters. This is where domain boundaries emerge.

5. **Name** -- Give each cluster a name. This name becomes the bounded context name.

6. **Map** -- Draw fact arrows between clusters. This is context mapping -- understanding how your bounded contexts communicate.

7. **Promote** -- Clusters become divisions. Each division enters its own independent lifecycle.

The AI assists throughout -- suggesting domain events you might have missed, helping identify bounded context boundaries, flagging potential coupling issues.

### Phase 3: Build

Each division enters the **8-process lifecycle wheel**:

![Lifecycle Wheel](assets/martha-talk-lifecycle-wheel.svg)

1. **Design** -- Design-level event storming within this bounded context. Identify aggregates, events, stream patterns.

2. **Planning** -- Inventory the desks (capabilities) needed. Map dependencies. Sequence sprints.

3. **Crafting** -- Code generation. Martha uses templates and your design decisions to scaffold the entire division.

4. **Refactoring** -- Can enter anywhere in the cycle. When structural issues are found, refactoring is a first-class process, not an afterthought.

5. **Debugging** -- Test suites, acceptance criteria, defect diagnosis. Quality is a phase, not a checkbox.

6. **Deployment** -- Release management, staged rollout. Ship with confidence.

7. **Monitoring** -- Health checks, SLA tracking, anomaly detection. Know when something's wrong before your users do.

8. **Rescue** -- Incident response, diagnosis, escalation. When things go wrong, Martha helps you respond systematically.

The lifecycle is a **cycle**, not a line. After rescue, you're back at design -- armed with production insights. Every iteration makes the software better.

---

## The Secret Sauce: Guided Conversations

Martha doesn't throw you into a blank canvas and say "design your architecture." That's overwhelming and leads to analysis paralysis.

Instead, she uses the **Guided Conversation Method**:

![Guided Conversation](assets/martha-talk-guided-conversation.svg)

The protocol:
1. **Frame a decision** -- One clear, bounded question.
2. **Present options** -- Tradeoffs as a table. Pros AND cons. No hidden opinions.
3. **You decide** -- Martha never decides for you.
4. **Record** -- Your decision becomes a constraint on all future decisions.
5. **Build forward** -- Each decision narrows the option space for the next.
6. **Produce an artifact** -- The output is structured data, not prose.

This creates a **decision cascade**. Your domain name constrains your discovery scope. Your discovered divisions constrain what needs designing. Your design constrains your planning. Your plan constrains what gets generated.

By the time you reach code generation, most decisions are already made. The code practically writes itself -- because the architecture was decided through structured, recorded conversations.

---

## Under the Hood

Martha runs entirely on your machine. No cloud. No telemetry. Your code, your data, your decisions.

![Architecture Overview](assets/martha-talk-architecture-overview.svg)

The architecture:

- **Frontend** (hecate-marthaw) -- A Svelte 5 component loaded at runtime by hecate-web. Organized as vertical slices -- one directory per ALC task.

- **Daemon** (hecate-marthad) -- An Erlang/OTP application running in a container. Event-sourced with CQRS (Command Query Responsibility Segregation). Every decision, every event, every phase transition is an immutable fact in the event store.

- **AI** -- Connects to your local LLMs through hecate-daemon's serve_llm. Ollama, llama.cpp, whatever you run locally. No API keys. No cloud calls.

- **Knowledge Base** -- Martha loads domain expertise from `hecate-corpus` -- a curated knowledge base of architecture patterns, naming conventions, anti-patterns, and code generation templates.

- **Zero-touch install** -- Install Martha from the Hecate appstore. One click. The container starts, the socket appears, hecate-web discovers it automatically.

---

## What Makes Martha Different

### Process Over Data

Traditional tools model development as data management: you create projects, update tasks, delete items. The verbs are CRUD.

Martha models development as **processes**. Each phase of building software is a first-class citizen with its own lifecycle, its own state, and its own history. The verbs are business actions: design, plan, craft, deploy.

### Events Are Sacred

Every decision Martha records is an immutable event. You can always trace back: *why* was this boundary drawn here? *What* tradeoffs were considered? *When* did we decide to split this aggregate?

This isn't just an audit trail. It's institutional memory. When a new team member joins, they can replay the decision history and understand not just *what* was built, but *why*.

### The Cycle Never Ends

Software isn't "done" when you ship v1. Martha's lifecycle wheel captures this truth. Monitoring feeds into rescue. Rescue feeds into design. Each cycle makes the software -- and your understanding of it -- deeper.

### Privacy First

Your domain vision, your domain events, your architectural decisions -- none of it leaves your machine. Martha runs locally. The AI runs locally. The event store is a local file. You own everything.

---

## Who Is Martha For?

- **Solo developers** building ambitious projects who need architectural discipline without a team of architects.

- **Small teams** who want structured domain discovery without the overhead of physical workshops.

- **Architects** who want to codify their process and make it reproducible.

- **Anyone** who's tired of starting to code before they understand the domain -- and paying for it later.

---

## The Name

Martha. Named for the tradition of guidance. Not a chatbot. Not an autocomplete. A guide who frames decisions, presents tradeoffs, and records your choices. She stands at the crossroads between intention and implementation, holding the keys to structured software development.

She is the tool that implements the discipline.

---

*Martha is a Hecate plugin. Hecate is your AI-powered development companion -- a desktop application that connects local AI agents to a decentralized mesh network. Learn more at [hecate.social](https://hecate.social).*
