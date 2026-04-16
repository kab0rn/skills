# Apply · Compliance Pack

Apply a compiled `.uipolicy` compliance pack to a UiPath tenant. Two phases: creation, then deployment.

> **Scope:** all `policyKind: "product"` entries for any AOPS product are processed through the shared CREATE recipe. Product-specific quirks (CLI value conventions, known error patterns) live in optional `products/<product>.md` files — only created when there's something real to document. `policyKind: "access"` entries are skipped (access policy CLI is on a separate branch, pending merge).

## Invocation modes

| Mode | Trigger | Runs |
|---|---|---|
| Full apply (default) | "apply ISO 27001" | Phase 1 (create) + Phase 2 (deploy) |
| Create-only | "just create", "--skip-deploy" | Phase 1 only |
| Deploy-only | "deploy this policy", "--skip-create" + policyId | Phase 2 only |

## Shared primitives used

- [../../auth-context.md](../../auth-context.md) — read `UIPATH_TENANT_ID`, etc.
- [../../policy-crud.md](../../policy-crud.md) — CREATE recipe (fetch defaults + merge + create)
- [../../policy-assign.md](../../policy-assign.md) — tenant / group / user binding
- [../../principals-lookup.md](../../principals-lookup.md) — when deployment level is group/user
- [../../property-labels.md](../../property-labels.md) — human labels for pre-flight and report output

## Workflow

### Step 0 — Preflight

Run the preflight check from [../../auth-context.md](../../auth-context.md). Require `Data.Status == "Logged in"` and `UIPATH_TENANT_ID` non-empty.

### Step 1 — Resolve the pack

[pack-resolution.md](pack-resolution.md). Accepts `--pack-file <path>` (V0), `--pack-url <url>`, or `--pack-id <id> [--pack-version <v>]`.

### Step 2 — Parse

[pack-format.md](pack-format.md). Read `manifest.json`, `clause-map.json`, every `policies/*.json`.

### Step 3 — Partition applicable vs. skipped

```
applicable = [ p for p in manifest.policies if p.policyKind == "product" (or absent) ]
skipped    = [ p for p in manifest.policies if p.policyKind == "access" ]
```

Access policy support is deferred — the `uip govern policy` CLI family lives on `jianjunwang/governance-policy-tool` and hasn't merged yet. When it does, add a `policyKind == "access"` branch.

Log the split to the user:
```
Pack: iso-27001-2022 v1.0.0
Applicable (product policies, 4 files):
  - policies/ai-trust-layer.json  (AITrustLayer)
  - policies/development.json     (Development)
  - policies/robot.json           (Robot)
  - policies/studio-web.json      (StudioWeb)
Skipped (access policies — out of scope, 0 files)
```

### Step 4 — Determine scope

[scope-selection.md](scope-selection.md). Default: all clauses. Narrow only on explicit signal.

### Step 5 — Pre-flight confirmation

```
Pack: iso-27001-2022 v1.0.0
Scope: all (9 clauses, 4 product policies to apply)
Will CREATE:  AITrustLayer, Development, Robot, StudioWeb
Will DEPLOY to: tenant DefaultTenant
Proceed? (y/n)
```

Require `y`. Anything else halts with no side effects.

### Step 6 — Synthesize

[synthesis-algorithm.md](synthesis-algorithm.md). Fast vs. subset path decided per policy file. Produces **override formData** (not a complete payload) — merged with product defaults in the CREATE recipe.

### Step 7 — Phase 1: CREATE

For each `applicable` policy file (sequential, fail-fast):

1. **Check for product-specific quirks** — look for `products/<normalized-product>.md` (e.g., AITL → `products/ai-trust-layer.md`). Normalize product name: lowercase, `PascalCase` → `kebab-case` (`AITrustLayer` → `ai-trust-layer`, `StudioWeb` → `studio-web`). Read the quirks file if it exists; otherwise proceed with the shared CREATE recipe only.
2. **Call the CREATE recipe** in [../../policy-crud.md](../../policy-crud.md) with `formData` = synthesized overrides.
3. **Collect** `{ policyId, status, warnings[] }`.

For each `skipped` (access) policy file:
- Append to `deploy-record.created[]` with `status: "skipped", reason: "access-policies-not-yet-supported"`.
- No CLI call.

Halt on any 4xx — remaining files become `status: "skipped", reason: "prior-failure"`.

### Step 8 — Phase 2: DEPLOY (bulk configure, one call per scope)

Skip if `--skip-deploy`. Otherwise:

1. **Group created policies by deployment scope.** Each policy has a target `(level, targetId)` derived from:
   - User prompt override ("apply to the Finance group") → that scope for all policies
   - Or `policy.deploymentLevel` in each pack file (typically `tenant` in V1 packs)
   - Default → tenant, using `UIPATH_TENANT_ID` + `UIPATH_TENANT_NAME` from `~/.uipath/.auth`

2. **For any group/user targets without a resolved `targetId`**, call [../../principals-lookup.md](../../principals-lookup.md) to fetch candidates and prompt for selection. Do this BEFORE Step 3 so each scope has a concrete GUID.

3. **For each distinct scope, issue ONE `deployment {tenant|group|user} configure` call.** This is the key change from the previous design:
   - The old flow called `assign-tenant` per policy (which was a full-replace API call) — sequential calls wiped prior assignments. **"Last deploy wins" bug.**
   - The new flow calls [../../policy-assign.md](../../policy-assign.md) ONCE per scope, passing the complete assignment array (all created policies for that scope, plus any existing assignments that should be preserved).

