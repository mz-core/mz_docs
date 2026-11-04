# P10 — Algemas, chaves e restrições (mz_police)

Status: implementação local concluída; **sem validação in-game**. Nenhum SQL foi executado, nenhum recurso foi reiniciado e nenhum commit/push/deploy foi feito.

## 1. Diagnóstico do estado anterior

Antes desta entrega, `mz_police` já possuía o owner canônico da custódia:

- `mz_police/server/custody.lua`: estado em memória por `targetSource`, revisão otimista, `requestId` idempotente, lock por alvo, `cuffed`, `cuffedBy`, escolta, veículo e sweep.
- `mz_police/server/policy.lua`: autorização por organização/duty/capability (`police.custody`).
- `mz_police/client/target.lua`: registro de ações policiais em `mz_target` para PLAYER.
- `mz_police/client/custody.lua`: pose/algemas visuais, escolta e bloqueio parcial de controles.
- `mz_police/server/audit.lua`: logs canônicos via `mz_core` (`CreateDetailedLog`).
- `mz_inventory`: itens únicos com `instance_uid` e `serial`, mutações transacionais (`executeInventoryMutation`), mas **não existia contrato para consumir/restaurar uma instância exata** por UID.
- `mz_target`: `AddGlobalPlayer` / `RemoveGlobalPlayer` com `entity` e `target.player_server_id`.
- `mz_animations`: expõe `Play/PlayByName/Stop` de emotes, não um adaptador de cena sincronizada de algema.
- `mz_prison`: owner de sentença/intake/confinamento; `mz_police` já chama `RequestSentence` e limpa custódia no intake.
- `mz_core`: owner global de Staff (`HasGlobalPermission`, `CanStaffActOnPlayer`, `GetStaffContext`).

Conclusões:

- Não foi criado segundo estado de custódia. O estado algemado continua em `MZPolice.Custody`; o P10 adiciona metadados de restrição (`origin`, `itemUid`, `serial`, `keyUid`) ao mesmo estado.
- A fase `APPLYING` é apenas um lock de operação (`pending`), não um estado paralelo de custódia.
- `mz_radialmenu` não foi alterado.
- `mz_animations` não foi alterado: não existe contrato de cena sincronizada de algema; foi usado `TaskPlayAnim` no client, como o `client/custody.lua` já fazia.

## 2. Decisões arquiteturais

1. **Owner único**: `MZPolice.Custody` continua sendo o único estado canônico de restrição.
2. **Orquestração**: novo `mz_police/server/restraints/service.lua` valida intenção, serial, item, Staff, distância, bucket, prisão e animação; delega a transição para `Custody.requestAuthorized`.
3. **Itens exatos**: `mz_inventory` ganhou dois exports server-only, restritos a `mz_police`:
   - `ConsumeRestraintItem(source, itemName, instanceUid, operationId)`
   - `GrantRestraintItem(source, itemName, metadata, operationId)`
   Ambos usam a transação/lock canônicos, allowlist `handcuffs`/`handcuff_key`, UID explícito e cache terminal por `operationId`.
4. **Serial**: `handcuffs` e `handcuff_key` são `unique = true`, `stack = false`, `generateSerial = true`. O UID nasce no inventário; a NUI não escolhe UID.
5. **Chave**: somente uma chave por par físico, criada no commit da aplicação. O vínculo é `metadata.handcuff_uid`, `metadata.cuff_serial` e `metadata.restraint_id`, todos gerados no servidor.
6. **Staff**: permissão concreta `staff.restraints.manage` adicionada ao catálogo canônico do `mz_core` e ao seed do cargo `administrador`. A autorização continua em `mz_core`; não há ACE ampla.
7. **Persistência**: opcional, desativada por padrão. Sem SQL aplicado, o estado físico é efêmero e é liberado com segurança em disconnect/restart; com SQL + flag, sobrevive a relog/restart.
8. **Prisão**: algemar não cria sentença; desalgemar não cancela sentença. Alvo com sentença ativa é negado (`target_in_prison`).

