# Policy CRUD — shared CLI recipes

Primitive operations on AOPS product policies. All three capabilities (Apply / Diagnose / Advise) call these recipes.

## Commands covered

| Operation | CLI |
|---|---|
| List | `uip admin aops-policy list [--product-name X] [--search Q] [--order-direction asc\|desc] --output json` |
| **Get (metadata + form data)** | `uip admin aops-policy get <policyIdentifier> --output json` — returns `{ name, identifier, description, priority, availability, product, data: {...} }` |
| Get tenant deployment map | `uip admin aops-policy deployment tenant get <tenantIdentifier> --output json` |
| Get template (schema + i18n) | `uip admin aops-policy template get <productIdentifier> --output json` |
| Get template defaults (form-data blueprint) | `uip admin aops-policy template get <productIdentifier> --output-form-data <path> --output json` |
| Get locale-resolved template reference | `uip admin aops-policy template get <productIdentifier> --output-template-locale-resource <path> --output json` |
| List license types | `uip admin aops-policy license-type list --output json` |
| Create | `uip admin aops-policy create ...` (recipe below) |
| Update | `uip admin aops-policy update ...` (recipe below) |
| Delete | `uip admin aops-policy delete <policyIdentifier> --output json` |

> **`policy get` is the single read path.** It combines `policyGetPolicyById` (metadata) with `policyGetFormDataByPolicyId` (values) into one response with the `data` field always populated. Use it for diagnosis, pre-update reads, and advise's current-state fetch.

All responses use `{ Result: "Success" | "Failure", Code, Data, Message }`. Parse accordingly.

---

## CREATE recipe

Used by **Apply** (creating policies from a pack) and **Advise** (when no policy is deployed for a target product).

### Input contract

```jsonc
{
  "policyName":           "<deterministic or user-supplied>",
  "productIdentifier":    "<AITrustLayer | Robot | ...>",
  "description":          "<pass-through>",
  "priority":             1,
  "availability":         365,
  "licenseTypeIdentifier": "NoLicense",
  "formData":             { /* overrides — NOT a complete payload */ }
}
```

### Steps

1. **Fetch defaults** — the pack / requirement provides only overrides, not a complete template payload.
   ```bash
   tmpDir="$(mktemp -d)"
   uip admin aops-policy template get "<productIdentifier>" \
     --output-form-data "$tmpDir/defaults.json" \
     --output json
   ```
   The `--output-form-data` flag writes the fillable blueprint (form-data skeleton with default values). This replaces the retired `template generate-data` subcommand.

2. **Deep-merge overrides onto defaults.** Pack / requirement values win on conflict; defaults fill every field the caller doesn't touch. Arrays replace wholesale; `null` in overrides means "explicitly clear".

3. **Write merged data to a temp file** (bare object, not wrapped in `{ "data": ... }`):
   ```bash
   dataFile="$tmpDir/formData.json"
   printf '%s' '<merged-formData-json>' > "$dataFile"
   ```

4. **Create:**
   ```bash
   uip admin aops-policy create \
     --name "<policyName>" \
     --product-name "<productIdentifier>" \
     --data-file "$dataFile" \
     --description "<description>" \
     --priority <priority> \
     --availability <availability> \
     --output json
   ```
   Parse `Data.identifier` → `policyId`.

### Error map

| HTTP | Action |
|---|---|
| `400` | Halt. Surface error — usually names the offending leaf. |
| `401 / 403` | Halt. Ask user to `uip login`. |
| `409` | Halt. Duplicate policy name — do NOT fall back to `update` (Apply contract). |
| `404` | Likely a missing / unreadable `--data-file` path. See [cli-known-issues.md](cli-known-issues.md). |
| `5xx` | Retry once after 3s. Halt on second failure. |

---

## UPDATE recipe

Used by **Diagnose** (applying approved fixes) and **Advise** (when a policy is already deployed for a target product).

### Input contract

