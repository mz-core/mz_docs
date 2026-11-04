---
title: MZ Police — P7.4 prisão rápida, casos automáticos e multas
status: IMPLEMENTATION_REPORT
authority: LEVEL_4_REFERENCE
owner: MZ Police
last_reviewed: 2026-10-17
applies_to:
  - mz_police
  - mz_prison
  - mz_target
  - mz_menu
  - mz_org
  - mz_economy
  - mz_docs
related_contracts:
  - MZ-IDENTITY-v2
  - MZ-TARGET-v1
---

# MZ Police — P7.4 prisão rápida, casos automáticos e multas

Data: 17/10/2026

Estado: **P7.4A, P7.4B e P7.4C implementadas no gate estático.** Runtime FiveM/OneSync/CEF/MySQL:
**NOT RUN**.

Nenhum `mz_core`, deploy, push, SQL ou migration foi executado. A migration aditiva da P7.4B foi
preparada em arquivo e **não executada**. As alterações preexistentes dos working trees foram
preservadas; não foi criada branch/worktree.

---

## 1. Resultado

A prisão comum agora pode ser feita pelo radial existente:

**Algemar → Prender → selecionar acusação → confirmar → encaminhar ao intake.**

O backend continua sendo o mesmo:

- `mz_police` é o owner de custódia, caso, pessoa, acusação, histórico e orquestração da prisão;
- `mz_prison` é o owner de sentença, intake, propriedade, atividades e soltura;
- `mz_inventory` continua o único owner de itens e propriedade retida;
- `mz_economy` continua o único writer financeiro;
- `mz_target` continua o único radial;
- `mz_menu` continua a interface contextual/dialogs;
- o MDT continua dentro de `mz_police/web`.

Não foi criado `mz_arrest`, `mz_mdt2`, `mz_prison2`, novo radial ou novo banco de ocorrências.

### Estados das sub-waves

| Sub-wave | Estado | Observação |
|---|---|---|
| P7.4A — uma acusação | Implementada; gate estático PASS | Fluxo radial + caso + charge + intake P7 |
| P7.4B — até três acusações | Implementada, **desligada por configuração** | Exige política de agregação aprovada |
| P7.4C — multas | Implementada | Citation pendente + pagamento idempotente separado |
| Confinamento | Fora do escopo | Não iniciado |
| Uniforme | Fora do escopo | Não iniciado |
| Transporte NPC | Fora do escopo | Não iniciado |

---

## 2. Arquivos alterados

### `mz_police`

Novos:

- `server/arrest.lua` — orquestração server-side de preview/quote/commit, caso atômico,
  idempotência e cálculo de totais;
- `client/arrest.lua` — formulário compacto via `mz_menu`, confirmação e feedback;
- `tests/p7_4_quick_arrest_harness.lua`;
- `tests/p7_4_multi_and_fines_harness.lua`;
- `tests/p7_4_quick_arrest_contract.test.js`;
- `tests/p7_4_multi_fines_contract.test.js`.

Modificados:

- `config/server.lua` — bloco `Arrest`, gate `multiChargeEnabled`, limites;
- `server/events.lua` — eventos `arrest:preview|quote|commit` e `citations:pay`;
- `server/interactions.lua` — ação `arrest` no contexto server-side;
- `client/interactions.lua` — ação **Prender** no radial e mensagens;
- `server/records/service.lua` — catálogo interno, citations de prisão e pagamento;
- `server/records/repository.lua` — transação `createQuickArrest`, leitura de citation por ID;
- `server/prison.lua` — intake agregado por caso, compatível com o fluxo de uma acusação;
- `server/bridge.lua` — `RequestSentenceFromCharges`;
- `server/audit.lua` — logs `police.arrest.*` e `chargeCount` no log prisional;
- `server/custody.lua` — `instanceId` imutável por custódia para idempotência;
- `server/main.lua` — cleanup do módulo de prisão rápida;
- `fxmanifest.lua` — carregamento dos novos arquivos;
- `web/app.js` — policiais responsáveis, histórico do caso, marcador de prisão rápida e multa na acusação;
- `tests/p8_target_access_harness.lua` — nova ação no radial.