## 3. Fluxos implementados

### Policial

- `/algemar [id]` e `/desalgemar [id]`: comandos server-side, exigem `police.custody` + duty.
- Target: as ações existentes `target.cuff` / `target.uncuff` continuam no `mz_target` e agora passam pelo orquestrador.
- `policeServiceCuffsWithoutItem=true`: policial em serviço não precisa de item e **não** gera item/chave.
- `policeServiceCuffsWithoutItem=false`: policial precisa usar o caminho físico (item).
- `policeServiceUnlock=true`: qualquer policial autorizado em serviço pode remover; `false`: apenas o responsável pela custódia.
- Sempre valida no servidor: identidade/sessão, duty, capability, alvo online, distância, bucket, estado de morte, prisão e revisão.
- Autoalgemar de policial é negado.

### Civil

- Item `handcuffs` no inventário; ação `Algemar (algemas)` no `mz_target`.
- O servidor escolhe a instância (menor slot) e valida UID + serial; o cliente não envia UID.
- `civiliansRequireSurrenderedTarget=true`: alvo precisa estar `downed` no estado canônico (`mz_core`/`mz_medical`), nunca correndo.
- No commit: consome 1 par exato, confirma a custódia e emite **uma** `handcuff_key` para o autor.
- A chave só abre a algema correspondente (`handcuff_uid` + `restraint_id` + `keyUid`).
- Desalgemar por chave consome a chave e restaura o par com o mesmo UID. Se a entrega não couber, o par entra em fila de restituição e é auditado; não há spawn no chão.

### Staff

- `/staffalgemar [id] [motivo]` e `/staffdesalgemar [id] [motivo]`.
- Exige `staff.restraints.manage` + `CanStaffActOnPlayer` (hierarquia).
- Não depende de emprego/item e não cria itens negociáveis.
- Auditado como `police.restraint.applied/removed` com `staffOverride=true`, ator, alvo, motivo, origem, comando (quando aplicável) e timestamp.
- Autoalgemar de Staff é permitido (override explícito) e auditado.

## 4. Configuração

Em `mz_police/config/shared.lua` (`MZPoliceConfig.Restraints`, lido pelo cliente e pelo servidor):

- `enabled`, `capability`, `staffPermission`
- `enablePoliceCommand`, `enableStaffCommand`
- `policeServiceCuffsWithoutItem`, `policeServiceUnlock`
- `civilianPhysicalCuffs`, `civiliansRequireSurrenderedTarget`, `allowFrontCuff` (futuro)
- `maximumDistance`, `animationDurationMs`, `removalDurationMs`, `applyTimeoutMs`
- `restrictVehicleEntry`, `blockWeaponUse`, `blockAttack`, `inventoryWhileCuffed`, `allowWalk`
- `logStaffOverrides`
- `items.handcuffs`, `items.key`
- `persistence.enabled` (padrão `false`), `persistence.table`, `retryIntervalMs`

`logStaffOverrides=true` dispara o evento interno `mz_police:server:restraintStaffOverride` para integração de auditoria; a auditoria canônica em `mz_core` é gravada sempre. Nenhum ban é automático.

## 5. Contratos públicos alterados

- `mz_inventory` (server-only):
  - `ConsumeRestraintItem`
  - `GrantRestraintItem`
- `mz_inventory` state bag consumido: `mz:restrainedInventoryBlocked` (projeção escrita por `mz_police`).
- `mz_core`:
  - nova permissão de catálogo `staff.restraints.manage`;
  - seed do cargo `administrador` inclui a permissão.
- `mz_police`:
  - eventos `mz_police:server:restraints:request`, `mz_police:server:restraints:refresh`;
  - eventos client `mz_police:client:restraints:*`;
  - `MZPolice.Restraints` (server) e `MZPoliceClient.Restraints` (client).