4. **Per-scope steps** (delegated to `policy-assign.md`):
   - Read current scope state (`deployment tenant get` etc.) so existing assignments for other `(product, licenseType)` slots aren't clobbered.
   - Merge: `currentAssignments ∪ newAssignments` with new entries winning on conflict.
   - Write the merged array to a temp JSON file.
   - Call `deployment tenant configure --input <file>` (or group/user variant).

Collect `{ status, scope, mergedAssignmentCount, warnings[] }` per scope. Halt on 4xx.

**Count discipline:** If a pack has 4 AITL + Robot + Studio + StudioWeb policies all targeting the same tenant, Phase 2 makes **1 API call**, not 4. If the pack splits across tenant + one group + one user scope, that's 3 calls (one per distinct scope).

### Step 9 — Write deploy record

[deploy-record.md](deploy-record.md). Single JSON with `created[]` + `deployed[]` arrays. Always write — success, partial failure, or halt.

### Step 10 — Report

```
Pack applied: iso-27001-2022 v1.0.0 → tenant DefaultTenant

Created (4):
  ✓ AITrustLayer  → iso-27001-2022-AITrustLayer (d0a68808-...)
  ✓ Development   → iso-27001-2022-Development  (f1b2c3d4-...)
  ✓ Robot         → iso-27001-2022-Robot        (7a8b9c0d-...)
  ✓ StudioWeb     → iso-27001-2022-StudioWeb    (e5f6a7b8-...)

Deployed (4, all tenant-level):
  ✓ iso-27001-2022-AITrustLayer
  ✓ iso-27001-2022-Development
  ✓ iso-27001-2022-Robot
  ✓ iso-27001-2022-StudioWeb

Deploy record: ./deploy-record-iso-27001-2022-<ts>.json
```

## Dispatch Table (within this capability)

| Phase | `policyKind` | `productIdentifier` | Action |
|---|---|---|---|
| Creation | `product` (or absent) | `AITrustLayer` | CREATE recipe + [products/ai-trust-layer.md](products/ai-trust-layer.md) |
| Creation | `product` (or absent) | `Assistant` | CREATE recipe + [products/assistant.md](products/assistant.md) |
| Creation | `product` (or absent) | `AssistantWeb` | CREATE recipe + [products/assistant-web.md](products/assistant-web.md) |
| Creation | `product` (or absent) | `Development` | CREATE recipe + [products/development.md](products/development.md) + shared [_studio-family.md](products/_studio-family.md) |
| Creation | `product` (or absent) | `Business` | CREATE recipe + [products/business.md](products/business.md) + shared [_studio-family.md](products/_studio-family.md) |
| Creation | `product` (or absent) | `Automate` | CREATE recipe + [products/automate.md](products/automate.md) + shared [_studio-family.md](products/_studio-family.md) |
| Creation | `product` (or absent) | `StudioPro` | CREATE recipe + [products/studio-pro.md](products/studio-pro.md) + shared [_studio-family.md](products/_studio-family.md) |
| Creation | `product` (or absent) | `Robot` / `StudioWeb` / `IntegrationService` | CREATE recipe only (no documented quirks — templates are minimal or not published) |
| Creation | `product` (or absent) | any unknown | CREATE recipe only; add a quirks file if you hit recurring errors |
| Creation | `access` | any | **SKIP** (access-policy CLI pending branch merge; record as `access-policies-not-yet-supported`) |
| Deployment | `product` (or absent) | any | [../../policy-assign.md](../../policy-assign.md) — tenant / group / user branch |

## Adding support for a new product's quirks

The shared CREATE recipe handles any AOPS product by default. Create a `products/<product>.md` file **only when you hit a real quirk worth documenting** — wrong-type CLI values, non-obvious enum constraints, conditional field dependencies, or recurring error patterns.

Template (follow [products/ai-trust-layer.md](products/ai-trust-layer.md) as the reference):

```markdown
# Compliance · <Product Label> — product quirks

Product-specific CLI value conventions and error patterns for `productIdentifier: "<Product>"`.
Follow the shared CREATE recipe in [../../../policy-crud.md](../../../policy-crud.md).

## Product identifiers
| Field | Value |
|---|---|
| `--product-name` | `<ExactProductIdentifier>` |
| Default license | `<NoLicense | Attended | Development | ...>` |

## CLI value quirks — will cause 400 if wrong
| Property | Expected type / values |
|---|---|
| ... | ... |

## Error triage
| Error message fragment | Likely cause | Action |
|---|---|---|
| ... | ... | ... |
```

Do NOT pre-create stub files for products without documented quirks. An empty file adds noise and no signal.

## Anti-Patterns

- **Never invent product quirks.** If the shared CREATE recipe works, no product file is needed.
- **Never prompt for scenario choice.** Derive from the user's prompt per [scope-selection.md](scope-selection.md).
- **Never run phases or policies in parallel.** Sequential = predictable fail-fast boundary.
- **Never auto-pick a group / user.** Always surface candidates for explicit selection.
- **Never hand-edit `formData`.** Pack is the source of truth; `policy-crud` handles the defaults merge.
- **Never silently drop a skipped policy.** Always record in `deploy-record.created[]` with a reason.
