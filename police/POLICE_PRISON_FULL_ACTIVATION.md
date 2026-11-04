# MZ Police — ativação completa de prisão + dispatch (staging)

> **Documento histórico, substituído pelo [runbook end-to-end](POLICE_DEPLOY_RUNBOOK.md).** As coordenadas candidatas abaixo não estão mais habilitadas no config atual e não devem ser implantadas sem medição no MLO.

Data: 16/10/2026

Estado: **configuração de staging aplicada e validada por harnesses locais. Runtime FiveM: NOT RUN.**

Este documento descreve apenas o que foi realmente habilitado, o que permanece bloqueado,
os dados que faltam e o procedimento de ativação na VPS. Nenhum SQL foi executado, nenhum
deploy/push foi feito e `mz_core` não foi modificado.

> Catálogo `MZ-RP-STAGING-1` é **exclusivo de staging** e não representa legislação real.
> Não usar em produção.
>
> **Atualização P13:** o catálogo atual é `MZ-RP-STAGING-2`, com 40 acusações, 11 multas e 40
> `sentenceRules` espelhadas. Consulte [P13_PENAL_CODE_AND_PATROL.md](./P13_PENAL_CODE_AND_PATROL.md).
>
> **Atualização P14:** a facility de staging `bolingbroke` está habilitada com booking, cela,
> áreas permitidas, rotina e release point. Consulte
> [P14_PRISON_CELLS_AND_EVENTS.md](./P14_PRISON_CELLS_AND_EVENTS.md).

---

## 1. Configurações realmente habilitadas

| Sistema | Arquivo | Estado |
|---|---|---|
| Dispatch P8 | `mz_dispatch/config.lua` | `enabled = true`; `server/prepare.lua` cria/valida o schema automaticamente no boot |
| Prisão (lado policial) | `mz_police/config/server.lua` | `Prison.enabled = true` |
| Catálogo jurídico | `mz_police/config/penal_code.lua` | `legalRulesEnabled = true`, `legalVersion = 'MZ-RP-STAGING-1'` (histórico), `jurisdictions.police.enabled = true` |
| Sentenças | `mz_prison/config.lua` | `sentenceRulesEnabled = true`, mesma versão/jurisdição, 1 regra |
| Intake | `mz_prison/config.lua` | `intakeEnabled = true`, 1 ponto candidato |
| Atividades | `mz_prison/config.lua` | `jobsEnabled = true`, `activityRateLimit` definido, job `rotina` habilitado |
| Soltura manual | `mz_prison/config.lua` | `manualReleaseEnabled = true`, motivo `staging_release` |
| Soltura automática | `mz_prison/config.lua` | fluxo ativo; **teleporte desabilitado** (destino candidato com `enabled = false`) |
| Evidências P5 | `mz_police/config/server.lua` | `occurrenceAssociationEnabled = true` (preservado) e contêiner `intake` |
| Mandados P6 | `mz_police/config/server.lua` | `warrants.enabled = true` preservado, **sem regras** (falha em `legal_rule_not_configured`) |
| P9 Advanced | `mz_police/config/server.lua` | `Advanced.enabled = true`; FLIR `polmav` e rastreamento de viaturas configurados; offset FLIR provisório; câmeras/perícia desativadas; runtime pendente |

### Correções de código comprovadas

1. `mz_prison/server/service.lua` — o `ruleCode` do catálogo prisional só aceitava `[a-z0-9_-]`.
   O `mz_police` persiste o código exatamente como configurado (`MZ_TEST_ARREST`, maiúsculo),
   então **toda** sentença falharia em `rule_not_configured`. O charset foi alinhado ao
   produtor canônico: `^[%w:_-]+$`.
2. `mz_prison/server/service.lua` — o `operationId` gerado pelo `mz_police` é
   `sentence_<session>_sentence:<timer>:<seq>` (contém `:`), mas o `mz_prison` aceitava apenas
   `[%w_-]` em `requestSentence`, `reduceSentence` e `releasePrisoner`. As três validações foram
   alinhadas para `^[%w:_-]+$`, mantendo limite de tamanho e allowlist de caracteres.
3. Nenhuma autorização, capability, custódia, proximidade, catálogo ou idempotência foi relaxada.

Regressão dedicada: `mz_prison/tests/p7_5_staging_catalog_harness.lua` reproduz o formato real
(ruleCode maiúsculo + `operationId` com `:`) e passou.

