---
title: MZ Framework — Auditoria P0 e plano incremental do sistema policial
status: REFERENCE
authority: LEVEL_4_REFERENCE
owner: MZ Police (planned)
last_reviewed: 2026-10-13
applies_to: [mz_police, mz_prison, mz_dispatch, mz_org, mz_inventory, mz_vehicles, mz_garagem, mz_economy, mz_core]
related_contracts: [MZ-DOCS-v1]
---

# Auditoria P0 e plano incremental do sistema policial

## A. Sumário executivo

Este documento registra somente a Fase P0: descoberta, confirmação de ownership, análise de referências e planejamento. Nenhum fluxo policial, migration, evento de gameplay, NUI ou facade no `mz_core` foi implementado.

O repositório já contém os donos canônicos necessários para identidade/sessão, organizações e plantão, inventário/armas, veículos/garagem, economia e logs. Não existe resource policial ativo. A solução deve nascer como `mz_police`, com `mz_prison` separado quando a onda de prisão começar. `mz_dispatch` deve ser criado apenas quando houver contrato genérico e consumidor real além da polícia; não há benefício em manter agora um adapter vazio.

Os bloqueios técnicos reais para as primeiras ondas são pequenos e localizados: ponto de plantão da organização `police`, acesso delegado e temporário ao inventário de outro jogador, transferência por UID para apreensão, autorização de guincho para um chamador de domínio confiável, liquidação econômica atômica de multa e um sinal público estável para mudanças relevantes de plantão/estado do jogador. Cada extensão deve permanecer no dono canônico do dado.

## B. Escopo auditado

### Código e documentação atuais

- `mz_core`: identidade, conta, personagem, sessão, estado fundamental do jogador, autorização staff global e log central.
- `mz_org`: organizações, vínculos, grades, capabilities, duty e cofres organizacionais como política.
- `mz_inventory`: itens, slots, containers, UIDs, metadata, hotbar, equipment e autorização de armas.
- `mz_vehicles`: agregado do veículo, ownership/acesso, estado e impound.
- `mz_garagem`: pontos, regras de garagem e orquestração sobre `mz_vehicles`.
- `mz_economy`: contas, mutações idempotentes, ledger e saldo organizacional.
- `mz_menu`, `mz_interact`, `mz_hud`, `mz_phone`, `mz_banguard` e `mz_medical`: integração potencial sem transferência de ownership.
- `mz_starter/cfg/resources.cfg`: ordem atual de inicialização e dependências operacionais.
- Documentos de extração de domínio em `mz_docs` e fontes reais dos resources.

### Referências externas locais

Foram inspecionados, sem copiar código ou assets, os pacotes `qb-policejob`, `qb-prison`, `vrp_policia`, `tabletpolicial_v2`, `alerts` e `hv-arsenal` em `ref/mz_police_handoff`.

`qb-policejob` e `qb-prison` declaram GPLv3. Nos demais pacotes não foi confirmada licença que autorize reutilização. Todos são tratados apenas como referência de fluxo e UX.

## C. Descobertas principais

1. Não existe `mz_police`, `mz_prison`, `mz_dispatch`, `mz_mdt` ou `mz_justice` ativo.
2. A organização `police` já é seed de `mz_org`, com dez grades e capabilities reutilizáveis para rádio, tablet, MDT, storage, armory, veículos, patrulha, relatórios, gestão e comando.
3. O seed policial não configura `dutyPoint`; portanto `SetSelfOrgDutyAtPoint` retorna `duty_point_not_configured` até que o ponto seja definido no owner `mz_org`.
4. `mz_inventory` mantém UID e metadata, mas seus descriptors públicos resolvem `player` e `equipment` sempre para o próprio ator. Não há grant seguro para busca/apreensão em outro jogador.
5. `mz_vehicles` é o único writer de impound. A política atual de `ImpoundVehicle` aceita console/ACE/staff, não um policial operacional autorizado por organização.
6. `mz_economy` possui idempotência e ledger, porém não oferece uma operação atômica de débito do jogador e crédito da organização para pagamento de multa.
7. `mz_core` expõe leitura de estado/morte, mas seus eventos internos de lifecycle não são contrato público para um novo resource policial.
8. `mz_phone` não contém hoje fluxo 911/dispatch que justifique antecipar um domínio `mz_dispatch`.
9. `mz_interact` e `mz_menu` bastam para as primeiras ondas. Uma NUI MDT completa deve esperar a onda P6.