### `mz_prison`

Novos:

- `sql/p7_4_additive.sql` — migration aditiva preparada, não executada;
- `tests/p7_4_multi_charge_harness.lua`.

Modificados:

- `config.lua` — bloco `multiCharge` desligado por padrão;
- `server/prepare.lua` — tabela `mz_prison_sentence_charges`;
- `server/repository.lua` — vínculo de charges, consulta por charge e criação multi;
- `server/service.lua` — `requestSentenceFromCharges`, replay, ativação compartilhada e retry;
- `server/exports.lua` — export `RequestSentenceFromCharges`;
- `tests/p7_prepare_harness.lua` — 6 tabelas.

### `mz_docs`

- `police/P7_4_QUICK_ARREST_IMPLEMENTATION.md` — este relatório;
- `police/POLICE_PRISON_SCHEMA_MATRIX.md` — tabela P7.4B adicionada;
- `INDEX.md` — referência aos relatórios P7.4 e de viabilidade.

---

## 3. Contratos usados

### 3.1 Contratos existentes reutilizados

- `mz_core:ResolvePlayerIdentity`, `GetDeathState`, `GetSourceByCitizenId`;
- `mz_org:GetPlayerOrgContext`, `CanOrg`, `GetOrganizationReadiness`;
- `mz_police:custody:request`, `Custody.get`;
- `mz_police:records:*` e `RecordsRepository`;
- `mz_police:prison:requestSentence` e `Prison.requestSentence`;
- `mz_prison:RequestSentence`, `GetSentence`;
- `mz_police_citations` + `mz_economy:SettlePlayerOrganizationCharge`;
- `mz_target:RegisterOption`;
- `mz_menu:InputDialog`, `AlertDialog`, `Notify`.

### 3.2 Contratos novos

| Contrato | Owner | Tipo | Mutação |
|---|---|---|---|
| `mz_police:server:arrest:preview` | `mz_police` | evento server-side | Não |
| `mz_police:server:arrest:quote` | `mz_police` | evento server-side | Não |
| `mz_police:server:arrest:commit` | `mz_police` | evento server-side | Sim |
| `mz_police:server:citations:pay` | `mz_police` | evento server-side | Sim, financeiro idempotente |
| `mz_prison:RequestSentenceFromCharges` | `mz_prison` | export server-only | Sim |
| `mz_prison_sentence_charges` | `mz_prison` | tabela aditiva | Persistência |
| `mz_police.Records.issueArrestCitations` | `mz_police` | método interno | Citation pendente |
| `mz_police.Records.payCitation` | `mz_police` | método interno/evento | Liquidação |

Nenhuma capability nova foi criada. O fluxo usa:

- `police.records.write` — abrir/confirmar o registro;
- `police.arrest.request` — iniciar a sentença;
- `police.citation.issue` — emitir citation de multa;
- `police.custody` — algemar/manter custódia;
- duty e organização canônicos do `mz_org`.

---

## 4. Ação no radial

O `mz_target` continua sendo o único radial. Em `mz_police/client/interactions.lua`:

- o submenu existente `police_player_actions` recebeu o filho `arrest`, com label **Prender**;
- o filho aparece quando o servidor retorna `actions.arrest = true`;
- o servidor só retorna `arrest = true` quando:
  - o alvo está sob custódia do policial (`cuffedBy`);
  - o alvo está algemado;
  - o policial tem `police.arrest.request`;
  - o policial tem `police.records.write`.
- o clique executa `MZPoliceClient.OpenQuickArrest`, não apenas uma mudança visual;
- o `mz_target` executa o `onSelect` depois de encerrar o modo de mira; o `mz_menu` assume o
  foco do formulário.

O fluxo não cria outro radial, outra tecla ou outra biblioteca de menu.

---

## 5. Formulário compacto

### 5.1 P7.4A (estado padrão)

`mz_menu:InputDialog` com:

1. **Acusação** — `select` obrigatório, carregado do catálogo server-side;
2. **Observação** — `textarea` opcional, limite de 4000 caracteres.

Cada opção do select exibe nome, pena e multa calculados pelo servidor:

