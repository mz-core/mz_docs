# MZ Police / MZ Prison — matriz de schema e paridade de bootstrap

Data: 16/10/2026

Estado: bootstrap automático padronizado em `mz_police`, `mz_prison` e `mz_dispatch`.
**Runtime FiveM/MySQL: NOT RUN.** Este documento é evidência estática + harnesses locais.

---

## 1. Como o bootstrap funciona agora

Os três resources usam o mesmo padrão:

1. aguardam o MySQL (`SELECT 1`, até 120 tentativas de 500 ms);
2. derivam o schema esperado do **mesmo DDL** usado para criar;
3. validam `information_schema` antes de executar DDL (banco saudável não recebe DDL);
4. se faltar tabela/coluna/índice/FK, executam apenas `CREATE TABLE IF NOT EXISTS`;
5. validam novamente e só então marcam `ready = true`;
6. qualquer divergência falha fechado, sem `ALTER` automático.

| Resource | Fonte única de DDL | Gate das operações |
|---|---|---|
| `mz_dispatch` | `mz_dispatch/sql/mz_dispatch.sql` (lido do resource) | `dispatch_unavailable` |
| `mz_police` | `server/records/schema.lua` (lista `statements`) | `records_unavailable` |
| `mz_prison` | `server/prepare.lua` (lista `statements`) | `prison_unavailable` |

O contrato de validação é derivado do DDL (colunas, tipo, tamanho, nulabilidade, PK, UNIQUE e FK),
não é uma segunda DDL escrita à mão. O harness de bootstrap lê o DDL real e falha se o contrato
divergir.

---

## 2. Matriz de tabelas — mz_police

Todas são criadas automaticamente e validadas. Nenhuma exige SQL manual.

| Tabela | Estrutura crítica validada | Validação | Bloqueio se divergir |
|---|---|---|---|
| `mz_police_records` | PK `(org_code, citizenid)` | colunas/tipos/nulabilidade/PK | `schema_invalid` |
| `mz_police_cases` | PK `public_id` | colunas/tipos/tamanhos/PK | `schema_invalid` |
| `mz_police_case_officers` | PK `(case_public_id, citizenid)`; FK → `mz_police_cases.public_id` | colunas/PK/FK | `schema_invalid` |
| `mz_police_case_people` | PK `(case_public_id, citizenid)`; FK → `mz_police_cases.public_id` | colunas/PK/FK | `schema_invalid` |
| `mz_police_case_charges` | PK `public_id`; FK → `mz_police_cases.public_id` | colunas/PK/FK | `schema_invalid` |
| `mz_police_case_evidence` | PK `(case_public_id, evidence_id)`; UNIQUE `(case_public_id, instance_uid)`; FK → cases | colunas/PK/UNIQUE/FK | `schema_invalid` |
| `mz_police_forensic_results` (P9) | PK `public_id`; UNIQUE `operation_id`; FK → cases | colunas/PK/UNIQUE/FK | `schema_invalid` |
| `mz_police_citations` | PK `public_id`; UNIQUE `operation_id` | colunas/PK/UNIQUE | `schema_invalid` |
| `mz_police_warrants` | PK `public_id` | colunas/PK | `schema_invalid` |
| `mz_police_history` | PK `id`; UNIQUE `operation_id` | colunas/PK/UNIQUE | `schema_invalid` |

Observações:
- `mz_police_forensic_results` (P9) **já é criada automaticamente** por `server/records/schema.lua`.
  `sql/records_schema_additive.sql` passou a ser apenas espelho de fallback.
- Operações de escrita/leitura de registros checam `RecordsPrepare.ready` em
  `server/records/service.lua`, `server/evidence.lua` e `server/advanced.lua`.

---

## 3. Matriz de tabelas — mz_prison

Todas são criadas automaticamente e validadas. Nenhuma exige SQL manual.

| Tabela | Estrutura crítica validada | Validação | Bloqueio se divergir |
|---|---|---|---|
| `mz_prison_sentences` | PK `public_id`; UNIQUE `operation_id`; coluna `charge_public_id` | colunas/tipos/tamanhos/nulabilidade/PK/UNIQUE | `schema_outdated_missing_charge_public_id` (coluna específica) ou `schema_invalid` |
| `mz_prison_sentence_charges` (P7.4B) | PK `(sentence_public_id, charge_public_id)`; UNIQUE `charge_public_id`; FK → `mz_prison_sentences.public_id` | colunas/PK/UNIQUE/FK | `schema_invalid` |
| `mz_prison_property` | PK `(sentence_public_id, retained_slot)`; FK → `mz_prison_sentences.public_id` | colunas/PK/FK | `schema_invalid` |
| `mz_prison_history` | PK `id`; UNIQUE `operation_id`; FK → `mz_prison_sentences.public_id` | colunas/PK/UNIQUE/FK | `schema_invalid` |
| `mz_prison_activities` | PK `public_id`; UNIQUE `operation_id`; FK → `mz_prison_sentences.public_id` | colunas/PK/UNIQUE/FK | `schema_invalid` |
| `mz_prison_release_deliveries` | PK `sentence_public_id`; FK → `mz_prison_sentences.public_id` | colunas/PK/FK | `schema_invalid` |