Nenhum contrato existente foi removido. `mz_radialmenu` permanece intacto.

## 6. SQL e rollout (NÃO EXECUTADO)

Arquivo: `mz_police/sql/restraints.sql` (CREATE-only, idempotente).

Ativação:

1. Revisar o SQL em staging.
2. Aplicar manualmente a tabela `mz_police_restraints`.
3. Definir `MZPoliceConfig.Restraints.persistence.enabled = true`.
4. Reiniciar **apenas o servidor de desenvolvimento** (não produção) para carregar a configuração.
5. Validar `mz_police_restraints` e os logs `police.restraint.*`.

Rollback:

1. Definir `persistence.enabled = false`.
2. Reiniciar o servidor de desenvolvimento.
3. As restrições físicas ativas em memória são liberadas com segurança e os itens são restituídos/auditados.
4. A tabela pode ser mantida (sem escritas) ou removida manualmente após aprovação. Não há `DROP` automatizado.

Limitações:

- Sem persistência, disconnect/restart libera a restrição (não prende ninguém para sempre) e restitui o par ao autor ou registra pendência.
- Com persistência, o alvo é restaurado no `playerLoaded`; o autor pode estar offline, mas a chave física continua válida para o portador.
- A fila de restituição por inventário lotado é em memória; em restart sem persistência ela é perdida, mas o UID fica no log de auditoria para recuperação manual.

## 7. Testes executados

> Os harnesses foram removidos da distribuição do resource. Os resultados abaixo são
> históricos e só podem ser reproduzidos a partir da cópia externa de QA/Git.

Executados com `lua.exe` local, em `mz_police`:

- `tests/p10_restraints_harness.lua`: **PASS — 21 casos, 45 auditorias**
- `tests/p10_restraints_repository_harness.lua`: **PASS**
- `tests/p10_static_harness.lua`: **PASS**
- `tests/p10_target_civil_harness.lua`: **PASS**
- Todos os 24 harnesses anteriores de `mz_police`: **PASS** (28/28 no total, incluindo os P10)

Executados em `mz_inventory`:

- `tests/inventory_restraint_access_harness.lua`: **PASS**
- 11 harnesses anteriores: **PASS**
- `tests/weapon_ammo_rate_limit_contract_harness.lua`: **PASS** quando executado de `mz_inventory/tests` (cwd esperado pelo próprio harness).
- Total: **12/12**.

Graphify (escopo local): `graphify update mz_police`, `graphify update mz_inventory` e `graphify update mz_core` reextraíram os grafos; os avisos de labels de comunidade ficaram pendentes (`graphify label`). O grafo raiz não foi reconstruído (não é checkpoint global).

Não executados:

- Runtime FiveM (2 clientes), CEF/NUI e banco MySQL real.
- `npm`/lint/build: `mz_police` não possui `package.json`; os testes JS existentes não foram executados nesta rodada.
- Graphify: não foi executado update (mudança local de recursos; sincronização pendente).

## 8. Roteiro manual para 2 clientes FiveM

Ambiente: servidor de desenvolvimento, `mz_core`, `mz_inventory`, `mz_target`, `mz_prison`, `mz_animations`, `mz_menu` iniciados.

### Policial x civil

1. Cliente A entra em duty no ponto da organização `police`.
2. Cliente B fica em pé. A usa `/algemar B`. Esperado: animação, B algemado, sem item/chave.
3. B tenta abrir inventário, atacar, sacar arma e entrar em veículo. Esperado: bloqueios conforme config.
4. A usa `/desalgemar B`. Esperado: B livre, controles restaurados.
5. Repetir com `policeServiceCuffsWithoutItem=false` (após restart de desenvolvimento): `/algemar` deve negar e orientar o uso de item.

### Civil x civil

