# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

Two related things published together:

1. **An mdBook** (`src/`) — a ~60,000-character Chinese-language book titled *一人企业方法论* (One-Person Company Methodology), v2.1. Built with [mdBook](https://rust-lang.github.io/mdBook/).

2. **A set of AI agent skills** (`skills/`) — Codex/OpenAI agent skill definitions that implement the same methodology as an interactive, guided workflow. Each skill maps to one stage of the OPC process.

## Build commands

```bash
# Build the mdBook (HTML output → book/)
mdbook build

# Build the EPUB
mdbook-epub --standalone true   # output lands in book/

# Count characters across all src/*.md files
./words.sh
```

There is no test suite and no linter. The only "build" artifacts are the book outputs.

## Repository structure

```
book.toml              # mdBook config; src dir is src/, language zh-cn
src/                   # Book chapters (Chinese markdown)
  SUMMARY.md           # mdBook table of contents — defines chapter order
  *.md                 # Book chapters
  images/              # Images referenced in chapters
skills/                # Agent skill definitions (one directory per skill)
  opc-orchestrator/    # Master orchestrator — entry point for the full workflow
  opc-resource-audit/  # Stage 01
  opc-niche-positioning/  # Stage 02
  opc-value-proposition/  # Stage 03
  opc-business-model-design/  # Stage 04
  opc-mvp-designer/    # Stage 06
  opc-conversion-loop/ # Stage 07
  opc-asset-ops/       # Stage 08 (repeatable)
  opc-dashboard-review/# Stage 09 (repeatable)
words.sh               # Character count utility
```

Each skill directory follows this layout:
```
skills/<skill-name>/
  SKILL.md             # Full skill instructions (the authoritative spec)
  agents/openai.yaml   # Interface metadata: display_name, short_description, default_prompt
  references/          # Supporting reference docs read by the skill
```

## Skills architecture

### How the skills work together

The skills implement a two-phase lifecycle:

**建盘期 (Setup phase) — linear, one-time, stages 01–07:**
```
01 opc-resource-audit
  → 02 opc-niche-positioning
    → 03 opc-value-proposition
      → 04 opc-business-model-design
        → 06 opc-mvp-designer
          → 07 opc-conversion-loop
```
Each stage must complete and write its outputs before the next stage begins. Stage 05 (opportunity scoring) is folded into stage 02.

**运营循环 (Operations loop) — triggered as needed, stages 08–09:**
- `opc-asset-ops` (08): triggered when repeatable outputs emerge and need systematizing
- `opc-dashboard-review` (09): triggered when operations stall or for periodic review

Both loop stages are repeatable; their output files use date-stamped filenames (format `YYYYMMDD`) so history is preserved.

### Shared file contract (`opc-doc/`)

All skills read from and write to an `opc-doc/` directory in the **user's current working directory** (not in this repo). The structure:

```
opc-doc/
  inputs/                    # Optional user-provided context files
  state/
    current-stage.json       # Which stage is active and its status
    decisions.json           # Log of confirmed key decisions
    assumptions.json         # Tracked hypotheses
    user-preferences.json    # Interaction mode, terminology preferences
  outputs/
    00-orchestrator/         # Session summaries
    01-resource-audit/       # inventory.md, scorecard.json
    02-niche-positioning/    # three-ring-analysis.md, candidates.md, target-segment.json, positioning-statement.md
    03-value-proposition/    # value-proposition-canvas.md, segment-vp-matrix.md, messaging.md
    04-business-model/       # lean-canvas.md, business-model-canvas-lite.md, pricing-notes.md, risky-assumptions.md
    06-mvp-design/           # mvp-spec.md, experiment-plan.md, human-ai-split.md
    07-conversion-loop/      # channel-strategy.md, content-plan.md, conversion-path.md
    08-asset-ops/            # asset-inventory-[date].md, action-plan-[date].md
    09-dashboard-review/     # review-[date].md
  reviews/
    dashboard.json           # Append-only historical review data
```

`opc-doc/` is not pre-created. Skills create it on demand. If the directory doesn't exist, the skill treats it as a fresh start.

### Interaction protocol (applies to all skills)

Every skill enforces the same interaction rules defined in `skills/opc-orchestrator/references/interaction-protocol.md`:

- **Dialogue before files**: present options and get user confirmation *before* writing any file. Describing a conclusion in chat is not the same as writing it to disk.
- **One question at a time** by default; 2–3 may be combined only if they are short and closely related.
- **Always offer 3 options + "4. 我有自己的方案"** for key decisions. Present analysis only — no direct recommendation.
- **Stage boundary enforcement**: each skill only does what its stage covers. If a topic belongs to a later stage, say so and defer it.
- **Round limits**: stages should conclude by round 10. Rounds 5–8 are a wind-down phase; round 9 forces a summary; round 10 defaults to completion.
- **Session recovery**: at the start of every session, read `opc-doc/state/current-stage.json` and `decisions.json` *before* asking any questions. Present a progress summary and ask whether to continue.

### Adding or modifying a skill

1. The source of truth for a skill's behavior is its `SKILL.md`.
2. `agents/openai.yaml` provides only display metadata (`display_name`, `short_description`, `default_prompt`); it does not contain logic.
3. Reference documents in `references/` are read by the skill at runtime — keep them factual and stable.
4. If adding a new stage, assign it an unused stage number and add it to `skills/opc-orchestrator/references/stage-map.md`.
5. Update `skills/opc-orchestrator/SKILL.md` if the new stage affects orchestration order or loop triggers.

### Adding book content

Book chapters must be registered in `src/SUMMARY.md` to appear in the built output. The title and chapter hierarchy in `SUMMARY.md` determine the mdBook navigation. Cover image is set in `book.toml` under `[output.epub]`.
