# BRD → COBOL + JCL Generator

An AI-assisted pipeline that reads a Business Requirements Document and produces
reviewable COBOL programs and JCL jobs for a z/OS batch environment.

**Status:** greenfield. Nothing built yet.

---

## 1. Core design principle

**The LLM does translation. Deterministic code does accuracy.**

The AI is used at exactly two stages: turning prose into a structured spec, and
writing PROCEDURE DIVISION logic. Everywhere else — copybook resolution, JCL
assembly, skeleton generation, validation — is plain code with no model in the
loop.

This is not a limitation to work around. It is the design. Any change that moves
work *into* the LLM and *out of* deterministic code is a regression.

### The non-negotiable rule

> The generator may never invent a field name, a PIC clause, a dataset name, a
> DD name, or a PROC name. Every one of those comes from a copybook lookup, the
> config file, or the approved spec. If a value cannot be resolved, the pipeline
> **fails loudly** rather than guessing.

Hallucinated identifiers are the single failure mode that destroys trust in this
tool. Guard against it structurally, not with prompt wording.

---

## 2. Pipeline

```
  [1] Intake            BRD file lands in /input
        ↓
  [2] Extract spec      LLM: prose → spec.json          ← AI
        ↓
  [3] Gap check         script: required fields present? questions raised?
        ↓
  [4] HUMAN APPROVAL    ★ THE GATE ★  human edits + signs off spec.json
        ↓
  [5] Resolve           script: copybook lookup, config merge
        ↓
  [6] Generate          ├─ JCL:            template fill
                        ├─ COBOL divisions: skeleton from spec + copybooks
                        └─ PROCEDURE DIV:   LLM                    ← AI
        ↓
  [7] Validate          JCL syntax check / COBOL compile
                        errors → back to [6], max 3 attempts, then stop
        ↓
  [8] Deliver           write to /output, present as diff for review
```

### Stage ownership

| Stage | Owner | Notes |
|-------|-------|-------|
| 1. Intake | script | |
| 2. Extract spec | **LLM** | only reads; never writes code |
| 3. Gap check | script | |
| 4. Approve | **human** | pipeline halts here, always |
| 5. Resolve copybooks | script | authoritative source of all field data |
| 6a. JCL | script + template | zero LLM involvement |
| 6b. COBOL skeleton | script | IDENTIFICATION / ENVIRONMENT / DATA divisions |
| 6c. COBOL logic | **LLM** | PROCEDURE DIVISION only |
| 7. Validate | script | |
| 8. Review | **human** | |

Stage 4 is a hard stop. The pipeline does not proceed past it automatically
under any circumstance, including when the spec looks complete.

---

## 3. Scope

### In scope (v1)

One program archetype only:

- Sequential input file → validation → sequential output file + report
- Fixed-length records
- Copybook-driven record layouts
- Standard batch JCL: job card, one or two EXEC steps, DD statements, SYSOUT

### Explicitly out of scope for v1

Do not build for, design for, or accommodate these yet:

- CICS
- DB2 / embedded SQL
- VSAM
- IMS
- Multi-program call chains
- Restart/checkpoint logic
- Sort steps beyond a single trivial SORT

Adding these later is a deliberate later decision. Building v1 flexible enough
to "maybe support DB2 someday" will produce a worse v1.

---

## 4. Environment constraints

These shape the architecture. Read them before proposing anything.

- **Editor:** VS Code
- **Model access:** GitHub Copilot only, via custom agents (`.agent.md` files)
- **No API keys anywhere.** No `ANTHROPIC_API_KEY`, no `OPENAI_API_KEY`, no
  `.env` with model credentials. The model comes from the user's Copilot
  subscription through VS Code. Any code that reaches for an API key is wrong.
- **Two machines.** Scaffolding is built on a personal laptop with synthetic
  data. Real copybooks, real BRDs, and real shop standards exist only on the
  office laptop. Code moves personal → office. Company data never moves the
  other way.
