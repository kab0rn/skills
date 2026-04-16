---
name: uipath-governance
description: "[PREVIEW] UiPath governance brain — APPLY compliance packs across AOPS products, DIAGNOSE governance errors with ranked fixes, ADVISE on policy configuration. Uses `uip admin aops-policy`. For pack authoring→uipath-compliance-pack-author."
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# UiPath Governance Brain

One skill, three capabilities over UiPath AOPS governance:

| Mode | What it does | Plugin |
|---|---|---|
| **Apply** | Apply a compliance pack (`.uipolicy`) — create + deploy AITL policies | [plugins/compliance/impl.md](references/plugins/compliance/impl.md) |
| **Diagnose** | Investigate a governance error and remediate with confidence-ranked fixes | [plugins/diagnosis/impl.md](references/plugins/diagnosis/impl.md) |
| **Advise** | Map a natural-language requirement to AOPS policy configuration across products | [plugins/advise/impl.md](references/plugins/advise/impl.md) |

All three capabilities share a small set of primitive operations (`references/*.md`) that talk to the `uip admin aops-policy` CLI.

## When to Use This Skill

- "Apply ISO 27001 to my tenant" / "Apply HIPAA compliance pack" → **Apply**
- "Deploy this already-created policy to the Finance group" → **Apply** (deploy-only sub-mode)
- "My agent failed with this governance error: ..." → **Diagnose**
- "Why is model gpt-4o blocked?" → **Diagnose**
- "I want to block AI usage outside US and EU — what do I configure?" → **Advise**
- "What policies do I need for 180-day log retention?" → **Advise**

## Mode Detection

Pick the mode by intent signal in the user's prompt:

| Prompt signals | Mode |
|---|---|
| `.uipolicy`, `compliance pack`, `ISO 27001`, `HIPAA`, `SOC 2`, `apply`, `deploy this policy` | Apply |
| pasted error text with `403`, `forbidden`, `governance policy`; words like `blocked`, `failed`, `why can't`, `diagnose` | Diagnose |
| `I want to`, `how do I configure`, `what policies do I need`, `block all X`, `restrict Y` (no pack, no error) | Advise |

If ambiguous: ask the user which mode, don't guess.

## Critical Rules (apply to every mode)

1. **Always `uip login status --output json` before any mutation.** Stop if not logged in.
2. **Read tenant / org identifiers from `~/.uipath/.auth`.** Never ask the user or decode JWTs — see [references/auth-context.md](references/auth-context.md).
3. **`--output json` on every `uip` call.** Parse `Data` / `Message` from the structured response.
4. **Never auto-apply changes without explicit user approval.** Every mode has an approval gate.
5. **Never fall back to `update` on a 409 during Apply.** For compliance-pack creation, halt with the conflict surfaced.
6. **One plugin file read per dispatch.** Do not preload other capability plugins.
7. **Write an audit record unconditionally** — deploy record for Apply, patch record for Diagnose fixes, either for Advise depending on create/update path.
8. **Fail-fast within a mode.** A 4xx halts the current phase; remaining items become `status: "skipped", reason: "prior-failure"`.

## Primitive Operations (shared across modes)

Every capability plugin delegates to these:

| File | Covers |
|---|---|
| [references/auth-context.md](references/auth-context.md) | Reading tenant / org / token from `~/.uipath/.auth` |
| [references/policy-crud.md](references/policy-crud.md) | `create` / `update` / `get-data` / `list` / `tenant get` / `template get` |
| [references/policy-assign.md](references/policy-assign.md) | `deployment tenant / group / user configure` — bulk assignment with merge-first recipe |
| [references/principals-lookup.md](references/principals-lookup.md) | Group / user GUID lookup via Identity Directory Search |
| [references/property-labels.md](references/property-labels.md) | Human-readable labels + descriptions for AOPS policy properties (i18n snapshot) |
| [references/cli-cheatsheet.md](references/cli-cheatsheet.md) | Complete `uip admin aops-policy` command reference |
| [references/cli-known-issues.md](references/cli-known-issues.md) | Known CLI bugs + workarounds |

## Workflow

### Step 0 — Preflight

```bash
uip login status --output json
```
Require `Data.Status == "Logged in"`. Then read `~/.uipath/.auth` per [auth-context.md](references/auth-context.md).

### Step 1 — Detect mode

Use the Mode Detection rules above.

### Step 2 — Dispatch to capability plugin

Read **exactly one** plugin file:
- `plugins/compliance/impl.md` — for Apply
- `plugins/diagnosis/impl.md` — for Diagnose
- `plugins/advise/impl.md` — for Advise

Each capability plugin handles its own workflow end-to-end, delegating to the primitive operations above for CLI calls.

### Step 3 — Report to user

Each capability's plugin specifies its own report format. The orchestrator just ensures the audit record was written.

## Anti-Patterns

- **Never blend modes.** One user request = one mode. If the user asks about Apply and Diagnose in the same prompt, handle Apply first, then ask about Diagnose.
- **Never preload all three capability plugins.** Dispatch to one based on mode detection. The other two stay on disk.
- **Never present diagnosis findings unranked.** Group by HIGH / MEDIUM / LOW confidence — the diagnosis plugin enforces this.
- **Never propose changes to LOW-confidence findings by default.** Informational only.
- **Never present a raw property name without its human label** when one exists in [property-labels.md](references/property-labels.md). Diagnose, advise, and the deploy/patch records all owe the user "Open AI" (label) + `azure-openai-control-toggle` (technical key), not just the technical key.
- **Never auto-pick a principal (group / user).** Always surface candidates for explicit selection.
- **Never hand-edit `formData` the user didn't ask to change.**
- **Never commit audit records (deploy / patch) to git.** Contains tenant / principal identifiers.
- **Never hardcode a product in the workflow.** Diagnose and Advise iterate all products. Apply partitions by product+scope per the pack.

## Capability References

- **Apply (compliance pack):** [impl.md](references/plugins/compliance/impl.md) · [pack-format](references/plugins/compliance/pack-format.md) · [pack-resolution](references/plugins/compliance/pack-resolution.md) · [scope-selection](references/plugins/compliance/scope-selection.md) · [synthesis-algorithm](references/plugins/compliance/synthesis-algorithm.md) · [deploy-record](references/plugins/compliance/deploy-record.md) · [products/](references/plugins/compliance/products/)
- **Diagnose:** [impl.md](references/plugins/diagnosis/impl.md) · [patch-record](references/plugins/diagnosis/patch-record.md)
- **Advise:** [impl.md](references/plugins/advise/impl.md)