## D. Ownership confirmado

| Capacidade/dado | Dono canônico | Responsabilidade de `mz_police` |
|---|---|---|
| Identidade, conta, personagem e sessão | `mz_core` | Consumir a identidade ativa; nunca persistir `source` como identidade |
| Estado fundamental, morte e ação disponível | `mz_core` | Consultar antes de ações e reagir por contrato público estável |
| Staff global | `mz_core` | Não criar permissões staff paralelas |
| Organização, vínculo, grade, capability e duty | `mz_org` | Revalidar contexto/capability em toda ação privilegiada |
| Inventário, equipamento, armas, UID e metadata | `mz_inventory` | Orquestrar grants, transferências e containers; nunca escrever tabelas do inventário |
| Veículo, ownership, acesso, estado e impound | `mz_vehicles` | Validar a ação policial e solicitar a mutação ao owner |
| Pontos e política de garagem | `mz_garagem` | Consumir quando necessário; não liberar ou gravar veículo diretamente |
| Dinheiro, conta organizacional, idempotência e ledger | `mz_economy` | Criar obrigação legal e solicitar liquidação atômica ao owner |
| Logs/auditoria central | `mz_core` | Emitir logs normalizados com correlação; manter timeline de domínio só quando necessária |
| Custódia policial em tempo real | `mz_police` | Estado runtime, política, sincronização e limpeza |
| Casos, evidências lógicas, multas e mandados | `mz_police` | Writer único do domínio policial |
| Sentença, intake, propriedade retida e soltura | `mz_prison` | Apenas solicitar por API autorizada |
| Chamados e atribuições genéricas | `mz_dispatch`, quando existir | Adapter policial; não ser o owner do chamado genérico |

## E. Reutilização obrigatória

- Resolver `source -> session -> activeCharacterId -> citizenid` pelos contratos de `mz_core`.
- Usar `GetPlayerOrgContext`, `CanOrg`, `HasGradeOrAbove` e duty de `mz_org`.
- Reusar as capabilities existentes: `org.view`, `radio.use`, `tablet.open`, `mdt.open`, `storage.*`, `armory.basic`, `vehicle.*`, `patrol.*`, `reports.approve`, `account.*`, `manage.team`, `boss.actions`, `highcommand` e `command.full`.
- Usar os containers, UID, metadata, locks e mutações de `mz_inventory`.
- Usar `ImpoundVehicle` e demais serviços de `mz_vehicles`; `mz_garagem` continua responsável por seus pontos e política de liberação.
- Usar operações idempotentes e ledger de `mz_economy`.
- Usar `CreateDetailedLog` para auditoria central, sem substituir os registros de domínio.
- Usar `mz_interact` para pontos/targets e `mz_menu` para UX inicial.
- Preservar `MZ_SECURITY_CANARY` como sensor inerte, sem transformá-lo em fluxo legítimo ou motivo isolado para ban.

## F. Gaps e decisões abertas

### Gaps de infraestrutura

- Configurar `dutyPoint` para a organização policial no owner `mz_org`.
- Criar em `mz_inventory` um grant curto, revogável e allowlisted para leitura e transferência de item de outro jogador por UID.
- Criar em `mz_inventory` um descriptor restrito para containers nomeados de evidência e propriedade prisional, sem novo writer.
- Permitir que `mz_police`, como chamador confiável, solicite impound em `mz_vehicles`; o resource de veículos mantém validação e escrita.
- Adicionar a `mz_economy` uma liquidação atômica jogador -> organização, com ledger, correlação e idempotência.
- Definir contrato público estável para mudanças de duty e de estado/morte, ou adotar polling curto e limitado até essa decisão. Eventos `internal` não são API.