---

## 2. Catálogo jurídico de teste (idêntico nos dois owners)

`mz_police/config/penal_code.lua` → `Records` (complementando `config/server.lua`):

```lua
legalRulesEnabled = true
legalVersion = 'MZ-RP-STAGING-1'
jurisdictions = { police = { enabled = true } }
charges = {
  MZ_TEST_ARREST = {
    enabled = true,
    name = 'Infração fictícia de teste',
    fineAmount = 0,
    sentenceSeconds = 180
  }
}
```

`mz_prison/config.lua`:

```lua
sentenceRulesEnabled = true
legalVersion = 'MZ-RP-STAGING-1'
jurisdictions = { police = { enabled = true } }
sentenceRules = {
  MZ_TEST_ARREST = {
    enabled = true,
    name = 'Infração fictícia de teste',
    durationSeconds = 180
  }
}
```

Correspondência garantida (e revalidada em runtime por `sentence_rule_mismatch`):

| Campo | mz_police | mz_prison |
|---|---|---|
| ruleCode | `MZ_TEST_ARREST` | `MZ_TEST_ARREST` |
| name | `Infração fictícia de teste` | `Infração fictícia de teste` |
| duração | `sentenceSeconds = 180` | `durationSeconds = 180` |
| versão | `MZ-RP-STAGING-1` | `MZ-RP-STAGING-1` |
| jurisdição | `police` | `police` |

Acusações existentes não foram substituídas nem apagadas: o catálogo apenas adiciona
`MZ_TEST_ARREST` a uma configuração que antes estava vazia.

---

## 3. Coordenadas candidatas (NÃO validadas no mapa/MLO da MZ)

| Uso | X | Y | Z | Heading | Estado |
|---|---|---|---|---|---|
| Delegacia Mission Row (referência) | 450.42 | -981.78 | 30.99 | — | não configurada em `Stations` (sem armaria/catálogo) |
| Intake Bolingbroke | 1847.14 | 2585.90 | 45.68 | — | **habilitado**, `routingBucket = 0`, `radius = 3.0` |
| Área prisional (job `rotina`) | 1691.65 | 2564.11 | 45.56 | — | **habilitado**, `routingBucket = 0`, `radius = 3.0` |
| Saída/soltura Bolingbroke | 1837.14 | 2591.40 | 45.01 | 175.68 | **preparado com `enabled = false`** — sem teleporte |

Regras aplicadas:

- Intake não teleporta: apenas valida proximidade e routing bucket no servidor.
- Soltura **não** teleporta enquanto o destino estiver `enabled = false`; a pena conclui,
  propriedades são devolvidas e o personagem permanece no local.
- Nenhum teleporte para destino ambíguo: o `releasePoint` só escolhe quando existe exatamente
  um ponto válido habilitado para a organização.

### Validação obrigatória antes de habilitar o teleporte de soltura

No FiveM, com um staff em staging:

> O comando `mz_police_p1_gate` foi removido na limpeza estrutural. Registre `x/y/z`,
> `heading` e `routingBucket` com a ferramenta de staff/administração vigente e valide
> manualmente no MLO antes de habilitar o release point.

1. Confirme se Bolingbroke usa MLO customizado e se o interior está no bucket `0`.
2. Confirme piso, colisão, portas e ausência de parede no ponto de saída.
3. Confirme que o ponto de intake é alcançável a pé e compatível com a viatura.
4. Confirme o ponto do job `rotina` (área prisional) e a altura do chão.
5. Só então altere `releasePoints.bolingbroke_exit.enabled = true` e reinicie `mz_prison`.

Se houver MLO/bucket diferente, altere `routingBucket` dos pontos **antes** de testar.

---

## 4. Atividades prisionais

Backend reutilizado: P7.1 (`mz_prison/server/service.lua` + `client/interactions.lua`).
Nada foi duplicado.

Habilitado:

| Job | Duração | Redução | Limite/sentença | Teto de redução | Cooldown | Ponto |
|---|---|---|---|---|---|---|
| `rotina` | 45 s | 30 s | 4 | 120 s | 20 s | `bolingbroke_yard` (candidato) |

Rate limit global: `maximum = 6`, `windowMs = 60000`.

Bloqueadas por falta de coordenada real (cada uma exige `coords.x/y/z`, `radius`, `routingBucket`):

