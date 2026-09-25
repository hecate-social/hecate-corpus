---
title: "ANTIPATTERNS: Verification — What We Wrote Down vs What We Built"
layer: skill
audience: [agent, human]
stage: stable
---

# ANTIPATTERNS: Verification — What We Wrote Down vs What We Built

*Demons about the gap between a repository's prose and its behaviour. Every other
file here catalogues a technical mistake. This one catalogues the mistake of
believing the technical mistakes were caught.*

[Back to Index](INDEX.md)

---

## 🔥🔥🔥 Demon 53: Comments That State Intent as Fact

**Date exorcised:** 2026-08-07
**Where it appeared:** `hecate-dronex` (four files in one day), `beam-campus-net/CLAUDE.md`
**Cost:** Every island in an archipelago restarted its population from seed on
every deploy, for weeks, while a module header explained in detail how it did not.

### The Lie

"The comment above this function describes what the function does."

### What Happened

Four in a single day, all in code written by the same author, all landing in the
same commit as the code they misdescribe:

| Prose | Code |
|---|---|
| "restore reads BACKWARD to the most recent snapshot and then forward from there" | `stream_forward(StoreId, Stream, 0, 5000)` — forward from the beginning, capped |
| "counters live on the island so they survive a deploy, because the roster snapshot that survives a restart IS the island" | the snapshot stored `entries` and `capacity`; every counter reset on every restart |
| "`max_ticks` is 1200 at 20 Hz" beside a call timeout justified as "longer than an engagement can take" | the fight had moved off the call months earlier |
| "`mix ecto.create` needs Postgres" | `adapter: Ecto.Adapters.SQLite3` |

None of these were lies about somebody else's code. In each case the author wrote
down the design, then wrote something else, and the design is what got preserved
**in the present tense**.

### Why It Survives

The house prose style is emphatic: capitals, ⚠ markers, a paragraph of hard-won
reasoning. That register is correct for **why** and actively harmful for **what**,
because a sentence that reads as settled does not get re-checked. In all four
cases the comment is the reason nobody looked. A hedged sentence would have
survived less well and cost less.

### The Rule

**A comment that describes BEHAVIOUR must have a test named after it.**

If the header says "reads backward to the newest snapshot", there is a test called
`reads_backward_to_the_newest_snapshot`. If that test cannot be written, the
sentence is a PLAN and must be marked as one:

```erlang
%% INTENDED: restore should read backward to the newest snapshot.
%% ACTUAL: reads forward from 0, capped at 5000. See TODO-nnn.
```

Reasons are durable and belong in comments. Descriptions are verifiable and belong
in tests. When in doubt, write less prose: **a shorter comment cannot be as wrong.**

---

## 🔥 Demon 54: A Test for the Name and None for the Fold

**Date exorcised:** 2026-08-07
**Where it appeared:** `hecate-dronex/apps/hecate_dronex/src/breed_a_roster/roster_log.erl`
**Cost:** the same as Demon 53 — every lineage, every deploy, for weeks

### The Lie

"This module is tested."

### What Happened

`roster_log` had exactly one test: that its stream id was a shape reckon-db would
accept. That was the part the author had recently argued with themselves about,
because a bad stream id had once wedged an island for four minutes.

The **fold** — the function that turns stored events back into a roster, the whole
reason the module exists — had no test at all. It raised `badmap` on the first
event of every restore it ever attempted.

### The Signature

**The untested part is the part you were confident about.** Attention went to the
recently painful thing; the rest was written on autopilot and shipped unexamined.
Look for a module whose tests cluster around one concern and are silent about its
main verb.

A second signature, from the same day: every fault sat at a **seam**. Record
versus map, ETS versus disk, colocated hook versus bundler path, a CSS margin
versus text content, a hex string versus a lightness band. Unit tests inside each
module were fine. Nothing crossed a join.

### The Rule

**Every seam gets one test built from the real shape, produced by the real
library.** Never a hand-written map standing in for a record:

