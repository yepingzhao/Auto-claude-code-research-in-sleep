# DeepXiv Optional Source Integration for ARIS

Date: 2026-04-08
Repo: `Auto-claude-code-research-in-sleep`
Scope: Add DeepXiv as a non-breaking optional data source and standalone skill in both the base ARIS skill set and the Codex mirror.

## Goal

Integrate `deepxiv_sdk` into ARIS so users can:

- call a standalone `/deepxiv` skill from ARIS
- call the mirrored Codex skill from `skills/skills-codex/`
- opt into DeepXiv-backed literature retrieval inside existing workflows, especially `/research-lit`

This integration must not change existing default behavior. DeepXiv is an additive source, not a replacement for `/arxiv`, `/semantic-scholar`, or the current web-search path.

## Non-Goals

- Do not replace the current `/arxiv` skill
- Do not replace the current `/semantic-scholar` skill
- Do not make DeepXiv part of the default `sources: all` behavior
- Do not vendor or fork the full `deepxiv_sdk` codebase into ARIS
- Do not require DeepXiv installation for existing ARIS workflows to keep working

## User-Facing Outcome

After this change, users can:

- install DeepXiv separately with `pip install deepxiv-sdk`
- invoke `/deepxiv "topic or paper id"` as a first-class ARIS skill
- use `/research-lit "topic" — sources: deepxiv`
- use `/research-lit "topic" — sources: all, deepxiv`
- continue using existing workflows without any behavior change when `deepxiv` is not requested

## Design Summary

The integration uses a thin adapter inside ARIS instead of directly hard-coding against raw `deepxiv` CLI output in multiple skills.

The adapter will:

- check whether the `deepxiv` executable is installed
- normalize a small set of CLI calls used by ARIS
- prefer machine-readable output where available
- return consistent failure modes for:
  - command not installed
  - auth/token issues
  - rate-limit issues
  - invalid paper IDs

This keeps ARIS skill logic stable even if the upstream CLI evolves.

## Architecture

### 1. Thin CLI Adapter

Add `tools/deepxiv_fetch.py`.

Responsibilities:

- provide a narrow ARIS-facing interface around the installed `deepxiv` CLI
- expose stable subcommands for the retrieval operations ARIS needs
- validate installation and surface actionable setup instructions
- normalize stdout into a predictable format for skill usage

Planned adapter operations:

- `search`
- `paper-brief`
- `paper-head`
- `paper-section`
- `trending`
- `wsearch`
- `sc`

Implementation rule:

- the adapter delegates to the installed `deepxiv` command and does not reimplement DeepXiv API logic

### 2. Base ARIS Skill

Add `skills/deepxiv/SKILL.md`.

Responsibilities:

- document when to use DeepXiv
- explain the progressive reading workflow
- instruct the agent to call the adapter instead of shelling out to raw `deepxiv` commands everywhere
- include setup and graceful fallback instructions

This skill is positioned as:

- stronger than `/arxiv` for layered reading and agent-friendly paper access
- complementary to `/semantic-scholar` for venue metadata and citation-rich published papers
- optional within literature workflows

### 3. Codex Mirror Skill

Add `skills/skills-codex/deepxiv/SKILL.md`.

Responsibilities:

- mirror the base DeepXiv skill in Codex-native packaging
- preserve command semantics and usage guidance
- maintain the same optional-source contract as the base skill

### 4. Workflow Integration

Primary integration target: `skills/research-lit/SKILL.md`

Changes:

- extend `sources:` parsing to recognize `deepxiv`
- update source selection docs and examples
- add a dedicated DeepXiv search branch that only runs when explicitly requested
- merge DeepXiv findings into the existing synthesis and de-duplication flow

Secondary integration target: `skills/skills-codex/research-lit/SKILL.md`

Changes mirror the base skill.

No default behavior change:

- if `deepxiv` is not present in `sources:`, no DeepXiv logic runs
- if DeepXiv is not installed, the skill reports a soft warning and falls back to the remaining requested sources when possible

### 5. Top-Level Workflow Awareness

For workflow docs that transitively depend on `/research-lit`, update wording only where needed so users understand that DeepXiv can be passed through as an optional source.

Initial touch points:

- base `skills/research-lit/SKILL.md`
- Codex mirror `skills/skills-codex/research-lit/SKILL.md`
- install/readme documentation where optional sources are described

High-level pipeline skills such as `/idea-discovery` do not need a new default path. They only need source-pass-through documentation if that is already part of the existing workflow contract.

## Source Semantics

DeepXiv enters ARIS as a new explicit source:

- `deepxiv`

Behavioral contract:

