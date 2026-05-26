---
title: Hecate Examples Library
layer: example
audience: [agent, human]
stage: stable
---

# Hecate Examples Library

*Canonical examples for training and documentation.*

---

## Purpose

This directory contains well-documented examples of correct patterns used in the Hecate ecosystem. These examples serve as:

1. **Training data** for fine-tuning Ollama models
2. **Reference implementations** for developers
3. **Teaching materials** for onboarding

---

## Examples Index

| Example | Pattern | Key Concepts |
|---------|---------|--------------|
| [VERTICAL_API_HANDLERS.md](VERTICAL_API_HANDLERS.md) | API in desks | Handlers in desks, shared utilities, dependency management |
| [MESH_INTEGRATION.md](MESH_INTEGRATION.md) | Emitter/Listener | FACTS vs EVENTS, cross-agent communication, command layer |
| [PROJECTIONS.md](PROJECTIONS.md) | Read models | Event → read model, naming, SQLite patterns |
| [BIT_FLAGS_STATUS.md](BIT_FLAGS_STATUS.md) | Aggregate status | Integer bit flags, evoq_bit_flags API, efficient queries |

**Moved to philosophy/ (core patterns, not just examples):**
- [PROCESS_MANAGERS.md](../philosophy/PROCESS_MANAGERS.md) — Cross-domain coordination
- [PARENT_CHILD_AGGREGATES.md](../philosophy/PARENT_CHILD_AGGREGATES.md) — Parent identifies, child initiates

**Merged into skills/NAMING_CONVENTIONS.md:**
- QUERY_NAMING.md — now part of consolidated naming reference

---

## Example Structure

Each example should include:

1. **The Pattern** - What we're teaching
2. **Wrong Way** - Antipattern to avoid
3. **Correct Way** - The proper implementation
4. **Code Examples** - Working Erlang/Elixir code
5. **Key Takeaways** - Summary points
6. **Training Note** - What the model should learn

---

## Adding Examples

When adding a new example:

1. Create `EXAMPLE_NAME.md` in this directory
2. Follow the structure above
3. Include complete, runnable code snippets
4. Add entry to this README's index
5. Tag with date and origin

---

## Future Examples (TODO)

- [ ] Event stream patterns (stream-per-aggregate vs stream-per-type)
- [x] Projection patterns (SQLite read models)
- [x] Mesh integration (Emitter/Listener/Requester/Responder)
- [x] Process manager patterns
- [x] Bit flag status fields
- [x] CQRS query optimization (see `skills/NAMING_CONVENTIONS.md`)

---

## Usage for Fine-Tuning

These examples can be converted to training data format:

```python
# Example conversion for fine-tuning
{
    "prompt": "How should API handlers be organized in a vertical slice architecture?",
    "completion": "<content from VERTICAL_API_HANDLERS.md>"
}
```

The examples are written to be self-contained and teachable.

---

*Maintained by: Hecate Team*
*Last updated: 2026-02-10*
