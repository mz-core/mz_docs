# MZ BanGuard V3 — relatório de prontidão pré-implementação

Data da análise: 2026-10-07. Escopo: dossiê `ref/banguard_v3`, código atual de `mz_banguard`, `mz_inventory`, `mz_admin`, `mz_core`, configuração versionada do servidor e documentação oficial Cfx.

## 1. Arquitetura atual real

- `mz_core` é a autoridade de identidade, sessão, permissões, lifecycle do jogador, state machine `alive/downed/dead/respawning` e audit detalhado.
- `mz_inventory` é o owner e único writer do inventário e da arma equipada. Mantém `EquippedWeaponsBySource` server-side e publica `GetEquippedWeaponState`/`IsWeaponAuthorized` como contratos server-only sanitizados.
- `mz_banguard` observa hooks Cfx e estado server-side, resolve leases/lifecycle antes dos detectores aplicáveis, agrega incidentes, persiste incidentes/açōes/LAB e expõe a Central para `mz_admin`.
- `mz_admin` autoriza a ação administrativa no servidor e emite leases curtos para teleporte/spawn de veículo. Estado permanente de staff não é fronteira de autorização do detector.
- `mz_banguard` client produz somente telemetria complementar e heartbeat falsificável; não é autoridade.

## 2. Autoridade da arma equipada

A autoridade já está corretamente em `mz_inventory/server/service.lua`. `mz_banguard/server/weapon_guard.lua` consulta o export `GetEquippedWeaponState`; não lê storage, não cria arma/item e não replica ownership.

## 3. Efeito da parada do client de `mz_inventory`

- Param de funcionar: UX/equip visual, aplicação client-side de arma/munição e pedidos originados pelo client.
- Não param: owner server-side, contrato `GetEquippedWeaponState`, `weaponDamageEvent` e comparação de hash no BanGuard.
- Gap real: o servidor não possui hoje presença específica do runtime client de `mz_inventory`; estado equipado pode continuar válido até uma transição server-side, e a ausência do client não fica registrada para correlação. Presença será apenas contexto, nunca autorização.

## 4. Efeito da parada do client de `mz_banguard`

Param heartbeat e sinais client-side de noclip/freecam/godmode/invisibilidade. Permanecem ativos: `weaponDamageEvent`, entity/event guards, sampling de posição server-side, autoridade do inventário, leases, state machine, incidentes, bans e audit. Logo, a segurança principal já não depende do client, mas perde cobertura/telemetria complementar.

## 5. Hooks Cfx e cancelabilidade

- `weaponDamageEvent`: disponível e oficialmente cancelável.
- `entityCreating`: disponível e oficialmente cancelável; cancelamento remove a entidade durante criação.
- `entityCreated`: observação posterior, não substitui bloqueio em `entityCreating`.
- `explosionEvent`: exemplo oficial demonstra interceptação/cancelamento, mas esta entrega não ampliará bloqueio sem fixture de compatibilidade.
- `ptFxEvent` e `startProjectileEvent`: disponíveis, porém a documentação oficial atual não promete cancelabilidade confiável; permanecem detect/correlate até validação no artifact.
- `onClientResourceStop`: disponível, mas é client-side e manipulável; serve somente como telemetria.

