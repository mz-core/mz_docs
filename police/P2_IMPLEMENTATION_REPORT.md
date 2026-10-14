---
title: MZ Framework — Relatório de implementação P2 do sistema policial
status: TEST_REPORT
authority: LEVEL_4_REFERENCE
owner: MZ Police
last_reviewed: 2026-10-14
applies_to: [mz_police, mz_inventory, mz_org]
related_contracts: [MZ-IDENTITY-v2, MZ-DOCS-v1]
---

# Relatório de implementação P2

## Resultado

P2A, P2B e P2C concluídas no gate estático. Runtime FiveM permanece **NÃO EXECUTADO**.

## Funcionalidades

- Custódia runtime revisionada: algemar, desalgemar, iniciar/parar escolta.
- Restrições e animação client-side derivadas apenas do estado sincronizado pelo servidor.
- Cleanup em disconnect, morte, troca/reuso de sessão, perda de capability/duty e restart do resource.
- Colocação/retirada de custodiado em veículo com veículo, bucket, distância, network ID e assento derivados no servidor.
- Confirmação do assento pelo alvo e reconciliação periódica contra corrida/estado divergente.
- Busca corporal somente de alvo sob custódia.
- Grant opaco, curto, revogável e vinculado às sessões no owner `mz_inventory`.
- Apreensão transacional por `instance_uid`, com metadata preservada e destino fechado `police_evidence`.
- Idempotência por `requestId`/`operationId`, locks canônicos do inventário, rate limiting e auditoria.

## Contratos

Eventos server-authoritative de `mz_police`:

- `mz_police:server:custody:request`
- `mz_police:server:custody:vehicleAck`
- `mz_police:server:search:open`
- `mz_police:server:search:seize`
- `mz_police:server:search:close`

Exports server-only adicionados ao owner `mz_inventory`:

- `CreatePlayerInventoryAccessGrant`
- `GetGrantedPlayerInventorySnapshot`
- `TransferGrantedPlayerItem`
- `RevokePlayerInventoryAccessGrant`

Capabilities adicionadas no owner `mz_org`: `police.custody`, `police.search` e `police.seize`. Custódia e busca são base do seed policial; apreensão começa no nível 2 e é herdada pela hierarquia existente.

## Arquivos

- `mz_police`: `fxmanifest.lua`, `client/custody.lua`, `server/{config,bridge,audit,custody,search,events,main}.lua` e harnesses P2.
- `mz_inventory`: `config.lua`, `shared/constants.lua`, `server/{core_bridge,service,police_access,exports}.lua` e harnesses de grant.
- `mz_org`: registry/allowlist de capabilities, seed policial e harness contratual.

## Gate estático

- Parse Lua de todos os arquivos de `mz_police`: PASS.
- P1 regressão: PASS, 23 casos.
- P2A custódia: PASS, 16 casos.
- P2B veículos: PASS, 9 casos.
- P2C orquestração: PASS, 6 casos.
- Grants do inventário: PASS, 11 casos + contrato estático.
- Regressões de owner/extraction do inventário: PASS.
- Registry, seed e allowlist do `mz_org`: PASS, incluindo Lot 6B/6D.

Mocks verificam lógica e contratos; não equivalem ao runtime FiveM.

## Segurança e limitações

- O cliente nunca informa `citizenid`, veículo, assento, bucket, destino físico ou estado de custódia.
- A apreensão aceita somente UID existente no inventário real do alvo e container allowlisted.
- Grants expiram, são revogados em cleanup e não são enviados ao cliente.
- Não há SQL cross-domain, transferência client-authoritative, bypass staff ou ban automático.
- A interação visual para iniciar os contratos será conectada a `mz_menu` na P3; os eventos P2 já são os contratos canônicos.
- Animação, attachment, seat task e sincronização OneSync ainda exigem teste real com dois jogadores.

## Próxima wave

P3 pode consumir estes contratos para pontos policiais, arsenal e veículos de serviço. Nenhuma coordenada será criada sem configuração real.

## Impacto arquitetural

- Contratos públicos: cinco eventos policiais e quatro exports server-only do inventário.
- Ownership: custódia em `mz_police`; itens/containers permanecem em `mz_inventory`; capabilities em `mz_org`.
- Writer/authority: inventário continua single-writer e usa seus locks/transações existentes.
- Dependências: `mz_police` passa a depender de `mz_inventory` além de Core/Org.
- Persistência: nenhuma tabela nova; apreensões usam `mz_inventory_items` pelo owner.
- Segurança: payload fechado, sessão, duty/capability, distância, bucket, rate limit, revisão e idempotência.
- Runtime: pendente.