`Infração fictícia de teste · 180s · R$ 0`

Depois do select, o cliente pede `arrest:quote` ao servidor e exibe um `AlertDialog`:

`Pena: 180s · Multa: R$ 0 · 1 acusação(ões)`

Somente depois do `Confirmar` o cliente envia `arrest:commit`.

### 5.2 P7.4B (quando habilitado)

O mesmo formulário passa a montar até três `select`s (o primeiro obrigatório; os demais com opção
**Nenhuma**). O servidor retorna `limits.maxCharges`; o cliente não inventa o limite.

A NUI **nunca** envia:

- duração;
- multa;
- nome de regra;
- versão legal;
- total de pena.

Ela envia apenas `targetSource`, `ruleCodes`, `notes`, `expectedCustodyRevision` e `requestId`.

---

## 6. Fluxo de registro e sentença

### 6.1 Preview

`arrest:preview`:

- resolve o policial por identidade/sessão;
- valida organização, duty e capabilities;
- resolve o alvo pela custódia (não por citizenid do cliente);
- exige cuffed + `cuffedBy` do mesmo policial;
- valida vida, distância e routing bucket;
- consulta a sentença atual no `mz_prison` (falha fechada se indisponível);
- retorna alvo, catálogo, revisão de custódia, limite de charges e flag de citation.

### 6.2 Quote

`arrest:quote`:

- revalida custódia;
- valida cada `ruleCode` no catálogo server-side;
- recalcula nome, pena e multa;
- rejeita código inexistente, repetido ou sem pena;
- exige `police.citation.issue` quando a multa total é maior que zero;
- não persiste nada.

### 6.3 Commit

`arrest:commit`:

1. valida payload fechado, notas e revisão de custódia;
2. revalida policy, alvo, custódia, vida, distância e bucket;
3. recalcula o catálogo e a sentença ativa;
4. usa `instanceId` da custódia + `targetCitizenid` como chave natural;
5. cria um `caseId` determinístico;
6. adquire lock por cidadão;
7. revalida custódia, revisão e sentença ativa dentro do lock;
8. verifica replay pelo histórico;
9. em transação única, grava:
   - `mz_police_cases`;
   - `mz_police_case_officers` (criador);
   - `mz_police_case_people` (subject);
   - uma linha em `mz_police_case_charges` por acusação;
   - `mz_police_history` com `quick_arrest_created`;
10. cria citations pendentes para charges com multa > 0;
11. audita `police.arrest.committed`;
12. retorna `caseId`, `chargeIds`, totais e avisos.

O caso é um `occurrence` comum. Não existe tabela de “caso simplificado”.

### 6.4 Encaminhamento

O cliente guarda `PendingIntake` com `targetSource`, `citizenid`, `caseId` e, quando há uma única
acusação, `chargeId`. O policial transporta o custodiado e usa o ponto de intake existente.

No intake:

- se `chargeId` foi informado, o fluxo P7 de uma acusação é usado;
- se não foi, o servidor agrega todas as charges **com pena** daquele caso/cidadão;
- a sentença só é criada depois de nova validação de intake, catálogo, custódia e bucket.

A custódia não é encerrada na prisão rápida; ela é encerrada somente no intake, como antes.

---

## 7. Caso automático, idempotência e retry

- `operationId = arrest:<policial>:<hash(instanceId + target)>`
- `caseId = case:<policial>:arrest:<hash>`
- `chargeId = charge:<hash>`
- O `custody.instanceId` é criado no ato do algemamento e não muda com escolta/viatura; uma nova
  custódia gera outro ID.
- O histórico único em `mz_police_history` fecha a operação.
- Replay do mesmo contexto retorna `quick_arrest_replayed`, sem criar segundo caso.
- Conteúdo divergente no mesmo `instanceId` retorna `request_id_conflict`.
- Duplo clique é serializado por lock por cidadão.
- Se o caso foi criado e a resposta se perdeu, o retry recupera o caso pelo histórico.
- A operação de sentença é determinística por caso + charge(s), inclusive no fluxo de uma única
  acusação; um retry após timeout/restart recupera a sentença existente em vez de criar outra.