- `limpeza` — falta ponto real (ex.: faxina de cela/corredor).
- `cozinha` — falta ponto real (refeitório/cozinha).
- `lavanderia` — falta ponto real.
- `manutencao` — falta ponto real.

Garantias do backend (inalteradas):

- recompensa e redução calculadas no servidor;
- `withLock` por cidadão, idempotência por `operationId`, cooldown e limites por sentença;
- conclusão exige `os.time() >= available_at` e presença no ponto/bucket;
- cliente só envia `jobCode`/`pointId`; nunca duração ou redução.

---

## 5. Soltura e propriedades

Fluxo canônico (inalterado):

`remaining_seconds = 0` → `release_pending` → `restoreProperty` no `mz_inventory` →
`finishRelease` → `released` → destino (se houver exatamente um válido).

Garantias verificadas no `mz_inventory`:

- retenção idempotente (`prison_property_conflict` em conflito; replay não duplica);
- devolução valida `expectedItemCount` e marcadores `prisonPropertyRestored`;
- `prison_property_count_mismatch` / `prison_property_restore_unverified` bloqueiam soltura
  em vez de perder itens silenciosamente;
- nenhuma soltura indevida: `releaseSentence` só conclui com ator autorizado ou término legítimo.

Soltura manual de staging: `manualReleaseReasons.staging_release` habilitado, exigindo a
capability `reports.approve`. **Não há botão no MDT para soltura manual**; o contrato
server-side (`mz_police:server:prison:releasePrisoner`) existe e é revalidado. A redução
manual também permanece sem regra (`reductionRulesEnabled = false`); a redução operacional
vem das atividades.

---

## 6. Capabilities reais do `mz_org` e matriz `/stafforg`

O seed padrão da organização `police` (`mz_org/server/domain/default_orgs.lua`) concede:

- **base**: `org.view`, `radio.use`, `tablet.open`, `mdt.open`, `storage.open`,
  `police.custody`, `police.search`;
- **por cargo**: `armory.basic` e `vehicle.basic` (nível 1), `patrol.basic` e `police.seize`
  (nível 2), `reports.approve` (nível 6+), entre outras.

**Não** são concedidas pelo seed e precisam ser adicionadas via `/stafforg`:

| Operação | Capability | Obrigatória para |
|---|---|---|
| Registros no MDT / criar caso / acusação | `police.records.write` | fluxo de prisão |
| Solicitar sentença | `police.arrest.request` | intake |
| Redução / soltura manual | `reports.approve` | P7.7 (já existe no nível 6+, confirme o cargo) |
| Apreensão | `police.seize` | revista com apreensão |
| Evidências | `police.evidence.collect`, `police.evidence.manage` | intake/transferência |
| Multas | `police.citation.issue` | P6 multas (sem regras ainda) |
| Mandados | `police.warrant.create`, `police.warrant.approve` | P6 mandados (sem regras ainda) |
| Arsenal | `armory.basic` | P3 (sem catálogo/coords ainda) |
| Viaturas | `vehicle.basic` / `vehicle.medium` | P2/P3 |
| Ferramentas de via | `police.road.tools` | P4 |
| P9 | `police.camera.*`, `police.air.support`, `police.tracker.*`, `police.forensics.request` | P9 (desligado) |

Como conceder: menu staff do `mz_org` → Organizações → cargo → permissões
(`orgs:addGradePermission` → export `AddOrgGradePermission`). Não edite o banco do `mz_org`.

### Ponto de duty (obrigatório)

O seed policial **não** contém `dutyPoint`. Sem ele, `SetSelfOrgDutyAtPoint` retorna
`duty_point_not_configured` e toda operação policial falha em `duty_required`.
Configure pelo menu staff (`org.duty_point.update`) com coordenadas reais:

```lua
dutyPoint = {
  enabled = true,
  coords = { x = <X_REAL>, y = <Y_REAL>, z = <Z_REAL> },
  radius = <1..5>,
  drawDistance = <5..50, >= radius>,
  label = '<ROTULO>'
}
```

---

## 7. SQL — verificação somente leitura e scripts aditivos

### 7.1 Leitura (não altera nada)

