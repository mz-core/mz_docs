# MZ Police — relatório de implementação P5

Data: 2026-10-14  
Wave: P5 — evidências  
Estado: **núcleo estático aprovado; coleta ambiental e associação com ocorrência bloqueadas por contratos ausentes; runtime pendente**

## Resultado implementado

- todo item apreendido pela P2 recebe identificação determinística `evidence:<instanceUid>`;
- UID e metadata originais são preservados no container restrito `police_evidence`;
- metadata `policeEvidence` registra organização, container, revisão, tipo e horário de coleta;
- listagem de evidências é server-only e minimizada;
- transferência entre containers configurados usa a transação/locks do `mz_inventory`;
- cada transferência incrementa a revisão, registra último custodiante e gera logs correlacionados em `inventory` e `police`;
- replay de transferência é idempotente e conflito de operation/request ID é rejeitado;
- containers não configurados e campos extras são rejeitados.

Também foi corrigida a seleção de slot livre no planner canônico: transferências server-side que não informam slot de destino agora selecionam um slot vazio antes da validação. Sem isso, a apreensão P2 real terminaria em `invalid_to_slot`, embora os mocks anteriores passassem.

## Ownership

- item, UID, metadata, containers, lock e transação: `mz_inventory`;
- autorização operacional, duty/capability e orquestração: `mz_police`;
- identidade/sessão e logs detalhados: contratos existentes do `mz_core`;
- associação com caso/ocorrência: reservada ao backend P6 do `mz_police`.

Exports server-only adicionados em `mz_inventory`:

- `ListPoliceEvidence(source, orgCode, containerId)`;
- `TransferPoliceEvidence(source, orgCode, fromContainer, toContainer, instanceUid, operationId)`.

Capabilities registradas, sem concessão automática a cargos:

- `police.evidence.collect`;
- `police.evidence.manage`.

## Configuração

Somente o container lógico `intake` permanece habilitado por padrão, porque já era o destino P2. Outros lockers devem ser cadastrados em `MZPoliceConfig.Evidence.containers` antes de permitir transferências.

`occurrenceAssociationEnabled` permanece `false`. O client não consegue enviar `occurrenceId` nesta wave.

## Bloqueios explícitos

A coleta ambiental (DNA, cápsula, projétil, impressão, sangue etc.) não foi inventada. Não há no framework atual um contrato canônico que produza evidência física confiável, com tipo e origem server-side. Criar itens a partir de um payload client seria inseguro. A integração deve ser concluída quando os produtores reais e o catálogo aprovado existirem.

A associação com ocorrência depende das tabelas e IDs canônicos da P6. Ela permanece desabilitada até esse owner existir.

## Testes

- P5 `mz_police`: 6 casos PASS;
- owner armory/evidence: 11 casos PASS;
- grant policial P2: 11 casos PASS;
- contratos estáticos, owner e extração de domínio do inventory: PASS;
- regressões P1–P4: PASS;
- sintaxe Lua e `git diff --check`: executados no gate.

Mocks não equivalem ao runtime FiveM/MySQL.

## Pendências de runtime

- confirmar criação da metadata no item apreendido real;
- validar transação para slot livre com MySQL;
- cadastrar ao menos um segundo container e validar transferência/restart;
- confirmar os logs de cadeia de custódia no backend real.

Nenhum runtime PASS é declarado.

## Próxima wave independente

P6 — registros e MDT. A criação do backend canônico permitirá finalizar a associação de evidência com ocorrência sem inventar identificadores.
