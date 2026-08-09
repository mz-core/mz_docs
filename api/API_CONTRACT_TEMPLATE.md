---
title: MZ Framework — Template de contrato de API pública
status: GUIDE
authority: LEVEL_3_GUIDE
owner: MZ Documentation
last_reviewed: 2026-08-09
applies_to: [public_exports, events, callbacks]
related_contracts: [MZ-R005, MZ-R008, MZ-R010, MZ-R015, MZ-R049, MZ-R050]
---

# API contract template

```markdown
---
title: <resource> — <API name>
status: CANONICAL | REFERENCE | DEPRECATED
authority: LEVEL_1_NORMATIVE | LEVEL_4_REFERENCE
owner: <resource>
last_reviewed: YYYY-MM-DD
applies_to: [<resource>, <known consumers>]
related_contracts: [<rule IDs>, <ADR IDs>]
---

# <API name>

- ID: <stable ID>
- Name: <observed name>
- Resource: <owner resource>
- Type: export | event | callback
- Side: server | client | shared/network boundary
- Status/version: <status and version, or UNKNOWN>
- Signature: <verified signature, or UNKNOWN>

## Parameters

| Name | Type/schema | Required | Validation | Authority |
|---|---|---|---|---|

## Return

## Mutation
## Authorization
## Contextual checks
## Failure behavior
## Idempotency
## Audit
## Consumers
## Related rules
## Examples
```

Exemplo não substitui schema nem autoriza inferir assinatura. Payload client-side permanece não confiável.