```erlang
%% WRONG — tests the invention, and the invention is the bug
Ev = #{event_type => <<"roster_snapshotted">>, data => #{entries => []}},

%% RIGHT — the library's own header, so a shape change fails the test
-include_lib("reckon_gater/include/reckon_gater_types.hrl").
Ev = #event{event_type = <<"roster_snapshotted">>, data = #{entries => []}},
```

A test that invents a convenient shape tests the invention. That is precisely the
mistake the code made, so the test passes cheerfully beside the bug.

---

## 🔥🔥🔥 Demon 55: Believing That Writing It Down Prevents It

**Date exorcised:** 2026-08-07
**Where it appeared:** this file's own index
**Cost:** Demon 23, repeated in full, five months and three weeks later

### The Lie

"It's in the antipatterns index, so we won't do it again."

### What Happened

**Demon 23, exorcised 2026-02-13:** *Raw `#event{}` Records Passed to Projections.
ReckonDB emitters send records; the consumer called map functions on a tuple; read
models were permanently empty and nothing errored.*

**2026-08-07, `hecate-dronex`:** `roster_log` read events from the same library,
called `maps:find/2` on the same record, restored nothing, and reported nothing.
A different repository, a different author-session, an identical bug.

Between those dates the demon sat in `INDEX.md`, correctly described, numbered and
dated. It changed nothing, because **the index is prose too**, and this whole file
is about prose not biting.

### The Rule

A demon is only exorcised when something MECHANICAL will refuse it. Ranked by how
much they actually bite:

1. **A type or a compile error.** Dialyzer caught three unreachable clauses in the
   same session that four paragraphs of comment had not.
2. **A test that fails without the fix.** Verify it RED before believing the green.
   Ten of eleven new tests went red against the reverted reader, with the
   production `badmap` in the output. That is what makes the green mean something.
3. **A lint rule.** Elvis, a custom check, a CI grep.
4. **A line in a document.** Nearly worthless on its own, and worth writing anyway
   for the reasoning it carries. Never mistake it for a guard.

**When adding a demon here, name the mechanism that will refuse it.** A demon with
no mechanism is a demon with a scheduled return date.

---

## 🔥 Demon 56: Correct Behaviour With No Reporting

**Date exorcised:** 2026-08-07
**Where it appeared:** `island_server:kept/2`; `Dronex` board (ETS with no read model)
**Cost:** weeks of silent lineage loss, and a fleet of exhibits resetting unnoticed

### The Lie

"It degrades gracefully."

### What Happened

```erlang
kept(Island, {ok, R})     -> island:with_roster(Island, R);
kept(Island, {error, _Why}) -> Island.       %% correct, and silent
```

An island that cannot read its log **must** still start. That behaviour is right.
The reporting was absent, so the failure had no voice for weeks.

What made it invisible was not the missing log alone. It was that **the only
published evidence agreed with both outcomes**: roster depth. A restored lineage
and a fresh island filling up from seed both show a number that climbs, and there
is no depth at which one looks wrong.

The same shape appeared the same day on the site: the /dronex board held every
raid in ETS and nowhere else, so each deploy emptied it, and a board filling up
again looks exactly like a board that was never empty.

### The Rule

Two questions, both required, whenever a fallback is written:

1. **Does it say so?** Every `rescue`, catch-all clause and error branch that
   returns a default logs once, at a level someone reads. See also Demons 24, 42
   and 48 — this family has now cost four separate outages.
2. **Could an observer tell?** If the healthy state and the failed state produce
   the same published numbers, add a field that distinguishes them. "Restored from
   the log" and "seeded fresh" must not render identically.

---

## 🔥🔥🔥 Demon 57: The Silent No-Op Edit

**Date exorcised:** 2026-08-08
**Where it appeared:** three times in one session, in three repositories
**Cost:** a red-check that was never red, one broken build, and every chart on a
live page drawing nothing

### The Lie

"The edit applied. The script did not error."

### What Happened

A scripted string replacement whose anchor does not match changes nothing and
says nothing. `str.replace` returns the input unchanged, `sed` exits zero, the
file is written, the command succeeds, and the next step proceeds on an
assumption that is now false.

