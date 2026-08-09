---
title: MZ Framework — Template de documentação de resource
status: GUIDE
authority: LEVEL_3_GUIDE
owner: MZ Documentation
last_reviewed: 2026-08-09
applies_to: [all_resource_docs]
related_contracts: [MZ-DOCS-v1, MZ-R011, MZ-R035-MZ-R040, MZ-R045, MZ-R049, MZ-R053]
---

# Resource document template

```markdown
---
title: <resource> — Resource documentation
status: CANONICAL | REFERENCE
authority: LEVEL_2_ARCHITECTURE | LEVEL_4_REFERENCE
owner: <resource/team>
last_reviewed: YYYY-MM-DD
applies_to: [<resource>, <consumers>]
related_contracts: [<rule IDs>, <ADR IDs>]
---

# <resource>

## Purpose
## Owns
## Does Not Own
## Dependencies
## Public API
## Events/Callbacks
## Database
## State
## Authorization
## Security
## Failure Behavior
## Invariants
## Integration
## Tests
## Related Rules
## Related ADRs
```

`Does Not Own` é obrigatório. Use `None verified` ou `UNKNOWN` quando a evidência for insuficiente; não invente conteúdo para completar a simetria.