```sql
-- Tabelas esperadas
SELECT TABLE_NAME FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
  AND (TABLE_NAME LIKE 'mz_police_%' OR TABLE_NAME LIKE 'mz_prison_%' OR TABLE_NAME LIKE 'mz_dispatch_%')
ORDER BY TABLE_NAME;

-- Colunas críticas
SELECT TABLE_NAME, COLUMN_NAME, COLUMN_TYPE, IS_NULLABLE
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = DATABASE() AND (
  (TABLE_NAME = 'mz_prison_sentences' AND COLUMN_NAME IN
    ('charge_public_id','legal_version','status','remaining_seconds','property_status'))
  OR (TABLE_NAME = 'mz_police_case_charges' AND COLUMN_NAME IN
    ('sentence_seconds','legal_version','rule_code'))
  OR (TABLE_NAME = 'mz_prison_property' AND COLUMN_NAME IN ('status','instance_uid'))
)
ORDER BY TABLE_NAME, COLUMN_NAME;

-- Integridade referencial (órfãos)
SELECT s.public_id FROM mz_prison_sentences s
LEFT JOIN mz_police_cases c ON c.public_id = s.case_public_id
WHERE c.public_id IS NULL LIMIT 50;

SELECT s.public_id FROM mz_prison_sentences s
LEFT JOIN mz_police_case_charges ch ON ch.public_id = s.charge_public_id
WHERE ch.public_id IS NULL LIMIT 50;

-- Estado das sentenças
SELECT status, COUNT(*) FROM mz_prison_sentences GROUP BY status;
SELECT public_id, status, last_error, remaining_seconds, rule_code, legal_version
FROM mz_prison_sentences
WHERE status IN ('intake_pending','release_pending') OR last_error IS NOT NULL
ORDER BY updated_at DESC LIMIT 50;

-- Propriedade retida
SELECT sentence_public_id, COUNT(*) AS items, status
FROM mz_prison_property GROUP BY sentence_public_id, status
ORDER BY sentence_public_id DESC LIMIT 50;

-- Dispatch
SELECT status, COUNT(*) FROM mz_dispatch_calls GROUP BY status;
```

### 7.2 Schema do dispatch: criação automática (SQL manual é fallback)

`mz_dispatch/server/prepare.lua` lê `mz_dispatch/sql/mz_dispatch.sql` do próprio resource e executa
apenas `CREATE TABLE IF NOT EXISTS` no boot, depois de aguardar o MySQL. Em banco saudável não há
DDL: o schema é validado e reutilizado. A validação cobre colunas, tipos, tamanhos críticos, índices
únicos e foreign keys; divergência falha fechado (`schema_invalid` ou `create_failed`) sem `ALTER`
automático.

O arquivo `mz_dispatch/sql/mz_dispatch.sql` permanece canônico e é o fallback manual quando o
usuário MySQL não possui `CREATE`/`REFERENCES` ou quando o operador preferir aplicar o schema antes
do boot.

`mz_police/sql/records_schema_additive.sql` é um espelho manual de fallback: a tabela
`mz_police_forensic_results` já é criada e validada automaticamente por
`mz_police/server/records/schema.lua`. Use o arquivo apenas se o usuário MySQL não puder executar
`CREATE`/`REFERENCES`.

`mz_prison` e os registros de `mz_police` criam o schema em runtime
(`mz_prison/server/prepare.lua`, `mz_police/server/records/schema.lua`), condicionados a
`CREATE TABLE`/`ALTER` do usuário MySQL.

### 7.3 Migração não destrutiva (somente se a verificação apontar schema legado)

`mz_prison` bloqueia a inicialização com `schema_outdated_missing_charge_public_id` quando a
coluna canônica não existe. Não apague sentenças. Procedimento aditivo:

```sql
-- 1) Adicionar a coluna sem remover dados (NULL para linhas antigas)
ALTER TABLE mz_prison_sentences
  ADD COLUMN charge_public_id VARCHAR(160) NULL AFTER case_public_id;

-- 2) Revisar o que ficou sem vínculo antes de qualquer decisão
SELECT public_id, case_public_id, status, target_citizenid
FROM mz_prison_sentences WHERE charge_public_id IS NULL;

-- 3) Só após decisão explícita: encerrar/arquivar sentenças antigas pelo fluxo canônico
--    (ou backfill aprovado a partir de mz_police_case_charges). NÃO usar DELETE/DROP.
```

Se `mz_police_case_charges.sentence_seconds` estiver ausente:

```sql
ALTER TABLE mz_police_case_charges
  ADD COLUMN sentence_seconds INT UNSIGNED NOT NULL DEFAULT 0 AFTER fine_amount;
```