Fontes oficiais: [Server events](https://docs.fivem.net/docs/scripting-reference/events/server-events/), [OneSync](https://docs.fivem.net/docs/scripting-reference/onesync/), [onClientResourceStop](https://docs.fivem.net/docs/scripting-reference/events/list/onClientResourceStop/), [Server security](https://docs.fivem.net/docs/developers/server-security/).

## 6. `sv_filterRequestControl` e lockdown

A configuração versionada usa OneSync e game build 3751. Não há `sv_filterRequestControl`, `sv_filterRequestControlSettleTimer` nem `sv_entityLockdown` ativos nos cfgs versionados; portanto, sem override externo, valem os defaults documentados (`sv_filterRequestControl=0`, lockdown inativo). Não há evidência suficiente para promover modo 4 ou lockdown global. Modos 1/2/3 e bucket `strict` exigem staging T22/T23 por resource. Fonte: [server commands](https://docs.fivem.net/docs/server-manual/server-commands/).

## 7. Gaps comprovados de persistência/evidência

- Incidentes agregados, actions, LAB e snapshots já persistem.
- Cada ocorrência ainda não possui uma linha raw imutável própria; deduplicação pode perder detalhes.
- Sampling retém até N evidências, mas não explicita retained/dropped além de `count` e `suppressedCount`.
- Linking cross-session por identidade canônica não existe.
- Crash captura o buffer do BanGuard, mas não integra audit de lifecycle/state e pode resultar em timeline vazia.
- Consulta forense por `session_id` ainda não existe.

## 8. Batches pequenos e reversíveis

1. Evidência: raw signals, contadores explícitos, semântica confidence/culpability e linking por identidade.
2. Contexto: presença de resources como telemetria, death state canônico e tratamento explícito unarmed/melee.
3. Mitigação cercada: cancelamento apenas em hooks oficialmente canceláveis, por modo + flag + validação runtime; LAB nunca bloqueia.
4. Forense: timeline de sessão construída de sinais/incidentes/actions, sem importar audit geral como risco.
5. Canário mínimo DEV no owner do domínio, inerte e coberto por teste anti-caller legítimo.

## 9. Riscos de falso positivo/regressão

- Unarmed/melee legítimo pode hoje virar `weapon_damage_no_equipped_weapon` (incidente real com hash `WEAPON_UNARMED`).
- Lockdown/request-control global pode quebrar veículos, NPCs, housing, jobs e admin.
- Cancelar por rate isolado ou por heartbeat ausente confundiria falha/runtime legítimo com abuso.
- Audit administrativo/lifecycle é contexto; convertê-lo automaticamente em score criaria culpabilidade artificial.
- Atribuição de owner/actor em hooks OneSync pode mudar ou faltar; correlação não equivale a causalidade.

## 10. Resources/arquivos previstos

- `mz_banguard`: config, DB/schema, incidentes/correlação, detector health, weapon/entity guards, API/status, testes e docs.
- `mz_inventory`: somente telemetria de presença e canário inerte junto ao owner, se a fixture provar o contrato.
- `mz_admin`: somente se a trilha real de noclip não emitir lease; não há motivo comprovado para mudar o fluxo atual de teleporte.
- `mz_core`: sem troca de ownership; apenas consumo dos contratos públicos existentes.
- `mz_docs`: threat model, ownership, API/capabilities, evidência/incidentes e modos.

## 11. Confidence versus culpabilidade

Hoje `confidence` descreve confiança do detector/classificação, mas a UI não explicita separadamente culpabilidade. Exemplo: `crash_observed/high` significa alta confiança de que o drop parece crash; não implica cheat. A implementação deve persistir/expor `culpability=none|contextual|suspected|supported` separadamente, com crash isolado em `none`.

## 12. Linking atual/cross-session

Há `linkedIncidentIds` no modelo e o crash cluster liga apenas o crash corrente. Não existe busca de incidentes anteriores do mesmo account/citizen em outra sessão. O gap é real; o vínculo deve ser por identidade canônica + janela configurada e nunca unificar causas automaticamente.

## 13. Audit disponível para timeline retroativa

`mz_core` registra `death_transition`, eventos de estado e stale session em seu log detalhado. O BanGuard não possui hoje um contrato público de query forense irrestrita adequado para incorporar tudo. Para evitar dependência SQL cross-domain, esta entrega correlacionará os eventos próprios persistidos e registrará referências estáveis; integração de audit do core só deve usar export público autorizado.

## 14. Ordem detector → contexto/lease → risk

Nos fluxos revisados de movimento, lifecycle/lease/admin state são resolvidos antes de `Incidents.Record`; entidade e eventos remotos também consomem lease antes de gerar incidente. O risk é calculado dentro de `Incidents.Record`. Esse desenho correto será preservado. O noclip administrativo observado indica ausência/expiração de contexto naquela execução, não justifica bypass permanente por staff.

## 15. Contrato de morte

`mz_core` impõe `alive -> downed|dead`, `downed -> dead|alive`, `dead -> respawning`, `respawning -> alive|dead`; `actionsBlocked` é derivado de estado diferente de `alive`. HP client isolado não é autoridade. O BanGuard pode anexar snapshot canônico como contexto, sem assumir ownership nem corrigir estado.

## 16. Unarmed/melee atual

O Weapon Guard exige arma equipada para qualquer `weaponDamageEvent` não nulo e não possui exceção canônica para `WEAPON_UNARMED`/melee. Isso explica o falso positivo de hash 2725352035. A correção mínima é classificar explicitamente unarmed/melee fora do contrato de item equipado e continuar observando rate/modifiers por detectores próprios.

## 17. Evidence sampling atual

O incidente incrementa `count`; dentro do cooldown incrementa `suppressedCount`; fora dele mantém FIFO de até `maxEvidencePerIncident`. Não expõe `retainedSampleCount` nem separa amostras descartadas pelo limite das suprimidas por cooldown. Esse accounting será tornado explícito sem duplicar todas as ocorrências dentro do agregado.

## 18. Canários mínimos sem violar ownership

Um único canário DEV no `mz_inventory`, marcado `MZ_SECURITY_CANARY`, sem mutação e sem retorno útil. O owner detecta a chamada; o BanGuard apenas recebe sinal sanitizado via contrato server-only, registra/correlaciona em observe e mantém score/punição desligados por padrão. Teste estático proíbe callers legítimos. Nenhum nome oculto será tratado como segurança.

## Decisão pré-código

Preservar contratos corretos e implementar somente gaps provados. Não alterar convars globais, não criar autoridade paralela, não fazer auto-ban por sinal único e não declarar validação runtime sem execução no FXServer de staging.