### Decisões de produto/lei

- Conteúdo e versionamento do código penal.
- Multa como obrigação pendente, pagamento voluntário, cobrança automática ou combinação.
- Fiança, apreensão de dinheiro, limites, prescrição e cancelamento.
- Regra de aprovação de mandado e necessidade de dupla aprovação.
- Retenção, destruição, devolução e visibilidade de evidência.
- Jurisdição e compartilhamento de dados entre corporações.
- Tempo de prisão online ou calendário. Recomendação inicial: contar somente tempo online.
- Comportamento de custódia diante de morte, incapacitação, reconnect e troca de personagem. Recomendação inicial: custódia é runtime e não sobrevive a reconnect/troca.

## G. Limite entre resources

### `mz_police`

Contém autorização contextual policial, custódia runtime, busca/apreensão como orquestração, arsenal como orquestração, flags e ações de fiscalização de veículo, evidência/cadeia de custódia, casos, pessoas relacionadas, acusações, multas, mandados e UI policial.

### `mz_prison`

Contém sentença, intake, retenção/devolução de propriedade, contagem de pena, trabalhos prisionais, lockdown/alarme e soltura. Não pertence a `mz_core` nem deve ser persistido como metadata solta do jogador.

### `mz_dispatch`

Deve ser genérico e existir quando houver consumidor real como EMS/phone. Será owner de chamados, unidades/atribuições e lifecycle do chamado. Até lá não será criado adapter vazio.

### `mz_core`

Permanece congelado. Uma eventual mudança só pode expor lifecycle genérico já pertencente ao Core, após Core Change Gate; nunca conterá lógica, tabela ou facade policial.

## H. Modelo mínimo de dados por onda

Nenhuma tabela deve ser criada em P1/P2. Custódia é estado runtime revisionado, associado à sessão atual e a snapshots de `citizenid`, com limpeza em disconnect, resource stop, morte/respawn, perda de duty e troca de personagem.

### P3 — emissão rastreável de arsenal, se necessária

`mz_police_armory_issues`: `public_id`, `operation_key` único, `org_code`, `issued_to_citizenid`, `issued_by_citizenid`, `item_name`, `amount`, `instance_uid`, `status`, `issued_at`, `returned_at`, `metadata_json`. A linha rastreia a emissão; o item continua pertencendo a `mz_inventory`.

### P4 — flags policiais de veículo

`mz_police_vehicle_flags`: `public_id`, `plate_normalized`, `flag_type`, `reason`, `status`, `case_public_id`, `org_code`, `created_by_citizenid`, `expires_at`, timestamps.

### P5 — evidência e cadeia de custódia

- `mz_police_evidence`: `public_id`, `evidence_type`, `case_public_id`, `source_citizenid`, `collected_by_citizenid`, `org_code`, contexto/coords, referências de arma/modelo/série, `metadata_json`, `status`, timestamps.
- `mz_police_evidence_custody`: `evidence_public_id`, transição, origem/destino, ator, `operation_id` único, `metadata_json`, `created_at`.

O item físico e sua localização permanecem em `mz_inventory`.

### P6 — MDT mínimo

- `mz_police_cases`
- `mz_police_case_officers`
- `mz_police_case_people`
- `mz_police_case_charges`, com snapshot imutável de código, nome, multa e pena aplicados
- `mz_police_citations`
- `mz_police_warrants`

O catálogo penal pode começar como configuração versionada. Não é necessária uma tabela editável de crimes no MVP.

### P7 — prisão

- `mz_prison_sentences`
- `mz_prison_property`, referenciando UID/container/slot/status sem duplicar o item

### P8 — dispatch, somente quando justificado

- `mz_dispatch_calls`
- `mz_dispatch_assignments`

Status transitório de unidade pode permanecer runtime até haver requisito comprovado de persistência.

