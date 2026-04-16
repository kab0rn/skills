# Advise · Requirement-to-Policy Mapping

Given a natural-language requirement, identify which AOPS product(s) + properties can satisfy it, and propose a configuration plan. May involve multiple products.

## When this plugin is invoked

Orchestrator detects advise mode from phrases like:
- "I want to block all AI usage outside US and EU"
- "How can I prevent developers from using non-Azure LLMs?"
- "What policies do I need to require PII masking for all agents?"
- "Our SOC 2 auditor wants 180-day log retention — what do I configure?"

Signal: user describes a **goal or requirement**, not a pack to apply, not an error to diagnose.

## Shared primitives used

- [../../auth-context.md](../../auth-context.md) — read `UIPATH_TENANT_ID`
- [../../policy-crud.md](../../policy-crud.md) — `product list`, `template get`, `deployment tenant get`, `policy get` (with full `data`), CREATE recipe, UPDATE recipe
- [../../policy-assign.md](../../policy-assign.md) — when a new policy needs deploying after creation
- [../../principals-lookup.md](../../principals-lookup.md) — when deployment level is group/user
- [../../property-labels.md](../../property-labels.md) — human labels + descriptions for every property shown in plan presentation, deploy/patch records

## Input contract

```jsonc
{
  "mode": "advise",
  "requirement": "<raw user description>",
  "adviseOnly": false,          // true = propose only, don't execute
  "tenantIdentifier": "<from ~/.uipath/.auth UIPATH_TENANT_ID>"
}
```

## Recipe

### 1. Fetch product catalog, templates, and tenant deployment (parallel)

```bash
uip admin aops-policy product list --output json
uip admin aops-policy deployment tenant get "$UIPATH_TENANT_ID" --output json

# For each product, in parallel:
uip admin aops-policy template get <productIdentifier> --output json

# For each deployed custom policy (tenantPolicies where policyIdentifier != null):
uip admin aops-policy get <policyIdentifier> --output json    # returns metadata + data.data
```

Build a catalog map:
```
{
  <productIdentifier>: {
    templateFields: { /* from template defaults */ },
    currentPolicy:  { identifier, name, formData } | null
  }
}
```

### 2. Correlate requirement against template fields

Template field names are self-descriptive. Scan them for keywords from the requirement:

| Requirement keyword | Template field patterns |
|---|---|
| `region`, `US`, `EU`, `Japan` | `allowed-llm-regions.*` (AITrustLayer) |
| `LLM`, `AI provider`, `Anthropic`, `OpenAI`, `Gemini` | `*-control-toggle` (AITrustLayer) |
| `PII`, `personal data`, `masking` | `pii-processing-mode`, `pii-execution-stage`, `container.pii-*` (AITrustLayer) |
| `retention`, `log`, `trace` | `traces-ttl*`, `traces-disable-insights` (AITrustLayer) |
| `prompt injection`, `harmful content` | `prompt-injection-*`, `harmful-content-*` (AITrustLayer) |
| `package`, `feed`, `source` | `PackageFeeds.*` (Development) |
| `runtime`, `web filter`, `URL allow` | `RT-UIA-001.*`, `runtime-governance-enabled` (Robot) |

Don't hardcode this — Claude reads actual template fields and correlates. The table is examples.

If no template has matching fields: "I couldn't find AOPS properties that map to this requirement. It may need access policies, orchestrator roles, or org-level settings."

### 3. Group findings by product and determine path

For each matching product, check the catalog:

| Current state | Path |
|---|---|
| No custom policy deployed | **CREATE** — new policy with proposed values. Ask user for policy name + deployment target. |
| Custom policy deployed, values already set | **NO-OP** — requirement already satisfied. |
| Custom policy deployed, values differ | **UPDATE** — diff current vs. proposed. |

### 4. Present the plan

Use [property-labels.md](../../property-labels.md) to render human labels alongside technical keys for every line. Format: `<Label> (<technical-key>): <from> → <to>`.

Example for multi-product requirement:

```
Requirement: "Block AI usage outside US and EU, and restrict developers to the tenant package feed only"

Matching products (2):

  [1] AITrustLayer
      Current policy: "iso-42001-ai-trust-layer" (d0a68808-...)
      Proposed UPDATE:
        United States  (allowed-llm-regions.united-states):  Off → On
        Europe         (allowed-llm-regions.europe):         Off → On
        (other regions already Off — no change)

  [2] Development
      Current policy: (none deployed)
      Proposed CREATE: new policy "custom-dev-package-restriction"
        Custom feed              (PackageFeeds.Custom):            On → Off
        Personal workspace feed  (PackageFeeds.PersonalWorkspace): On → Off
        Tenant processes feed    (PackageFeeds.TenantPackages):    (kept On)
        Needed from you: policy name confirmation, priority, deployment level

Apply this plan? (y / partial / n)
```

For products not yet covered in `property-labels.json` (Robot, IntegrationService, etc. as of today), fall back to the raw key — note this in the plan: `(no human label available — using raw key)`.

Accept partial selection — user can say "apply just [1], skip [2]" or "apply only the region changes."

### 5. Execute (skip if advise-only)

Per approved product:

- **UPDATE path** → use UPDATE recipe in [../../policy-crud.md](../../policy-crud.md#update-recipe). Write a patch record.
- **CREATE path** → use CREATE recipe in [../../policy-crud.md](../../policy-crud.md#create-recipe), then assign via [../../policy-assign.md](../../policy-assign.md). Write a deploy record for the new policy.

If the plan creates a new policy with group or user scope, call [../../principals-lookup.md](../../principals-lookup.md) to resolve the target GUID before assigning.

### 6. Report

```
✓ AITrustLayer policy "iso-42001-ai-trust-layer" updated
  allowed-llm-regions.united-states: false → true
  allowed-llm-regions.europe:        false → true

✓ Development policy "custom-dev-package-restriction" created (<guid>) and deployed to tenant DefaultTenant
  PackageFeeds.Custom:            false
  PackageFeeds.PersonalWorkspace: false

Requirement satisfied. Records written to:
  ./patch-record-iso-42001-ai-trust-layer-<ts>.json
  ./deploy-record-custom-dev-package-restriction-<ts>.json
```

## Advise-only mode

If the user phrased as "just tell me what's possible" / "how can I do X" / "what would I need":
- Skip Step 5 (no execution)
- End after Step 4 with the plan
- Offer: "Want me to apply this? Say yes and I'll create/update the policies."

## Error map

| Situation | Action |
|---|---|
| `template get` fails for a product | Skip that product with a warning. Continue. |
| Requirement maps to zero products | Tell user it's outside AOPS scope. Suggest access policies or orchestrator roles. |
| User approves CREATE but won't supply policy name | Halt. Name is required. |
| UPDATE fails mid-plan (after successes) | Halt remaining steps. Report partial state. |

## Anti-Patterns

- **Never execute without explicit `y`.** Plan must be shown first.
- **Never skip the tenant fetch.** Without it you can't distinguish UPDATE from CREATE.
- **Never invent policy names.** Ask the user for CREATE names.
- **Never propose changes to fields that already match the desired value.** Flag them as "already satisfied."