Após qualquer ALTER autorizado, reinicie o owner correspondente.

---

## 8. Funcionalidades ainda bloqueadas e motivo

| Funcionalidade | Motivo | Dado necessário |
|---|---|---|
| Teleporte de soltura | destino não validado | confirmar coords/heading/bucket no MLO de Bolingbroke |
| Jobs `limpeza`, `cozinha`, `lavanderia`, `manutencao` | sem pontos reais | `coords`, `radius`, `routingBucket` por posto |
| Redução manual | sem regra aprovada | `reductionRules.enabled` + `seconds` |
| Multas | sem regras | `citations[ruleCode]` com `fineAmount` |
| Mandados | sem regras | `warrants.rules[ruleCode]` |
| Arsenal/delegacia (P3) | sem armaria/catálogo | `Stations[id].armory.coords` + `Armory.catalog` com itens do inventário |
| P4 flags/apreensões/road tools | sem valores | `flags[code]`, `impoundReasons[code]`, `RoadObjects.types` |
| P9 bodycam/dashcam/CCTV | sem modelos/câmeras | `dashcamVehicles`, `cctv[id]` com coords/rotação/FOV |
| P9 FLIR | sem aeronave/offset | `helicopterModels`, `cameraOffset` |
| P9 tracker | sem modelos | `vehicleModels` |
| P9 perícia | desligada junto com Advanced | habilitar `Advanced.enabled` + `Forensics.enabled` após validar evidências |

`Advanced.enabled` permanece **false** enquanto os pré-requisitos acima estiverem ausentes.
Nenhuma gravação de vídeo é alegada: bodycam/dashcam são estado operacional, sem backend de vídeo.

---

## 9. Arquivos modificados

- `mz_police/config/penal_code.lua` — catálogo de staging;
  `mz_police/config/server.lua` — `Prison.enabled = true` e operação server-side;
  P9 rebaixado para `false` com comentários de pré-requisitos.
- `mz_prison/config.lua` — intake, sentença, atividades, soltura manual e destino candidato.
- `mz_prison/server/service.lua` — charset de `ruleCode` e de `operationId` alinhado ao produtor.
- `mz_prison/tests/p7_static_harness.lua` — passa a validar o contrato de staging.
- `mz_prison/tests/p7_5_staging_catalog_harness.lua` — **novo** teste de regressão de IDs.
- `mz_police/tests/p6_static_harness.lua` — passa a validar o catálogo de staging.
- `mz_dispatch/config.lua`, `mz_dispatch/server/prepare.lua`, `mz_dispatch/server/service.lua`,
  `mz_dispatch/fxmanifest.lua`, `mz_dispatch/tests/*` — bootstrap automático do schema
  (CREATE TABLE IF NOT EXISTS a partir do SQL canônico + validação estrutural + gate fail-closed).
- `mz_dispatch/tests/p8_prepare_harness.lua` — **novo** teste do bootstrap (criação, reutilização,
  schema legado, permissão negada, falha parcial, concorrência e chamada antes do schema pronto).
- `mz_starter/cfg/resources.cfg` + instaladores/check_structure — `ensure mz_dispatch` na ordem
  `prison → dispatch → police` (etapa anterior preservada).

---

## 10. Testes executados

Todos locais; **não substituem runtime FiveM**.

| Suíte | Resultado |
|---|---|
| `mz_dispatch` (4 harness Lua + NUI) | PASS |
| `mz_prison` (6 harness Lua, inclui P7.5 novo) | PASS |
| `mz_police` (17 harness Lua + 9 contratos NUI) | PASS |
| `mz_starter` (node) | PASS (5/5) |
| `luac -p` nos Lua alterados | PASS |
| `node --check` dos JS alterados | PASS |
| `git diff --check` | ver relatório da sessão |

`Runtime FiveM: NOT RUN`. Nenhum item é declarado operacional sem teste real.

---

## 11. Ordem de publicação na VPS

1. Backup recuperável do banco e dos resources.
2. Publicar `mz_dispatch`. O schema é criado/validado automaticamente no boot; aplicar
   `sql/mz_dispatch.sql` manualmente é apenas fallback (usuário MySQL sem `CREATE`/`REFERENCES`).
3. Confirmar as 3 tabelas e os logs `schema_ready=true` (seções 7.1 e 7.2).
4. Publicar as pastas atualizadas: `mz_prison`, `mz_dispatch`, `mz_police`, `mz_starter`,
   `mz_phone`/`mz_medical` (adapters já existentes).