Todas as funções de `server/service.lua` (sentença, leitura, redução, soltura, atividades,
release ack, resume e tick) começam com o gate `prison_unavailable`.

---

## 4. Matriz de tabelas — mz_dispatch (referência)

| Tabela | Estrutura crítica | Criação/validação |
|---|---|---|
| `mz_dispatch_calls` | PK `id`; UNIQUE `public_id` | automática |
| `mz_dispatch_assignments` | PK `id`; UNIQUE `(call_public_id, responder_citizenid)`; FK → calls | automática |
| `mz_dispatch_history` | PK `id`; UNIQUE `operation_id`; FK → calls | automática |

---

## 5. Checklist de paridade (requisitos da auditoria)

| # | Requisito | mz_dispatch | mz_police | mz_prison |
|---|---|---|---|---|
| 1 | Cria todas as tabelas automaticamente | sim (3) | sim (10) | sim (6) |
| 2 | Prepara schema antes de aceitar operações | sim | sim (`records_unavailable`) | sim (`prison_unavailable`) |
| 3 | Valida colunas/tipos/índices/UNIQUE/FK | sim | sim | sim |
| 4 | Preserva tabelas/dados existentes | sim | sim | sim |
| 5 | Schema antigo incompatível falha fechado | sim | sim | sim |
| 6 | Aguarda oxmysql | sim | sim | sim |
| 7 | Registra falha de MySQL | sim | sim | sim |
| 8 | Restart reutiliza dados | sim | sim | sim |
| 9 | Tabela P1–P9 exigindo SQL manual | nenhuma | nenhuma | nenhuma |
| 10 | Chamada antes de `schema_ready` | bloqueada | bloqueada | bloqueada |

---

## 6. Diferenças comprovadas e corrigidas nesta auditoria

1. **`mz_police` não aguardava o MySQL.** `server/records/schema.lua` só verificava a presença da API;
   se a conexão não estivesse pronta, o primeiro boot falhava e exigia restart. Agora há probe +
   retry e `main.lua` chama `bootstrap()` em thread.
2. **`mz_police` não validava estrutura.** Uma tabela antiga sem `sentence_seconds` (ou com
   tamanho/tipo divergente) era considerada pronta e falhava em runtime com `persistence_failed`.
   Agora valida colunas, tipos, tamanhos, nulabilidade, PK, UNIQUE e FKs, e falha fechado no boot.
3. **`mz_prison` validava apenas `charge_public_id`.** Outras colunas/índices/FKs podiam estar
   ausentes. Agora valida o schema completo (mantendo o código específico
   `schema_outdated_missing_charge_public_id`).
4. **`mz_prison` não bloqueava operações antes do schema.** As exports chamavam o `Service`
   diretamente. Agora todas as funções checam `Prepare.ready` e retornam `prison_unavailable`.
5. **`mz_police/sql/records_schema_additive.sql` era tratado como passo manual obrigatório**, mas a tabela já era
   criada por `server/records/schema.lua`. O arquivo e a documentação foram corrigidos para fallback.

Nenhuma validação de segurança foi removida. Nenhum `DROP`/`DELETE`/`TRUNCATE`/`ALTER` é
executado automaticamente.

---

## 7. O que ainda exige intervenção manual

- **Criação de tabelas:** nada. Os três resources preparam o próprio schema quando o usuário
  MySQL tem `CREATE` (e `REFERENCES` para FKs).
- **Permissão negada:** aplicar o SQL canônico manualmente (`mz_dispatch/sql/mz_dispatch.sql`,
  `mz_police/sql/records_schema_additive.sql` apenas para a tabela P9) ou conceder `CREATE`/`REFERENCES`.
- **Schema legado incompatível:** o bootstrap **não** altera tabelas. É necessário revisar o
  problema reportado e aplicar uma migração aditiva aprovada manualmente. Exemplo já documentado:
  adicionar `mz_prison_sentences.charge_public_id` sem apagar sentenças
  (`POLICE_PRISON_FULL_ACTIVATION.md`, seção 7.3).
- **Migração de dados:** sempre manual e sob aprovação; nenhum backfill é automático.

---

## 8. Validação executada

| Suíte | Resultado |
|---|---|
| `mz_police/tests/p6_prepare_harness.lua` (novo) | PASS — 10 casos |
| `mz_prison/tests/p7_prepare_harness.lua` (reescrito) | PASS — 10 casos |
| Harnesses `mz_police` | PASS |
| Harnesses `mz_prison` | PASS |
| Harnesses `mz_dispatch` | PASS |
| Runner global `mz_testkit -Suite covered` | PASS — 51/51 |
| `luac -p` nos arquivos alterados | PASS |
| `git diff --check` | PASS |

Os harnesses de bootstrap cobrem: banco vazio, criação de todas as tabelas, reutilização/restart,
preservação de dados, coluna ausente, UNIQUE ausente, FK ausente, permissão `CREATE` negada,
falha parcial + retry, inicialização simultânea, logs e chamada antes de `schema_ready`.