- `sources: deepxiv` means use only DeepXiv-backed retrieval
- `sources: web, deepxiv` means use both
- `sources: all` remains unchanged and does not implicitly include DeepXiv
- `sources: all, deepxiv` means current default sources plus DeepXiv

This mirrors the existing `semantic-scholar` opt-in behavior and avoids hidden changes to retrieval breadth, latency, or rate limits.

## Data Flow

### Standalone `/deepxiv`

1. user invokes `/deepxiv "query or paper id"`
2. skill parses intent
3. skill calls `tools/deepxiv_fetch.py`
4. adapter calls installed `deepxiv`
5. skill presents results in ARIS-native markdown tables and summaries

### `/research-lit` with `deepxiv`

1. user invokes `/research-lit "... — sources: deepxiv"` or includes `deepxiv` in a multi-source list
2. existing source parser includes the `deepxiv` branch
3. skill calls the adapter for search and progressive reading
4. papers are merged into the same analysis pipeline used by other sources
5. synthesis output labels result provenance as `deepxiv`

## De-Duplication Strategy

DeepXiv may overlap with arXiv and Semantic Scholar results. De-duplication will use:

- arXiv ID when available
- DOI when available
- normalized title fallback when neither is available

Priority rules:

- if a result is the same paper as an existing arXiv result, keep one canonical entry and preserve DeepXiv as an additional source note
- if a result overlaps with Semantic Scholar and S2 has richer venue metadata, retain the richer metadata while preserving the DeepXiv reading affordance in notes
- venue-only or PMC-only findings from DeepXiv are retained as distinct additions

## Error Handling

The integration must degrade gracefully.

Cases:

- `deepxiv` not installed:
  - standalone `/deepxiv` tells the user how to install it
  - `/research-lit` logs a soft warning and continues with other requested sources if any remain
- token registration/auth error:
  - adapter surfaces the upstream message
  - skill suggests rerunning a basic DeepXiv command or checking `DEEPXIV_TOKEN`
- rate limit reached:
  - adapter reports it explicitly
  - workflow continues with non-DeepXiv sources where possible
- invalid paper ID or empty search:
  - report clearly without crashing the whole workflow

## Testing Plan

### Adapter Verification

- installed check works when `deepxiv` is absent
- search path works with a valid query
- paper brief/head/section paths work with a known paper ID
- trending path works
- adapter returns a stable non-zero exit path on common failures

### Skill Verification

- standalone `/deepxiv` instructions are internally consistent
- standalone `/deepxiv` includes setup and fallback guidance
- base and Codex mirror skill files remain aligned

### Workflow Verification

- `/research-lit "... — sources: deepxiv"` executes the DeepXiv branch
- `/research-lit "... — sources: all, deepxiv"` keeps existing `all` semantics and adds DeepXiv
- `/research-lit "... — sources: all"` remains unchanged
- existing workflows without `deepxiv` continue to behave as before

### Documentation Verification

- README or install docs mention DeepXiv as optional
- examples are consistent across base and Codex docs

## Files Expected To Change

New files:

- `tools/deepxiv_fetch.py`
- `skills/deepxiv/SKILL.md`
- `skills/skills-codex/deepxiv/SKILL.md`

Likely modified files:

- `skills/research-lit/SKILL.md`
- `skills/skills-codex/research-lit/SKILL.md`
- `README.md`
- `README_CN.md`
- `skills/skills-codex/README.md`

Possible modified files:

- additional docs only if needed to document optional source usage

## Key Tradeoffs

### Thin Adapter vs Direct CLI Calls

Chosen: thin adapter.

Reason:

- centralizes installation checks and error normalization
- avoids scattering DeepXiv-specific shell logic across multiple skills
- lowers maintenance cost if the CLI output changes

### Optional Source vs Default Source

Chosen: optional source.

Reason:

- preserves current ARIS defaults
- avoids hidden retrieval-cost and latency changes
- matches the existing `semantic-scholar` opt-in pattern

### Standalone Skill + Workflow Embedding

Chosen: both.

Reason:

- standalone skill supports explicit expert use
- workflow embedding makes DeepXiv useful without forcing users to manually orchestrate it each time

## Open Constraints

Assumptions made for implementation:

- `deepxiv-sdk` remains an external dependency installed by the user
- the installed `deepxiv` command is reachable on PATH
- the CLI supports the commands already documented in `deepxiv_sdk`

If any of these assumptions fail during implementation, the adapter will be designed to fail softly and provide recovery instructions rather than breaking ARIS workflows.

## Implementation Boundary

This design covers only:

- optional DeepXiv source integration
- standalone skill addition
- Codex mirror addition
- minimal documentation updates

It does not cover:

- migrating existing ARIS literature tooling onto DeepXiv by default
- replacing current fetch scripts
- adding new DeepXiv-specific automation beyond literature retrieval and progressive reading