- **Compile access:** to be confirmed. Assume none initially; design the JCL
  path to be fully validatable without a mainframe.

---

## 5. Repository layout

```
.
├── .github/
│   ├── copilot-instructions.md      # shop standards, always-on context
│   ├── agents/
│   │   ├── brd-analyst.agent.md     # stage 2: BRD → spec.json
│   │   ├── cobol-writer.agent.md    # stage 6c: PROCEDURE DIVISION
│   │   └── jcl-writer.agent.md      # supervises stage 6a
│   └── skills/
│       └── shop-standards/SKILL.md  # naming rules, conventions
│
├── config/
│   ├── shop.yml                     # GITIGNORED — real values
│   └── shop.sample.yml              # committed — synthetic values
│
├── copybooks/                       # GITIGNORED — real copybooks
├── input/                           # GITIGNORED — real BRDs
├── output/                          # GITIGNORED — generated artifacts
│
├── samples/                         # committed — synthetic test data
│   ├── copybooks/
│   ├── brds/
│   └── expected/                    # known-good outputs for regression
│
├── templates/
│   ├── batch-job.jcl.j2
│   └── cobol-skeleton.cbl.j2
│
├── schema/
│   └── spec.schema.json
│
├── scripts/
│   ├── validate_jcl.py
│   ├── validate_spec.py
│   ├── resolve_copybook.py
│   ├── render_jcl.py
│   └── render_cobol_skeleton.py
│
└── tests/
```

### .gitignore — write this first

```gitignore
config/shop.yml
copybooks/
input/
output/
*.cpy
*.CPY
```

Create the `.gitignore` **before** creating the directories it protects. It is
much easier to never commit company data than to scrub it from history.

---

## 6. Configuration

Nothing shop-specific is ever hardcoded. All of it lives in `config/shop.yml`:

```yaml
job_card:
  account: "{{ACCOUNT}}"
  class: A
  msgclass: X
  notify: "&SYSUID"

datasets:
  hlq: SYNTH.TEST
  work_unit: SYSDA

procs:
  compile: SYNTHCOB
  link: SYNTHLNK

naming:
  program_prefix: SN
  max_program_name_length: 8

standards:
  indent_area_a: 8
  indent_area_b: 12
  max_line_length: 72
  sequence_numbers: false
```

Personal laptop has a synthetic `shop.yml`. Office laptop has the real one at
the same path. **Switching machines requires zero code changes** — only a
different config file being present.

---

## 7. The spec (stage 2 output)

This JSON file is the contract between the AI half and the deterministic half.
Get its schema right and everything downstream is straightforward.

```json
{
  "program": {
    "name": "SNCUST01",
    "description": "Validate customer master records and produce reject report",
    "frequency": "daily"
  },
  "inputs": [
    {
      "logical_name": "CUSTIN",
      "copybook": "CUSTREC",
      "dataset_suffix": "CUST.MASTER.IN",
      "organization": "sequential",
      "record_length": 200
    }
  ],
  "outputs": [
    {
      "logical_name": "CUSTOUT",
      "copybook": "CUSTREC",
      "dataset_suffix": "CUST.MASTER.OUT",
      "organization": "sequential",
      "disposition": "NEW"
    },
    {
      "logical_name": "REJRPT",
      "type": "report",
      "dataset_suffix": "CUST.REJECT.RPT"
    }
  ],
  "validation_rules": [
    {
      "id": "V001",
      "field": "CUST-ID",
      "rule": "must be numeric and non-zero",
      "on_failure": "reject record, write to REJRPT with reason 'INVALID ID'"
    }
  ],
  "business_rules": [
    {
      "id": "B001",
      "description": "Records with status code 'C' are excluded from output",
      "source_text": "<verbatim quote from the BRD>"
    }
  ],
  "error_handling": {
    "on_input_file_missing": "abend with RC=12",
    "on_zero_records": "write warning to SYSOUT, RC=4"
  },
  "totals_required": ["records_read", "records_written", "records_rejected"],
  "open_questions": [
    "BRD does not specify behaviour when CUST-ID is present but duplicated"
  ]
}
```

