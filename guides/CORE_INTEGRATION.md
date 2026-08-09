---
title: mz_core — Guia de integração
status: GUIDE
authority: LEVEL_3_GUIDE
owner: mz_core
last_reviewed: 2026-08-09
applies_to: [core_consumers]
related_contracts: [MZ-CORE-BOUNDARY-v1, MZ-R001-MZ-R010, MZ-R015, MZ-R049, MZ-R050]
---

# Integração com `mz_core`

1. Declare apenas dependências realmente usadas.
2. Aguarde readiness do core antes de chamar contratos que dependem de banco/bootstrap.
3. Resolva player/session por API server-side; não trate `source` como ID persistente.
4. Para autorização, peça capability global ao core e aplique checks contextuais no owner.
5. Use API pública do owner; não leia tabelas, repositories ou cache interno.
6. Valide payloads no boundary e defina failure behavior observável.
7. Considere reconnect, unload, source reuse, retry e operação duplicada.
8. Documente caller, actor, mutation, authorization, idempotency, audit e consumers.
9. Adicione harnesses próximos; rotule separadamente evidência estática e runtime.

O nome/signature exatos de exports, events e callbacks devem ser confirmados no código e no catálogo aplicável. Este guia não inventa uma API agregada.