## I. APIs, exports e eventos mínimos propostos

Estes nomes são TARGET, não contratos implementados.

### Política interna de `mz_police`

- `Policy.authorize(source, capability, context)`: resolve sessão/personagem, organização elegível, duty, capability e contexto físico.
- `Custody.get(targetSource)`: leitura do estado runtime revisionado.

Não criar export público enquanto não existir consumidor real.

### Eventos de rede P2

- `mz_police:server:custody:request`: payload fechado com `action`, `targetSource`, `requestId` e revisão esperada. Ações: cuff/uncuff, iniciar/parar escort e colocar/retirar de veículo.
- `mz_police:server:search:open`: `targetSource` e `requestId`.
- `mz_police:client:custody:sync`: estado produzido pelo servidor e sua revisão.

IDs, posição, bucket, veículo, duty e capability são derivados/revalidados no servidor. O client nunca fornece `citizenid`, pena, valor, inventário ou estado autoritativo.

### Extensão mínima em `mz_inventory`

- `CreatePlayerInventoryAccessGrant(actorSource, targetSource, purpose, ttlSeconds)`
- `GetGrantedPlayerInventorySnapshot(actorSource, grantToken)`
- `TransferGrantedPlayerItem(actorSource, grantToken, instanceUid, amount, destination, operationId)`
- `RevokePlayerInventoryAccessGrant(actorSource, grantToken)`

O owner valida allowlist do resource chamador, sessão ativa, source reuse, distância, routing bucket, expiração, locks, operação idempotente, posição atual do UID e destino permitido.

### Extensão mínima em `mz_vehicles`

Manter `ImpoundVehicle` como mutação canônica e adicionar caminho allowlisted para `mz_police`, recebendo ator, motivo normalizado, localização e referência/correlação. A validação policial continua em `mz_police`; a integridade do veículo continua em `mz_vehicles`.

### Extensão mínima em `mz_economy`

`SettlePlayerOrganizationCharge(targetSource, orgCode, amount, options)`: débito e crédito atômicos, ledger nos dois lados, chave idempotente e correlação. O valor vem do código penal validado pelo servidor policial.

### `mz_prison` futuro

- `RequestSentence(actorSource, targetSource, request)`
- `GetSentence(citizenid)`
- `ReduceSentence(actorSource, citizenid, request)`
- `ReleasePrisoner(actorSource, citizenid, request)`

Chamadores são allowlisted e toda mutação é idempotente/auditada.

## J. Matriz mínima de capabilities

Reusar capabilities existentes sempre que elas expressarem a ação. Adicionar apenas:

| Capability proposta | Uso |
|---|---|
| `police.custody` | Algema, escolta e colocação/retirada de veículo |
| `police.search` | Identificação contextual e consulta corporal |
| `police.seize` | Transferência de item autorizada por UID |
| `police.citation.issue` | Emissão de multa/obrigação |
| `police.arrest.request` | Solicitação de sentença a `mz_prison` |
| `police.impound` | Solicitação operacional de guincho |
| `police.evidence.collect` | Coleta de evidência |
| `police.evidence.manage` | Transferência, descarte/devolução autorizados |
| `police.records.write` | Criação/edição de caso e registro |
| `police.warrant.create` | Criação de mandado |
| `police.warrant.approve` | Aprovação/cancelamento conforme política |
| `police.road.tools` | Cones, barreiras e spike strips |
| `police.anpr` | Consulta/flag por placa |
| `police.radar` | Uso e gestão operacional de radar |

Não criar `police.command`, `police.armory.issue` ou cópias de capabilities de gestão já presentes. A elegibilidade deve ser uma configuração de organizações policiais, inicialmente contendo apenas `police`, em vez de comparações dispersas e rígidas por código.

## K. Ondas de implementação e gates

### P0 — auditoria e plano

Este documento. Gate: ownership confirmado em fonte, referências classificadas e gaps explícitos.

### P1 — fundação segura

