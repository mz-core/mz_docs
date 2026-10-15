# MZ Police — Release Candidate P1–P7 + MDT

Data da consolidação: 2026-10-15  
Escopo: distribuição P1–P7 e frontend MDT sobre o backend canônico P6  
Estado: **pacote consistente para staging; runtime FiveM/MySQL ainda não aprovado**

## Resultado executivo

- Foi criado um snapshot distribuível do estado runtime efetivo dos resources, incluindo integrações
  essenciais que ainda existem apenas nos working trees dos owners.
- O snapshot possui manifesto de revisions/estado, lista SHA-256 de cada arquivo e arquivo ZIP com hash
  externo. Configurações privadas, credenciais e arquivos Git não entram no pacote.
- O MDT foi implementado dentro de `mz_police`, sem backend paralelo e sem alteração em `mz_core`.
- Todas as operações mutáveis continuam no servidor P6; o cliente só envia payloads para os eventos
  canônicos e não decide capability, duty, sentença, pena ou valor financeiro.
- Não houve deploy, push, reset, clean ou execução de migration. P8/P9 não foram iniciadas.

## Checkpoints criados

| Repositório | Commit | Conteúdo isolado |
|---|---|---|
| `mz_starter` | `83943b0c8270fa823855a9bfe3f80656565c4323` | builder do RC e teste do empacotamento |
| `mz_police` | `137faa85f182dc3b4fe4cb315e9611d2d470e6d6` | NUI MDT autorizada, assets locais e testes |

Os commits anteriores necessários continuam sendo `mz_police@a0ff0ce` (vínculo canônico de sentença)
e `mz_prison@7d1b3b8` (schema/serviço P7). O commit `137faa8` sucede `a0ff0ce`.

Não foi criado commit em `mz_inventory`, `mz_org`, `mz_economy` ou `mz_testkit`. Nesses repositórios,
a separação da integração policial ainda não é segura porque ela depende de extrações maiores e
preexistentes. Nenhuma dessas mudanças foi reescrita ou incluída em commits parciais.

## Artefato de distribuição

- Diretório: `release-artifacts/mz-police-p1-p7-mdt-rc-20261015/`
- Arquivo: `release-artifacts/mz-police-p1-p7-mdt-rc-20261015.zip`
- SHA-256 do ZIP: `1779af652d2eefde691cf80a9b8a66c1d4d8a723ae24059c2608dbef27b56761`
- Arquivos runtime: 2.327
- Manifesto: `release-manifest.json`
- Integridade completa: `SHA256SUMS` (2.327 entradas, todas recalculadas com PASS)

O pacote é construído a partir dos arquivos do working tree, não apenas dos HEADs. O manifesto marca
cada resource sujo e registra quantas entradas havia em `git status`. Assim, o ZIP é um snapshot
reproduzível do estado validado, embora parte desse estado ainda não tenha checkpoint Git isolável.

### Resources e revisions do snapshot

| Resource | Versão | Commit base | Estado capturado |
|---|---:|---:|---|
| `mz_core` | 1.0.0 | `d595b39f7c92` | working tree, 128 entradas; não alterado nesta execução |
| `mz_economy` | 1.0.0 | `cf2c8b1b8389` | working tree, 12 entradas |
| `mz_menu` | 0.4.0-test | `b4cbfa98a9af` | working tree, 6 entradas |
| `mz_interact` | 0.2.0 | `bbd64d9afe7b` | working tree, 3 entradas |
| `mz_vehicles` | 0.1.0 | `e44347f6dd3a` | working tree, 1 entrada |
| `mz_org` | 0.1.0 | `e00eb758ddd8` | working tree, 34 entradas |
| `mz_inventory` | 1.0.0 | `8627a2c067cd` | working tree, 32 entradas |
| `mz_fuel` | 1.0.0 | `942104b7c8ce` | working tree, 8 entradas |
| `mz_garagem` | 0.1.0 | `d5f52518ff1d` | working tree, 12 entradas |
| `mz_police` | 0.7.1 | `137faa85f182` | limpo |
| `mz_prison` | 0.1.0 | `7d1b3b817a06` | limpo |
| starter | n/a | `83943b0c8270` | arquivos runtime selecionados do working tree |

### Integrações policiais ainda não isoláveis

| Owner | Integração efetiva necessária |
|---|---|
| `mz_inventory` | `config.lua`, `shared/*`, owner em `server/*`, `server/police_access.lua`, exports, retenção/restauração prisional e harnesses |
| `mz_org` | `shared/capabilities.lua`, default org policial, domain service, duty e `CanOrg` |
| `mz_economy` | extração de accounts e `server/accounts/police_charges.lua` |
| `mz_starter` | ordem em `cfg/resources.cfg` e ferramentas de instalação/preflight; somente o builder novo foi isolado |
| `mz_testkit` | registry/runner e fixtures centralizadas que sustentam o baseline 51/51 |