1. **A red-check that was never red.** Verifying that a new test caught a bug
   meant reintroducing the bug. The anchor had the wrong indentation, so nothing
   was reintroduced, the suite passed, and the test appeared to have caught a bug
   it had never seen. Only expecting a SPECIFIC failure and not getting it
   revealed this.
2. **A broken build.** An extraction cut one line further than intended and left
   a closing brace behind.
3. **Every chart blank on a live page.** An edit deleting one helper cut from a
   comment that had drifted above a DIFFERENT helper and took it too. A later
   edit meant to add a third anchored on the deleted one, matched nothing, and
   said nothing. Two functions silently ceased to exist, the bundler left them as
   undefined globals, and every chart threw a `ReferenceError` into a guard that
   logged and drew an empty box.

### The Rule

**Assert the anchor before replacing, and assert the result after.**

```python
old = "..."
assert old in s, "anchor moved"     # before
s = s.replace(old, new, 1)          # count it: 1, never replace-all
assert new in s                     # after
```

And when the point of an edit is to make something FAIL, expecting a specific
failure is the only proof it applied. "The suite passed" after reintroducing a
bug is not success; it is the edit having done nothing.

⚠ The habit worth keeping: a tool that reports success on having done nothing
needs its outcome checked separately from its exit code. True of `str.replace`,
of `sed -i`, of `grep -c` on a minified file, and of every helpful default.

## 🔥 Demon 62: A Test That Passes For a Different Layer's Reason

**Date exorcised:** 2026-09-05
**Where it appeared:** `hecate-social/hecate-om`, `hecate_om_read_model.erl`'s new
per-database TTL-sweep-config passthrough
**Cost:** would have shipped, standing green, a passthrough that did nothing —
caught before merge, not after, only because the author checked

### The Lie

"The test passed, so the fix works."

### What Happened

The task: thread `ttl_sweep_interval`/`ttl_sweep_batch` config through
`hecate_om:boot/1` into `barrel_docdb:create_db/2`'s options, so a service can
opt a database into barrel's own document-expiry sweep. The first test: write a
doc with `expires_at` a few hundred ms in the future, sleep past it, assert the
doc is gone.

It passed. It also passed with the new passthrough code **completely stubbed
out** — because `barrel_docdb`'s own read path treats an expired document as
gone unconditionally, regardless of whether any sweep config was ever set. The
sweep config only controls whether a background timer later reclaims the disk
for a document already invisible to every reader. The test asserted a fact
about `barrel_docdb`'s permanent behaviour and never once touched the code
that was actually written.

### The Signature

**A test written against the visible SYMPTOM of a feature, rather than the
boundary the new code itself owns, will pass whether or not that code runs at
all.** The tell is that the assertion (`doc == not_found`) would read as
correct even with the diff reverted — the two are close enough in outward
behaviour that only checking the actual dependency chain reveals they're
unrelated. The instinct to sleep-and-check felt like an integration test and
was actually testing a different subsystem's own always-on guarantee.

### The Rule

**Stub the exact thing you just wrote and confirm the test goes RED before
trusting it GREEN.** Concretely here: point the assertion at the boundary the
new code actually owns — `barrel_docdb:db_info/1`'s reported `config` map
carrying the sweep settings — not at a downstream effect that a completely
different, pre-existing mechanism could produce on its own. If a test still
passes with your diff commented out, it is not testing your diff; find the
real seam and assert there instead. Same mechanism as Demon 57, aimed at a
different failure to reach it: `str.replace` returning unchanged input and a
document's *visible* expiry both give a green check that proves nothing about
the code under test.

---

## The Addendum to Demon 62 (2026-09-25): Four Pins That Agree Can All Disagree With the Fleet

The scaffold template pinned its builder/runtime image pair in four places —
Containerfile, lint workflow, `.tool-versions`, and a generated guard test
that compared all four against the running VM — and that test was green,
every time, while the pair itself (hexpm/alpine) had already been retired
from the fleet standard (the macula-ci-images team pair) and 16 of 17
services had moved. mcl-bookclub was scaffolded from the stale template and
shipped the retired pair, guard tests all passing.