Criar o resource mínimo, config de organizações elegíveis, política de autorização, rate limiting, logging/correlação, testes de contrato e ponto de duty no owner. Sem DB e sem NUI própria.

Gate: organização/duty/capabilities validados; recursos canônicos iniciam antes; nenhum fluxo depende de estado do client.

### P2 — custódia, busca e apreensão

Implementar estado runtime revisionado, validações de distância/bucket/estado, cleanup e extensão de grant no inventário.

Gate: testes de replay, source reuse, simultaneidade, desconexão, morte, perda de duty e rollback de inventário.

### P3 — estações, arsenal e veículos de serviço

Usar `mz_interact`, `mz_menu`, storage/armory/vehicle capabilities e containers canônicos. Persistir emissão somente se houver requisito de devolução/rastreio.

Gate: todo item possui origem, UID/metadata preservados e auditoria; nenhum `GiveWeaponToPed` autoritativo.

### P4 — fiscalização veicular

Impound confiável, flags, ANPR, radar e objetos viários limitados pelo servidor.

Gate: placa/entidade/proximidade derivadas no servidor; limites de objetos; writer único de veículo preservado.

### P5 — evidências

Coleta, empacotamento, locker e cadeia de custódia sobre containers de `mz_inventory`.

Gate: toda transição é idempotente, auditável e não duplica item.

### P6 — MDT

Casos, busca minimizada de cidadão/veículo, acusações, multas e mandados. Criar NUI apenas nesta onda.

Gate: regras legais aprovadas, filtros por organização/jurisdição, payloads de leitura minimizados e liquidação econômica atômica.

### P7 — prisão

Criar `mz_prison`, intake, sentença, propriedade, contagem, atividades e soltura.

Gate: regras de pena aprovadas, propriedade reconciliável por UID e nenhuma identidade baseada em `source`.

### P8 — dispatch

Criar domínio genérico somente com consumidores reais; integrar phone, polícia e EMS por adapters.

Gate: ownership/lifecycle do chamado definido e isolamento entre serviços validado.

### P9 — sistemas avançados

Helicóptero, câmeras, tracker, GSR avançado, fingerprint lab e recursos semelhantes, somente após telemetria e necessidade real.

## L. Ordem de dependências

Ordem alvo mínima:

1. `mz_core`
2. `mz_economy`
3. `mz_menu` e `mz_interact`
4. `mz_vehicles`
5. `mz_org`
6. `mz_inventory`
7. `mz_garagem`
8. `mz_police`
9. `mz_prison`, quando P7 existir
10. `mz_dispatch`, quando P8 existir

`mz_police` deve declarar somente dependências realmente usadas. Dependência opcional deve degradar de modo explícito, não silencioso.

## M. Modelo de ameaça e controles

| Risco | Controle mínimo |
|---|---|
| Evento/NUI malicioso | Payload fechado, tipos/limites, sessão e capability revalidadas no servidor |
| Spoof de target/citizenid | Client envia no máximo `targetSource`; servidor resolve identidade ativa |
| Source reuse/reconnect | Grants ligados a sessão/character e TTL curto; nunca persistir `source` |
| Ação remota/cross-instance | Distância, routing bucket, entidade e estado verificados no servidor |
| Replay/duplo clique | `requestId`/`operationId` idempotente e revisão esperada |
| Corrida em cuff/item/veículo | Locks ou transações no owner canônico; estado revisionado |
| Multa/pena arbitrária | Código penal server-side, snapshots imutáveis e limites aprovados |
| Duplicação de item/evidência | Transferência por UID, destino fechado e cadeia de custódia transacional |
| Guincho de placa forjada | Placa/entidade normalizadas no servidor e mutação em `mz_vehicles` |
| Vazamento entre corporações | Filtro exato por org/jurisdição e minimização de resposta |
| Spam de objetos/chamados | Rate limit por ator/ação/alvo, quota e cleanup |
| Resource stop/morte/perda de duty | Cleanup idempotente e reconciliação |
| Abuso repetido | Log estruturado com correlação; nunca auto-ban por canary isolado |