O `mz_testkit` não é resource de produção e, por isso, não é copiado para o ZIP. Ele permanece parte
do conjunto de validação local. Os três owners runtime são copiados em seu estado efetivo completo.

## Dependências, boot e publicação

Dependências externas: FXServer, MySQL/MariaDB, `oxmysql`, `ox_lib` e `spawnmanager`.

Ordem do núcleo policial no snapshot:

```text
ensure oxmysql
ensure ox_lib
ensure mz_core
ensure mz_economy
ensure mz_menu
ensure mz_interact
ensure mz_vehicles
ensure mz_org
ensure mz_inventory
ensure mz_fuel
ensure mz_garagem
ensure mz_police
ensure mz_prison
```

O `cfg/resources.cfg` distribuído preserva a ordem completa do servidor. Publicar todos os diretórios
do ZIP como uma unidade; não misturar `mz_police@137faa8` com HEADs limpos antigos de inventory, org
ou economy. O pacote não contém `server.cfg`, `database.cfg`, `permissions.cfg`, `.env`, chaves ou
assinaturas conhecidas de segredo. Esses arquivos permanecem sob gestão da VPS.

### Atualização não destrutiva

1. Abrir janela sem jogadores e registrar backup verificável do banco e dos resources atuais.
2. Validar o SHA-256 do ZIP e todas as entradas de `SHA256SUMS` em diretório temporário.
3. Comparar as configurações locais com os exemplos; não sobrescrever segredos nem coordenadas reais.
4. Executar apenas o preflight de schema abaixo. Migration continua sendo decisão manual.
5. Publicar todos os resources do snapshot juntos e iniciar na ordem acima.
6. Confirmar readiness no console antes de executar os gates e o MDT.
7. Em falha, restaurar todos os diretórios do mesmo snapshot anterior. Não fazer rollback parcial de
   um owner. A coluna aditiva pode permanecer; para rollback integral do banco, restaurar o backup.

## Schema prisional e legado

O schema atual exige `mz_prison_sentences.charge_public_id VARCHAR(160) NOT NULL`. O startup consulta
`information_schema.COLUMNS` e falha fechado com `schema_outdated_missing_charge_public_id` quando a
coluna não existe. O repositório grava o ID público canônico produzido por
`mz_police_case_charges.public_id`.

Preflight somente leitura:

```sql
SELECT COLUMN_NAME, COLUMN_TYPE, IS_NULLABLE
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'mz_prison_sentences'
  AND COLUMN_NAME = 'charge_public_id';

SELECT COUNT(*) AS legacy_rows FROM mz_prison_sentences;
```

Se a coluna já existir como `VARCHAR(160) NOT NULL`, não executar alteração. Se estiver ausente e a
tabela estiver vazia, a alteração aditiva candidata é:

```sql
ALTER TABLE mz_prison_sentences
  ADD COLUMN charge_public_id VARCHAR(160) NOT NULL AFTER case_public_id;
```

Executar somente após backup, revisão humana e janela de manutenção. DDL MySQL pode fazer commit
implícito.

Se existirem linhas legadas, **não executar o ALTER acima automaticamente**. Não há regra segura para
inventar qual acusação P6 originou cada sentença. O procedimento é: exportar as linhas, adicionar a
coluna inicialmente anulável em uma migration revisada, mapear cada registro a um
`mz_police_case_charges.public_id` real e inequívoco, validar zero nulos/órfãos e somente então alterar
para `NOT NULL`. Ambiguidade ou ausência de acusação mantém P7 bloqueado. Nenhum SQL foi executado.

## Configurações obrigatórias e flags

- `mz_org`: `dutyPoint` real da organização `police`, criado pela interface `/stafforg` no local real.
  O contrato exige `enabled`, `coords`, `radius`, `drawDistance` e `label`; nenhum valor foi inventado.
- `mz_police.Stations`: vazio por padrão; operações dependentes de estação/armário ficam indisponíveis.
- `MZPoliceConfig.Records.legalRulesEnabled = false`, `legalVersion = 'unapproved'`, jurisdições e
  catálogos vazios.
- Mandados: `warrants.enabled = false`.
- Associação evidência→caso: `occurrenceAssociationEnabled = false`.
- Prisão em police: `MZPoliceConfig.Prison.enabled = false`.
- `mz_prison`: intake, regras de sentença/redução, soltura manual e trabalhos desativados; pontos de
  intake/release e catálogos vazios.