The shape is Demon 62's: a test that passes for a different layer's reason.
The guard proved **internal agreement** — N copies of the same wrong choice
agree with each other — while the claim it seemed to make ("the runtime is
pinned to the team standard") needed agreement with something *outside* the
repository, which no self-comparison can prove.

The mechanism that caught it was a one-liner against the estate, not against
the repo: a fleet-wide grep of every Containerfile's `FROM` line compared to
the team pair's tag and digest. A pin guard proves the pins agree; only a
comparison against the outside proves they are right.

---

## 🔥🔥 Demon 71: The Lint Gate That Linted Nothing

**Date exorcised:** 2026-09-25
**Where it appeared:** `macula-services/mcl-bookclub-gleam` — the elvis
`mcl_min` gate on the Gleam sources' generated Erlang
**Cost:** The house nesting rule was enforced nowhere while the gate was
green; the same config flip-flopped between a silent pass (dev) and a red
CI full of level-3 nesting; ~24 violations across 35 generated modules had
to be refactored only after the gate was made real.

### The Lie

"elvis runs on the generated code and the gate is green, so the code
respects the house nesting rule."

### What Happened

The Gleam twin's "source" is generated Erlang under `build/`, and
`build/` is git-ignored — it must be, it is generated. elvis_core >= 5.0
injects the output of `git check-ignore **/*` into **every rule group's
ignore list** whenever it loads its config from a file. So the artefact
glob silently resolved to **zero files** on the dev box: a green no-op.
CI's copy of the same injection misfired differently (its
`git check-ignore` returned nothing useful), so CI linted everything and
failed on the test modules' generated level-3 nesting.

Same config, same commit, two opposite verdicts — and both wrong. The
gate was environment-fragile by construction, and the commits that
claimed to "respect the house nesting rule, elvis-enforced" had never
been linted at all.

### The Fix

Three parts, all worth copying:

1. **Invoke elvis with an explicit in-memory config** — `rock({config,
   RuleGroups})` — instead of letting it read `rebar.config` (or any
   file). That code path never consults `git check-ignore`, so the gate
   is the same machine in dev and CI: lint the generated artefacts in
   place, no staging dirs, no `.gitignore` games.
2. **Make "zero files" a failure.** A canary in the lint script resolves
   each rule group first and refuses to pass on an empty file list. A
   gate that can lint nothing will eventually lint nothing; fail loudly
   instead.
3. **The generated code obeys the rule, not just the source.** Gleam
   `let assert` chains and nested `case`-in-clause bodies generate
   case-in-case Erlang (level-3 nesting). The fix mirrors the twins'
   `record_when` style: hoist every nested decision into a
   one-decision-per-function helper, so the generated Erlang stays at
   level 2.

### The Rule

> **A gate that can resolve zero files is not a gate — it is a permission
> slip.** When a linter's mechanism has its own opinions about which files
> exist (elvis's gitignore injection, globs over git-ignored build
> output), verify what it actually saw, in BOTH environments, and make an
> empty result a hard failure. Generated code inherits your source's
> nesting — enforce the structural rule on it too, and hoist decisions
> into helpers instead of chaining asserts.

---

## The Session This File Came From

2026-08-07, one working day, one author. Findings: an archipelago whose attack
graph was decided by `hd/1` over a sorted flatmap; a lineage that had never once
been restored; counters that reset on every deploy under a comment promising they
would not; an unjustified physics constant deciding every long fight; a public
exhibit holding its entire dataset in memory; four documents describing designs
the code did not implement; and a table column that asserted a ceiling the caption
beside it denied.

**Every one was found by measuring. Every one had been hidden by prose.**

The good moments were all instruments: a probe against a live node, a query against
the running board, a palette validator, a dry-run of the updater, an anonymous
registry pull. The bad ones were all assertions from memory, written confidently.

That is the whole of this file. Measure the thing. Then write down only what the
measurement said.
