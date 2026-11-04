# MZ Police — Relatório dos complementos P5.1 / P7.1 / P7.2

Data: 15/10/2026

Estado: gates estáticos aprovados; runtime FiveM **NOT RUN**

## Resultado

Os complementos foram concluídos sem iniciar P8/P9, sem alterar `mz_core`, sem executar deploy ou migrations e sem modificar a release candidate anterior.

- P5.1: associação persistente de evidência existente a caso policial, consulta e histórico no MDT, identidade do responsável, proteção contra duplicidade/replay e validação autoritativa pelo inventário.
- P7.1: atividades prisionais configuráveis, desativadas por padrão, com duração e posição verificadas no servidor, routing bucket, antispam, limites de redução, persistência após restart e auditoria.
- P7.2: soltura idempotente, devolução verificável da propriedade retida, recuperação após reconnect/restart, entrega configurável e confirmação server-side do destino.

## Ownership e contratos

- `mz_inventory` continua sendo o único owner dos itens físicos e da devolução de propriedade. O export `RestorePrisonerProperty` recebe a contagem esperada e confirma replay somente quando encontra exatamente os marcadores persistidos da sentença.
- `mz_police` armazena apenas o vínculo investigativo e seu snapshot (`evidence_id`, `instance_uid`, contêiner, revisão e policial responsável).
- `mz_prison` continua sendo o único owner de sentenças, atividades e estado de soltura.
- Todas as operações iniciadas pelo cliente resolvem identidade/sessão no servidor e falham fechadas em capability, organização, duty, posição, bucket ou dependência inválida.
- Nenhum valor de sentença, redução, multa ou coordenada foi criado nesta execução.

## Configuração obrigatória antes de ativar

### Evidências

Em `mz_police/config/server.lua`, `Evidence.occurrenceAssociationEnabled` permanece `false`. Somente após aprovação, habilitar a flag e manter os contêineres reais em `Evidence.containers`. A consulta exige `police.evidence.collect`; a associação exige `police.evidence.manage` e duty válido pela policy canônica.

### Atividades prisionais

Em `mz_prison/config.lua`, `jobsEnabled` permanece `false`, `activityRateLimit = {}` e `jobs = {}`. Antes de habilitar, o operador deve informar:

- `activityRateLimit.windowMs` e `activityRateLimit.maximum`;
- para cada job aprovado: `enabled`, `durationSeconds`, `reductionSeconds`, `maximumCompletionsPerSentence`, `maximumReductionSecondsPerSentence` e `cooldownSeconds`;
- para cada ponto do job: `enabled`, coordenadas reais `coords.x/y/z`, `radius` e `routingBucket`.

Os nomes funcionais solicitados (limpeza, lavanderia, cozinha, manutenção e rotina/exercícios) não foram pré-cadastrados, porque não há posições nem regras de redução aprovadas.

### Destino de soltura

`releasePoints = {}` permanece vazio. Cada destino aprovado precisa de `enabled`, `orgCode`, coordenadas reais `coords.x/y/z`, `radius`, `routingBucket` e, opcionalmente, `heading`.

Para escolha automática, deve existir exatamente um destino válido para a organização da sentença. Zero destinos conclui a soltura e a devolução sem teleporte. Mais de um destino válido é tratado como configuração ambígua e nenhum é escolhido arbitrariamente.

`manualReleaseEnabled` também permanece `false`; motivos manuais continuam dependentes de aprovação em `manualReleaseReasons`.

## Persistência

O prepare normal dos resources usa somente `CREATE TABLE IF NOT EXISTS` para as novas estruturas:

- `mz_police_case_evidence`;
- `mz_prison_activities`;
- `mz_prison_release_deliveries`.

O schema prisional existente continua exigindo `charge_public_id`. Nenhuma migration ou instrução destrutiva foi executada nesta entrega. A atualização em banco legado deve ser revisada e aplicada pelo operador antes do restart; o resource falha fechado quando o schema obrigatório não está presente.

## Validação executada