## N. Requisitos de idempotência, locks e logs

- Toda mutação persistente recebe chave idempotente definida pelo servidor ou validada em formato/escopo.
- A chave inclui escopo de domínio e não confia apenas no valor fornecido pelo client.
- Inventário, economia e veículos usam seus próprios locks/transações; `mz_police` não cria lock paralelo sobre dados alheios.
- Custódia usa revisão monotônica e transições válidas; repetir uma solicitação retorna o resultado anterior ou noop seguro.
- Logs críticos incluem ator/target resolvidos, organização, capability, ação, resultado, reason code, correlação e public IDs; nunca incluem segredo ou payload bruto desnecessário.
- Logs centrais são auditoria. Casos, evidências e sentenças continuam sendo dados de domínio, não reconstruídos a partir de log.

## O. Referências — reutilizar ideias, rejeitar acoplamentos

### Ideias úteis

- `qb-policejob`: catálogo de interações, evidência, flags, impound, objetos viários, panic/tracker e câmera.
- `qb-prison`: lifecycle de sentença, propriedade, trabalhos, lockdown, alarme e escape hooks.
- `tabletpolicial_v2`: navegação de busca, ficha, ocorrências, multas e prisão.
- `alerts`: fila, aceite e lifecycle visual de chamados.
- `hv-arsenal`: catálogo por grade, timing/cooldown e opções por item.
- `vrp_policia`: linguagem operacional brasileira e amplitude de fluxos.

### Padrões rejeitados

- Acoplamento a QBCore/vRP, Tunnel/Proxy ou tabelas alheias.
- SQL direto em veículos, inventário, economia, identidade ou organização.
- Limpar inventário/armas sem transação e rastreio por UID.
- Valores, pena, evidência, estado ou identidade fornecidos pelo client.
- Prisão, multas, dispatch e polícia em um resource monolítico.
- Estado crítico apenas em memória sem lifecycle/cleanup definido.
- Webhooks, URLs, permissões ou teleport hardcoded.
- Eventos públicos amplos, duplicados ou sem rate limit/idempotência.
- Assets/código sem licença confirmada.

## P. Critérios para iniciar P1

P1 só deve começar após:

1. Aprovação deste boundary e da lista mínima de capabilities.
2. Definição da(s) organização(ões) elegível(is). O `dutyPoint` pode permanecer pendente durante a fundação P1, mas bloqueia validação operacional de entrada em serviço.
3. Decisão sobre o contrato público de duty/estado ou aceite explícito do polling limitado.
4. Confirmação de que custódia não persiste entre sessões no MVP.
5. Execução dos testes reais existentes de `mz_org`, `mz_inventory`, `mz_vehicles`, `mz_economy` e `mz_core` no ambiente aplicável.

Validação estática, testes de browser e inspeção documental não equivalem a teste no runtime FiveM. Cada onda deve registrar scripts executados, ambiente, limitações e resultado antes de avançar.

## Fontes internas principais

- [`mz_docs/organizations/DOMAIN_EXTRACTION.md`](../organizations/DOMAIN_EXTRACTION.md)
- [`mz_docs/inventory/DOMAIN_EXTRACTION.md`](../inventory/DOMAIN_EXTRACTION.md)
- [`mz_docs/vehicles/DOMAIN_EXTRACTION.md`](../vehicles/DOMAIN_EXTRACTION.md)
- [`mz_docs/economy/DOMAIN_EXTRACTION.md`](../economy/DOMAIN_EXTRACTION.md)
- [`mz_docs/core/ACCOUNT_CHARACTER_SESSION.md`](../core/ACCOUNT_CHARACTER_SESSION.md)
- [`mz_docs/core/AUTHORIZATION.md`](../core/AUTHORIZATION.md)
- [`mz_docs/core/CORE_ARCHITECTURAL_FREEZE.md`](../core/CORE_ARCHITECTURAL_FREEZE.md)

