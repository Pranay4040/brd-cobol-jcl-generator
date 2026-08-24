# Build Plan

Ordered implementation steps. Read `PROJECT.md` first — it is the source of
truth for architecture and constraints.

**Work one step at a time.** Complete the acceptance criteria before moving on.
Do not skip ahead, and do not build several steps at once.

**Build order is deliberately backwards from the pipeline.** Validators come
before generators, because you cannot tell whether a generator works until
something can grade its output. Building the generator first means tuning blind.

---

## Phase 0 — Foundation

### Step 0.1 — Repository skeleton

Create the directory structure and `.gitignore` from `PROJECT.md` §5.

**Order matters:** write `.gitignore` first, then create directories.

- [ ] `.gitignore` exists and lists `config/shop.yml`, `copybooks/`, `input/`, `output/`, `*.cpy`
- [ ] All directories from §5 created, with `.gitkeep` in the empty ones
- [ ] `git status` shows no ignored paths as untracked
- [ ] `README.md` with a one-paragraph description

### Step 0.2 — Synthetic test data

This is the fuel for every subsequent step. Do not rush it.

Create in `samples/`:

1. **A copybook, `SNCUST.cpy`** — must be *realistically awkward*, not a clean
   toy. Include at minimum: 20+ fields, a nested group, one `OCCURS`, one
   `REDEFINES`, one `COMP-3`, one `SIGN TRAILING SEPARATE`. If your synthetic
   copybook is simpler than the real ones, the generator will look great locally
   and collapse on real data.

2. **A second copybook, `SNRPT.cpy`** — a report line layout.

3. **A BRD, `sample-brd-01.md`** — 2–3 pages, prose, written the way a real BRD
   is written. Deliberately include: one ambiguous requirement, one unstated
   edge case, and one rule expressed only in a table.

4. **`config/shop.sample.yml`** — synthetic values matching `PROJECT.md` §6.

- [ ] Copybook parses as valid COBOL data division syntax
- [ ] Copybook contains OCCURS, REDEFINES, and COMP-3
- [ ] BRD contains at least one deliberate ambiguity (note it in a comment at the bottom of the file)
- [ ] `config/shop.sample.yml` covers every key referenced elsewhere

### Step 0.3 — A hand-written golden JCL job

Write, by hand, the JCL you *want* the generator to produce for
`sample-brd-01.md`. Save as `samples/expected/SNCUST01.jcl`.

This is your target. Everything in Phase 1 aims at reproducing it.

- [ ] Job card, at least one EXEC step, DD statements for input/output/SYSOUT
- [ ] Uses only values that exist in `config/shop.sample.yml`
- [ ] You would submit this job as-is

---

## Phase 1 — JCL path (the shippable milestone)

### Step 1.1 — JCL validator

`scripts/validate_jcl.py`. Pure Python, no dependencies beyond stdlib if
possible. Takes a `.jcl` file path, returns exit 0 or a list of errors with line
numbers.

Checks required (see `PROJECT.md` §9):
- Job card present, well-formed, name ≤ 8 chars
- Every `EXEC` followed by its required DDs
- No `DD` referenced in a step that isn't defined
- Dataset names match the `naming` rules in config
- Content within columns 1–71; continuation cards correct
- No unresolved `{{` placeholders

**Test it against real JCL you already have** — it should pass known-good jobs
and catch deliberately broken ones.

- [ ] Passes `samples/expected/SNCUST01.jcl`
- [ ] Catches all of: missing job card, undefined DD reference, line >71 chars, unresolved placeholder
- [ ] Errors include line numbers and are readable by a human
- [ ] Unit tests in `tests/test_validate_jcl.py`, all passing

### Step 1.2 — Spec schema and validator

`schema/spec.schema.json` per `PROJECT.md` §7, plus `scripts/validate_spec.py`.

Beyond JSON Schema, add semantic checks:
- Every referenced copybook file exists
- Every validation rule has a non-empty `on_failure`
- Every business rule has non-empty `source_text`
- Warn if `open_questions` is empty

- [ ] Schema validates a hand-written spec for `sample-brd-01.md`
- [ ] Rejects a spec with a missing `source_text`
- [ ] Rejects a spec referencing a non-existent copybook
- [ ] Unit tests passing

### Step 1.3 — Hand-write the spec

Manually write `samples/expected/spec-01.json` for `sample-brd-01.md`.

Do this by hand *before* building the extraction agent. It forces you to
discover what the schema actually needs, and gives you something to compare the
AI's output against.

- [ ] Passes `validate_spec.py`
- [ ] Contains every rule present in the sample BRD
- [ ] `open_questions` names the deliberate ambiguity you planted

### Step 1.4 — JCL template + renderer

Extract `templates/batch-job.jcl.j2` from your golden JCL by replacing the
varying parts with placeholders. Then `scripts/render_jcl.py` fills it from
spec + config.

**No LLM in this step.** Template fill only.

- [ ] `render_jcl.py samples/expected/spec-01.json` reproduces `samples/expected/SNCUST01.jcl` (diff is empty or trivially whitespace)
- [ ] Output passes `validate_jcl.py`
- [ ] Missing config value causes a clear error, never a blank or guessed substitution

**Checkpoint:** you now have a deterministic spec → JCL pipeline with
validation. Everything from here adds AI on top of a working foundation.

### Step 1.5 — Shop standards context file

`.github/copilot-instructions.md`. Plain English. Naming conventions, dataset
rules, column discipline, house style, what never to do.

Keep it under ~200 lines. Long instruction files get diluted.