- Se a citation falhou após o caso, o retry tenta novamente a citation de forma idempotente e
  devolve aviso; o caso não é duplicado.

---

## 8. P7.4B — múltiplas acusações

### 8.1 Política

O suporte está implementado, mas **desligado**:

```lua
-- mz_police/config/server.lua
Arrest = {
  maximumCharges = 3,
  multiChargeEnabled = false
}

-- mz_prison/config.lua
multiCharge = {
  enabled = false,
  maximumCharges = 3,
  aggregation = 'sum'
}
```

Para habilitar, as duas flags precisam ser aprovadas e ligadas em conjunto. A única política
implementada é `sum`; qualquer outro valor retorna `aggregation_policy_not_supported`. Não foi
inventado limite total de pena; `maximumCharges` limita apenas a quantidade de linhas por sentença.

### 8.2 Sentença

`mz_prison_sentence_charges`:

- PK `(sentence_public_id, charge_public_id)`;
- UNIQUE `charge_public_id`;
- FK para `mz_prison_sentences`;
- guarda `rule_code`, `rule_name`, `legal_version`, `sentence_seconds`, `is_primary`.

`RequestSentenceFromCharges`:

- valida ator, alvo, intake, organização, jurisdição e versão legal;
- valida cada charge contra `MZPrisonConfig.sentenceRules`;
- soma as penas;
- rejeita charge repetida, charge já sentenciada e segunda sentença ativa;
- cria uma única sentença com `duration_seconds = soma`;
- grava as linhas de vínculo na mesma transação;
- retém propriedade e ativa como o fluxo P7 existente.

O fluxo de uma acusação continua usando `RequestSentence`; novas sentenças de uma acusação também
gravam uma linha de vínculo com `is_primary = 1`.

### 8.3 Compatibilidade

- sentenças antigas continuam válidas;
- `charge_public_id`, `rule_code` e `rule_name` continuam na linha principal;
- sentenças antigas não possuem linhas em `mz_prison_sentence_charges`;
- não foi executado backfill; o MDT usa os campos principais como fallback;
- para que sentenças antigas também bloqueiem reutilização de charge, um backfill manual futuro
  precisa ser aprovado. Nenhum dado foi alterado nesta entrega.

### 8.4 Migration

`mz_prison/sql/p7_4_additive.sql` contém apenas `CREATE TABLE IF NOT EXISTS`. Não foi executado.
O bootstrap do `mz_prison` também cria/valida a tabela no boot quando o usuário MySQL tem
`CREATE`/`REFERENCES`.

---

## 9. P7.4C — multas

- O commit da prisão cria **citations pendentes** para cada charge com `fineAmount > 0`;
- a citation usa `mz_police_citations`, com `public_id = citation:<chargeId>` e
  `operation_id = charge_fine:<chargeId>`;
- a criação exige `police.citation.issue`;
- **não há débito automático** na prisão;
- o pagamento é um contrato separado: `mz_police:server:citations:pay`;
- o pagamento é feito pelo próprio cidadão, resolve identidade canônica, exige citation pendente
  do próprio cidadão e usa `SettlePlayerOrganizationCharge` com chave idempotente estável;
- replay de pagamento retorna `citation_paid` sem segunda movimentação;
- falha de saldo mantém a citation `pending` e registra o erro;
- o catálogo de staging atual tem multa R$ 0, então nenhuma citation real é criada no fluxo de
  teste.

A UI de pagamento pelo cidadão não foi criada nesta wave; o contrato server-side está pronto e
testado. O MDT já lista citations do cidadão na tela existente.

---

## 10. MDT

A prisão rápida não cria um segundo banco nem uma segunda NUI.

O caso aparece em `listCases` e `getCase` como qualquer ocorrência. A tela de caso ganhou:

- painel **Policiais responsáveis** com `mz_police_case_officers`;
- painel **Histórico do caso** com `mz_police_history`;
- aviso **Registro criado pela prisão rápida** quando há `quick_arrest_created`;
- exibição da multa na linha da acusação.

O MDT continua permitindo:

