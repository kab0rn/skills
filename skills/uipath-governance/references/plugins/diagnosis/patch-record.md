# Patch Record

Audit artifact written when the diagnosis mode updates a deployed policy. Captures what changed, why, and who approved it.

## Schema

```jsonc
{
  "fileKind": "compliance-patch-record",
  "schemaVersion": "1.0.0",

  "timestamp": "2026-04-17T14:30:00Z",
  "patchedBy": "nishank.siddharth@uipath.com",
  "organization": "procodeapps",
  "tenant": {
    "identifier": "<guid>",
    "name": "DefaultTenant"
  },

  "trigger": {
    "kind": "governance-error",
    "errorText": "User is forbidden by governance policy to use: gpt-4o-2024-11-20 in EU",
    "source": "user-reported"
  },

  "policy": {
    "identifier": "<guid>",
    "name": "iso-42001-ai-trust-layer",
    "productIdentifier": "AITrustLayer",
    "licenseTypeIdentifier": "NoLicense"
  },

  "changes": [
    {
      "path": "allowed-llm-regions.europe",
      "label": "Europe",
      "from": false,
      "to": true,
      "confidence": "HIGH",
      "reason": "Error text 'in EU' directly matches region path"
    },
    {
      "path": "azure-openai-control-toggle",
      "label": "Open AI",
      "labelDescription": "Disabling models will impact genAI features across products.",
      "from": false,
      "to": true,
      "confidence": "MEDIUM",
      "reason": "gpt-4o-2024-11-20 is an Azure OpenAI model; provider was disabled"
    }
  ],

  "approval": "user-confirmed",
  "status": "success"
}
```

## Write rules

1. Write after a successful `update` call.
2. Default path: `./patch-record-<policyName>-<timestamp>.json`.
3. On update failure, still write the record with `"status": "failed"` and an `"error"` field.
4. Do NOT commit to git — contains tenant identifiers.
5. **Resolve `label` (and optional `labelDescription`)** for every change via [../../property-labels.md](../../property-labels.md). If no label exists, omit the field — the `path` is always authoritative. Storing the label snapshot makes the patch record readable even years later when the i18n bundle has drifted.

## Relationship to deploy records

Deploy records (`compliance-deploy-record`) capture pack application. Patch records capture post-deployment fixes. Together they form the full audit trail:
- Deploy record says: "these policies were created from this pack"
- Patch record says: "this policy was later modified because of this error"

An auditor reads both to understand the current policy state vs. the original compliance baseline.
