# Diagnose · Governance Error Investigation + Remediation

Diagnose why an agent, process, or automation was blocked by a governance policy. Fetch all deployed policies on the tenant, correlate the error to the blocking property with confidence ranking, show a diff, and update on approval.

## When this plugin is invoked

Orchestrator detects diagnose mode from phrases like:
- "my agent failed with this governance error: ..."
- "why is model X blocked?"
- "I got a 403 forbidden by governance policy"
- Any error text containing `governance policy`, `forbidden`, or AOPS product keywords

## Shared primitives used

This capability delegates to:
- [../../auth-context.md](../../auth-context.md) — read `UIPATH_TENANT_ID`, `UIPATH_ACCESS_TOKEN`
- [../../policy-crud.md](../../policy-crud.md) — `deployment tenant get`, `policy get` (with full `data`), UPDATE recipe
- [../../property-labels.md](../../property-labels.md) — human labels + descriptions for every property shown in findings, diff, and patch record

## Input contract

```jsonc
{
  "mode": "diagnose",
  "errorText": "<raw error message or natural-language description>",
  "tenantIdentifier": "<from ~/.uipath/.auth UIPATH_TENANT_ID>"
}
```

## Recipe

### 1. Extract signal from the error

Parse `errorText` for structured hints:

| Signal | Example | Maps to |
|---|---|---|
| Model name | `gpt-4o-2024-11-20`, `claude-3`, `gemini-pro` | Provider toggle (`azure-openai-control-toggle`, etc.) |
| Region | `in EU`, `in US`, `in Japan` | `allowed-llm-regions.*` |
| Product context | `governance policy`, `AI Trust Layer` | `productIdentifier` |
| HTTP status | `403`, `forbidden` | Policy enforcement block |
| Feature | `PII masking`, `prompt injection`, `harmful content` | `*-execution-stage`, `*-container.*` |

Don't hardcode mappings. Use these as search hints when scanning actual policy data in Step 3. The property names are self-descriptive — correlate `gpt-4o` → `azure-openai-control-toggle` by reading field names.

If the error text is too vague: ask "Can you share the full error message or describe what you were trying to do?"

### 2. Fetch the tenant deployment map