- consultar cidadão, caso, acusações e sentença;
- vincular evidências P5 posteriormente;
- usar perícia P9 quando habilitada;
- criar mandados quando houver regra;
- complementar o caso manualmente.

---

## 11. Configuração operacional

### 11.1 Pré-requisitos

- `mz_core`, `mz_org`, `mz_inventory`, `mz_economy`, `mz_prison`, `mz_target`, `mz_menu` iniciados;
- policial com personagem/sessão ativos, organização autorizada e duty;
- capabilities:
  - `police.custody`;
  - `police.records.write`;
  - `police.arrest.request`;
  - `police.citation.issue` (quando houver multa).
- ponto de intake do `mz_prison` configurado e dentro do routing bucket esperado.

### 11.2 Catálogo

O catálogo atual é `MZ-RP-STAGING-1` e a acusação `MZ_TEST_ARREST` tem multa R$ 0. Não é
legislação de produção. O operador deve substituir por um catálogo aprovado, mantendo o espelho
entre `mz_police` e `mz_prison`.

### 11.3 Ligar P7.4B

Somente depois de aprovar a política de agregação:

```lua
-- mz_police/config/server.lua
Arrest.multiChargeEnabled = true

-- mz_prison/config.lua
multiCharge.enabled = true
```

Manter `maximumCharges = 3` ou reduzir, conforme a política aprovada. Não alterar
`aggregation = 'sum'` sem uma nova implementação.

### 11.4 P7.4C

O catálogo de teste não tem multa. Para testar citation, usar uma regra de teste com
`fineAmount > 0` e conceder `police.citation.issue` ao cargo. Não usar valores de produção
inventados.

---

## 12. Testes executados

### 12.1 `mz_police`

- 23 arquivos de harness Lua: **PASS**;
- 6 arquivos de contrato Node: **19 testes PASS**;
- novos:
  - `p7_4_quick_arrest_harness.lua` — 14 casos;
  - `p7_4_multi_and_fines_harness.lua` — 11 casos;
  - `p7_4_quick_arrest_contract.test.js` — 3 testes;
  - `p7_4_multi_fines_contract.test.js` — 2 testes;
- regressões P1–P9, custódia, registros, prisão, target, dispatch e P9: **PASS**.

Cobertura nova inclui:

- prisão com uma acusação;
- caso automático com pessoa/charge/histórico;
- replay do mesmo commit;
- conteúdo divergente no mesmo `instanceId`;
- ausência de capability;
- alvo não algemado;
- alvo fora de alcance;
- catálogo desativado;
- acusação inválida;
- revisão de custódia divergente;
- sentença ativa;
- observação acima do limite;
- pena enviada pelo cliente (payload fechado);
- falha de persistência;
- falha de retenção no intake e retry;
- múltiplas acusações quando habilitadas;
- citation pendente, pagamento idempotente e tentativa de pagamento alheio.

### 12.2 `mz_prison`

- 7 arquivos de harness Lua: **PASS**;
- novo `p7_4_multi_charge_harness.lua`: 12 casos;
- regressões P7.1, P7.2, P7.5, prepare, service e static: **PASS**.

### 12.3 Sintaxe e diff

- `luac -p` nos arquivos Lua alterados: **PASS**;
- `node --check web/app.js`: **PASS**;
- `git diff --check` em `mz_police`, `mz_prison` e `mz_docs`: **PASS**, apenas avisos de
  normalização LF/CRLF do Git.

### 12.4 Runner global

`mz_testkit` `-Suite covered` **não concluiu** nesta sessão: falhou em teste preexistente do
`mz_inventory` (`server/service.lua:2510`, global `Player` nil) antes de chegar aos novos testes.
O `mz_inventory` não foi alterado nesta wave. O resultado não é declarado PASS global.

### 12.5 Runtime

**NOT RUN.** Não há evidência de:

- FXServer;
- OneSync/culling;
- CEF/foco do `mz_menu` ou do MDT;
- MySQL real/transações;
- restart/reconnect;
- dois jogadores reais;
- intake real;
- citation/economy real.

---

## 13. Como prender pelo radial

