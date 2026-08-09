---
title: MZ Framework — AI context
status: CANONICAL
authority: LEVEL_3_GUIDE
owner: MZ Documentation
last_reviewed: 2026-08-09
applies_to: [coding_agents, review_agents]
related_contracts: [MZ-DOCS-v1, MZ-R011-MZ-R018, MZ-R035-MZ-R043, MZ-R053]
---

# AI context

MZ Framework é uma framework modular para um único runtime FiveM. Owners controlam invariantes e persistência; products/orchestrators integram owners por contratos públicos.

## Antes de trabalhar

1. Identifique task, resources e domínios afetados.
2. Leia regras canônicas aplicáveis e depois a arquitetura/documentação do owner.
3. Verifique código e testes atuais antes de afirmar comportamento runtime.
4. Diferencie `CURRENT`, `TARGET`, proposta e histórico.
5. Não invente API, event, table, signature, owner ou garantia.
6. Preserve owner e canonical writer; client payload não prova authority.
7. Preserve mudanças locais e o escopo autorizado.
8. Rotule evidência como static analysis, harness ou runtime.
9. Atualize documentação quando uma mudança autorizada alterar contrato.
10. Interrompa mutação quando a decisão necessária exceder a autoridade recebida.

Use [Resource Index](RESOURCE_INDEX.md), [Terminology](TERMINOLOGY.md) e [Context Routing](CONTEXT_ROUTING.md) para carregar somente o contexto necessário.
