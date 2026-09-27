---
name: onboarding-kit
description: >-
  Use when the user wants to generate a complete onboarding guide for a new
  contributor to this repository. Orchestrates the four Ramp onboarding modes
  (ramp-architecture-mapper, ramp-docs-auditor, ramp-env-verifier,
  ramp-task-curator) and compiles their outputs into a single ONBOARDING.md
  at the repository root. Trigger phrases: onboarding kit, generate onboarding
  guide, create onboarding, onboard new contributor, ramp onboarding.
---

# Onboarding Kit Skill

Orchestrates the four Ramp onboarding modes to produce a complete contributor
onboarding guide. Never duplicate the logic of those modes -- activate them in
order and compile their outputs.

## Constraints (apply throughout)

- Do not modify application or source code files.
- Generated docs go only under `docs/onboarding/` or to `ONBOARDING.md` at the
  repo root.
- Never invent facts. Every claim must trace to an output doc or repository file.
- Follow `.bob/rules/onboarding-rules.md`.

---

## Step 1 -- Determine contributor experience level

If the user has not stated an experience level, use `ask_followup_question` to
ask before proceeding:

> "What is the new contributor's experience level?"
> Suggestions: beginner, junior, mid-level, senior

Record the level -- it is used in Step 6 to tailor starter-task guidance.

---

## Step 2 -- Run ramp-architecture-mapper

Switch to the **Ramp Architecture Mapper** mode using `switch_mode` with
`mode_id: ramp-architecture-mapper`.

Prompt it to:
- Explore the repository structure, entry points, services, databases, and
  runtime boundaries.
- Trace the most important user/business flows.
- Create `docs/onboarding/architecture.md` with Mermaid diagrams.

Wait for `docs/onboarding/architecture.md` to exist before continuing.
If the mode cannot complete, note the failure and continue -- record what is
missing in the final ONBOARDING.md.

---

## Step 3 -- Run ramp-docs-auditor

Switch to the **Ramp Docs Auditor** mode using `switch_mode` with
`mode_id: ramp-docs-auditor`.

Prompt it to:
- Inspect all documentation sources (README files, AGENTS.md, service docs).
- If `docs/converted/` exists, include it as a documentation source.
- Compare claims against the current implementation.
- Create `docs/onboarding/docs-drift-report.md`.

Wait for `docs/onboarding/docs-drift-report.md` to exist before continuing.

---

## Step 4 -- Run ramp-env-verifier

Switch to the **Ramp Env Verifier** mode using `switch_mode` with
`mode_id: ramp-env-verifier`.

Prompt it to:
- Inspect README files, AGENTS.md, Dockerfiles, package manifests, and CI
  config to identify prerequisites and commands.
- Execute setup, install, build, test, and lint steps.
- Record pass/fail status and exact errors for every step.
- Create `docs/onboarding/verified-setup.md`.

Wait for `docs/onboarding/verified-setup.md` to exist before continuing.

---

## Step 5 -- Run ramp-task-curator

Switch to the **Ramp Task Curator** mode using `switch_mode` with
`mode_id: ramp-task-curator`.

Prompt it to:
- Read `docs/onboarding/architecture.md`, `docs/onboarding/verified-setup.md`,
  and `docs/onboarding/docs-drift-report.md`.
- Scan source files for TODO/FIXME comments.
- Query the issue tracker if an issue-tracker MCP is configured; otherwise
  state that and rely on repository evidence.
- Select 3-5 ranked starter tasks appropriate for the contributor's experience
  level (pass the level from Step 1).
- Create `docs/onboarding/starter-tasks.md`.

Wait for `docs/onboarding/starter-tasks.md` to exist before continuing.

---

## Step 6 -- Read the four generated documents

Switch back to agent mode using `switch_mode` with `mode_id: agent`.

Use `read_file` to read all four documents in full:
1. `docs/onboarding/architecture.md`
2. `docs/onboarding/docs-drift-report.md`
3. `docs/onboarding/verified-setup.md`
4. `docs/onboarding/starter-tasks.md`

If any document is missing or empty, note what is unavailable and continue.

---

## Step 7 -- Compile ONBOARDING.md

Write `ONBOARDING.md` at the repository root using `write_file`. Include all
sections below. Pull every claim from the four generated documents or direct
repository evidence -- do not invent anything.

### Required sections in order

1. **Project overview**
   Brief description of what the project does, its purpose, and its demo
   context. Source: README.md and architecture.md.

2. **Architecture summary**
   High-level description of the major services/components, runtime boundaries,
   and how they relate. Reproduce or summarise key Mermaid diagrams from
   architecture.md.

3. **Services and responsibilities**
   One paragraph per service/application listing: what it does, its entry
   point, port, database, and key dependencies. Source: architecture.md.

4. **Main application flows**
   Walk through the 1-2 most important end-to-end flows (e.g. direct booking,
   hold-and-confirm). Source: architecture.md.

5. **Local development prerequisites**
   Required runtime versions, tools, and any platform-specific notes. Source:
   verified-setup.md Section 1.

6. **Environment variables**
   Table of all relevant env vars, their defaults, and which service uses them.
   Source: verified-setup.md Section 3.

7. **Installation and setup**
   Exact commands to install dependencies for each service, with working
   directories. Indicate verified/failed/untested status from verified-setup.md.

8. **Build, test, lint, and run**
   All commands needed to build, test, lint, and start each service locally.
   Mark each command's verified status. Include any known failures with their
   root causes. Source: verified-setup.md Sections 5-7 and 9.

9. **Known setup failures and limitations**
   Concise list of every failed or untested step from verified-setup.md,
   including root causes and recommended fixes where documented.

10. **Documentation drift to know about**
    Table of high- and medium-impact drifted claims from docs-drift-report.md.
    Contributor should be aware of these before relying on existing docs.

11. **Starter tasks**
    The 3-5 ranked tasks from starter-tasks.md. For each task include: title,
    source, relevant files, what to understand, why it is suitable, and
    dependencies/risks. Tailor the framing to the contributor's stated
    experience level (from Step 1):
    - beginner/junior: emphasise which files to read first and what concepts
      to look up before starting.
    - mid-level: include the architectural context and testing requirements.
    - senior: include the broader risk assessment and cross-service implications.

12. **Next steps and useful references**
    Links to the four generated onboarding docs, key source files, and any
    important READMEs.

---

## Step 8 -- Final notice

After writing ONBOARDING.md, tell the contributor:

> "Your onboarding guide is ready at ONBOARDING.md.
>
> The four detailed onboarding documents are available under docs/onboarding/:
> - architecture.md
> - docs-drift-report.md
> - verified-setup.md
> - starter-tasks.md
>
> If you are capturing this session for the IBM Bob 2.0 Hackathon demo, please
> take a screenshot or save a session summary now before closing the
> conversation."