### Rules for the spec

- **`source_text` is mandatory on every business rule.** A verbatim quote from
  the BRD. This is the traceability that makes human review fast — the reviewer
  checks the quote against the rule, not against the whole document.
- **`open_questions` must never be silently empty.** If the LLM found no
  ambiguity in a real BRD, it did not look hard enough. An empty array on a
  non-trivial BRD is a signal to re-run stage 2.
- The spec references copybooks **by name only**. It never contains field lists,
  PIC clauses, or record layouts — those are resolved at stage 5 from the actual
  copybook files.

---

## 8. COBOL generation: two zones

Treat these differently. They have very different reliability profiles.

**Zone 1 — deterministic (high confidence)**

IDENTIFICATION DIVISION, ENVIRONMENT DIVISION, FILE SECTION, WORKING-STORAGE
derived from copybooks, FD entries, SELECT clauses.

Generated from template + spec + resolved copybooks. No LLM. Review is a glance.

**Zone 2 — LLM-generated (needs real review)**

PROCEDURE DIVISION only. The read loop, validation paragraphs, business rule
application, error handling, totals, and report writing.

Expect roughly 70–80% correct on first pass. The failures are *quiet* — code
that compiles cleanly and produces a wrong number. Review must be done by
someone who understands the business rules, not just COBOL syntax.

The generator writes Zone 2 into a clearly marked region so reviewers know where
to spend their attention:

```cobol
      *================================================================
      * AI-GENERATED LOGIC — REVIEW AGAINST SPEC RULES V001-V004, B001
      *================================================================
```

---

## 9. Validation

**JCL** — `scripts/validate_jcl.py`. Checks:
- Job card present and well-formed
- Every EXEC has required DDs
- No DD name referenced that isn't defined
- Dataset names conform to `naming` rules in config
- Columns 1–71 respected, continuation cards correct
- No unresolved template placeholders (`{{`) remain

**Spec** — `scripts/validate_spec.py`. JSON Schema validation plus semantic
checks: every referenced copybook exists, every validation rule has an
`on_failure`, every business rule has `source_text`.

**COBOL** — compile gate. GnuCOBOL `cobc -fsyntax-only` if available locally,
real compile PROC on the office laptop if an LPAR is reachable. If neither is
available, COBOL generation stays experimental and JCL is the shippable
deliverable.

### Retry loop

Validation failures feed back to generation with the error text attached.
**Hard cap of 3 attempts.** After the third failure the pipeline stops and
reports what it could not fix. It does not loop indefinitely, and it does not
silently emit invalid output.

---

## 10. Definition of done (v1)

Measured against a golden set of 10 real BRD + COBOL + JCL triplets:

| Metric | Target |
|--------|--------|
| JCL passes validator first attempt | ≥ 90% |
| JCL usable with only trivial edits | ≥ 80% |
| COBOL compiles within 3 attempts | ≥ 70% |
| Reviewer edits to PROCEDURE DIVISION | < 30% of lines |
| Hallucinated identifiers | **0 — this is a hard gate** |

The last row is pass/fail, not a percentage to optimise. A single invented
copybook field in a demo costs more trust than good numbers on the other rows
can buy back.

---

## 11. Guardrails for anyone (human or AI) working on this repo

- Do not add API key handling. Model access is Copilot-only.
- Do not commit anything from `config/shop.yml`, `copybooks/`, `input/`, or
  `output/`. Check `git status` before every commit.
- Do not move business logic from scripts into LLM prompts to "simplify."
- Do not remove the stage 4 human gate, or add a flag that bypasses it.
- Do not extend scope to CICS/DB2/VSAM before v1 metrics are met.
- Do not hardcode dataset names, PROC names, or job card values.
- Do not raise the retry cap above 3.
- When a value cannot be resolved, **fail loudly**. Never substitute a plausible
  default.