Use the TENANT-GET recipe in [../../policy-crud.md](../../policy-crud.md#tenant-get-recipe):

```bash
uip admin aops-policy deployment tenant get "$UIPATH_TENANT_ID" --output json
```

Filter `Data.tenantPolicies[]` to entries where `policyIdentifier != null`. Present the landscape:

```
Deployed policies on tenant DefaultTenant:
  AITrustLayer (NoLicense)  → policy "iso-42001-ai-trust-layer" (d0a68808-...)
  Robot (Attended)           → (no custom policy)
  ...

Found 1 deployed custom policy. Fetching configuration...
```

If zero deployed policies: "No custom governance policies on this tenant. The error may be from org-level / global defaults — check the AOPS admin console."

### 3. Fetch full policy (metadata + formData) for each deployed policy (parallel)

Use the read-full-policy recipe from [../../policy-crud.md](../../policy-crud.md#read-full-policy-recipe):

```bash
uip admin aops-policy get <policyIdentifier> --output json
```

`Data.data` contains the full formData. Collect: `{ policyIdentifier → { productIdentifier, policyName, formData: Data.data } }`.

### 4. Correlate with confidence ranking

Scan each policy's `formData` for properties in a blocking state. Rank each finding by confidence:

| Tier | When | Example |
|---|---|---|
| **HIGH** | Direct text match between error and property path segment | `"in EU"` ↔ `allowed-llm-regions.europe` |
| **MEDIUM** | Domain-knowledge inference (one step of reasoning) | `gpt-4o` → Azure OpenAI → `azure-openai-control-toggle` |
| **LOW** | Property is blocking but unrelated to the error | `"in EU"` error + `anthropic-control-toggle: false` |

**Scoring rules (apply in order):**

1. **HIGH** — error contains a synonym/direct reference to a path segment:
   - `"EU"` / `"Europe"` → `allowed-llm-regions.europe` ✓
   - `"PII"` / `"personal data"` / `"masking"` → `pii-processing-mode`, `container.pii-*` ✓
   - Region names, feature names, enum values literally present in the error

2. **MEDIUM** — requires product knowledge:
   - Model name → provider (`gpt-*` → Azure OpenAI, `claude-*` → Anthropic, `gemini-*` → Gemini)
   - Concept → property family (`"log retention"` → `traces-ttl*`)
   - HTTP status + generic phrase → probable property family

3. **LOW** — property is `false` / restrictive but has no textual connection. Only include if user asked "show everything blocking."

4. **Tie-breaker** — within a tier, prefer findings from the product the error most clearly references.

**Presentation — HIGH first, always explain reasoning. Use [property-labels.md](../../property-labels.md) to render human labels alongside technical keys:**

```
Diagnosis — policy "iso-42001-ai-trust-layer" (AITrustLayer, tenant-level):

MOST LIKELY (1 finding):
  Europe (allowed-llm-regions.europe): Off
    ← Error says "in EU" — direct match on region path

POSSIBLY (1 finding):
  Open AI (azure-openai-control-toggle): Off
    ← gpt-4o-2024-11-20 is an Azure OpenAI model; provider is disabled
    Description: Disabling models will impact genAI features across products

UNLIKELY / UNRELATED (2 findings, hidden by default):
  Anthropic (anthropic-control-toggle): Off    — unrelated to the error
  Gemini   (gemini-control-toggle):    Off    — unrelated to the error
```

Format: `<Label> (<technical-key>): <human-value>`. Look up `<Label>` per the lookup convention in [property-labels.md](../../property-labels.md). For boolean values, render `true → On`, `false → Off`. For enums, look up the value as a label too (e.g., `"DetectionAndMasking"` → `PII Masking`). Include the `-description` line for the top finding when available — gives the user product context without forcing them to recall what the property does.

Only show UNLIKELY if the user asks for it or HIGH + MEDIUM are both empty.

If nothing matches: "I couldn't identify a property causing this error in deployed tenant-level policies. The block may come from a different policy layer (group/user/global) or non-AOPS governance."

### 5. Diff preview + approval

By default, propose fixes only for **HIGH** and **MEDIUM** tiers. LOW is informational.

```
Proposed fix for policy "iso-42001-ai-trust-layer":

  [MOST LIKELY] Europe   (allowed-llm-regions.europe):    Off → On
  [POSSIBLY]    Open AI  (azure-openai-control-toggle):   Off → On

All other fields remain unchanged (78 fields).
Apply this fix? (y / partial / n)
```

(Same label lookup as the diagnosis output. The diff stays scannable for technical reviewers because the raw key is right next to the label.)

Three user options:
- `y` — apply all proposed changes
- `partial` / `"just fix the region"` — apply a subset (user specifies)
- `n` — no changes

> **Tip surface when both tiers have findings:** "MOST LIKELY alone often unblocks. Say `just apply MOST LIKELY` for the minimal fix — you can re-run diagnosis if the error persists."

Wait for explicit approval. No side effects before then.

### 6. Apply the fix

Use the UPDATE recipe in [../../policy-crud.md](../../policy-crud.md#update-recipe):

1. Current formData is already cached (Step 3).
2. Apply approved changes onto the formData object.
3. Call `uip admin aops-policy update` with the patched formData.

### 7. Write patch record

Write `./patch-record-<policyName>-<timestamp>.json`. See [patch-record.md](patch-record.md) for the schema. Each applied change carries its confidence tier and reasoning.

### 8. Report

```
✓ Policy "iso-42001-ai-trust-layer" updated successfully.

Changes applied:
  [MOST LIKELY] Europe   (allowed-llm-regions.europe):   Off → On
  [POSSIBLY]    Open AI  (azure-openai-control-toggle):  Off → On

Patch record: ./patch-record-iso-42001-ai-trust-layer-<ts>.json

The agent should now be able to use gpt-4o-2024-11-20 in EU.
If the error persists, check group-level or user-level policy overrides.
```

## Error map

| Situation | Action |
|---|---|
| `tenant get` fails (403/404) | Halt. Verify `UIPATH_TENANT_ID` in `~/.uipath/.auth`. |
| `policy get` fails for one policy | Skip that policy with a warning. Continue with others. |
| `update` returns 400 | Halt. Surface error — likely schema mismatch. |
| `update` returns 401/403 | Halt. User may lack update permission. |
| `update` returns 409 | Halt. Concurrent modification. Ask user to retry. |
| No deployed policies found | Tell user. Suggest checking org-level / global policies. |
| Error signal too vague | Ask user for the full error message. |