- Harness P1: `mz_police_p1_gate_enabled=0` por padrão e restrito a staging/console.

## MDT frontend

### Implementação

O comando cliente `/mz_police_mdt` solicita primeiro `records:listCases`. O foco NUI só é adquirido
quando o servidor responde `ok=true`; negações de sessão, organização, duty ou capability não abrem a
interface. Fechamento, ESC e stop do resource liberam o foco.

A NUI usa somente os eventos já existentes:

| Ação da interface | Contrato canônico |
|---|---|
| abrir/listar casos | `mz_police:server:records:listCases` |
| buscar/abrir cidadão | `records:searchCitizens`, `records:getCitizen` |
| salvar ficha | `records:upsertRecord` |
| criar/ler/alterar caso | `records:createCase`, `records:getCase`, `records:updateCase` |
| vincular pessoa | `records:addCasePerson` |
| consultar intake | `mz_police:server:evidence:list` |

Cada contrato acima retorna à policy P1/P6, que resolve identidade/sessão em `mz_core`, membership,
duty e capability em `mz_org`. O payload do navegador nunca escolhe organização ou autoridade. Há
allowlist cliente, correlação de request, bloqueio de duplicidade por ação e timeout de 15 segundos.

### Telas e limites reais

- Dashboard, consulta e ficha do cidadão, casos/ocorrências, acusações, multas, mandados, evidências e
  histórico estão presentes.
- Ficha, casos e vínculo de pessoas usam operações P6 reais.
- Acusações, multas e mandados exibem apenas dados persistidos. A criação fica indisponível porque o
  backend não expõe catálogo jurídico aprovado; a NUI não inventa código, pena ou valor.
- Evidências consultam o intake existente. Associação de evidência a caso continua explicitamente
  indisponível.
- O modo `?preview=1` é isolado, mostra somente estados vazios e identifica “sem dados reais”. Ele não
  contém registros fictícios e não existe como evento público de runtime.
- Lucide é servido localmente; não há CDN ou nova dependência runtime.

## Validação executada

| Validação | Resultado |
|---|---|
| Runner integrado `mz_testkit -Suite covered` | **51/51 PASS** |
| Harnesses Lua `mz_police` P1–P7 | **13 arquivos PASS** |
| Contrato NUI Node | **2/2 PASS** |
| Integrações inventory | **4/4 PASS**, 30 casos + estático |
| Integração org | **1/1 PASS** |
| Integração economy | **1/1 PASS**, 4 casos |
| Harnesses `mz_prison` | **3/3 PASS**, 30 casos |
| Builder do release | **1/1 PASS** |
| `luac -p client/mdt.lua` e `node --check web/app.js` | PASS |
| `git diff --check` nos sete repositórios relevantes | PASS; apenas avisos LF/CRLF |
| SHA256SUMS do RC | **2.327/2.327 PASS** |
| Graphify | atualização escopada de `mz_police`, 367 nós/742 arestas |

QA de navegador: dashboard e navegação de cidadãos foram inspecionados em Chromium com árvore de
acessibilidade; foco, labels, estados vazios, botão de fechar e navegação foram renderizados. O teste
de viewport explícito ficou inconclusivo por erro 403 do serviço de telemetria do controlador, sem
falha observada na página local. Os breakpoints do CSS foram revisados estaticamente. Isso não é teste
CEF dentro do FiveM.

## Arquivos modificados nesta execução

`mz_starter`:

- `tools/build_police_release.ps1`
- `tests/police_release_builder.test.js`

`mz_police`:

- `fxmanifest.lua`
- `client/mdt.lua`
- `web/index.html`
- `web/style.css`
- `web/app.js`
- `web/lucide.min.js`
- `tests/p1_harness.lua`
- `tests/p6_mdt_nui_contract.test.js`

`mz_docs`:

- `police/P1_P7_RELEASE_CANDIDATE_MDT_REPORT.md`

## Pendências exclusivas de runtime

1. Publicar o snapshot em staging e reiniciar na ordem registrada.
2. Validar FXServer + MySQL reais, readiness, restart/reconnect e falhas de dependências.
3. Executar G01–G10 com personagens reais e `dutyPoint` real.
4. Abrir `/mz_police_mdt` em duty e validar busca, ficha, casos, evidência, ESC/foco e negações com
   capacidades reais.
5. Validar o CEF em resoluções efetivamente usadas no servidor.
6. Manter catálogos e P7 desativados até aprovação jurídica/configuração real.

Conclusão: a distribuição está consistente e verificável para **staging**, inclusive quando os owners
ainda possuem integrações não commitadas. Não existe PASS operacional, e a habilitação jurídica ou
prisional permanece bloqueada por configuração e validação reais.