5. Confirmar a ordem de boot: `... → mz_garagem → mz_prison → mz_dispatch → mz_police → mz_medical → mz_phone`.
6. Reiniciar os resources (seção 12) e conferir os logs.
7. No `/stafforg`: configurar `dutyPoint` e conceder as capabilities da seção 6.
8. Validar coordenadas candidatas (seção 3) antes de habilitar o teleporte de soltura.
9. Executar a primeira prisão (seção 13).

### Logs esperados no console

```text
[mz_prison] ready clock=online intake_enabled=true rules_enabled=true
[mz_dispatch] database connection ready
[mz_dispatch] schema initialization started
[mz_dispatch] schema validated
[mz_dispatch] schema_ready=true enabled=true
[mz_police][P6] initialized; organization_ready=true records_ready=true stations=0
```

Se aparecer `[mz_prison] initialization blocked: ...`, o schema não foi criado (ver seção 7.3).
Se aparecer `[mz_dispatch] schema initialization failed: ...`, leia o motivo: permissão `CREATE`,
arquivo SQL ausente, banco indisponível ou estrutura incompatível. O serviço permanece fail-closed.
Aplicar `sql/mz_dispatch.sql` manualmente resolve o caso de permissão.

---

## 12. Comandos de restart

```text
restart mz_prison
restart mz_dispatch
restart mz_police
restart mz_medical
restart mz_phone
```

Ordem importa: `mz_prison` antes de `mz_dispatch`, e ambos antes de `mz_police`.
Em restart completo do FXServer, `mz_starter/cfg/resources.cfg` já garante a ordem.

---

## 13. Primeira prisão real (passo a passo)

Pré-requisitos: dois personagens reais; policial com `dutyPoint` configurado, em duty, e com
`mdt.open`, `police.custody`, `police.search`, `police.records.write`, `police.arrest.request`
(mais `police.seize` se for apreender).

1. Policial entra em duty no ponto real do `mz_org`.
2. Aproxima-se do cidadão e usa `mz_target` (ALT ESQ) ou o radial rápido.
3. **Algemar** → **Revistar** (opcional) → **Escoltar** ou **Colocar na viatura**.
4. **Abrir ficha no MDT** pelo radial (o `targetSource` só existe nesse caminho).
5. No MDT: criar/selecionar ocorrência e adicionar o cidadão como `subject` se necessário.
6. Aba **Acusações** → **Carregar catálogo** → selecionar `MZ_TEST_ARREST` → **Adicionar ao caso**.
7. Na lista de acusações com pena, clicar **Preparar intake**.
8. Levar o custodiado (algemado) até `1847.14, 2585.90, 45.68` e usar a interação espacial
   de recebimento. O servidor revalida organização, duty, capability, custódia, proximidade,
   bucket, caso, acusação e catálogo.
9. Confirmação esperada: “Entrada prisional confirmada e sentença iniciada.” A custódia é
   encerrada e a sentença inicia com 180 s.
10. No presídio, ir ao ponto do job `rotina` (`1691.65, 2564.11, 45.56`) e concluir a atividade
    (45 s). Cada conclusão reduz 30 s, até 4 vezes / 120 s.
11. Consultar a sentença no MDT (**Situação prisional** → Consultar) para confirmar
    `status` e `remaining_seconds`.
12. Ao zerar o tempo (ou pela atividade), a soltura conclui, as propriedades são devolvidas e
    o personagem é liberado **sem teleporte** (destino desabilitado).
13. Repetir o teste com reconnect durante a sentença e com restart de `mz_prison` para validar
    retomada (`resumeSource` / `tickOnce`).

### Verificação de permissões antes do teste

O comando `mz_police_p1_gate` foi removido. Valide as capabilities
`police.records.write`, `police.arrest.request` e `reports.approve` pelo fluxo de
staff/`mz_org` e confirme o resultado pelos logs de auditoria de `mz_police`.

---

## 14. Limites e segurança

- Nenhum bypass de capability, organização, duty, custódia, proximidade, catálogo ou idempotência.
- Nenhum comando administrativo substitui o gameplay.
- Nenhum menu paralelo foi criado.
- `mz_core` não foi alterado.
- SQL/migration/deploy não foram executados.
- Runtime FiveM continua **NOT RUN** até evidência real no FXServer.
                                                                                                                        