```jsonc
{
  "policyIdentifier":    "<existing policy GUID>",
  "policyName":          "<current name>",
  "productIdentifier":   "<current product>",
  "description":         "<current description>",   // REQUIRED — see warning below
  "priority":            1,                          // REQUIRED — see warning below
  "availability":        365,                        // REQUIRED — see warning below
  "changes": [
    { "path": "allowed-llm-regions.europe", "to": true },
    { "path": "azure-openai-control-toggle", "to": true }
  ]
}
```

> **⚠ All metadata flags are required on UPDATE.** The CLI accepts `--description`, `--priority`, `--availability` as "optional" in `--help`, but the API returns **500** if any are omitted (see [cli-known-issues.md #2](cli-known-issues.md)). Always read the current values via `uip admin aops-policy get <id>` and pass them through unchanged unless the caller explicitly wants to update them.

### Steps

1. **Read the full current policy** — metadata + form data in one call:
   ```bash
   uip admin aops-policy get "<policyIdentifier>" --output json
   # Capture: Data.name, Data.product.name, Data.description, Data.priority, Data.availability, Data.data
   ```
   `Data.data` holds the full form-data object. The CLI fetches metadata (`policyGetPolicyById`) and form data (`policyGetFormDataByPolicyId`) in parallel and merges them.

2. **Apply each approved change** onto the full formData object from `Data.data` (not a patch — always send the complete object back).

3. **Write merged formData to a temp file.** Same format rules as CREATE (bare object).

4. **Update — pass ALL metadata flags, not just the ones you're changing:**
   ```bash
   uip admin aops-policy update \
     --policy-identifier "<policyIdentifier>" \
     --name "<policyName>" \
     --product-name "<productIdentifier>" \
     --description "<description>" \
     --priority <priority> \
     --availability <availability> \
     --data-file "$dataFile" \
     --output json
   ```
   Even if you're only patching `formData`, pass through `--description`, `--priority`, and `--availability` from Step 1. Omitting any returns 500.

### Error map

| HTTP | Action |
|---|---|
| `400` | Halt. Likely schema mismatch. Surface the error. |
| `401 / 403` | Halt. Permission — user may not have update rights. |
| `404` | Target identifier not found. (Missing `--data-file` paths now produce a clear filesystem error, not 404.) |
| `409` | Concurrent modification — another admin updated the policy. Tell user to retry. |
| `500` | Most often: missing `--description` / `--priority` / `--availability` flag. See [cli-known-issues.md #2](cli-known-issues.md). Re-run with all metadata flags from Step 1. |

---

## Read-full-policy recipe

Used by **Diagnose** and **Advise** to read the live configuration of any deployed policy:

```bash
uip admin aops-policy get <policyIdentifier> --output json
```

The response includes both metadata and the full `data` (formData) in a single call. `Data.data` is the unwrapped formData object with every field and value — ready for correlation (diagnosis), plan construction (advise), or read-modify-write (update). No separate `form-data get` step is needed.

---

## TENANT-GET recipe

Returns the full deployment landscape for a tenant — every `(product, license)` slot and whether a custom policy is deployed.

```bash
uip admin aops-policy deployment tenant get "$UIPATH_TENANT_ID" --output json
```

Response shape:
```jsonc
{
  "Data": {
    "name": "DefaultTenant",
    "identifier": "<guid>",
    "tenantPolicies": [
      { "productIdentifier": "AITrustLayer", "licenseTypeIdentifier": "NoLicense",
        "policyIdentifier": "<guid>" },                  // custom policy deployed
      { "productIdentifier": "Robot", "licenseTypeIdentifier": "Attended",
        "policyIdentifier": null }                        // using global default
    ]
  }
}
```

**Filter to `policyIdentifier != null`** to get only custom-deployed policies. Used by Diagnose (to know what to inspect) and Advise (to know CREATE vs. UPDATE path).

## Product-specific quirks

Per-product CLI value conventions (like AITL's `"yes"/"no"` strings) live in `plugins/compliance/products/<product>.md`. The CRUD recipe itself is uniform across products.