1. B fica em pé; C (civil com `handcuffs`) tenta algemar. Esperado: negado (`target_not_surrendered`).
2. B é derrubado/incapacitado (`downed`). C algema. Esperado: 1 par consumido, 1 chave emitida.
3. C move a chave para o slot 2, tenta usar chave de outro par. Esperado: negado.
4. C usa a chave correta. Esperado: chave consumida, par restaurado com o mesmo UID, B livre.
5. D rouba/transfere a chave para outro jogador; o novo portador tenta abrir. Esperado: funciona somente se for a instância emitida e a algema ainda existir.

### Admin x jogador

1. Staff sem `staff.restraints.manage` usa `/staffalgemar`. Esperado: negado.
2. Staff com permissão usa `/staffalgemar B motivo`. Esperado: B algemado sem item, log com motivo e `staffOverride=true`.
3. Staff de nível inferior tenta agir em Staff superior. Esperado: `staff_hierarchy_denied`.
4. `/staffdesalgemar B`. Esperado: livre; auditoria de remoção.

### Disconnect/relog/restart

1. Com persistência **desligada**: C algema B; B desconecta. Esperado: B não fica preso; par é restituído a C ou fica em pendência auditada.
2. Com persistência **ligada** (após SQL + flag): C algema B; B desconecta e reloga. Esperado: B volta algemado, chave de C continua válida; C abre e recebe o par.
3. Reiniciar `mz_police` em desenvolvimento com B algemado e persistência ligada. Esperado: B é reconciliado no `playerLoaded`/restart; não há duplicação de par/chave.
4. Reiniciar `mz_target` e `mz_animations`. Esperado: opções de target são reregistradas; nenhum ped fica travado.

### Inventário lotado / concorrência

1. C com inventário no limite algema B. Esperado: se a chave não couber, a ação é revertida e o par é restaurado; B não fica algemado.
2. C e D tentam algemar B ao mesmo tempo. Esperado: apenas uma transição; um par/chave.
3. C tenta desalgemar enquanto a animação de aplicação roda. Esperado: `target_busy`/`operation_pending`; sem estado impossível.
4. C move o par de slot durante a animação. Esperado: aplicação aborta sem consumir; ou consome a instância correta se ainda estiver no inventário.

### Prisão/veículo

1. B algemado, A coloca na viatura, retira e escolta. Esperado: fluxo existente preservado.
2. B com sentença ativa tenta ser algemado. Esperado: `target_in_prison`.
3. Intake prisional de B algemado. Esperado: custódia limpa, itens físicos restituídos/auditados, sentença não é criada por algemar.

## 9. Riscos residuais

- **Sem teste runtime**: animações, state bags, CEF e distâncias reais não foram validados em FiveM.
- **Persistência**: o caminho com SQL não foi executado; `ON DUPLICATE KEY UPDATE` e o schema precisam de validação em MySQL de staging.
- **Fila de restituição**: em memória; restart sem persistência pode perder a pendência (UID fica no log). Não há spawn no chão nem duplicação.
- **Chave órfã**: em liberação forçada com portador offline, a chave pode permanecer no inventário sem vínculo; ela não abre nada e é rejeitada no próximo uso.
- **Animações**: nomes `mp_arrest_paired`/`mp_arresting` são nativos; se algum não existir na build, a gameplay continua sem bloquear.
- **Legacy custody path**: outros recursos que chamem `mz_police:server:custody:request` com `cuff`/`uncuff` passam pelo P10 e podem receber `restraint_pending`; não há consumidor legado identificado além do próprio target/comandos.
- **Core**: a permissão `staff.restraints.manage` foi adicionada ao catálogo/seed do `mz_core`; isso é uma mudança de dado no owner canônico de Staff, necessária para autorização concreta. Requer aprovação arquitetural explícita no checkpoint.

## 10. Comando seguro de desenvolvimento

Não executar em produção. Em um servidor de desenvolvimento:

```text
restart mz_police
```

Se a mudança do catálogo Staff precisar ser recarregada, reinicie o servidor de desenvolvimento (não use `restart mz_core` em produção com jogadores online). Não há deploy/restart automático nesta entrega.
