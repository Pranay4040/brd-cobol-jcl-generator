# BRD → COBOL + JCL Generator

> ## ⚠️ INTERNAL / ENTERPRISE USE ONLY
>
> This repository is licensed for **internal enterprise and office use only.**
> It is not open source, not public domain, and not licensed for personal,
> commercial resale, redistribution, or third-party use of any kind.
> See [LICENSE](LICENSE) and [USAGE-POLICY.md](USAGE-POLICY.md) before using,
> copying, or sharing any part of it.
>
> **No company data belongs in this repository.** Real copybooks, real BRDs,
> real shop standards, and real dataset names stay on the office machine and are
> excluded by `.gitignore`. Check `git status` before every commit.

An AI-assisted pipeline that reads a Business Requirements Document and produces
**reviewable** COBOL programs and JCL jobs for a z/OS batch environment.

**Status:** greenfield scaffold. Repository skeleton only — no pipeline code yet.
Implementation follows [BUILD_PLAN.md](BUILD_PLAN.md), one step at a time.

---

## The one idea that matters

**The LLM does translation. Deterministic code does accuracy.**

The model is used at exactly two stages: turning prose into a structured spec,
and writing PROCEDURE DIVISION logic. Everything else — copybook resolution, JCL
assembly, skeleton generation, validation — is plain code with no model in the
loop.

This is the design, not a limitation to be engineered around. Any change that
moves work *into* the LLM and *out of* deterministic code is a regression.

### The non-negotiable rule

> The generator may never invent a field name, a PIC clause, a dataset name, a
> DD name, or a PROC name. Every one of those comes from a copybook lookup, the
> config file, or the approved spec. If a value cannot be resolved, the pipeline
> **fails loudly** rather than guessing.

Hallucinated identifiers are the single failure mode that destroys trust in this
tool. It is guarded structurally — read-only agent tools, copybook-resolved
field lists, template-only JCL — not with prompt wording.

---

## Pipeline

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

**Stage 4 is a hard stop.** The pipeline never proceeds past it automatically,
under any circumstance, including when the spec looks complete. There is no
bypass flag, and adding one is a guardrail violation.

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

---

## Scope (v1)

**In scope — one program archetype only:**

- Sequential input file → validation → sequential output file + report
- Fixed-length records
- Copybook-driven record layouts
- Standard batch JCL: job card, one or two EXEC steps, DD statements, SYSOUT

**Explicitly out of scope.** Do not build for, design for, or accommodate these:
CICS · DB2 / embedded SQL · VSAM · IMS · multi-program call chains ·
restart/checkpoint logic · sort steps beyond a single trivial SORT.

Building v1 flexible enough to "maybe support DB2 someday" produces a worse v1.
Adding these later is a deliberate later decision.

---

## Environment constraints

These shape the architecture. Read them before proposing anything.

- **Editor:** VS Code
- **Model access:** GitHub Copilot only, via custom agents (`.agent.md` files)
- **No API keys anywhere.** No `ANTHROPIC_API_KEY`, no `OPENAI_API_KEY`, no
  `.env` with model credentials. The model comes from the user's Copilot
  subscription through VS Code. Any code that reaches for an API key is wrong.
- **Two machines.** Scaffolding is built on a personal laptop with synthetic
  data. Real copybooks, real BRDs, and real shop standards exist only on the
  office laptop. **Code moves personal → office. Company data never moves the
  other way.**
- **Compile access:** to be confirmed. Assume none initially; the JCL path is
  designed to be fully validatable without a mainframe.

---

## Repository layout

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

Directories marked GITIGNORED exist locally on each machine and are never
committed. The committed tree carries `.gitkeep` files where a directory is
still empty.

---

## Configuration

Nothing shop-specific is ever hardcoded. All of it lives in `config/shop.yml`,
which is gitignored. A synthetic [`config/shop.sample.yml`](config/shop.sample.yml)
is committed as the template.

```bash
cp config/shop.sample.yml config/shop.yml
```

The personal laptop has a synthetic `shop.yml`. The office laptop has the real
one at the same path. **Switching machines requires zero code changes** — only a
different config file being present.

---

## Validation

**JCL** — `scripts/validate_jcl.py`: job card present and well-formed; every
EXEC has its required DDs; no DD name referenced that isn't defined; dataset
names conform to the `naming` rules in config; columns 1–71 respected with
correct continuation cards; no unresolved `{{` placeholders remain.

**Spec** — `scripts/validate_spec.py`: JSON Schema validation plus semantic
checks — every referenced copybook exists, every validation rule has an
`on_failure`, every business rule has `source_text`.

**COBOL** — compile gate. GnuCOBOL `cobc -fsyntax-only` if available locally,
the real compile PROC on the office laptop if an LPAR is reachable. If neither
is available, COBOL generation stays experimental and JCL is the shippable
deliverable.

**Retry loop:** validation failures feed back to generation with the error text
attached. **Hard cap of 3 attempts.** After the third failure the pipeline stops
and reports what it could not fix. It does not loop indefinitely, and it does
not silently emit invalid output.

---

## Definition of done (v1)

Measured against a golden set of 10 real BRD + COBOL + JCL triplets, assembled
on the office machine and gitignored.

| Metric | Target |
|--------|--------|
| JCL passes validator first attempt | ≥ 90% |
| JCL usable with only trivial edits | ≥ 80% |
| COBOL compiles within 3 attempts | ≥ 70% |
| Reviewer edits to PROCEDURE DIVISION | < 30% of lines |
| Hallucinated identifiers | **0 — hard gate** |

The last row is pass/fail, not a percentage to optimise. A single invented
copybook field in a demo costs more trust than good numbers on the other rows
can buy back.

---

## Guardrails

Binding on anyone working in this repo, human or AI. Full text in
[CONTRIBUTING.md](CONTRIBUTING.md).

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

---

## Documents

| File | What it is |
|------|-----------|
| [PROJECT.md](PROJECT.md) | Source of truth for architecture, scope, and constraints |
| [BUILD_PLAN.md](BUILD_PLAN.md) | Ordered implementation steps with acceptance criteria |
| [CONTRIBUTING.md](CONTRIBUTING.md) | Working rules, commit hygiene, data-handling policy |
| [USAGE-POLICY.md](USAGE-POLICY.md) | Permitted-use terms — enterprise / office only |
| [LICENSE](LICENSE) | Proprietary internal-use licence |

Read `PROJECT.md` before `BUILD_PLAN.md`. Build order in the plan is
deliberately backwards from the pipeline — validators before generators, because
you cannot tell whether a generator works until something can grade its output.

---

## Licence

Proprietary. Internal enterprise and office use only. All rights reserved.
See [LICENSE](LICENSE).
