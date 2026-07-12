---
last_updated: 2026-07-06
last_updated_by: kai
last_reviewed: 2026-07-06
last_reviewed_by: kai
convention: pai-freshness-v1
version: 1.0.0
---

# Ledger — the LifeOS Versioning System

> **Ledger is the subsystem that assigns the right version to every change and keeps every version number in LifeOS coherent.** It classifies each change as Major, Feature, or Patch, bumps the OS umbrella version and every touched component line, records the change in an append-only update registry, and syncs and tags the private repos. It is the versioning authority for the whole system.

LifeOS has always done this; it just never had a name or a version of its own — the same gap the Router had before it was named. The tools lived in `skills/_LIFEOS/Tools/` and `LIFEOS/TOOLS/`, the policy lived in scattered rules, and no single doc said "this is one subsystem." This doc names it **Ledger**, versions it (**1.0.0**), and draws the boundary around it.

The one-line test for what belongs to Ledger: **does it decide, record, or propagate a version?** If yes, it's Ledger.

---

## The scheme — always three levels, Major.Feature.Patch

Every live version identifier in LifeOS is exactly three levels: **`Major.Feature.Patch`**. No two-level, no four. This binds the whole version surface — the OS version (`LIFEOS/VERSION`), the Algorithm (`LIFEOS/ALGORITHM/LATEST`), the ISA Format spec, Memory, every skill's `version:`, every hook's `@version`, and any new component line. The standing rule lives in `OPERATIONAL_RULES.md`; Ledger is what enforces it.

| Level | Meaning | Gate |
|-------|---------|------|
| **Major** | Breaking or structural change — a format-schema change, a removed contract, a restructured subsystem | Human conversation required before the bump |
| **Feature** | Additive capability — a new section, a new workflow, a new subsystem | One-line confirm at ship time |
| **Patch** | Fix, doc, or no-behavior-change edit | Auto-applies with a visible notice |

The middle number is **Feature**, not "minor" — the word is standardized everywhere it's written out. `ClassifyChange.ts` already emits `patch | feature | major`; Ledger just names the scheme around it.

**Scope:** the rule governs every *live* marker and all *new* bumps. Dated changelog entries that record a past state (a doc's "v7.4 — 2026-03-31" history line) stay as recorded — back-filling `.0` into a historical record is revisionism, not versioning.

---

## Umbrella + component model

Two layers, both three-level:

- **The OS umbrella version** — `LIFEOS/VERSION` (today `6.1.2`). The single rolled-up number for the whole system. Every substantive change rolls up into it.
- **Component version lines** — independently versioned subsystems that move at their own pace and roll up into the umbrella: the **Algorithm** (`LATEST` → `v{X.Y.Z}.md`), the **system prompt** (`LIFEOS_SYSTEM_PROMPT.md` frontmatter `version:`), the **ISA Format** spec, **Memory**, each **skill** (`version:` field), each **hook** (`@version`). A component can bump on its own (the Algorithm going `6.24.0 → 6.25.0`) and that bump also rolls into the OS umbrella.

This is why a single skill edit both bumps that skill's `version:` and increments the OS version: the component line and the umbrella are different granularities of the same fact.

---

## The components (the tools)

| Tool | Role |
|------|------|
| `skills/_LIFEOS/Tools/ClassifyChange.ts` | **Classify.** Reads the diff since the last tag, proposes a level (`patch \| feature \| major`). Major is always human-gated. |
| `skills/_LIFEOS/Tools/UpdateLifeosVersion.ts` | **Bump the umbrella.** Writes the new `LIFEOS/VERSION` at the classified level (single source of truth). |
| `skills/_LIFEOS/Tools/BumpAlgorithmVersion.ts` | **Bump the Algorithm** component line + `LATEST`. |
| `skills/_LIFEOS/Tools/BumpSystemPromptVersion.ts` | **Bump the system prompt's** frontmatter `version:` (default patch; `--sp-level` elevates; a single markdown doc can't be auto-classified, so the human sets feature/major). |
| `skills/_LIFEOS/Tools/BumpHookVersions.ts` | **Bump touched hooks'** `@version` (git-derived). |
| `skills/_LIFEOS/Tools/BumpSkillVersions.ts` | **Bump touched skills'** `version:` field, scoped per changed skill. |
| `skills/_LIFEOS/Tools/CreateUpdate.ts` | **Record.** Appends the classified change to the update registry with its version field. |
| `skills/_LIFEOS/Tools/UpdateKaiRepo.ts` | **Sync + tag.** Verified two-repo sync of the private system repo + the private user-data repo, with `--bump` running the component bumps and auto-tagging both. |
| `skills/_LIFEOS/Tools/SeedVersions.ts` · `SeedHookVersions.ts` | **Seed.** One-time stamping of version lines onto components that lacked them. |

Workflows that orchestrate them: `skills/_LIFEOS/Workflows/VersionBump.md` (`/vb`) and `UpdateLifeosVersion.md`. The `rc` shortcut cuts a release candidate (VersionBump then CreateShadowRelease).

---

## The flow

```
change lands
  → ClassifyChange.ts          # diff → {patch | feature | major}
  → (major? → human conversation before proceeding)
  → UpdateLifeosVersion.ts     # bump LIFEOS/VERSION at that level
  → Bump{Algorithm,SystemPrompt,Hook,Skill}Versions.ts   # roll the touched component lines
  → CreateUpdate.ts            # append to the update registry
  → UpdateKaiRepo.ts --bump    # sync + tag both private remotes
```

Component bumping now runs automatically inside `UpdateKaiRepo --bump` — it is no longer a separate prose step that could be skipped.

---

## Operation vocabulary (never conflate these three)

Ledger governs the private-version side. Two later stages are release stages, deliberately separate and increasingly sensitive:

1. **UpdateKaiRepo** — private-repo sync + version-tag to the two private remotes (the system repo and the user-data repo). Everyday sensitivity. This is Ledger's terminal step.
2. **CreateShadowRelease** — "cut" a release: stage to the shadow-release directory only, run the release gates, no push. Moderate sensitivity.
3. **CreateRelease** — "publish" to the public LifeOS repo. Extremely sensitive, always a deliberate separate step requiring explicit go-ahead.

**"Cut" never means "publish." Private-sync ≠ cut ≠ publish.**

---

## What Ledger is NOT

- **Not the release/publish system.** Ledger stamps and records versions and syncs the private repos. Pushing a public release is `CreateRelease`, a separate deliberate act.
- **Not the Router.** The Router decides how to handle a prompt; Ledger decides what version a change is.
- **Not a scheduler.** It doesn't time releases; it classifies and stamps the ones that happen.

---

## Release role

Ledger ships with LifeOS as a documented subsystem. This doc lives under `LIFEOS/DOCUMENTATION/`, mirrored by the public-docs sync (with its scrub rules) and covered by the shadow-release gates — no new packaging machinery. It's listed in the routing table (`CLAUDE.md`), the master architecture doc, `ARCHITECTURE_SUMMARY.md`, and carries its own component version (`1.0.0`), exactly like every other subsystem.

---

## Cross-references

- Standing rule (three-level, Major.Feature.Patch) — `LIFEOS/USER/CONFIG/OPERATIONAL_RULES.md` § Versioning
- Version-bump workflow — `skills/_LIFEOS/Workflows/VersionBump.md` (`/vb`)
- Per-skill versioning — `LIFEOS/DOCUMENTATION/Skills/SkillSystem.md`
- OS version source of truth — `LIFEOS/VERSION`
- Master architecture — `LIFEOS/DOCUMENTATION/LifeosSystemArchitecture.md`