- [ ] Covers naming, dataset conventions, standard PROCs, column rules
- [ ] Contains an explicit "never invent identifiers" rule
- [ ] Contains no real company values (those live in config)

### Step 1.6 — BRD analyst agent

`.github/agents/brd-analyst.agent.md`.

```yaml
---
name: brd-analyst
description: Reads a BRD and produces a structured spec.json. Never writes code.
tools: [read, search]
---
```

Read-only tools deliberately — this agent physically cannot write COBOL or JCL.

The prompt must instruct it to: emit only valid JSON against the schema, quote
BRD text verbatim into `source_text`, list ambiguities in `open_questions`, and
never invent copybook names.

- [ ] Agent appears in the VS Code agent picker
- [ ] Run on `sample-brd-01.md` produces JSON passing `validate_spec.py`
- [ ] Output captures the planted ambiguity in `open_questions`
- [ ] Compared against your hand-written spec: substantially matching rules

### Step 1.7 — Wire the JCL path end to end

Chain: BRD → agent → spec → human approval → render → validate → output.

The human approval step must be an actual stop, not a printed reminder.

- [ ] Single command runs BRD to validated JCL, pausing for approval
- [ ] Pipeline halts if spec validation fails
- [ ] Pipeline halts if JCL validation fails after 3 attempts
- [ ] Generated JCL for the sample BRD is usable

**MILESTONE — this is a genuinely useful tool.** Demo it before continuing.
Feedback here will change what you build next.

---

## Phase 2 — COBOL path

Do not start until Phase 1 metrics hold on real BRDs at the office.

### Step 2.1 — Copybook resolver

`scripts/resolve_copybook.py`. Parses a `.cpy` file into structured field data:
name, level, PIC, usage, occurs, redefines, computed offsets and lengths.

This is the single most important script in the repo. It is what makes
hallucinated field names structurally impossible.

- [ ] Correctly parses `samples/copybooks/SNCUST.cpy` including OCCURS, REDEFINES, COMP-3
- [ ] Computes correct offsets and total record length
- [ ] Raises a clear error on unknown copybook, never returns a guess
- [ ] Unit tests covering each construct

### Step 2.2 — COBOL skeleton generator

`scripts/render_cobol_skeleton.py` + `templates/cobol-skeleton.cbl.j2`.

Produces Zone 1 only (`PROJECT.md` §8): IDENTIFICATION, ENVIRONMENT, DATA
divisions, FDs, SELECTs, WORKING-STORAGE from resolved copybooks. Emits a marked
empty PROCEDURE DIVISION region.

**No LLM.**

- [ ] Output compiles (or passes syntax check) with a stub PROCEDURE DIVISION
- [ ] Column discipline correct — Area A / Area B, nothing past column 72
- [ ] Every field traceable to the copybook; nothing invented
- [ ] AI-generated region clearly marked per §8

### Step 2.3 — Compile gate

`scripts/compile_check.py`. Wraps GnuCOBOL locally, or the compile PROC via
Zowe CLI on the office laptop. Returns structured errors with line numbers.

If neither is available: **stop here.** Do not build Step 2.4 without a way to
verify its output. Ship the JCL tool instead.

- [ ] Compiles a known-good program successfully
- [ ] Returns parseable errors for a known-broken program
- [ ] Works from a config flag, no code change between machines

### Step 2.4 — COBOL logic agent

`.github/agents/cobol-writer.agent.md`. Writes PROCEDURE DIVISION only, given:
the skeleton, the approved spec, and resolved copybook fields.

Prompt must require: use only fields present in the resolved copybook list,
implement every rule by its spec ID, add a comment referencing the rule ID on
each rule's implementation, respect column discipline.

- [ ] Generated logic compiles within 3 attempts on the sample
- [ ] Every validation and business rule from the spec is implemented
- [ ] Every rule implementation carries its spec ID in a comment
- [ ] Zero references to fields not in the copybook

### Step 2.5 — Retry loop

Wire compile errors back to the agent. Hard cap 3. On exhaustion: stop, write
partial output to `output/`, report clearly what failed.

- [ ] Recovers from a deliberately introduced compile error
- [ ] Stops cleanly after 3 failures without looping
- [ ] Failure report names the unresolved errors

---

## Phase 3 — Measurement

### Step 3.1 — Golden set

On the office laptop, assemble 10 real BRD + COBOL + JCL triplets in
`input/golden/` (gitignored).

### Step 3.2 — Evaluation harness

`scripts/evaluate.py` runs the pipeline across the golden set and reports the
metrics from `PROJECT.md` §10.

- [ ] Reports all five metrics
- [ ] Flags any hallucinated identifier as a hard failure
- [ ] Output is a table you can put in front of your team

### Step 3.3 — Tune

Iterate on instructions, agent prompts, and templates against the metrics.
Change one thing at a time and re-run. Do not tune prompts and templates
simultaneously — you will not know which change helped.

---

## Working notes for Claude Code

- Read `PROJECT.md` §11 guardrails before each step and respect them.
- Never add API key handling. Model access is Copilot-only.
- Prefer stdlib. Every dependency added is one more thing to justify on a
  corporate machine.
- Every script: clear `--help`, non-zero exit on failure, errors with line
  numbers.
- Write the test before the implementation where practical.
- If a step's acceptance criteria cannot be met, stop and say so. Do not
  proceed with a partial implementation and note it as a TODO.
- Ask before adding scope. The scope in `PROJECT.md` §3 is deliberately narrow.
