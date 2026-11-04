---
title: MZ BanGuard — segurança orientada a incidentes
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: MZ Security
last_reviewed: 2026-10-11
applies_to: [mz_banguard, mz_inventory, mz_admin, mz_core, security_incidents]
---

# MZ BanGuard — segurança orientada a incidentes

## Ownership

`mz_banguard` possui detecção, correlação, incidentes, evidência, bans e mitigação de eventos Cfx. Ele não possui identidade/sessão/player state (`mz_core`), inventário/arma equipada (`mz_inventory`) nem autorização administrativa (`mz_admin`). Não existe writer SQL cross-domain.

## Evidência

- `confidence` mede confiança na classificação técnica; `culpability` mede suporte a conduta maliciosa (`none`, `contextual`, `suspected`, `supported`).
- `crash_observed/high` confirma a classificação do drop como crash, mas permanece `culpability=none`.
- O agregado expõe `occurrenceCount`, `retainedSampleCount`, `droppedSampleCount` e `suppressedSampleCount`.
- `mz_security_signals` preserva sinais sanitizados por `session_id`; `mz_security_incidents` mantém o agregado.
- Linking cross-session usa account/citizen e janela temporal; não afirma mesma assinatura, causa ou atacante.

## Contratos públicos

| Contrato | Caller | Resultado |
|---|---|---|
| `mz_inventory:GetEquippedWeaponState(source)` | server | snapshot sanitizado da arma equipada canônica |
| `mz_inventory:IsWeaponAuthorized(source, hash)` | server | decisão da autoridade de inventário |
| `mz_banguard:RegisterSecurityLease(...)` | resources allowlisted | capability curta, contextual e consumível |
| `mz_banguard:GetSecuritySessionTimeline(actor, session, limit)` | `mz_admin` autorizado | sinais/incidentes sanitizados em ordem temporal |
| `mz_banguard:RecordSecurityCanary(...)` | owner configurado | registro de canário; nenhum efeito de negócio |

Heartbeat, `onClientResourceStop` e presença de resource são telemetria falsificável. Nunca equipam arma, concedem capability, alteram state ou autorizam ação.

## Capabilities e decisão

O detector resolve lifecycle/capability/contexto antes de criar incidente e calcular risk. Noclip administrativo usa autorização server-side, audit e `movement.noclip.staff`; não usa bypass permanente por cargo. State machine de morte vem de `mz_core`; HP client isolado não é prova.

## Modos de atuação (Wave 2A)

| Modo | Efeito |
|---|---|
| `passive` | registra/correlaciona; não chama mitigação nem pune |
| `mitigate` | herda registro e chama somente capability server-side suportada |
| `kick` | herda mitigação; exige chave global, evidência persistida e identidade revalidada |
| `ban` | herda mitigação; persiste no ban local existente antes de desconectar |

Defaults DEV: `DefaultMode=passive`, `AutomaticKickEnabled=false`, `AutomaticBanEnabled=false`, `cancelHighConfidence=false` e `runtimeValidated=false`. Os modos legados dos guards continuam descrevendo readiness técnico; a decisão de atuação pertence ao motor unificado.

O motor associa execução a `incident_id`, Account, Session e Character público; source nunca é identidade persistente. Kick/ban revalidam `account_id`, `session_id`, `citizenid` e identifier estável imediatamente antes da atuação. Solicitação persistida é idempotency key at-most-once. Heartbeat/resource presence, crash isolado, telemetria client-side, godmode/invisibility/câmera/aim e canários não são elegíveis a punição automática isolada.

Floods são agregados por sessão/detector, com burst inicial, amostras representativas, contagem de descartes, fila limitada e backpressure. Sinais que já originam incidente não são duplicados pela correlação SQL por padrão.

## Cfx, canários e operação

`weaponDamageEvent` e `entityCreating` possuem caminho de cancelamento documentado. `ptFxEvent` e `startProjectileEvent` permanecem detect/correlate até prova no artifact. O repositório não ativa globalmente `sv_filterRequestControl` nem `sv_entityLockdown`.

O canário fica no owner, é marcado `MZ_SECURITY_CANARY`, não realiza mutação e é desligável. A ativação prova chamada ao sensor, não identifica automaticamente ferramenta específica e nunca gera auto-ban isoladamente.

Instalação nova usa `mz_banguard/sql/install_mysql.sql`; instalação 1.5 aplica manualmente `upgrade_1_6_incident_v3_mysql.sql`. Harness local não aprova comportamento runtime/FiveM.
