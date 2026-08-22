# Exam Mistake Memory — Language Edition

A system prompt that gives an AI tutor permanent memory of your exam mistakes — and, for language exams, makes it
get the **diagnosis** right before it picks the fix.

Built for [Walrus Session 7 — Prompt Evolution](https://www.deepsurge.xyz/hackathons/f313beb4-290d-46d9-ac73-3e216fdba8d1).
An evolution of [Exam Mistake Memory](https://github.com/EAZITECH1/exam-mistake-memory) by
[@EAZITECH1](https://github.com/EAZITECH1), a Session 5 winner. Runs on
[Walrus Memory](https://walrus.xyz/products/walrus-memory/) (MemWal MCP) in Claude Code, Codex, Cursor, or any
MCP client.

**→ The prompt: [`CLAUDE.md`](CLAUDE.md)**

---

## Why I picked this prompt

I'm retaking the **Goethe-Zertifikat B1** at Goethe-Institut Jakarta. B1 is modular: four modules — Lesen (65
min), Hören (40 min), Schreiben (60 min), Sprechen (~15 min, paired) — each scored out of 100, each passed
independently at 60. There is no compensation between them: a 90 in Lesen does not rescue a 55 in Schreiben. Fail
one module and you retake only that one; the rest stay banked.

So I don't need a study assistant that knows German. I need one that knows **which of my mistakes are still
costing me points**, and stops spending my last week on the ones that aren't.

The original prompt already does the hard architectural work: it writes structured mistake records to Walrus,
dedupes before writing, ranks weaknesses, and opens every session with a briefing instead of asking me what I
want to study. That design is right, and most of it survives here unchanged — the health-check flow, the batch
writes, the error handling, the refusal to store credentials. What I changed is what happens **inside a mistake
record**.

## The problem: a language mistake is not a missing fact

The original stores each error like this:

```
topic, question, my_error, correct, severity, misses
severity: high (conceptual gap) / medium (mix-up) / low (careless)
```

That works beautifully for the exams it was designed against — pharmacology, USMLE, CFA — where a mistake really
is a fact you didn't have, and the fix really is a flashcard.

It breaks on a language exam, for one root reason: **`severity` is a diagnosis wearing a priority's clothes.**
"Conceptual / mix-up / careless" describes *why* the error happened. The prompt then multiplies it by `misses` to
decide *what to study next*. Fusing those two jobs into one field causes four specific failures:

**1. Every cause gets the same remedy.** In German, five completely different problems produce identical-looking
errors, and each needs a different fix. `mit mein Bruder` might be a rule I never learned, an Indonesian
structure I transferred, a slip under time pressure, or a word I was reaching for while my attention was
elsewhere. The original hands all four the same flashcard.

**2. Careless is ranked last — but careless is what fails you.** Schreiben is three tasks in 60 minutes. Errors
you'd never make untimed are exactly the ones that cost the module. `severity: low` buries them.

**3. Mastery is declared on the wrong evidence.** The original masters a topic after three correct answers across
two sessions. Every one of those can be multiple-choice. I can recognise the dative three times running and still
write it wrong in a Forumsbeitrag — recognition and production are different skills, and only one of them is on
the exam.

**4. The database grows forever.** Roughly half my errors are predictable from Bahasa Indonesia: no grammatical
gender, no case marking, no verb inflection for tense, SVO word order throughout where German is V2 in main
clauses and verb-final in subordinate ones. Stored as independent surface errors, that's an unbounded list. They
are ~8 root contrasts wearing dozens of costumes.

## What changed

| # | Change | Why |
|---|---|---|
| 1 | `severity` split into **`cause`** (stored) and **priority** (computed at read time, never stored) | Diagnosis and ranking were the same field, so the remedy could never depend on the cause |
| 2 | **Remedy routing** — five causes, five different responses | A slip needs a timed re-run, not an explanation. A missed Leitpunkt needs no grammar at all |
| 3 | Flashcard → **minimal pair**: `wrong` and `right` stored verbatim, differing by only the error | You repair `mit mein Bruder` → `mit meinem Bruder`. You do not repair "the dative case" |
| 4 | **Production-gated mastery**: two `produced-free` correct answers, two sessions, three days apart | Stops the prompt certifying mastery on evidence the exam never asks for |
| 5 | **`l1` namespace** — root Indonesian↔German contrasts written once, surface errors point at them via `l1_ref` | Turns a linearly growing mistake log into a converging one |
| 6 | **Stable `code` as dedup key**, module namespaces, banked modules never ranked | Semantic dedup was approximate, so `misses` counts drifted; and a banked module shouldn't outrank the one you're actually retaking |
| 7 | **`ASK` command** + a `curriculum` namespace | The original had no way to *start* a drill from memory in one word, and no record of what I'd been taught — so it could test me on things I'd never studied |

### 1–2. Cause decides the remedy

`cause` is one of five, and it routes the response:

| cause | what it means | what the tutor does |
|---|---|---|
| `rule-gap` | I don't know the rule, or know it wrong | State the rule in one sentence, three contrastive examples, then I produce two of my own |
| `l1-transfer` | I applied an Indonesian structure to German | Show both structures side by side, name the difference, then drill *from Indonesian* so the interference fires and gets caught in the act |
| `slip` | I know it and produced it wrong anyway | **Teach nothing.** Re-run under the clock and see if it survives time pressure |
| `retrieval` | I knew what I meant, didn't have the word or phrase | Give the chunk, require it in free production this session and next |
| `strategy` | Nothing to do with German — missed a Leitpunkt, wrong word count, wrong register, ran out of time | **No grammar at all.** Re-run under the real limit with the constraint stated up front |

The worked example that made this concrete for me: I write a fluent Forumsbeitrag and skip one of the
Leitpunkte. Goethe marks Schreiben on Erfüllung, Kohärenz, Wortschatz and Strukturen — so I lose Erfüllung points
no matter how good the German was. That is `cause: strategy`. Correcting my grammar in response would be the
wrong intervention entirely, and the original prompt has no way to express that the error wasn't linguistic.

One safeguard: if a code reaches **4 misses while filed as `slip`**, it is not carelessness. The prompt
reclassifies it as `rule-gap` and says so. "Careless" means it happened once.

### 4. Mastery has to be earned in free production

Every correct answer is now recorded at the tier of evidence it actually provides:

- `recognised` — right in multiple-choice or gap-fill, where the target was obvious
- `produced-prompted` — right when told which structure to use
- `produced-free` — right in unprompted writing or speaking, where nothing signalled the rule

Only `produced-free` counts toward mastery. This has a consequence I didn't anticipate when I wrote it: **a
session with no free-production task cannot master anything**, however many questions I answer correctly. So the
prompt now requires every session to end with one real task under the clock. Changing the mastery rule changed
the shape of the study session.

### 5. The L1 layer

The `l1` namespace holds one entry per structural contrast — `l1.no-gender`, `l1.no-case`,
`l1.no-verb-inflection`, `l1.verb-position`, `l1.no-plural-marking`. When a mistake is caused by one of them, the
record sets `cause: l1-transfer` and `l1_ref`, and **does not restate the contrast**. It points.

When an `l1` entry collects five or more dependents, the briefing surfaces it: that root is the single
highest-value thing I could fix, and I'd never see it from a flat list of surface errors.

This is also the part that matters most for Walrus Memory specifically. The Session 5 brief was that memory only
helps *if the agent knows what's worth remembering* — a prompt that writes a new blob for every observed error
degrades its own recall quality as it grows.

### 7. `ASK` — the daily loop

The original prompt is conversational: you tell it what you want to study. That's fine on a planned study
evening and useless on a Tuesday when you have ten minutes. There was also nothing recording *what I'd been
taught*, so it could happily test me on the Konjunktiv II before I'd studied it — which writes a mistake record
for a topic I never learned, and that fake weakness then outranks a real one.

So: a `curriculum` namespace holds what I've actually been taught, and one keyword drives everything.

```
ASK
```

1. Recalls `curriculum` plus every open weakness
2. Picks 1–3 questions, highest-priority weaknesses first, interleaved
3. **Picks the question format from the weakness** — weak on case → `luecke`; weak on vocabulary → `vokabel`;
   weak on word order → `umformen`; strong in drills but never tested in the wild → `absatz`
4. Scores each answer 0 / 1 / 2, reports `Punkte: N/M` and a minimal pair per question
5. Writes the results straight back to Walrus in one bulk call
6. The next `ASK` starts from what it just wrote

Variants: `ASK 1` for a single question, `ASK vokabel` or `ASK schreiben` to narrow the pool without changing the
ranking inside it.

The rule that keeps this honest is #4: a format can never certify a tier above its own. Fill-in-the-gap proves
`recognised` and nothing more, so a week of clean `luecke` answers escalates you to `absatz` instead of quietly
declaring mastery.

## The memory record

```
code:       <module>.<domain>.<specific>    stable dedup key
module:     lesen | hoeren | schreiben | sprechen | wortschatz
topic_ref:  curriculum.<topic-slug>         the curriculum topic this code belongs to
cause:      rule-gap | l1-transfer | slip | retrieval | strategy
wrong:      exactly what I produced, verbatim, error intact
right:      the minimal repair — same sentence, only the error fixed
trigger:    the cue that should fire the rule next time
tier:       untested | recognised | produced-prompted | produced-free
misses:     running count
last_miss:  YYYY-MM-DD
last_ok:    YYYY-MM-DD                      most recent correct answer at any tier
free_hits:  [YYYY-MM-DD, ...]               produced-free correct answers, append-only
l1_ref:     code of the l1 entry, when cause = l1-transfer
```

Priority is **not** a field. It's computed at session start, ranked only within modules still to be passed
(`wortschatz` and `l1` are always rankable — they can never be banked):

```
priority = misses ÷ (days since last_miss + 1) × cause_weight
cause_weight   1.5  rule-gap, l1-transfer, strategy
               1.0  retrieval
               0.5  slip
```

## How to use it

**Install MemWal.** In Claude Code:

```
/plugin marketplace add MystenLabs/MemWal
/plugin install memwal@memwal-plugins
```

Restart, then `memwal_login` — it opens a browser wallet sign-in, valid for 5 minutes. (If you're using the
hosted claude.ai connector instead, run `/mcp` and authorize **claude.ai Memwal**.)

**Drop in the prompt.** Keep [`CLAUDE.md`](CLAUDE.md) in the directory you study from, or paste everything below
its `---` into any MCP client's system prompt.

**Fill in the config block first** — exam date, and each module's status (passed with score / to retake). Change
6 depends on it: the prompt will not rank a banked module into your top 5.

**Tell it what you've learned.** The `curriculum` namespace is the pool `ASK` draws from, and it starts empty:

> I've finished Modalverben im Präteritum and the Wechselpräpositionen.

**Then just type `ASK`.** Every day. That's the loop — it asks, you answer, it scores, it writes to Walrus, and
tomorrow's questions come from today's score. Say `done` to close the session and get a written summary of every
blob stored.

The shipped config is Goethe-Zertifikat B1 with Bahasa Indonesia as L1. The structure is not German-specific —
swap the module list and the `l1` contrasts for telc, TestDaF, IELTS, JLPT, DELE or TOPIK, and for your own
first language.

## What's in this repo

| File | |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | The prompt. This is the deliverable — and it's live: keep it here and Claude Code loads it |
| [`README.md`](README.md) | This file — what changed and why |
| [`REFERENCES.md`](REFERENCES.md) | Research notes behind the rewrite — the SLA methods each change draws on |

## Validation

Before using it, I ran the `ASK` loop adversarially — three simulated sessions against a stubbed memory store,
with the "student" deliberately failing a Wechselpräposition on case and blanking a vocabulary item in session 1,
to check whether session 2 actually reprioritised and whether session 3 refused to declare mastery off drill
answers.

Selection and ranking passed. The bookkeeping did not, and the failures were structural:

- **Mastery was uncomputable.** The rule needs two free-production hits three days apart; the schema stored only
  `tier`, `misses`, `last_miss`. A record with one hit was indistinguishable from one with five. → added
  `free_hits` and `last_ok`; mastery is now computed from `free_hits`, never inferred from `tier`.
- **`tier` had no value for "failed, never yet correct"**, so new records had to open at `recognised` — asserting
  evidence the student hadn't given. That made a correct re-test a literal no-op: the record was byte-identical
  before and after. → added `untested`, and restructured the format table so the weakness picks the row and the
  **tier picks the column**, giving every code a defined path from `luecke` to `absatz` instead of stalling.
- **`wortschatz` would have been silently unrankable.** The ranking said "only modules I still have to pass", and
  vocabulary is not a module you can pass — so half the test payload would have vanished on day one.
- **The shipped example code was wrong.** `kasus.dativ-nach-praeposition` merges `mit` + Dativ (a memorised
  fixed-case list) with `auf` + Wo? → Dativ (a semantic judgement). Opposite drills, one code. → added a
  granularity rule: `<specific>` names *the decision I had to make*, not the surface form.
- **`ASK` diagnosed and never treated.** "Report the cause, nothing else" contradicted "cause decides the remedy,
  always" — across three sessions the student got zero explanations. → `ASK` now applies remedy routing to the
  single highest-priority `0`. One, not three: diagnosis without treatment isn't a session, treatment on
  everything is a lecture.

Also fixed: `Punkte: N/M` was ambiguous (2/3 vs 2/6, both compliant, wildly different signal — M is now defined
as max points), and `misses × recency, weighted by cause` was not an operator and ranked two ways on the same
data.

### Then the live run found one more, and it was the biggest

Both the original prompt and my rewrite told the agent to *"update the record — increment `misses`, refresh the
date."* Once I connected to MemWal and wrote the first ten blobs, the API turned out to be
`memwal_remember_bulk(facts: string[])` — free text, append-only. **There is no update primitive and no delete.**
Walrus blobs are immutable, so a read-modify-write is not something the storage can do.

So the dedup rule at the heart of both versions was written against an API that doesn't exist. Every "update"
silently became a second blob, with the stale copy left in the index competing for recall relevance forever —
and since there's no way to enumerate a namespace, you can't even see it happening.

The fix is the standard append-only pattern, now in the prompt: records carry `updated` and `supersedes`, every
write is a **complete state snapshot rather than a delta** ("misses=3" is storable, "increment misses" is not),
recall groups by `code` and keeps only the newest `updated`, and writes are batched to one per code per session
to limit how fast the index fills with superseded copies.

This one applies to the original prompt too, so it's going upstream as a GitHub issue.

## Evidence of use

Two `ASK` sessions were run end to end on 2026-08-20.

**14 records across 9 distinct codes.** Four were seeded before the first session — three `curriculum`
topics and one `exam-intel` note that Lesen was already banked — so **10 were written by the agent
itself**, none of them requested. Nothing is ever edited or removed: a correction is a new complete
record layered on top of the old one, which is why 9 codes account for 14 records.

| Namespace | Records | Distinct codes |
|---|---:|---:|
| `curriculum` | 5 | 3 |
| `wortschatz` | 4 | 2 |
| `schreiben` | 3 | 2 |
| `exam-intel` | 2 | 2 |
| **Total** | **14** | **9** |

Session 1 wrote 6 records, session 2 wrote 4. The second session wrote fewer because most of what
happened in it was not new — three codes already existed and moved up a level, which is an append,
not a discovery.

The codes that accumulated:

```
schreiben.praeposition.wechsel-wo-dativ                    Level 1 → 2
schreiben.wortstellung.hauptsatz-nach-vorfeld-nebensatz    Level 1 (unchanged)
wortschatz.reflexiv.sich-konzentrieren                     Level 1 → 2
wortschatz.kollokation.benutzen-als-funktion               Level 1 → 2
curriculum.wechselpraepositionen                           tested
curriculum.nebensatz-wortstellung                          tested + 1 free hit
curriculum.perfekt-hilfsverb                               never asked, never decided
exam-intel                                                 Lesen banked, exam date
```

Two of those lines are the point of the whole design.
`schreiben.wortstellung.hauptsatz-nach-vorfeld-nebensatz` did **not** advance, because the specific
configuration that failed the first time never reappeared — three sessions of drills would have logged
that as "improving". And `curriculum.perfekt-hilfsverb` stayed at `tested: never` because no
Bewegungsverb appeared in the writing, so haben-vs-sein was never actually decided. Neither outcome is
flattering, and both are correct.

## Credits

Original prompt: [Exam Mistake Memory](https://github.com/EAZITECH1/exam-mistake-memory) by
[@EAZITECH1](https://github.com/EAZITECH1), Walrus Session 5. The memory lifecycle in this version — health
checks, restore fallback, batch writes, session-start briefing, the refusal to persist credentials — is theirs,
and it's the reason this rewrite could focus entirely on the error model.

Walrus Memory / MemWal: [MystenLabs/MemWal](https://github.com/MystenLabs/MemWal)
