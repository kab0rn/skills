# CLI Known Issues

Live bugs in `@uipath/aops-governance-tool` that affect this skill.

## ~~1. Missing `--data-file` path returns misleading 404~~ — FIXED

The CLI now validates `--data-file` before the API call and surfaces `"Data file not found: <resolved path>"`. Verified against the updated commit.

## 2. Omitting metadata flags returns 500 — affects both `create` AND `update`

**Affected commands:**
- `uip admin aops-policy create --name X --product-name AITrustLayer --data-file ./valid.json` (no metadata flags)
- `uip admin aops-policy update --policy-identifier <id> --name X --product-name X --data-file ./valid.json` (no metadata flags)

**Expected:** 400 with validation message listing the missing fields, or the CLI should apply defaults / read-modify-write the existing values for `update`.

**Actual:** `500 Internal Server Error` — the API chokes on null metadata fields.

**CLI flags treat them as optional in `--help`** but the API requires all three: `--description`, `--priority`, `--availability`.

**Impact on skill:**
- **CREATE path:** The creation plugin always passes all flags because the pack provides them — usually safe.
- **UPDATE path:** Particularly nasty. If the caller is patching only `formData` (e.g., diagnosis fix), it's natural to omit metadata flags — but doing so destroys the policy with a 500. Workaround documented in [policy-crud.md UPDATE recipe](policy-crud.md#update-recipe): always read current metadata via `aops-policy get` and pass it through unchanged.

**Validated:** This bit us during a real diagnosis-mode update — first call returned 500, retry with full metadata succeeded.

**Fix:** Either the CLI should supply sensible defaults / read existing values on `update`, or the API should return 400 with specific field names. Marking the flags as `requiredOption` in commander would also surface the issue at validation time.