- Harnesses Lua completos de `mz_police`: PASS, incluindo P5.1 com 8 casos.
- Contrato NUI do MDT: 2/2 PASS.
- Harnesses de `mz_prison`: P7.1 11/11, P7.2 11/11, prepare 2/2, service 14/14 e estático 16/16.
- Contratos relacionados de `mz_inventory`: propriedade 10/10, grant policial 11/11, acesso estático PASS e armaria/evidência 11/11.
- Sintaxe Lua dos arquivos de `mz_police` e `mz_prison`, mais os arquivos tocados de `mz_inventory`: PASS com `luac -p`.
- Runner global `mz_testkit -Suite covered`: **51/51 PASS**.
- `git diff --check`: PASS; apenas avisos de normalização LF/CRLF do Git.
- QA visual local do MDT: PASS no navegador para navegação e estados vazios/indisponíveis; isto não equivale a teste NUI dentro do FiveM.
- Graphify atualizado de forma escopada e conferido contra o código real.

## Checkpoints Git

- `mz_police` `0.8.0`: `1e9dfed3e1af34c0b86db000c4fbb23899741871` (`feat(police): link evidence to cases`).
- `mz_prison` `0.2.0`: `f54763279804e2148bdb30c4fc85efb363102e54` (`feat(prison): complete activities and release recovery`).
- `mz_inventory` `1.0.0`: integração incluída na RC a partir do working tree sobre `8627a2c067cdc016b89451c8ef173051a79dd552`; não foi criado commit porque os três arquivos tocados fazem parte de um conjunto preexistente não rastreado/alterado que não permite separação segura.

Os 27 arquivos do escopo foram preservados: 11 em `mz_police`, 13 em `mz_prison` e 3 em `mz_inventory`. Alterações preexistentes fora desse conjunto não foram incorporadas aos checkpoints.

## Release candidate

Nova RC: `release-artifacts/mz-police-p5-1-p7-1-p7-2-mdt-rc-20261015.zip`

- SHA-256 do ZIP: `a29c7e077dcc74fe1c8f3847faccf57870c86b025aa54215abbf7884fefb0a0c`;
- 2.329 arquivos verificados pelo `SHA256SUMS` e 2.334 entradas no ZIP;
- o manifesto registra commits, versões, branch, dirty state e ordem de boot;
- os arquivos runtime novos de `mz_police`, `mz_prison` e a integração working-tree de `mz_inventory` estão presentes no diretório e no ZIP;
- nenhuma credencial, configuração privada ou arquivo bloqueado foi incluído pelo builder.

A RC anterior `mz-police-p1-p7-mdt-rc-20261015` permaneceu imutável. Hashes reconferidos:

- ZIP: `1779af652d2eefde691cf80a9b8a66c1d4d8a723ae24059c2608dbef27b56761`;
- manifesto: `bdb5bff43c27abdf87305f39b801d74ba4ea6b077204a1cb2cbb01843f75cd8a`;
- `SHA256SUMS`: `74ba124f877ad5fe557a695a5539c8a9b6a75e84a6f21d9b80b3e514e2a10296`.

## Atualização e rollback em staging

1. Fazer backup recuperável dos diretórios atuais dos resources e da base antes da manutenção.
2. Conferir o SHA-256 do ZIP e todos os registros de `SHA256SUMS`.
3. Publicar o conteúdo completo da RC, sem misturar arquivos de HEADs diferentes.
4. Revisar o schema de banco de forma não destrutiva; não remover tabelas nem dados existentes.
5. Manter as flags acima desativadas até inserir e aprovar valores reais.
6. Respeitar a ordem do manifesto: dependências, owners canônicos, `mz_police` e então `mz_prison`.
7. Reiniciar o FXServer e executar testes reais de autorização, evidência, atividade, soltura, reconnect e restart.
8. Em rollback, restaurar os diretórios salvos e a configuração anterior. As tabelas aditivas podem permanecer sem uso; não executar `DROP` automático.

## Pendências exclusivamente de runtime

- Inicialização real com FXServer, oxmysql e MySQL/MariaDB.
- Sessão/personagem reais, policy de organização, duty e capabilities.
- Persistência após restart e reconnect em ambiente real.
- OneSync: posição, routing bucket e teleporte de soltura com coordenadas aprovadas.
- Interação NUI/CEF dentro do FiveM e foco/ESC.
- Concorrência real entre inventário, prisão e banco.

**Runtime FiveM: NOT RUN.** Nenhum deploy ou migration foi executado.
