---
title: mz_phone — Physical charging architecture
status: REFERENCE
authority: LEVEL_2_ARCHITECTURE
owner: mz_phone
last_reviewed: 2026-09-26
applies_to: [mz_phone, mz_admin, mz_world_editor, mz_interact]
related_contracts: [MZ-DOCS-v1]
---

# mz_phone physical charging

## Purpose

Permitir que um aparelho real seja conectado a uma tomada entity ou a uma ancora
customizada invisivel, colocado fisicamente no mundo e carregado pela bateria
canonica do `DeviceFoundation`.

## Owns

- normalizacao de outlets entity/custom;
- tabela `mz_phone_charging_outlets` e seu cache server-side;
- state machine runtime da charging session;
- props visuais locais do carregador/telefone e Rope;
- validacoes espaciais, de ownership, transicao, rate limit e replay.

## Does Not Own

- identidade, ownership ou bateria do aparelho: pertencem ao `DeviceFoundation`;
- interacao espacial: pertence ao `mz_interact`;
- camera, raycast e ghost placement: pertencem ao `mz_world_editor`;
- permissoes staff: pertencem ao `mz_core` e sao expostas pelo `mz_admin`.

## Outlet sources

`entity` e usado quando `GetClosestObjectOfType` devolve um objeto valido de um
modelo configurado. Objetos incorporados a uma mesh de interior podem produzir hit
e surface normal no shape test sem produzir um entity handle utilizavel. Esses
casos usam `custom`: uma ancora persistida com transform completo e distancia de
interacao. Nao ha alteracao de MLO/YTYP.

Os modelos iniciais ficam exclusivamente em `Config.Charging.Outlets`:

- `v_res_tre_plugsocket`;
- `v_serv_1socket`;
- `v_serv_2socket`;
- `lr_smodrv_2socket_001`.

Um custom outlet dentro de `OutletDedupeRadius` substitui o automatico, evitando
duas interacoes. Depois da resolucao, ambas as origens usam o mesmo contrato com
`id`, `sourceType`, position/rotation, charger transform, cable point e interaction
distance.

## Admin workflow and authorization

O caminho e **Admin > Mundo e cenarios > Tomadas de carregamento**. O client usa o
ghost `mz_phone_charger` do `mz_world_editor`, snap por raycast/surface normal,
movimento XYZ e rotacao Pitch/Roll/Yaw. Cancelamento ou resource stop apaga o
preview. Criacao usa um draft server-side de uso unico com TTL.

As operacoes sao autorizadas duas vezes: `mz_admin` valida a action e `mz_phone`
valida a permissao global correspondente. O sistema reutiliza:

- view: `staff.world.view`;
- create: `staff.world.props.create`;
- manage/move: `staff.world.props.manage`;
- archive/restore: `staff.world.props.archive`.

O servidor valida formato, limites numericos, distancia do staff ao placement,
unicidade do codigo e tamanho de metadata. “Remover” e implementado como
arquivamento recuperavel (`enabled = 0`), mantendo autoria e historico.

## Public server exports

Fornecidos por `mz_phone` para o adaptador administrativo:

- `IsChargingOutletsReady()`;
- `GetChargingOutletEditorSettings()`;
- `ListChargingOutletsForAdmin(source)`;
- `CreateChargingOutletForAdmin(source, payload)`;
- `UpdateChargingOutletForAdmin(source, outletId, payload)`;
- `SetChargingOutletEnabledForAdmin(source, outletId, enabled)`.

Esses exports nao substituem autorizacao: cada mutacao revalida `source` no
resource proprietario.

## State and events

Charging sessions sao runtime-only e indexadas por source, device e outlet. Um
device e um outlet so admitem uma sessao. O servidor transmite snapshots; clients
em scope recriam carregador, telefone e Rope sem alterar o estado canonico.

Eventos client-to-server aceitos passam por schema fechado, rate limit e cache de
replay: `connect`, `placePhone`, `cancelPlacement`, `takePhone`, `disconnect` e
`outletMissing`. Payloads nao aceitam battery, owner ou charging rate.

`mz_phone:server:chargingOutletChanged` encerra sessoes quando a ancora muda ou e
arquivada. Disconnect, morte, liberacao do device context e stop de `mz_phone`
tambem fazem cleanup. Abrir a NUI enquanto o telefone esta colocado e recusado para
nao substituir o contexto fisico.

## Battery and Rope

`DeviceFoundation.SetPhysicalCharging` e o unico caminho que liga/desliga carga.
A progressao usa timestamp/delta e checkpoints; nao grava por frame e aplica clamp
de 0 a 10000 basis points. O Rope e estritamente visual e pode desaparecer ou ser
recriado a partir do snapshot sem mudar a bateria.

## Security boundary

Custom outlets sao resolvidos inteiramente no servidor a partir do ID persistido;
o client nao escolhe seu transform. Para entity outlets do mapa, o servidor valida
modelo permitido, hash declarado, bounds dos anchors, distancia do player,
ocupacao, ownership e transicao. Como map objects nao networked nem sempre existem
no entity pool server-side, a existencia visual exata da entity continua sendo uma
atestacao limitada do client; ela nao concede bateria, ownership ou coordenadas
fora dos limites aceitos. Locais sensiveis devem usar custom outlets, cujo ID e
transform sao autoritativos.

## Database

`mz_phone_charging_outlets` armazena codigo unico, label, XYZ, rotacao XYZ, surface
normal, interaction distance, enabled, metadata e auditoria de criacao/edicao/
arquivamento. A tabela e criada pela migration SQL e defensivamente no startup.

## Assets and builds

O servidor Legacy usa `sv_enforceGameBuild 3751`. Somente o artefato Gen8 esta na
raiz efetivamente streamada; Gen8 e Gen9 permanecem preservados em
`assets/charger/`. Runtime sempre usa o nome `mz_phone_charger`.

Na revisao de 2026-09-26 nao havia `.ytyp` no repositorio. Portanto o manifest nao
declara `DLC_ITYP_REQUEST`. Quando o YTYP original for adicionado, ele deve entrar em
`files` e em um unico `data_file 'DLC_ITYP_REQUEST'`.

## Failure behavior

- entity sem handle: nenhuma interacao automatica; use custom outlet;
- outlet custom arquivado/alterado: sessao encerrada e snapshot atualizado;
- entity automatica desaparecida: apos confirmacoes configuradas, owner reporta e
  o servidor encerra a sessao;
- editor interrompido: ghost removido e draft cancelado;
- restart: sessions e visuais somem; device permanece integro e nao carrega em
  background; custom outlets voltam do banco.

## Tests

- `mz_phone/tests/charging_sessions.lua`: T01–T06 e T12–T30, incluindo modelos,
  dedupe, state machine, cabo, authority, replay, ownership e cleanup;
- `mz_phone/tests/charging_outlets.lua`: T07–T11 com repository/permission/database
  simulados, incluindo reload;
- validacao runtime in-game ainda e obrigatoria para pivots/offsets dos quatro
  modelos, comportamento especifico de cada MLO e aparencia do Rope.
