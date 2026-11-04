---
title: MZ Framework — API documentation
status: REFERENCE
authority: LEVEL_4_REFERENCE
owner: MZ Documentation
last_reviewed: 2026-08-09
applies_to: [public_exports, events, callbacks, schemas]
related_contracts: [MZ-R005, MZ-R015, MZ-R031, MZ-R032, MZ-R044, MZ-R049, MZ-R050]
---

# API documentation

Os contratos atuais de incidentes, capabilities, session timeline e canários estão em [BanGuard e segurança orientada a incidentes](../security/BANGUARD_INCIDENT_SECURITY.md).

Os contratos vivos de Vehicles e Garage estão em [Vehicles e Garage domain extraction](../vehicles/DOMAIN_EXTRACTION.md).

O contrato client-side de opções contextuais está em [Target API](TARGET_API.md).

## Estado atual

O catálogo em `_architecture/mz_framework_v1/wave_0/catalog/apis.json` registra exports, events e callbacks observados no snapshot Wave 0. Ele é evidência histórica, não promessa automática de estabilidade nem catálogo vivo permanente.

Antes de integrar, confirme a assinatura e o side no código atual do owner. Ausência de evidência deve ser registrada como `UNKNOWN`; não complete parâmetros por inferência.

## Catálogo vivo futuro

O catálogo vivo deverá:

1. extrair surfaces dos manifests/código atuais;
2. combinar documentação do owner para authority, mutation, failure behavior e version;
3. validar IDs, consumers e references em CI;
4. publicar diff revisável quando a surface mudar;
5. preservar snapshots anteriores como histórico, sem sobrescrevê-los.

CI e gerador vivo não são implementados na Wave 1A.

Use o [template de contrato](API_CONTRACT_TEMPLATE.md) para documentar APIs públicas.
