---
name: documentation-tiers
description: Create or update long-form documentation under the docs root following a hub-and-spoke, tiered model. Use when creating, restructuring, or maintaining reference documentation. Covers naming, sizing, hierarchy, thematic placement, and cross-references.
---

## When to Use

- Creating or updating any document under `<DOCS_ROOT>`
- Deciding where new content belongs (which file, which domain)
- Extracting sections into child documents or consolidating children into parents
- Adding cross-references or glossary links
- Reviewing documentation changes for compliance

This is the discipline the **ship** stage's docs fan-out routes to when the sink is
architecture/reference documentation (see `prompts/sdlc/ship.md`). It keeps `<DOCS_SINKS>` from
drifting into a flat pile of overlapping files.

## Documentation tiers

Reference documentation is not one undifferentiated bucket. Place content by how often it is read
and how stable it is:

| Tier | What lives here | Read cadence |
|---|---|---|
| **L1 — always-loaded** | Principles + routing only (the agent/contributor instructions file). Points *at* the deeper tiers; never duplicates them. | Every task |
| **L2 — task-triggered** | Skills / focused how-to guides, auto-loaded when a task matches. Detailed patterns for one concern. | When the task matches |
| **L3 — on-demand reference** | The `<DOCS_ROOT>` tree below — architecture, design, and domain reference read explicitly when needed. | When explicitly needed |

This skill governs **L3**. The rules below keep the L3 tree navigable; the rest of the tier model
just tells you what does *not* belong here (routing prose → L1, single-concern patterns → L2).

## Decision Flowchart

When adding L3 documentation content, follow this sequence:

```
1. Is there an existing file for this topic?
   ├─ YES → Write content in that file → go to step 3
   └─ NO  → go to step 2

2. Create a new file?
   ├─ Will the new file be ≥ 30 lines?
   │  ├─ YES → Does a parent file exist at the preceding level?
   │  │  ├─ YES → Create the child file
   │  │  └─ NO  → Create the parent first, or add as a section in nearest ancestor
   │  └─ NO  → Add as a section in the most relevant existing file instead
   └─ Done → go to step 3

3. Check sizing thresholds:
   ├─ Is any single section > 50 lines OR file > 300 lines?
   │  ├─ YES → Extract section to a new child file (verify ≥ 30 lines)
   │  │        Replace with ≤ 5 sentence summary ending with: See: <ChildFile>.md
   │  └─ NO  → Done
   └─ Is a child file < 30 lines AND parent < 250 lines after merge?
      ├─ YES → Consolidate child back into parent; remove child file
      └─ NO  → Done
```

## File Naming & Hierarchy

**Pattern**: `Topic.md`, `Topic_Subtopic.md`, `Topic_Subtopic_Detail.md`

Rules:
1. PascalCase components separated by single underscores between hierarchy levels
2. Each file begins with an H1 matching (or slightly expanding) the filename
3. 1-2 line Purpose tagline directly under the H1
4. Prefer conceptually stable names (avoid volatile implementation abbreviations)
5. Keep diagrams / large tables in child docs; hub files summarize and link
6. Typically three levels max; deeper nesting requires strong justification

### Hierarchy Completeness

Every child document must have a parent at the immediately preceding level:
- `Topic_Subtopic.md` requires `Topic.md`
- `Topic_Subtopic_Detail.md` requires `Topic_Subtopic.md`
- Never skip hierarchy levels

**Justification principle**: A child document is only warranted when its content would cause the parent to exceed size thresholds. If the parent has capacity, the content belongs in the parent as a section, not as a separate file.

## Content Sizing

| Metric | Threshold | Action |
|--------|-----------|--------|
| Hub document | ≤ 300 lines | Extract sections if exceeded |
| Single section | > 50 lines | Extract to child file |
| Child document | ≥ 30 lines minimum | Merge into parent if below |
| Parent after merge | < 250 lines | Safe to consolidate child |

- Do not create placeholder files unless substantial (≥ 50 lines) content is imminent.
- Aim for parents 50–300 lines with children ≥ 30 lines each.

## Thematic Placement

Assign content by primary intent. Define your project's domain prefixes once in `<DOC_DOMAINS>` and
route every file to exactly one. A typical set:

| Domain | Content |
|--------|---------|
| Architecture_* | Design, structure, algorithms, rationale |
| Development_* | Contribution process, style, tooling, docs meta |
| Testing_* | Strategy, patterns, coverage, debugging workflows |
| Security_* | Threat model, validation, authN/Z |
| Deployment_* | Build, packaging, migrations, scaling, observability |
| UserGuide_* | User-facing explanations (not internal design rationale) |
| Glossary.md | Authoritative term definitions (link targets, not narrative) |

Adjust the domains to the project — a game adds `GameDesign_*`, an API adds `API_*`. Whatever the
set, the rule is fixed: **cross-domain features choose one primary owner domain; other domains carry
only a brief pointer.**

## Document Structure

### Required Elements

1. **H1 Title**: Match or slightly expand the filename
2. **Purpose Statement**: 1-2 line description directly under H1
3. **Back-Link** (child docs only): `← Back to [Parent](Parent.md)`
4. **Overview Section**: Key concepts and high-level explanation
5. **Cross-References**: Links to related docs

### Optional Elements

- Diagrams for complex flows
- Tables for comparisons
- Code examples, annotated with the execution context when the project has more than one (e.g.
  which tier / environment a snippet runs in)

## Linking & Cross References

- **Glossary**: First occurrence of a defined term links to `Glossary.md#term` (lowercase anchor). Don't over-link repeated terms.
- **Prefer links over duplication**: Link to authoritative concept pages rather than re-explaining.
- **Shallow cross-linking**: hub → child, hub → sibling hub, child → authoritative sources. Avoid dense sibling interlink webs.
- **Parent → child**: End summary sections with `See: <ChildFile>.md`.
- **Child → parent**: Include `← Back to [Parent](Parent.md)` near the top, after the Purpose tagline.

## Quality Checklist

Before committing documentation changes:

- [ ] Naming follows hierarchy pattern (PascalCase, underscores)
- [ ] No orphan short files (< 30 lines)
- [ ] Parent docs remain concise (< 300 lines)
- [ ] First glossary term occurrences link to Glossary.md
- [ ] Relative links resolve correctly
- [ ] No duplicated explanations (link instead)
- [ ] Purpose tagline present
- [ ] Hierarchy is complete (every child has its required parent)
- [ ] Back-links present on all child docs
- [ ] Content is at the right tier — routing prose stayed in L1, single-concern patterns in L2

## Placeholders

| Placeholder | Meaning | Example |
|---|---|---|
| `<DOCS_ROOT>` | Root directory of the L3 documentation tree | `docs/` |
| `<DOC_DOMAINS>` | The thematic domain prefixes this project routes files into | `Architecture_*, Testing_*, Security_*, UserGuide_*` |
| `<DOCS_SINKS>` | (from ship stage) documentation targets fan-out routes to | `README.md, docs/API.md` |
