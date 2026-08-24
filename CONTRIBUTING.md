# Contributing

> Internal enterprise / office use only. Read [USAGE-POLICY.md](USAGE-POLICY.md)
> before your first commit — particularly the data rule.

This document is binding on anyone working in this repository, **human or AI
assistant.**

---

## Before you start

1. Read [PROJECT.md](PROJECT.md). It is the source of truth for architecture,
   scope, and constraints.
2. Read [BUILD_PLAN.md](BUILD_PLAN.md). Find the current step.
3. Re-read `PROJECT.md` §11 guardrails. They are repeated below.

---

## Guardrails

Non-negotiable. A change that violates any of these is rejected regardless of
how well it works.

- **Do not add API key handling.** Model access is Copilot-only.
- **Do not commit anything** from `config/shop.yml`, `copybooks/`, `input/`, or
  `output/`. Check `git status` before every commit.
- **Do not move business logic from scripts into LLM prompts** to "simplify."
  Deterministic code doing accuracy is the design, not an implementation detail.
- **Do not remove the stage 4 human gate,** or add a flag that bypasses it.
- **Do not extend scope** to CICS / DB2 / VSAM before v1 metrics are met.
- **Do not hardcode** dataset names, PROC names, or job card values. They come
  from config.
- **Do not raise the retry cap above 3.**
- **When a value cannot be resolved, fail loudly.** Never substitute a plausible
  default. A clear error beats a confident wrong answer every time.

### The rule behind the rules

> The generator may never invent a field name, a PIC clause, a dataset name, a
> DD name, or a PROC name.

Every such value comes from a copybook lookup, the config file, or the approved
spec. Hallucinated identifiers are the one failure mode that destroys trust in
this tool, and they are prevented structurally — read-only agent tooling,
copybook-resolved field lists, template-only JCL — not by asking a model nicely.

---

## How work proceeds

**One build-plan step at a time.** Complete a step's acceptance criteria before
starting the next. Do not skip ahead and do not build several steps at once.

**Build order is deliberately backwards from the pipeline.** Validators come
before generators, because you cannot tell whether a generator works until
something can grade its output. Building the generator first means tuning blind.

**If a step's acceptance criteria cannot be met, stop and say so.** Do not ship a
partial implementation with a `TODO` and call the step done.

**Ask before adding scope.** The scope in `PROJECT.md` §3 is deliberately narrow.

---

## Code standards

- **Prefer the standard library.** Every dependency added is one more thing to
  justify on a corporate machine.
- **Every script needs:** a clear `--help`, a non-zero exit code on failure, and
  errors that name the file and line number.
- **Write the test before the implementation** where practical.
- **Errors are for humans.** "Copybook CUSTREC not found in copybooks/" beats
  `KeyError: 'CUSTREC'`.
- **No machine-specific paths or values in code.** Switching between the personal
  laptop and the office laptop must require zero code changes — only a different
  `config/shop.yml` being present.

---

## Commit hygiene

**Enable the hooks once per clone,** on every machine:

```bash
git config core.hooksPath .githooks
```

Git does not install hooks for you, and a clone without this line has only
`.gitignore` standing between it and a committed copybook. The `pre-commit` hook
blocks staged company data, copybooks outside `samples/`, credential-shaped
strings, and CRLF in column-sensitive artefacts. `--no-verify` bypasses it; if
you reach for that flag, read the diff line by line first and be certain the
hook is wrong.

The hook is a backstop, not a substitute for looking. Before every commit:

```bash
git status
```

Read the output. Confirm that nothing from `config/shop.yml`, `copybooks/`,
`input/`, or `output/` is staged, and that no real dataset name, account code,
userid, or hostname appears in the diff or in your commit message.

Commit messages describe the change and reference the build-plan step where one
applies, for example:

```
Step 1.1: add JCL validator with column and placeholder checks
```

---

## Working with AI assistants in this repo

The same guardrails apply to generated contributions.

- The BRD analyst agent has **read-only tools by design.** It physically cannot
  write COBOL or JCL. Do not grant it write tools.
- The COBOL writer agent produces **PROCEDURE DIVISION only**, using only fields
  present in the resolved copybook list.
- Every rule implementation carries its spec ID in a comment, so a reviewer can
  trace generated logic back to the approved spec and to the quoted BRD text.
- Generated logic is written into a clearly marked region. Leave the markers in
  place — they tell reviewers where to spend attention.

Instruction files (`.github/copilot-instructions.md`, agent prompts) contain **no
real company values.** Those live in config. Keep the instruction file under
roughly 200 lines; long instruction files get diluted.