1. Entre em duty no ponto real da organização.
2. Confirme que o cargo tem `police.custody`, `police.records.write` e `police.arrest.request`
   (e `police.citation.issue` se a acusação tiver multa).
3. Aproxime-se do cidadão e use o target (`ALT ESQ`) ou o quick radial (`F3`).
4. Escolha **Ações policiais → Algemar**.
5. Com o cidadão algemado e sob sua custódia, abra novamente o radial e escolha **Prender**.
6. No formulário:
   - selecione a acusação;
   - escreva uma observação, se necessário;
   - confirme a pena/multa exibida.
7. O servidor cria o caso e as charges. O feedback informa o `caseId`.
8. Leve o custodiado ao ponto de intake configurado.
9. Use a interação de recebimento prisional.
10. A sentença só começa depois da validação do intake. A custódia é encerrada nesse momento.
11. Consulte o caso no MDT para complementar ficha, evidências, perícia, mandados e histórico.

Se algo falhar, o feedback usa os códigos de `mz_police/client/arrest.lua` e
`mz_police/client/interactions.lua`: duty, capability, custódia, distância, bucket, catálogo,
sentença ativa, intake, persistência ou timeout.

---

## 14. Pendências operacionais

1. Aprovar o catálogo jurídico de produção e manter `mz_police`/`mz_prison` espelhados.
2. Configurar duty point, intake real, raio e routing bucket.
3. Conceder as capabilities aos cargos aprovados.
4. Decidir e aprovar a política de agregação P7.4B antes de ligar as duas flags.
5. Testar citation com uma regra de multa real de staging e `police.citation.issue`.
6. Validar `mz_prison_sentence_charges` no MySQL real; o bootstrap só cria se houver permissão.
7. Validar transações, locks, retry e concorrência em staging.
8. Validar CEF/foco/ESC do `mz_menu` e do MDT.
9. Validar restart/reconnect e dois jogadores reais.
10. Decidir se o pagamento de citation terá UI própria (telefone/banco/MDT) em wave futura.
11. Decidir se sentenças antigas receberão backfill de `mz_prison_sentence_charges`; nada foi
    executado.

---

## 15. Limitações e riscos

- O runtime não foi executado; nenhum PASS de FiveM é declarado.
- O fluxo multi-acusação está implementado, mas desligado por configuração; habilitar sem a
  política aprovada é uma decisão operacional incorreta.
- Sentenças antigas não têm linhas de vínculo; o bloqueio de reutilização de charge é completo
  apenas para sentenças novas.
- A citation de prisão é pendente; não há débito automático. A UI de pagamento é futura.
- O `mz_menu` é um dialog; catálogos muito grandes podem justificar o modal compacto do MDT em
  wave posterior.
- O radial tem limite visual; a ação `arrest` só aparece quando as ações de custódia já estão
  filtradas, mantendo o conjunto visível abaixo do limite radial.

---

## 16. Impacto arquitetural

- **Contratos públicos:** adicionados eventos de prisão rápida, pagamento de citation e export
  `RequestSentenceFromCharges`; contratos P6/P7 existentes preservados.
- **Ownership:** inalterado. `mz_police` é o único writer de caso/charge/citation; `mz_prison` é o
  único writer de sentença e vínculo de charges; `mz_economy` continua o único writer financeiro.
- **Writer/authority:** inalterado; nenhum writer paralelo.
- **Cross-resource:** `mz_police` → `mz_prison` (sentença), `mz_police` → `mz_economy` (pagamento
  de citation), `mz_police` → `mz_inventory` (propriedade no intake, via prison).
- **Persistência:** nova tabela `mz_prison_sentence_charges`; nenhuma alteração destrutiva;
  migration preparada e não executada.
- **Segurança/permissões:** nenhuma permissão enfraquecida; capabilities existentes.
- **Novos exports/eventos:** listados na seção 3.2.
- **Dependências externas:** nenhuma nova.
- **Graphify:** atualização scoped concluída em `mz_police` (625 nós/1287 arestas) e `mz_prison`
  (156 nós/292 arestas). `graphify explain server_arrest` e
  `graphify explain Service.requestSentenceFromCharges` localizaram os novos nós.
- **Runtime:** NOT RUN.
