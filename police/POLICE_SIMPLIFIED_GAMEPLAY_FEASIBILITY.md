---
title: MZ Police — Estudo de viabilidade de prisão simplificada + investigação avançada
status: FEASIBILITY_STUDY
authority: LEVEL_4_REFERENCE
owner: MZ Police
last_reviewed: 2026-10-17
applies_to:
  - mz_police
  - mz_prison
  - mz_inventory
  - mz_clothing
  - mz_target
  - mz_menu
  - mz_org
  - mz_economy
  - mz_vehicles
  - mz_starter
  - mz_docs
related_contracts:
  - MZ-IDENTITY-v2
  - MZ-TARGET-v1
---

# MZ Police — Estudo de viabilidade: prisão simplificada + investigação avançada

Data: 17/10/2026

Estado: **ESTUDO DOCUMENTAL. NENHUMA IMPLEMENTAÇÃO, SQL, MIGRATION, DEPLOY OU ALTERAÇÃO DE CONFIGURAÇÃO FOI REALIZADA.**

Runtime FiveM/OneSync/CEF/MySQL: **NOT RUN**.

> Atualização 2026-10-17: a P7.4A–P7.4C foi implementada após aprovação deste estudo. Consulte
> `mz_docs/police/P7_4_QUICK_ARREST_IMPLEMENTATION.md` para os contratos, testes e pendências
> operacionais. Este documento permanece como o estudo de viabilidade original.

Este documento responde se é possível deixar o trabalho policial tão simples quanto no vRP sem perder
MDT, casos, provas, investigações, sentenças, persistência e autoridade server-side da MZ Framework.
Ele foi produzido a partir do código real dos owners, não de presunções sobre a documentação P1–P9.

---

## 0. Veredito executivo

**É viável.** A simplificação deve ocorrer na **orquestração e na interface**, não na remoção de
verificações. O caminho mais seguro é:

1. manter `mz_police` como owner de custódia, casos, acusações e operação;
2. manter `mz_prison` como owner de sentença, intake, propriedade, atividades e soltura;
3. manter `mz_inventory` como único owner de itens, UID, metadata e propriedade retida;
4. adicionar uma **orquestração de prisão rápida** dentro de `mz_police`, reutilizando as tabelas de
   caso e acusação existentes;
5. adicionar **sentença multi-acusação** em `mz_prison` com uma tabela de vínculo e um export novo;
6. adicionar **confinamento** em `mz_prison` com routing bucket + áreas permitidas + recuperação;
7. adicionar **uniforme** como item `clothing_garment` emitido por contrato restrito do
   `mz_inventory`, sem criar inventário paralelo;
8. manter o **transporte manual** como fluxo padrão; NPC apenas opcional e depois de runtime
   validado; “encaminhamento simplificado” por teleporte não é aceitável.

O MDT completo permanece. Um caso criado pela prisão rápida é um **caso normal** de
`mz_police_cases` (kind `occurrence`), com pessoa, acusações e histórico; ele pode ser aberto,
complementado, promovido a investigação e usado por evidências, perícia e mandados.

### Decisões arquiteturais recomendadas

| Tema | Decisão recomendada |
|---|---|
| Segundo sistema policial | **Não criar.** Nenhum `mz_arrest`, `mz_mdt2`, `mz_prison2` ou writer paralelo. |
| Caso automático | Criar caso `occurrence` normal, com pessoa `subject`, acusações e histórico, em transação única. |
| Sentença multi-acusação | Uma sentença por cidadão, com tabela `mz_prison_sentence_charges` e soma validada no servidor. |
| Multa | Reutilizar `mz_police_citations` + `SettlePlayerOrganizationCharge`; uma citation por acusação com multa. |
| Idempotência | Chave natural baseada em `custodyRevision` para a prisão rápida e em hash do conjunto de charges para a sentença. |
| Transporte padrão | Manual (algemar → escoltar/viatura → intake). |
| Transporte NPC | Opcional, condicionado a runtime; não iniciar sentença por NPC. |
| Confinamento | Routing bucket dedicado + áreas permitidas + correção com loop guard; restaurar bucket na soltura. |
| Uniforme | Item `clothing_garment` com metadata `prisonUniform`, emitido/removido por export restrito a `mz_prison`. |
| Interface rápida | `mz_target` + `mz_menu` (3 selects) na primeira wave; modal compacto no NUI do MDT apenas se o catálogo crescer. |
| MDT | Continua sendo a interface avançada; não vira obrigatório para prender. |

### Riscos que decidem a ordem das waves

1. **Confinamento** é o maior risco de runtime: bucket, teleporte, reconnect, morte e soltura.
2. **Sentença multi-acusação** é o maior risco de regressão do P7.
3. **Transporte NPC** é o maior risco de perda de entidade e de experiência travada.
4. **Uniforme** é o maior risco de duplicação/perda de item se a ordem de retenção/devolução falhar.
5. **Runtime FiveM** continua sendo o gate final; nenhum harness local prova OneSync, CEF ou MySQL.

---

## 1. Método, fontes e limites

### 1.1 Leitura obrigatória

Foram lidos:

- `AGENTS.md`;
- `.agents/rules/architecture.md`;
- `.agents/rules/graphify.md`;
- `.agents/rules/validation-and-reporting.md`;
- `mz_docs/police/P1_IMPLEMENTATION_REPORT.md` a `P9_IMPLEMENTATION_REPORT.md`;
- `mz_docs/police/P7_3_OPERATIONAL_GAMEPLAY_REPORT.md`;
- `mz_docs/police/POLICE_GAMEPLAY_GUIDE.md`;
- `mz_docs/police/POLICE_PRISON_FULL_ACTIVATION.md`;
- `mz_docs/police/POLICE_PRISON_SCHEMA_MATRIX.md`;
- `mz_docs/inventory/CLOTHING_EQUIPMENT.md`;
- `mz_docs/api/TARGET_API.md`;
- código real de `mz_police`, `mz_prison`, `mz_inventory`, `mz_clothing`, `mz_org`,
  `mz_economy`, `mz_target`, `mz_menu`, `mz_notify`, `mz_transport` e `mz_vehicles`.

### 1.2 Graphify

`graphify query` foi executado, mas o grafo disponível não indexa de forma útil os arquivos Lua
relevantes: as consultas retornaram nós de documentação (`ref/...`, `mz_docs/...`) ou resultados
ambíguos. `graphify path` entre `mz_police/server/prison.lua` e `mz_prison/server/service.lua`
não encontrou caminho.

Conclusão: **Graphify foi insuficiente para este escopo**. A evidência usada é o código-fonte real.
Nenhuma atualização de Graphify foi executada, porque esta tarefa é somente leitura.

### 1.3 Limite de runtime

Os relatórios P1–P9 declaram explicitamente **runtime FiveM NOT RUN**. Este estudo mantém a mesma
regra: tudo abaixo é viabilidade estática e desenho de contrato. Não há PASS de OneSync, CEF,
routing bucket, MySQL, restart, reconnect ou entidade de rede.

### 1.4 Configuração atual é de staging, não de produção

O código atual não está no estado “tudo desativado” descrito em alguns relatórios antigos:

- `mz_police/config/server.lua:84-116` habilita catálogo `MZ-RP-STAGING-1`, jurisdição `police`,
  a acusação fictícia `MZ_TEST_ARREST` e `Prison.enabled = true`;
- `mz_prison/config.lua:5-84` habilita intake, sentença, atividades e soltura manual com
  coordenadas **candidatas** de staging.

Isso é configuração de teste. Não é aprovação de produção e não deve ser tratado como catálogo
legal real. O estudo assume que valores finais de produção ainda precisam ser aprovados.

---

## 2. Arquitetura atual relevante

### 2.1 Owners e fronteiras

| Domínio | Owner canônico | Contrato real observado |
|---|---|---|
| Identidade, sessão, personagem, logs | `mz_core` | `ResolvePlayerIdentity`, `GetDeathState`, `CreateDetailedLog`, `GetSourceByCitizenId` |
| Organização, cargo, duty, capability | `mz_org` | `GetPlayerOrgContext`, `CanOrg`, `GetOrgByCode`, `GetOrganizationReadiness` |
| Custódia, abordagem, registros, casos, acusações | `mz_police` | eventos `mz_police:server:*`, `RecordsRepository`, `mz_police_cases` |
| Sentença, intake, propriedade retida, atividades, soltura | `mz_prison` | `RequestSentence`, `GetSentence`, `ReduceSentence`, `ReleasePrisoner` |
| Item, UID, metadata, equipamento, containers | `mz_inventory` | `RetainPrisonerProperty`, `RestorePrisonerProperty`, `GetPlayerEquipmentState` |
| Roupa, catálogo de garment, outfits, projeção visual | `mz_clothing` | `GetPlayerClothingProjection`, `ApplyPlayerOutfit`, reconcile por equipamento |
| Alvo/radial | `mz_target` | `RegisterOption`, `RemoveOption`, `OpenPlayerMenu` |
| Menu contextual | `mz_menu` | `OpenContext`, `InputDialog`, `AlertDialog`, `Confirm`, `Notify` |
| Chamados | `mz_dispatch` | fila, atribuição, blips, histórico |
| Economia e ledger | `mz_economy` | `SettlePlayerOrganizationCharge`, contas e ledger |
| Veículos | `mz_vehicles` / `mz_garagem` | flags, impound, frota, garagem |

O MDT **não é um resource separado**. Ele está dentro de `mz_police/web/` e usa o bridge
`RegisterNUICallback('request')` de `mz_police/client/mdt.lua:176-251`.

### 2.2 Contratos já prontos que podem ser reutilizados

- Catálogo jurídico server-side: `mz_police/server/records/service.lua:152-177`.
- Criação de caso: `mz_police/server/records/service.lua:207-233`.
- Vínculo de pessoa: `mz_police/server/records/service.lua:299-326`.
- Acusação com snapshot imutável: `mz_police/server/records/service.lua:328-360`.
- Leitura de caso com pessoas, charges, evidências, perícias e histórico:
  `mz_police/server/records/repository.lua:62-89`.
- Custódia revisionada: `mz_police/server/custody.lua:265-341`.
- Intake prisional com validação de ponto, raio e bucket: `mz_prison/server/service.lua:210-232`.
- Sentença idempotente e uma sentença ativa por cidadão:
  `mz_prison/server/service.lua:336-358`.
- Retenção e devolução de propriedade: `mz_inventory/server/service.lua:1979-2160`.
- Soltura com devolução e destino: `mz_prison/server/service.lua:234-291`.
- Roupas como itens e outfits: `mz_docs/inventory/CLOTHING_EQUIPMENT.md:13-18, 48-52, 66-76`.

### 2.3 O que já está “implementado” versus “aprovado para produção”

| Item | Estado real |
|---|---|
| Custódia, algema, escolta, viatura | Implementado em código + harness; runtime NOT RUN |
| Catálogo jurídico e sentença | Implementado em código; valores atuais são staging |
| Caso, pessoa, acusação, evidência, mandado | Implementado em código + harness |
| MDT | Implementado em `mz_police/web`; QA de browser, CEF NOT RUN |
| Atividades prisionais | Implementado; runtime NOT RUN |
| Soltura e propriedade | Implementado; runtime NOT RUN |
| Confinamento físico | **Ausente** |
| Uniforme prisional | **Ausente como fluxo completo**; só existe a infraestrutura de roupa como item |
| Transporte NPC policial | **Ausente**; `mz_transport` é táxi e o NPC é client-owned |
| Sentença multi-acusação | **Ausente**; a sentença atual é de uma acusação |
| Multa por acusação de prisão | **Ausente como liquidação**; `fine_amount` é persistido, mas não cobrado |

---

## 3. Fluxo atual de prisão (linha de base)

1. Policial entra em duty e recebe `police.custody`/`police.search` via `mz_org`.
2. Mira no jogador e abre **Ações policiais** no `mz_target`
   (`mz_police/client/interactions.lua:225-235`, `:256-274`).
3. Algemar/desalgemar, escoltar e colocar/retirar da viatura são transições server-side
   (`mz_police/server/custody.lua:265-341`).
4. **Abrir ficha no MDT** usa o `targetSource` do radial
   (`mz_police/client/interactions.lua:217-218`, `mz_police/client/mdt.lua:284-290`).
5. No MDT, o policial cria/seleciona um caso e adiciona acusações
   (`mz_police/web/app.js:243-318`, `mz_police/server/records/service.lua:207-360`).
6. O botão **Preparar intake** grava o contexto **somente no cliente**
   (`mz_police/client/mdt.lua:194-204`, `mz_police/client/interactions.lua:295-304`).
7. O policial leva o custodiado ao ponto de intake; a interação espacial dispara
   `mz_police:client:prison:intake` (`mz_prison/client/interactions.lua:48-63`).
8. O servidor policial revalida custódia, proximidade, bucket, caso e acusação
   (`mz_police/server/prison.lua:66-116`).
9. `mz_prison` revalida intake, catálogo, duração e versão legal e cria a sentença
   (`mz_prison/server/service.lua:293-378`).
10. A propriedade é retida e a sentença é ativada
    (`mz_prison/server/service.lua:359-374`, `mz_inventory/server/service.lua:1979-2038`).
11. O relógio anda apenas com o jogador online (`mz_prison/server/service.lua:662-672`).
12. Ao zerar, a soltura devolve propriedade e, se houver destino válido, teleporta
    (`mz_prison/server/service.lua:234-291`, `mz_prison/client/release.lua:1-16`).

### 3.1 Gargalos comprovados

- **MDT obrigatório para prender.** O fluxo de acusação e “Preparar intake” só existe no MDT.
- **Uma acusação por sentença.** `mz_prison_sentences.charge_public_id` é uma coluna única
  (`mz_prison/server/prepare.lua:23-46`) e `createSentence` grava um único `chargeId`
  (`mz_prison/server/repository.lua:107-120`).
- **Intake preparado apenas no cliente.** Se o cliente cair, o contexto é perdido; o caso
  permanece, mas o encaminhamento precisa ser refeito.
- **Custódia de um único policial.** `requestSentence` exige `cuffedBy.source == source`
  (`mz_police/server/prison.lua:77-79`). Outro policial não pode iniciar a sentença sem
  re-algemar.
- **Multa de acusação não é cobrada.** `fine_amount` é gravado em `mz_police_case_charges`
  (`mz_police/server/records/schema.lua:66-80`), mas o fluxo de multa usa o catálogo separado
  `citations` (`mz_police/server/records/service.lua:362-424`).
- **Sem confinamento.** O intake ativa a sentença, mas não prende fisicamente; o preso fica livre
  no ponto de intake.
- **Soltura pode deixar bucket errado.** `SetPlayerRoutingBucket` só é chamado quando existe
  destino de soltura (`mz_prison/server/service.lua:124-143`, `:275-290`). Se o confinamento
  usar bucket e a soltura não tiver destino, o jogador pode permanecer no bucket prisional.

---

## 4. Proposta A — Prisão rápida pelo radial

### 4.1 Fluxo desejado, mapeado para contratos existentes

| Etapa desejada | Contrato existente | Situação |
|---|---|---|
| Policial aborda e algema | `mz_police:server:custody:request` | Pronto |
| Seleciona **Prender** no radial | `mz_target` + `mz_police/client/interactions.lua` | **Falta ação** |
| Escolhe acusações do catálogo | `getLegalCatalog` / `Records` | Catálogo pronto; falta UI |
| Servidor calcula pena/multa | `configuredRule('charges', ...)` | Cálculo existe; falta orquestração |
| Adiciona observações | `mz_police_cases.summary` | Campo existe |
| Confirma | Nenhum contrato atômico | **Falta** |
| Cria/vincula caso automaticamente | `createCase` + `addCasePerson` + `addCaseCharge` | Existe, mas em operações separadas |
| Registra cidadão, policial, acusações e histórico | Tabelas P6 | Pronto |
| Prepara encaminhamento prisional | `PendingIntake` no cliente | **Parcial, não durável** |
| Sentença só no intake | `mz_police/server/prison.lua` + `mz_prison` | Pronto para uma acusação |

### 4.2 O que falta, em ordem de dependência

1. **Ação `arrest` no radial e no contexto server-side.**
   - `mz_police/client/interactions.lua:225-235` não tem `arrest`.
   - `mz_police/server/interactions.lua:82-92` não retorna `arrest`.
   - Adicionar a ação apenas quando o alvo está algemado, sob custódia do policial, vivo e no
     mesmo bucket. A visibilidade é conveniência; o commit revalida tudo.

2. **Orquestração server-side atômica de caso + pessoa + acusações.**
   - Hoje `createCase`, `addCasePerson` e `addCaseCharge` são chamadas separadas
     (`mz_police/server/records/service.lua:207-360`) e cada uma tem sua própria transação.
   - A prisão rápida precisa de uma única operação com lock por cidadão e transação contendo:
     `mz_police_cases`, `mz_police_case_officers`, `mz_police_case_people`,
     `mz_police_case_charges` e `mz_police_history`.
   - Não é necessário criar tabela nova de “prisão” ou “booking”. O caso é o artefato durável do
     encaminhamento.

3. **Sentença multi-acusação.**
   - O contrato atual aceita um `chargeId` (`mz_police/server/prison.lua:67`) e o owner prisional
     grava um `charge_public_id` único (`mz_prison/server/repository.lua:107-120`).
   - É necessário um export novo em `mz_prison` e uma tabela de vínculo
     `mz_prison_sentence_charges`, com `UNIQUE(charge_public_id)` para impedir que a mesma
     acusação seja sentenciada duas vezes.

4. **Liquidação de multa por acusação.**
   - `mz_police_case_charges.fine_amount` existe, mas não há writer de cobrança para charges.
   - Reutilizar `mz_police_citations` e `mz_economy:SettlePlayerOrganizationCharge`
     (`mz_economy/server/accounts/police_charges.lua:136-141`), criando uma citation por charge
     com multa. Não criar tabela de multa paralela.

5. **Idempotência e retry.**
   - A chave natural da prisão rápida deve ser `(actorCitizenid, targetCitizenid, custodyRevision)`.
     A revisão de custódia é server-side (`mz_police/server/custody.lua:265-341`) e muda a cada
     transição; um retry do mesmo contexto reaplica o mesmo caso, enquanto uma nova custódia
     produz uma nova operação.
   - O `casePublicId` pode ser determinístico:
     `case:<actorCitizenid>:arrest:<targetCitizenid>:<custodyRevision>`.
   - As charges usam `charge:<casePublicId>:<index>`.
   - A operação de histórico usa `arrest:<actorCitizenid>:<custodyRevision>`; o histórico é único
     por `operation_id` (`mz_police/server/records/schema.lua:151-166`).
   - Reuso divergente do mesmo ID deve retornar `request_id_conflict`.

6. **Transferência de custódia para múltiplos policiais.**
   - Sem isso, o policial que não algemou não consegue iniciar a sentença
     (`mz_police/server/prison.lua:77-79`).
   - Adicionar a ação `transfer` ao `ACTIONS` de `mz_police/server/custody.lua:10-17`:
     valida capability, duty, proximidade, bucket, alvo algemado; troca `cuffedBy` para o novo
     policial; limpa escolta; incrementa revisão; sincroniza antigo/novo; audita
     `police.custody.transfer`. Continua existindo **um único** `cuffedBy`.

7. **Cotação server-side antes da confirmação.**
   - A NUI não pode enviar duração, multa, nome de regra ou versão legal.
   - O servidor recebe apenas `ruleCodes` e `notes`, recalcula tudo pelo catálogo
     (`mz_police/server/records/service.lua:87-104`) e devolve totais para exibição.
   - A confirmação usa o mesmo conjunto de `ruleCodes`; o servidor recalcula de novo e ignora
     qualquer total vindo do cliente.

### 4.3 Contratos propostos para a prisão rápida

Todos os novos eventos são server-side, com payload fechado, rate limit, identidade/sessão,
organização, duty, capability, custódia, proximidade, bucket, alvo vivo e auditoria.

| Contrato | Tipo | Função | Mutação? |
|---|---|---|---|
| `mz_police:server:arrest:preview` | evento server-side | Retorna alvo, catálogo, capabilities e situação prisional | Não |
| `mz_police:server:arrest:quote` | evento server-side | Valida `ruleCodes` e devolve pena/multa totais | Não |
| `mz_police:server:arrest:commit` | evento server-side | Cria caso + pessoa + charges + histórico atomicamente | Sim |
| `mz_police:server:prison:requestSentence` | evento existente, payload estendido | Agrega charges não sentenciadas do caso e inicia a sentença no intake | Sim |
| `mz_police:server:custody:request` ação `transfer` | evento existente, ação nova | Transfere a custódia entre policiais | Sim |
| `RequestSentenceFromCharges` | export server-only de `mz_prison` | Cria uma sentença com múltiplas charges | Sim |
| `IssuePrisonUniform` / `RemovePrisonUniform` | exports server-only de `mz_inventory` | Emite/remove o uniforme de forma idempotente | Sim |
| `IssueCaseChargeFine` | método interno de `mz_police` | Cria citation por charge e tenta liquidar | Sim |

#### 4.3.1 `arrest:preview`

Payload fechado:

```lua
{ targetSource = <number> }
```

Retorno mínimo:

```lua
{
  ok = true,
  target = { citizenid, firstname, lastname },
  catalog = { legalVersion, charges = { { code, name, fineAmount, sentenceSeconds } } },
  capabilities = { write, arrest, citation },
  custody = { revision, cuffed, cuffedByMe },
  sentence = { active, status, remainingSeconds },
  warnings = { ... }
}
```

Regras:

- exige `police.records.write` e custódia do alvo;
- não cria caso, não cobra multa, não inicia sentença;
- não confia em nome, duração ou multa enviados pelo cliente;
- se o alvo já tiver sentença ativa, retorna `sentence.active = true` e a UI bloqueia.

#### 4.3.2 `arrest:quote`

Payload fechado:

```lua
{ targetSource = <number>, ruleCodes = { '<CODE>', ... } }
```

Retorno:

```lua
{
  ok = true,
  charges = { { code, name, fineAmount, sentenceSeconds } },
  totals = { sentenceSeconds, fineAmount, chargeCount },
  warnings = { no_custodial_charge, citation_capability_required, ... }
}
```

Regras:

- `ruleCodes` é uma lista fechada, com limite configurável (recomendado 8);
- remove duplicatas ou rejeita duplicatas explícitas (decisão de produto);
- cada código precisa existir no catálogo da organização, com `enabled = true`;
- recalcula nome, multa e duração no servidor;
- pelo menos uma charge precisa ter `sentenceSeconds > 0`;
- se houver multa > 0 e o policial não tiver `police.citation.issue`, retorna aviso/bloqueio;
- não persiste nada.

#### 4.3.3 `arrest:commit`

Payload fechado:

```lua
{
  targetSource = <number>,
  ruleCodes = { '<CODE>', ... },
  notes = '<texto>',
  expectedCustodyRevision = <number>,
  requestId = '<id>'
}
```

Algoritmo autoritativo:

```lua
-- mz_police/server/arrest.lua (módulo novo dentro do owner existente)
function Arrest.commit(source, payload)
  -- 1. payload fechado + rate limit
  -- 2. policy: police.records.write + police.arrest.request
  -- 3. identidade/sessão canônica do policial
  -- 4. alvo resolvido pela custódia, nunca por citizenid do cliente
  -- 5. custódia: cuffed, cuffedBy == source, revision == expectedCustodyRevision
  -- 6. alvo vivo, proximidade e mesmo routing bucket
  -- 7. catálogo: cada ruleCode válido, versão legal, soma de pena e multa
  -- 8. lock por citizenid do alvo
  -- 9. operação determinística por actor + target + custodyRevision
  -- 10. transação: case + officer + person + charges + history
  -- 11. pós-transação: citations de multa, best-effort e idempotentes
  -- 12. auditoria e retorno do caseId/totais
end
```

Regras de negócio:

- **Não** aceitar `sentenceSeconds`, `fineAmount`, `ruleName`, `legalVersion` do cliente;
- no máximo N acusações por prisão rápida (config; recomendado 8);
- todas as charges pertencem ao mesmo caso e ao mesmo `targetCitizenid`;
- o caso é `case_kind = 'occurrence'`;
- o título é gerado no servidor (ex.: `Prisão rápida — <nome>`), não enviado pela NUI;
- `notes` vai para `mz_police_cases.summary` com limite de 4000 caracteres;
- o policial é gravado em `mz_police_case_officers` como `creator`;
- o cidadão é gravado em `mz_police_case_people` como `subject`;
- o histórico grava uma linha `quick_arrest_created` com `caseId`, `chargeIds`, totais e
  `custodyRevision` em `details_json`;
- a resposta inclui `caseId`, `chargeIds`, totais e `next = 'intake'`;
- se a criação do caso falhar, nada é parcialmente persistido;
- se a multa falhar, o caso e as charges permanecem; a resposta inclui `warnings` e a citation
  fica `pending` (não bloqueia a sentença).

#### 4.3.4 Intake multi-acusação

O evento `mz_police:server:prison:requestSentence` passa a aceitar:

```lua
{ targetSource = <number>, caseId = '<case>', chargeId = '<opcional>' }
```

- se `chargeId` vier preenchido, mantém o fluxo P7.3 de uma acusação;
- se não vier, o servidor agrega **todas as charges ainda não sentenciadas** daquele caso para
  aquele cidadão;
- a agregação é feita no servidor; o cliente não envia lista de charges nem totais;
- charges com `sentenceSeconds = 0` não entram na pena, mas podem gerar multa;
- se nenhuma charge tiver pena, retorna `no_custodial_charge` e não chama o prison;
- o servidor monta snapshots imutáveis `{ chargeId, ruleCode, ruleName, sentenceSeconds,
  legalVersion }` e chama `RequestSentenceFromCharges`;
- a operação é determinística por hash do caso + conjunto de charges, para que um retry após
  timeout recupere a sentença existente em vez de responder `active_sentence_exists`.

#### 4.3.5 Sentença multi-acusação no `mz_prison`

Novo export:

```lua
RequestSentenceFromCharges(actorSource, targetSource, request)
```

`request` fechado:

```lua
{
  caseId = '<case>',
  primaryChargeId = '<charge>',
  charges = {
    { chargeId, ruleCode, ruleName, sentenceSeconds, legalVersion },
    ...
  },
  operationId = '<id>',
  orgCode = 'police'
}
```

O owner prisional:

1. resolve ator e alvo por identidade canônica;
2. exige intake point, raio e bucket;
3. valida organização, jurisdição, `sentenceRulesEnabled`, `legalVersion`;
4. valida **cada** charge contra `MZPrisonConfig.sentenceRules`, incluindo nome e duração;
5. soma as durações e compara com a solicitação;
6. rejeita charge repetida e charge já vinculada a outra sentença;
7. rejeita se já existir sentença ativa para o cidadão;
8. cria a sentença com `charge_public_id = primaryChargeId` e `duration_seconds = total`;
9. insere as linhas de `mz_prison_sentence_charges` na mesma transação;
10. retém propriedade e ativa a sentença como hoje;
11. retorna a sentença com a lista de charges.

A tabela nova (sem ALTER em tabelas existentes):

```sql
CREATE TABLE IF NOT EXISTS mz_prison_sentence_charges (
  sentence_public_id VARCHAR(96) NOT NULL,
  charge_public_id VARCHAR(160) NOT NULL,
  rule_code VARCHAR(64) NOT NULL,
  rule_name VARCHAR(160) NOT NULL,
  legal_version VARCHAR(64) NOT NULL,
  sentence_seconds INT UNSIGNED NOT NULL,
  is_primary TINYINT UNSIGNED NOT NULL DEFAULT 0,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (sentence_public_id, charge_public_id),
  UNIQUE KEY uq_mz_prison_sentence_charge (charge_public_id),
  CONSTRAINT fk_mz_prison_sentence_charge_sentence FOREIGN KEY (sentence_public_id)
    REFERENCES mz_prison_sentences(public_id) ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```

Por que não alterar `mz_prison_sentences`:

- o bootstrap do `mz_prison` valida a estrutura e **não executa ALTER**
  (`mz_prison/server/prepare.lua:10-19`);
- adicionar coluna exigiria migração manual e atualização do contrato de validação;
- uma tabela nova é criada por `CREATE TABLE IF NOT EXISTS` e mantém a compatibilidade;
- `charge_public_id` continua sendo a charge primária, e a lista completa fica na tabela de
  vínculo.

#### 4.3.6 Multa por acusação, sem tabela paralela

Reutilizar `mz_police_citations` (`mz_police/server/records/schema.lua:115-133`):

- uma citation por charge com `fineAmount > 0`;
- `public_id = 'citation:' .. chargePublicId`;
- `operation_id = 'charge_fine:' .. chargePublicId` (limite de 160);
- `case_public_id`, `target_citizenid`, `rule_code`, `rule_name`, `amount`,
  `legal_version`, `issued_by_citizenid`;
- status inicial `pending`;
- liquidação via `mz_economy:SettlePlayerOrganizationCharge` com chave idempotente curta
  (hash da charge), referência `caseId/chargeId`;
- se o alvo não tiver saldo, a citation permanece `pending` com `economy_error`;
- não criar tabela `mz_police_fines` ou writer econômico novo.

Observação: o fluxo atual `issueCitation` usa o catálogo `citations`, não `charges`
(`mz_police/server/records/service.lua:370`). A liquidação de charge precisa de um método interno
específico que use o snapshot da charge, mas mantenha a mesma tabela e o mesmo owner.

#### 4.3.7 Permissões

| Operação | Capability mínima |
|---|---|
| Ver preview/catálogo no fluxo | `police.records.write` |
| Criar caso e acusações | `police.records.write` |
| Iniciar sentença | `police.arrest.request` |
| Emitir multa de charge | `police.citation.issue` |
| Custódia/transferência | `police.custody` |
| Revista | `police.search` |
| Apreensão | `police.seize` |

Todas exigem organização autorizada, membro ativo e **duty** (`mz_police/server/policy.lua:76-142`).
O seed policial atual só concede `police.custody` e `police.search`
(`mz_org/server/domain/default_orgs.lua:304-317`); as demais precisam ser atribuídas no `/stafforg`
antes do teste.

Decisão pendente: se uma charge tiver multa e o policial não tiver `police.citation.issue`,
o commit deve ser bloqueado (`citation_capability_required`) ou a multa deve ficar pendente?
Recomendação: **bloquear a seleção dessa charge** na UI e retornar erro no servidor; não
emitir multa sem autoridade.

#### 4.3.8 Prevenção de duplicidade

| Risco | Mitigação |
|---|---|
| Duplo clique | lock por cidadão + operação determinística |
| Retry após timeout | mesmo `casePublicId` e mesma operação; replay retorna o caso |
| Duas abas/NUI | o servidor não confia no cliente; o lock e a custódia revisionada vencem |
| Mesma charge em duas sentenças | `UNIQUE(charge_public_id)` em `mz_prison_sentence_charges` |
| Duas sentenças ativas | `Repository.getCurrent` + lock por cidadão (`mz_prison/server/service.lua:336-358`) |
| Multa duplicada | `UNIQUE(operation_id)` em `mz_police_citations` + chave econômica idempotente |
| Caso duplicado por novo requestId | chave natural por `custodyRevision`, não por requestId do cliente |

#### 4.3.9 Auditoria

- `police.arrest.preview` e `quote` podem ser auditados apenas em falha/abuso, para não poluir;
- `police.arrest.committed`: ator, alvo, `caseId`, `chargeIds`, totais, `custodyRevision`,
  `legalVersion`, resultado;
- `police.arrest.failed`: motivo (`custody_required`, `target_too_far`,
  `routing_bucket_mismatch`, `legal_rule_not_configured`, `citation_capability_required`,
  `active_sentence_exists`, etc.);
- `police.custody.transfer`: ator antigo, ator novo, alvo, revisão;
- `mz_police_history`: `quick_arrest_created` com detalhes;
- `mz_core:CreateDetailedLog`: log central do domínio.

#### 4.3.10 Feedback

- preview: nome, citizenid, situação prisional;
- quote: “Pena total: 180s; multa: R$ 0; 1 acusação”;
- commit: “Caso CASE-… criado. Leve o custodiado ao intake.”;
- intake: “Sentença iniciada. 180s.”;
- multa pendente: “Multa de R$ X registrada como pendente.”;
- erros: mensagens existentes em `mz_police/client/interactions.lua:13-28`, acrescidas das novas.

#### 4.3.11 Compatibilidade com o MDT

- o caso aparece em `listCases` como `occurrence`;
- `getCase` retorna charges, pessoa, evidências, perícias e histórico;
- o MDT existente já renderiza acusações e “Preparar intake”
  (`mz_police/web/app.js:293-317`, `:549-558`);
- o caso pode ser aberto depois para complementar resumo, pessoas, evidências e mandados;
- o marcador de “prisão rápida” fica no histórico (`quick_arrest_created`), sem necessidade de
  coluna nova; se for necessário filtrar na lista, uma coluna `origin` exigirá migração aditiva
  manual, pois o bootstrap não faz ALTER.

### 4.4 Veredito da Proposta A

**Viável, complexidade média, risco médio.** Não exige novo resource, nova tabela de caso ou
novo sistema de permissão. Exige um módulo de orquestração em `mz_police`, um export/tabela de
vínculo em `mz_prison`, um método de multa reutilizando citations e testes de concorrência.

---

## 5. Proposta B — Investigação completa no MDT

### 5.1 O que já existe

| Funcionalidade | Estado | Evidência |
|---|---|---|
| Casos persistentes | Implementado | `mz_police_cases`, `mz_police/server/records/schema.lua:33-45` |
| Suspeitos e envolvidos | Implementado | `mz_police_case_people`, `server/records/service.lua:299-326` |
| Acusações | Implementado | `mz_police_case_charges`, `server/records/service.lua:328-360` |
| Evidências físicas | Implementado | P5 + `mz_police_case_evidence`, `evidence.lua:88-166` |
| Cadeia de custódia | Implementado | revisão/last handler em `mz_police_case_evidence` e `mz_inventory` |
| Perícia | Parcial | `weapon_identity` em P9; globalmente desligado (`mz_police/config/server.lua:121-158`) |
| Mandados | Parcial | tabelas e eventos existem; regras vazias (`mz_police/config/server.lua:100-103`) |
| Histórico de decisões | Parcial | `mz_police_history` e `mz_prison_history`; MDT só lê o histórico do caso |
| Relatórios policiais | Parcial | `summary` + notas + histórico; não há documento/relatório estruturado |
| Ocorrências relacionadas | **Ausente** | não há tabela caso-caso |
| Veículos relacionados | **Ausente** | não há tabela caso-veículo |
| Promoção de ocorrência para investigação | **Ausente** | `case_kind` é imutável no update |

### 5.2 Lacunas e contratos propostos

#### 5.2.1 Promover caso `occurrence` → `investigation`

Hoje `updateCase` aceita apenas `title`, `summary`, `status`, `expectedRevision`
(`mz_police/server/records/service.lua:261-297`). `CASE_KINDS` inclui `investigation`
(`mz_police/server/records/service.lua:5`), mas não há transição.

Proposta:

```lua
-- evento server-side
mz_police:server:records:promoteCase
{ caseId, expectedRevision, requestId }
```

- exige `police.records.write`, organização e duty;
- aceita `occurrence` ou `case` como origem;
- atualiza `case_kind = 'investigation'` e `revision = revision + 1`;
- grava histórico `case_promoted`;
- mantém o mesmo `public_id`, pessoas, charges, evidências e perícias;
- idempotente por `requestId` e `expectedRevision`;
- não cria um segundo caso.

Isso resolve “transformar um caso simplificado em investigação detalhada” sem duplicar tabelas.

#### 5.2.2 Ocorrências relacionadas

Nova tabela aditiva:

```sql
CREATE TABLE IF NOT EXISTS mz_police_case_links (
  case_public_id VARCHAR(160) NOT NULL,
  related_case_public_id VARCHAR(160) NOT NULL,
  relation VARCHAR(32) NOT NULL,
  created_by_citizenid VARCHAR(64) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (case_public_id, related_case_public_id, relation),
  CONSTRAINT fk_mz_police_case_links_case FOREIGN KEY (case_public_id)
    REFERENCES mz_police_cases(public_id) ON DELETE CASCADE,
  CONSTRAINT fk_mz_police_case_links_related FOREIGN KEY (related_case_public_id)
    REFERENCES mz_police_cases(public_id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```

Eventos `linkRelatedCase` / `unlinkRelatedCase`, com validação de mesma organização e histórico.
Não substitui `case_people`; é relação entre casos.

#### 5.2.3 Veículos relacionados

Nova tabela aditiva:

```sql
CREATE TABLE IF NOT EXISTS mz_police_case_vehicles (
  case_public_id VARCHAR(160) NOT NULL,
  plate VARCHAR(16) NOT NULL,
  model VARCHAR(64) NULL,
  owner_citizenid VARCHAR(64) NULL,
  linked_by_citizenid VARCHAR(64) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (case_public_id, plate),
  CONSTRAINT fk_mz_police_case_vehicles_case FOREIGN KEY (case_public_id)
    REFERENCES mz_police_cases(public_id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```

Para validar a placa/modelo no owner, é necessário um contrato de leitura em `mz_vehicles`.
Hoje P4 tem `PoliceScanNearestVehicle` e P9 tem `PoliceResolveOccupiedVehicle`; não há
`ResolveVehicleByPlate`. Proposta: export server-only `PoliceResolveVehicleReference(actorSource,
plate)` restrito a `mz_police`, que devolve placa/modelo/owner sanitizados e valida organização/
duty no owner. O `mz_police` não escreve no banco de veículos.

#### 5.2.4 Histórico da sentença no MDT

`mz_prison_history` já registra sentença, retenção, atividades, redução e soltura
(`mz_prison/server/prepare.lua:63-76`). O MDT hoje só consulta `getSentence`, que retorna a
sentença e propriedade/atividades (`mz_prison/server/service.lua:380-388`), mas não o histórico.

Proposta: export de leitura `GetSentenceDetails(citizenid)` ou evento
`mz_police:server:prison:getSentenceHistory`, restrito a `mdt.open`, retornando histórico
paginado/limitado. Não altera a sentença e não cria writer novo.

#### 5.2.5 Relatórios policiais

Não há tabela de relatório/documento. O caso tem `summary` e o histórico. Para “relatórios
policiais” estruturados, uma wave futura pode adicionar `mz_police_case_reports` com título,
corpo, autor, revisão e status. Não é necessário para a prisão rápida; o `summary` cobre a
observação inicial.

### 5.3 Veredito da Proposta B

**Viável e de baixo risco na maior parte.** A promoção de caso é uma extensão pequena e de alto
valor. Ocorrências relacionadas e veículos são tabelas aditivas. Relatórios estruturados podem
ficar para depois. O MDT continua sendo o sistema avançado, e a prisão rápida apenas alimenta o
mesmo caso.

---

## 6. Proposta C — Transporte prisional

### 6.1 Modalidade A — Transporte manual

**Estado: implementado em código, runtime NOT RUN.**

O policial já pode:

- algemar e escoltar (`mz_police/client/custody.lua:58-67`,
  `mz_police/server/custody.lua:286-297`);
- colocar/retirar o custodiado da viatura, com veículo, assento, bucket e network ID derivados
  no servidor (`mz_police/server/custody.lua:143-190`, `:298-311`);
- dirigir até o intake e usar a interação espacial (`mz_prison/client/interactions.lua:48-63`);
- ter a sentença revalidada no ponto (`mz_prison/server/service.lua:210-232`, `:293-378`).

O que falta é **UX e multi-policial**, não o transporte em si:

- um waypoint/blip de intake para o policial (o ponto já tem marker/blip configurável em
  `mz_prison/client/interactions.lua:55-58`; pode ser habilitado na configuração);
- lembrete para retirar o preso da viatura antes do intake;
- transferência de custódia para outro policial (seção 4.2, item 6).

**Recomendação: fluxo padrão.** É o mais realista, o mais seguro e o que menos depende de
entidades de rede.

### 6.2 Modalidade B — Transporte assistido por NPC

**Estado: ausente para polícia.** Existe um provider de NPC em `mz_transport`, mas ele é de táxi
e é **client-owned**:

- o servidor manda o cliente do passageiro criar o NPC
  (`mz_transport/server/providers/population_standalone.lua:8-44`);
- o cliente cria veículo/ped, dirige com `TaskVehicleDriveToCoordLongrange` e reporta status
  (`mz_transport/client/providers/population_standalone.lua:65-155`);
- o servidor valida entidades, assento e distância, mas a IA e a posse da entidade são do
  cliente (`mz_transport/server/providers/population_standalone.lua:46-154`);
- em resource stop, o cliente remove as entidades
  (`mz_transport/client/providers/population_standalone.lua:161-167`).

Para transporte policial, seria necessário:

1. um contrato/serviço de transporte policial, com sessão server-side (ator, alvo, veículo,
   motorista, origem, destino, status, expiração);
2. spawn no cliente do **policial** (não no cliente do preso), para reduzir risco de o preso
   controlar a entidade;
3. validação server-side de que o alvo está no veículo e o veículo chegou ao intake;
4. watchdog de timeout, entidade perdida, desconexão, restart e bucket;
5. fallback manual sempre disponível;
6. **não** iniciar sentença pelo NPC; o intake continua sendo a única porta.

Riscos técnicos reais:

- o NPC só existe enquanto o cliente que o criou estiver vivo e com a entidade na rede;
- OneSync pode perder/migrar ownership; culling e routing buckets podem esconder o veículo;
- se o policial cair, o NPC e a viatura podem desaparecer com o preso;
- se o preso cair, o assento fica inconsistente;
- restart do resource remove as entidades;
- um preso em outro bucket não vê o veículo;
- a IA pode falhar em chegar (trânsito, blip, pathfinding, MLO).

**Recomendação: opcional e posterior.** Não deve ser o fluxo padrão. Se for aprovado, deve ser
uma wave separada, com feature flag, fallback manual e teste in-game extensivo.

### 6.3 Modalidade C — Encaminhamento simplificado

“Transferência controlada ao presídio após confirmação autorizada” só é aceitável se houver
**movimento real** do personagem. Um teleporte com fade, cutscene ou desculpa de RP é
exatamente o “teleporte administrativo disfarçado” que a restrição proíbe.

Alternativas legítimas:

- **Intake remoto legítimo**: o preso já está fisicamente no ponto de intake; a interação normal
  inicia a sentença. Isso não é teleporte; é o fluxo atual.
- **Transporte por NPC** (modalidade B) com movimento real.
- **Transporte manual** (modalidade A).

**Recomendação: não implementar C como teleporte.** Se o objetivo for reduzir tempo, a solução é
a prisão rápida + transporte manual com bom UX, não mover o jogador sem RP.

### 6.4 Comparação

| Critério | Manual | NPC assistido | Encaminhamento simplificado |
|---|---|---|---|
| Experiência do jogador | Alta imersão; depende de direção | Boa se funcionar; frustrante se falhar | Rápida, mas artificial |
| Realismo RP | Alto | Médio | Baixo |
| Segurança | Alta; servidor valida tudo | Média; entidade client-owned | Baixa; teleporte disfarçado |
| Performance | Baixa (um jogador dirigindo) | Média (NPC + veículo + IA) | Baixa, mas teleporte |
| Complexidade | Baixa | Alta | Média/alta para “esconder” |
| Confiabilidade | Alta | Baixa/média | Alta, mas inaceitável |
| Compatibilidade MZ | Total | Requer contrato novo | Viola a restrição |
| Recomendação | **Padrão** | **Opcional futuro** | **Não implementar** |

### 6.5 Veredito da Proposta C

**Manual é a melhor escolha.** NPC é viável tecnicamente, mas caro e frágil. O encaminhamento
simplificado por teleporte não deve existir. O ganho de simplicidade deve vir do radial, do caso
automático e da sentença multi-acusação, não de mover o preso sem transporte.

---

## 7. Proposta D — Confinamento prisional

### 7.1 Estado atual

Não existe confinamento. O `mz_prison`:

- valida o ponto de intake e ativa a sentença
  (`mz_prison/server/service.lua:210-232`, `:293-378`);
- retém a propriedade e ativa (`mz_prison/server/repository.lua:122-147`);
- anda o relógio com o jogador online (`mz_prison/server/service.lua:662-672`);
- não tem cela, área, bucket, teleporte de entrada ou sweep de fuga;
- só usa `SetPlayerRoutingBucket` na soltura, quando há destino
  (`mz_prison/server/service.lua:124-143`).

Portanto, hoje o preso pode simplesmente andar para fora do presídio após o intake.

### 7.2 Opções avaliadas

| Opção | Vantagens | Limitações | Veredito |
|---|---|---|---|
| Área física + portas | Sem teleporte; imersivo | Depende de MLO/portas; jogador pode pular/correr; `mz_doors` não tem contrato de lock de gameplay para prisão | Complementar, não suficiente |
| Routing bucket apenas | Isola do mundo; simples | Não impede andar para fora; não move o jogador; release pode deixar no bucket | Insuficiente sozinho |
| Bucket + áreas permitidas + correção | Contém de fato; independente de MLO; auditável | Requer coordenadas reais, sweep, loop guard e restore de bucket | **Recomendado** |
| Bucket + portas | Mais imersivo | Mais dependências e pontos de falha | Opcional depois |

### 7.3 Desenho recomendado

#### 7.3.1 Configuração (sem inventar coordenadas)

```lua
MZPrisonConfig.confinement = {
  enabled = false,             -- fail-closed
  bucket = nil,                -- bucket reservado; nil = não trocar bucket
  sweepIntervalMs = 3000,
  graceMs = 1500,
  correctionCooldownMs = 3000,
  maxCorrections = 8,
  cells = {
    -- cell_a = { coords = { x = <REAL>, y = <REAL>, z = <REAL> }, heading = <REAL>, radius = <REAL>, areaId = 'cell_a' }
  },
  areas = {
    -- cell_a = { center = { x = <REAL>, y = <REAL>, z = <REAL> }, radius = <REAL> },
    -- yard   = { center = { x = <REAL>, y = <REAL>, z = <REAL> }, radius = <REAL> }
  }
}
```

Regras de configuração:

- nenhum valor é preenchido por padrão;
- `enabled = true` exige pelo menos uma célula e uma área válidas;
- o bucket deve ser reservado/documentado (faixa própria) para não colidir com casas,
  interiores ou outros sistemas;
- o raio máximo deve ser limitado (ex.: 50 m) e validado no boot;
- se a célula não estiver contida em nenhuma área, o boot/entrada falha fechado com
  `confinement_area_invalid`, em vez de iniciar um loop de teleporte.

#### 7.3.2 Persistência

Nova tabela aditiva, sem ALTER:

```sql
CREATE TABLE IF NOT EXISTS mz_prison_confinement (
  sentence_public_id VARCHAR(96) NOT NULL PRIMARY KEY,
  cell_id VARCHAR(64) NOT NULL,
  bucket INT NOT NULL,
  previous_bucket INT NOT NULL DEFAULT 0,
  status VARCHAR(24) NOT NULL DEFAULT 'confined',
  last_error VARCHAR(64) NULL,
  last_correction_at TIMESTAMP NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  KEY idx_mz_prison_confinement_bucket (bucket, status),
  CONSTRAINT fk_mz_prison_confinement_sentence FOREIGN KEY (sentence_public_id)
    REFERENCES mz_prison_sentences(public_id) ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```

Campos:

- `cell_id`: célula atribuída;
- `bucket`: bucket prisional em uso;
- `previous_bucket`: bucket do jogador antes do confinamento, para restaurar na soltura;
- `status`: `confined`, `released`, `error`;
- `last_error`: motivo auditável;
- `last_correction_at`: controle de loop/cooldown.

A tabela registra o estado de confinamento; **não** é um inventário nem um segundo sistema de
sentença.

#### 7.3.3 Ciclo de vida

**Entrada (após `activateWithProperty`):**

1. resolve a célula (primeira livre; se todas ocupadas, cela compartilhada ou fila);
2. grava `previous_bucket = GetPlayerRoutingBucket(source)` e o bucket prisional;
3. se `bucket` configurado, `SetPlayerRoutingBucket(source, bucket)`;
4. envia `mz_prison:client:confine` com coordenadas/heading da célula (server-issued);
5. o cliente sai de veículo, se necessário, e aplica as coordenadas;
6. o servidor confirma a posição no sweep seguinte.

**Sweep (a cada `sweepIntervalMs`):**

1. carrega sentenças `active`/`release_pending` com confinamento;
2. resolve o `source` pelo `citizenid` canônico;
3. se offline, não teleporta; mantém o estado persistido;
4. se morto/respawnando (`GetDeathState`), não corrige;
5. verifica bucket e área permitida;
6. se fora, aplica correção após `graceMs`, com cooldown;
7. se corrections > `maxCorrections` na janela, grava `confinement_correction_loop`,
   suspende correções por um cooldown e audita; **não** bane, **não** pune;
8. se a configuração for inválida, grava `confinement_invalid` e não teleporta.

**Soltura:**

1. remove o uniforme (se houver);
2. devolve a propriedade;
3. encerra o confinamento;
4. restaura `previous_bucket` (ou 0) **mesmo sem destino de soltura**;
5. se houver destino válido, teleporta e confirma; se não, mantém a posição atual no bucket
   restaurado;
6. marca `mz_prison_confinement.status = 'released'`.

**Reconnect/restart:**

- `mz_prison/server/main.lua:47-49` já chama `resumeSource` no `playerLoaded`;
- `resumeSource` precisa reaplicar o confinamento para sentença `active`;
- no boot, o sweep precisa reconstruir o estado a partir de `mz_prison_confinement` +
  `mz_prison_sentences`;
- se o bucket anterior não existir mais, usar 0 e auditar.

**Morte/respawn:**

- enquanto `GetDeathState` não for `alive`, o sweep não corrige;
- ao voltar `alive`, reaplica célula/bucket;
- se o respawn do `mz_medical` levar o jogador para fora, a primeira correção o traz de volta;
- não alterar o lifecycle do `mz_core`; usar apenas exports públicos (`GetDeathState`,
  `ResolvePlayerIdentity`, `GetSourceByCitizenId`).

**Troca de personagem / source reuse:**

- o estado é por `citizenid`/sentença, não por `source`;
- ao trocar de personagem, o sweep resolve a identidade atual e não reaplica a sentença de outro
  personagem;
- o `source` antigo é limpo no `playerDropped` (`mz_prison/server/main.lua:43-45`).

#### 7.3.4 Anti-loop e kill switch

- `maxCorrections` + `correctionCooldownMs`;
- se a célula/área for inválida, não teleportar;
- se o jogador estiver em veículo, o cliente sai antes de teleportar; se falhar, o servidor
  registra e tenta novamente com cooldown;
- se o jogador estiver em cutscene/character switch, pular;
- feature flag `confinement.enabled` permite desligar sem alterar a sentença;
- um comando de diagnóstico read-only pode mostrar `sentence/cell/bucket/area/lastError`
  sem conceder nada; o antigo `p1_gate` foi removido do runtime.

#### 7.3.5 Release e bucket: correção obrigatória

O código atual só chama `SetPlayerRoutingBucket` dentro de `dispatchRelease`, que depende de um
`releasePoint` válido (`mz_prison/server/service.lua:124-143`, `:275-290`). Com confinamento, isso
é insuficiente: uma soltura sem destino deixaria o jogador no bucket prisional.

Correção necessária (proposta):

```lua
-- antes de dispatchRelease / em releaseSentence
local confinement = MZPrison.Repository.getConfinement(sentence.public_id)
if confinement then
  SetPlayerRoutingBucket(targetSource, tonumber(confinement.previous_bucket) or 0)
  MZPrison.Repository.finishConfinement(sentence.public_id, 'released')
end
```

A ordem exata (antes/depois do teleporte) deve ser validada in-game. O importante é que a
restauração do bucket **não dependa** de existir destino de soltura.

### 7.4 Limitações FiveM/OneSync

- `SetPlayerRoutingBucket` isola entidades, mas **não move** o jogador; é preciso teleportar.
- O servidor pode ler posição/bucket, mas o cliente é quem aplica `SetEntityCoords` de forma
  confiável; usar evento server→client e revalidar.
- Routing buckets podem esconder o preso de policiais que não estão no mesmo bucket; o intake
  deve ocorrer antes da troca de bucket ou a polícia precisa entrar no bucket.
- Culling/focus zone pode afetar a visão de NPCs e veículos.
- `GetEntityCoords` do servidor pode falhar durante transições; tratar como
  `position_unavailable` e tentar depois.
- Restart do resource perde estado em memória; a persistência é obrigatória.
- MLO/portas podem ter colisão/altura diferentes; validar no mapa real.

### 7.5 Veredito da Proposta D

**Viável, mas é a proposta de maior risco de runtime.** Deve vir depois da prisão rápida e da
sentença multi-acusação. Exige configuração real, tabela nova, sweep, restore de bucket e um gate
in-game com dois jogadores, reconnect, restart, morte e soltura sem destino.

---

## 8. Proposta E — Uniforme de presidiário

### 8.1 O que já existe

- Roupa é item físico único `clothing_garment`, com UID e metadata
  (`mz_inventory/shared/items.lua:2-14`).
- Equipar/retirar e aplicar outfit passam por locks/transações do `mz_inventory`
  (`mz_docs/inventory/CLOTHING_EQUIPMENT.md:48-52`).
- `mz_clothing` aplica a projeção visual a partir do equipamento
  (`mz_clothing/server/main.lua:432-471`, `:514-520`).
- `mz_prison` já retém e devolve **todos** os itens de `main` + `equipment`
  (`mz_inventory/server/service.lua:1979-2038`, `:2055-2160`).
- O container `prison_property` existe (`mz_inventory/shared/constants.lua:6-17`).

### 8.2 O que falta

1. Um **garment aprovado** de uniforme prisional. O catálogo atual de `mz_clothing` é de
   categorias de loja, não de uniformes (`mz_clothing/shared/catalog.lua:1-27`); não há asset
   aprovado.
2. Um contrato restrito para emitir/remover o uniforme sem usar `AddPlayerItem` aberto
   (`mz_inventory/server/exports.lua:133-139`).
3. Ordem correta de retenção/emissão/devolução.
4. Persistência/reconciliação pós-restart.
5. Regra de não duplicar, não destruir roupa original e não devolver uniforme como propriedade.

### 8.3 Desenho recomendado

#### 8.3.1 Configuração

```lua
MZPrisonConfig.uniform = {
  enabled = false,
  item = 'clothing_garment',
  slot = 'torso',
  garment = {
    garment_id = '<ID_APROVADO>',
    slot = 'torso',
    model_family = 'freemode',
    variants = {
      -- mp_m_freemode_01 = { kind = 'component', native_id = 11, drawable = <REAL>, texture = <REAL> },
      -- mp_f_freemode_01 = { kind = 'component', native_id = 11, drawable = <REAL>, texture = <REAL> }
    }
  }
}
```

Sem asset/drawable aprovado, `enabled` deve permanecer `false`. Não inventar IDs.

#### 8.3.2 Contratos restritos no `mz_inventory`

```lua
IssuePrisonUniform(targetSource, sentenceId, itemName, metadata)
RemovePrisonUniform(targetSource, sentenceId)
GetPrisonerUniform(targetSource, sentenceId)
```

Regras:

- `GetInvokingResource() == 'mz_prison'` (como os exports de propriedade em
  `mz_inventory/server/exports.lua:185-202`);
- valida item `unique = true`, `type = 'clothing'`, `equipment = true`;
- o servidor monta o metadata `clothing` a partir da configuração prisional; o cliente não envia;
- marca `metadata.prisonUniform = { sentenceId, issuedAt, schema = 1 }`;
- verifica se já existe item com esse marcador; se existir, retorna replay;
- adiciona/equipa em transação, preservando UID;
- `RemovePrisonUniform` remove **somente** o item com o marcador da sentença; se não existir,
  retorna sucesso idempotente;
- não usa `AddPlayerItem`/`RemovePlayerItem` abertos, para não permitir que qualquer resource
  emita uniforme.

#### 8.3.3 Ordem no intake

1. `RetainPrisonerProperty` retém main + equipment (incluindo roupas originais);
2. `activateWithProperty` ativa a sentença;
3. `IssuePrisonUniform` emite o uniforme e equipa;
4. `Confinement.begin` inicia o confinamento.

Por que emitir **depois** da retenção:

- se emitir antes, o uniforme seria retido como propriedade do preso e devolvido na soltura;
- depois da retenção, o uniforme é um item do presídio, não do preso;
- a roupa original permanece intacta em `prison_property` com UID/metadata.

#### 8.3.4 Ordem na soltura

1. `RemovePrisonUniform` (idempotente);
2. `RestorePrisonerProperty` (devolve roupas/itens originais);
3. `Confinement.end` / restore bucket;
4. destino de soltura, se configurado.

Se a remoção do uniforme falhar:

- manter `release_pending` e tentar novamente;
- não apagar roupa original;
- registrar `uniform_remove_failed`;
- se persistir, exigir intervenção staff, sem destruir itens.

#### 8.3.5 Reconnect/restart

- o item do uniforme é persistente no inventário;
- uma tabela de reconciliação `mz_prison_uniform` pode registrar
  `(sentence_public_id, instance_uid, item, status, issued_at, removed_at)` — não é um segundo
  inventário, é o mesmo padrão de `mz_prison_property`;
- no `resumeSource`, se a sentença está ativa e o uniforme não está equipado, reemitir de forma
  idempotente ou registrar `uniform_missing`;
- se a sentença já foi liberada e o uniforme ainda existe, remover;
- limitar reemissões para não criar loop de itens.

#### 8.3.6 Visual

- ao equipar o garment, o `mz_inventory` publica a mudança de equipamento;
- `mz_clothing` reconcilia a projeção visual
  (`mz_docs/inventory/CLOTHING_EQUIPMENT.md:76`);
- não criar um segundo sistema de aparência, nem alterar `mz_creator`;
- a aparência base (rosto/corpo/cabelo) continua do `mz_creator`.

### 8.4 Riscos

| Risco | Mitigação |
|---|---|
| Duplicar uniforme | marcador único por `sentenceId` + verificação transacional |
| Devolver uniforme como propriedade | emitir depois da retenção; remover antes da devolução |
| Destruir roupa original | nunca apagar itens sem marcador; retenção/devolução canônicas |
| Perder uniforme no restart | reconciliação por `mz_prison_uniform` + metadata |
| Slot ocupado | equipar em slot configurado; se ocupado, usar slot livre ou falhar com auditoria |
| Item aberto a qualquer resource | export restrito a `mz_prison`, não `AddPlayerItem` |

### 8.5 Veredito da Proposta E

**Viável, complexidade média, risco médio.** Depende de um garment aprovado (asset/config) e de
dois exports restritos no `mz_inventory`. Pode ser implementado depois do confinamento, mas a
ordem de retenção/emissão/devolução deve ser testada com inventário cheio e equipamento ativo.

---

## 9. Experiência visual

### 9.1 Reaproveitamento

| Recurso | Uso proposto | Não fazer |
|---|---|---|
| `mz_target` | Ação **Prender** dentro de **Ações policiais** | Novo radial ou nova tecla |
| `mz_menu` | `InputDialog` para acusações/observação, `AlertDialog`/`Confirm` para revisão, `Notify` para feedback | Nova biblioteca de menu |
| MDT NUI (`mz_police/web`) | Interface avançada; caso, charges, evidências, investigação | Reescrever o MDT para a prisão rápida |
| `mz_notify` | Pode ser usado para feedback server-side se o padrão for unificado | Dois toasts para a mesma ação |
| `mz_frontend` | Não usar nesta fase: é base de demonstração, não produção
(`mz_frontend/README.md:1-20`) | Introduzir React/Vite no MDT atual |

### 9.2 Opção 1 (recomendada para a primeira wave): `mz_menu`

Fluxo:

1. radial **Prender**;
2. `InputDialog` com:
   - nome/citizenid do alvo (read-only);
   - três `select` de acusação (1 obrigatório, 2 e 3 opcionais com “Nenhuma”);
   - `textarea` de observação;
3. `mz_menu:InputDialog` devolve os valores; o cliente envia apenas `ruleCodes` e `notes` ao
   servidor;
4. o servidor responde com totais; o cliente mostra `AlertDialog`:
   “Pena total: 180s · Multa: R$ 0 · 1 acusação. Confirmar?”;
5. confirmação envia `arrest:commit`;
6. feedback e próximo passo.

Por que três selects:

- `mz_menu` NUI suporta `select`, `textarea` e `checkbox`
  (`mz_menu/web/app.js:530-559`);
- não há multi-select nativo com busca;
- a maioria das prisões tem 1–3 acusações;
- para 4+ acusações, o MDT continua disponível;
- não cria NUI nova, não cria biblioteca, não cria radial.

Limitações:

- catálogos grandes ficam ruins em `select`;
- não há soma dinâmica dentro do dialog;
- a revisão com totais exige uma segunda janela.

Mitigação: configurar `maxChargesPerQuickArrest = 3` e usar o MDT para casos complexos; ou
partir para a Opção 2 se o catálogo real tiver muitos crimes.

### 9.3 Opção 2 (se o catálogo crescer): modal compacto no NUI do MDT

Reutilizar o mesmo bundle `mz_police/web` (HTML/CSS/JS/lucide), com um modo compacto sem sidebar:

- lista pesquisável de acusações;
- checkbox por acusação;
- totais calculados a partir do snapshot server-side;
- observação;
- confirmação;
- ESC/focus tratados pelo `client/mdt.lua`.

Não é uma nova biblioteca: é uma rota/tela do NUI existente. Custo médio, benefício de UX alto.
Recomendação: **não fazer na P7.4**; avaliar após o catálogo real.

### 9.4 Feedback e estados

- `Aguardando resposta do servidor` (loading);
- `Caso criado` (sucesso);
- `Encaminhamento preparado; leve ao intake`;
- `Sentença iniciada`;
- `Multa pendente`;
- erros com código traduzido (`mz_police/web/app.js:45-76` já tem um dicionário; acrescentar os
  novos códigos);
- nunca exibir “preso” antes de a sentença existir; o caso é um registro, não uma sentença.

### 9.5 Foco, ESC e CEF

- `mz_menu` já gerencia foco e ESC; usar seus exports
  (`mz_menu/client/main.lua:1294-1325`);
- se usar o NUI do MDT, preservar `SetNuiFocus(true,true)` no open e `false,false` no close
  (`mz_police/client/mdt.lua:72-77`, `:108-110`);
- não abrir dois focos ao mesmo tempo (menu + MDT);
- testar 1366×768, 1920×1080 e 2560×1440; o MDT já tem QA de browser, mas CEF é NOT RUN.

### 9.6 Veredito visual

**Viável e de baixo risco na Opção 1.** A simplicidade vem de um dialog curto, não de um
dashboard. A Opção 2 é uma melhoria posterior, não um pré-requisito.

---

## 10. Segurança e arquitetura

### 10.1 Invariantes preservados

| Invariante | Como a proposta preserva |
|---|---|
| Server-authoritative | Todos os cálculos, validações e mutações no servidor |
| Fail-closed | Sem catálogo, custódia, prisão, inventário, economia ou config, a operação falha |
| Identidade/sessão canônicas | `ResolvePlayerIdentity`, `citizenid`, sessão; nunca `source` persistido |
| Organização/duty/capability | `mz_police.Policy.authorize` em toda operação |
| Distância/bucket | Revalidados no commit e no intake |
| Custódia | `cuffed`, `cuffedBy`, `revision`, alvo vivo |
| Código penal versionado | Snapshot `legalVersion` + comparação com o catálogo prisional |
| Sentença | `mz_prison` continua único owner |
| Propriedade | `mz_inventory` continua único owner |
| Idempotência | Chave natural por custódia/hash de charges; UNIQUEs |
| Auditoria | `mz_core:CreateDetailedLog` + históricos de domínio |
| Replay | Fingerprint e operação determinística; conflito explícito |
| Locks/transações | Lock por cidadão/caso; transação nas mutações |
| `mz_core` | Nenhuma alteração |
| Sem writers paralelos | Casos/charges em `mz_police`; sentença em `mz_prison`; itens em `mz_inventory` |

### 10.2 Modelo de ameaça

| Ameaça | Mitigação |
|---|---|
| Cliente envia duração/multa arbitrária | Servidor ignora totais e recalcula do catálogo |
| Cliente envia `ruleCode` inexistente/desabilitado | `configuredRule` falha fechado |
| Cliente falsifica alvo | Alvo resolvido pela custódia server-side, não por citizenid do payload |
| Cliente reaplica operação | `custodyRevision` + fingerprint + UNIQUE |
| Cliente cria caso duplicado | Case ID determinístico + lock |
| Cliente duplica multa | `UNIQUE(operation_id)` + chave econômica idempotente |
| Cliente duplica sentença | `getCurrent` + lock + `UNIQUE(charge_public_id)` |
| Cliente remove uniforme de outro | Export restrito a `mz_prison`, marcador por sentença |
| Cliente escapa do confinamento | Sweep server-side de posição/bucket + correção |
| Cliente troca bucket | Sweep reaplica; release restaura |
| Staff bypass | Nenhum comando novo de mutação; logs only |
| Replay de economia | `SettlePlayerOrganizationCharge` já tem idempotência |
| Replay de intake | Operação determinística por caso + charges |

### 10.3 Contratos que precisam ser adicionados

| Contrato | Owner | Por que | Risco |
|---|---|---|---|
| `arrest:preview` / `quote` / `commit` | `mz_police` | Orquestração atômica da prisão rápida | Médio |
| `custody:request` ação `transfer` | `mz_police` | Múltiplos policiais | Baixo |
| `requestSentence` agregado | `mz_police` | Intake multi-acusação | Médio |
| `RequestSentenceFromCharges` | `mz_prison` | Sentença multi-acusação | Alto |
| `mz_prison_sentence_charges` | `mz_prison` | Vínculo e unicidade por charge | Alto |
| `IssueChargeFine` (interno) | `mz_police` | Multa por charge sem tabela paralela | Médio |
| `promoteCase` | `mz_police` | Ocorrência → investigação | Baixo |
| `mz_police_case_links` | `mz_police` | Ocorrências relacionadas | Baixo |
| `mz_police_case_vehicles` | `mz_police` | Veículos relacionados | Médio |
| `PoliceResolveVehicleReference` | `mz_vehicles` | Validar placa no owner | Médio |
| `GetSentenceDetails` | `mz_prison` | Histórico da sentença no MDT | Baixo |
| `mz_prison_confinement` | `mz_prison` | Estado de confinamento | Alto |
| `IssuePrisonUniform` / `RemovePrisonUniform` | `mz_inventory` | Uniforme idempotente | Médio |
| `mz_prison_uniform` | `mz_prison` | Reconciliação do uniforme | Médio |

Nenhum desses contratos cria permissão, economia, notificação, log ou abstração de estado
paralela. Todos usam os owners existentes.

---

## 11. Limitações reais do FiveM/OneSync

1. **Server/client native split.** Muitas natives de câmera, IA, animação e controle são
   client-side. O servidor não dirige um NPC de forma confiável.
2. **Routing bucket.** Isola entidades, mas não contém o jogador; é necessário teleportar e
   monitorar área.
3. **Culling/focus.** Jogadores/entidades fora da focus zone podem não existir no cliente.
4. **Entity ownership.** O cliente que cria uma entidade pode perdê-la ao desconectar; NPCs
   client-owned não são confiáveis para transporte crítico.
5. **Reconnect/restart.** Estado em memória desaparece; só o banco sobrevive.
6. **CEF/NUI.** Foco, ESC e input precisam ser tratados; QA de browser não é CEF.
7. **MySQL/transações.** Harnesses com mock não provam locks, constraints e deadlocks reais.
8. **State bags.** Replicam estado, mas não são autoridade; o servidor deve revalidar.
9. **Posição do jogador.** O core persiste a última posição; o confinamento precisa reaplicar
   após spawn/reconnect.
10. **Death/respawn.** O `mz_medical` controla o respawn; o `mz_prison` deve observar
    `GetDeathState` e reaplicar, sem alterar o lifecycle do `mz_core`.

Consequências:

- transporte NPC é o item mais dependente de runtime;
- confinamento precisa de teste real de bucket, culling e reconnect;
- o uniforme precisa de teste real de equipamento e projeção visual;
- a prisão rápida em si é a parte menos dependente de runtime (lógica server-side + menu).

---

## 12. Matriz de viabilidade

Legenda:

- **Implementado (estático)**: código e harness existem; runtime NOT RUN.
- **Parcial**: parte existe, parte falta.
- **Ausente**: não existe contrato/fluxo.
- **Bloqueado por configuração**: existe, mas depende de catálogo/coordenada/asset.
- **Depende de runtime FiveM**: só pode ser confirmado in-game.

| Recurso | Já existe? | O que falta? | Resources afetados | Complexidade | Risco |
|---|---|---|---|---|---|
| **Prisão rápida** | **Parcial** | Ação `arrest`; orquestração atômica; quote/commit; UI; multi-acusação; multa | `mz_police`, `mz_prison`, `mz_menu`, `mz_target` | Média | Médio |
| **Caso automático** | **Parcial** | Transação única caso + pessoa + charges; marcador de origem; retry durável | `mz_police` | Média | Médio |
| **Investigação integrada** | **Parcial** | Promoção de caso; vínculos caso-caso; veículos; histórico de sentença; relatórios | `mz_police`, `mz_prison`, `mz_vehicles` | Média | Baixo/Médio |
| **Transporte manual** | **Implementado (estático)** | UX de rota; transferência de custódia; lembrete de intake | `mz_police`, `mz_prison` | Baixa | Baixo |
| **Transporte NPC** | **Ausente** | Sessão server-side; spawn no cliente do policial; watchdog; fallback; validação de chegada | `mz_police`, `mz_transport`/novo, `mz_garagem` | Alta | Alto |
| **Confinamento** | **Ausente** | Célula/área; bucket; sweep; correção; loop guard; restore de bucket; recover | `mz_prison`, `mz_prison` client, config | Alta | Alto |
| **Uniforme prisional** | **Parcial / bloqueado por asset** | Garment aprovado; exports restritos; ordem retenção/emissão/devolução; reconciliação | `mz_inventory`, `mz_clothing`, `mz_prison` | Média | Médio |
| **Recuperação pós-restart** | **Parcial** | Estado de confinamento; uniforme; bucket; retomada de sentença ativa | `mz_prison`, `mz_inventory` | Média/Alta | Alto |
| **Sentença multi-acusação** | **Ausente** | Tabela de vínculo; export; soma; unicidade por charge | `mz_prison`, `mz_police` | Alta | Alto |
| **Multa por acusação** | **Parcial / bloqueado por configuração** | Método de liquidação de charge; citations; capability; UI | `mz_police`, `mz_economy` | Média | Médio |
| **Transferência de custódia** | **Ausente** | Ação `transfer`; sincronização; auditoria | `mz_police` | Baixa | Médio |
| **Promoção de caso** | **Ausente** | Evento/método; update de `case_kind`; histórico | `mz_police`, MDT | Baixa | Baixo |
| **Ocorrências relacionadas** | **Ausente** | Tabela e eventos de vínculo | `mz_police`, MDT | Média | Baixo |
| **Veículos relacionados** | **Ausente** | Tabela; contrato de resolução no `mz_vehicles`; MDT | `mz_police`, `mz_vehicles` | Média | Médio |
| **Histórico de sentença no MDT** | **Parcial** | Export de leitura; UI | `mz_prison`, `mz_police` | Baixa | Baixo |

### 12.1 Status por proposta

| Proposta | Veredito | Observação |
|---|---|---|
| A — Prisão rápida | **Viável** | Melhor custo/benefício; começar por ela |
| B — Investigação | **Viável** | Promoção de caso é o quick win; veículos/relacionados depois |
| C — Transporte | **Manual** | NPC opcional; teleporte não |
| D — Confinamento | **Viável, alto risco** | Só depois de A e com gate in-game |
| E — Uniforme | **Viável, médio risco** | Depende de asset aprovado |

---

## 13. Waves propostas

A ordem abaixo segue dependências reais, não apenas nomes:

### P7.4 — Prisão rápida + caso automático + sentença multi-acusação

**Objetivo:** prender em um radial, com caso e acusações criados atomicamente, sem iniciar
sentença fora do intake.

Escopo:

- ação `arrest` no `mz_target` e no contexto server-side;
- `arrest:preview` / `quote` / `commit`;
- transação caso + pessoa + charges + histórico;
- `mz_prison_sentence_charges` + `RequestSentenceFromCharges`;
- intake agregado por caso;
- multa por charge reutilizando citations/economy;
- auditoria e mensagens;
- testes de harness/Node.

Dependências: nenhuma wave nova; usa P1–P7.

Risco: médio. É a base das demais.

Critério de saída:

- harness de autorização, payload, custódia, concorrência, replay, catálogo, multi-charge e multa;
- contrato Node do NUI/MDT;
- `mz_testkit covered` PASS;
- runtime: duas prisões reais, uma com 1 e outra com 2 acusações; reconnect; timeout/retry.

### P7.5 — Custódia multi-policial + transporte manual

**Objetivo:** permitir que outro policial transporte e faça o intake, sem re-algemar.

Escopo:

- ação `transfer` na custódia;
- UX de rota/lembrete de intake;
- teste com dois policiais e dois veículos;
- (opcional) NPC atrás de flag, somente após P7.4/P7.5 estáveis.

Dependências: P7.4 (caso/charges) e custódia P2.

Risco: baixo/médio. NPC é a parte de alto risco; separar.

Critério de saída:

- custódia transferida, antigo policial perde autoridade, novo assume;
- intake pelo novo policial;
- desconexão do antigo não trava o preso;
- runtime com dois policiais.

### P7.6 — Confinamento + recuperação

**Objetivo:** prender fisicamente, com bucket/área, sem quebrar o personagem.

Escopo:

- `mz_prison_confinement`;
- config de células/áreas/bucket;
- sweep, correção, loop guard;
- restore de bucket na soltura (mesmo sem destino);
- reconnect/restart/morte/respawn;
- comando de diagnóstico read-only.

Dependências: P7.4 (sentença) e P7.5 (transporte/entrega).

Risco: alto. Exige gate in-game dedicado.

Critério de saída:

- preso entra na cela, não sai sem autorização;
- fuga é corrigida sem loop;
- soltura restaura bucket mesmo sem destino;
- reconnect/restart mantêm estado;
- morte/respawn não prendem o jogador no bucket errado;
- dois presos, dois buckets/áreas, sem colisão.

### P7.7 — Uniforme + propriedades

**Objetivo:** aplicar uniforme prisional sem duplicar/perder itens.

Escopo:

- garment aprovado/config;
- `IssuePrisonUniform` / `RemovePrisonUniform`;
- `mz_prison_uniform` para reconciliação;
- ordem retenção → emissão → devolução;
- reconnect/restart/inventário cheio.

Dependências: P7.6 (confinamento) para o fluxo completo; o contrato de inventário pode ser
implementado/testado antes.

Risco: médio. Exige asset e teste com inventário cheio.

Critério de saída:

- roupa original preservada com UID;
- uniforme não duplicado;
- uniforme removido na soltura;
- restart/reconnect recuperam corretamente;
- projeção visual aplicada (CEF/runtime).

### P7.8 — Investigação integrada + experiência final

**Objetivo:** transformar o caso rápido em investigação completa e fechar a UX.

Escopo:

- `promoteCase`;
- `mz_police_case_links` (ocorrências relacionadas);
- `mz_police_case_vehicles` + `PoliceResolveVehicleReference`;
- histórico de sentença no MDT;
- (opcional) relatórios estruturados;
- modal compacto se o catálogo exigir;
- QA visual/CEF.

Dependências: P7.4–P7.7.

Risco: baixo/médio. É majoritariamente aditivo.

Critério de saída:

- caso rápido abre no MDT, recebe evidências/perícias/mandados e é promovido sem duplicar;
- veículos e ocorrências relacionadas aparecem;
- histórico de sentença visível;
- NUI/CEF sem overflow/foco preso.

### P7.9 — Transporte NPC (opcional)

**Objetivo:** transporte assistido por NPC, se houver demanda real.

Escopo:

- sessão de transporte server-side;
- spawn no cliente do policial;
- watchdog, timeout, entity lost, disconnect, restart;
- validação de chegada no intake;
- fallback manual.

Dependências: P7.5/P7.6 validados.

Risco: alto. Só avançar com runtime estável e aprovação explícita.

### 13.1 O que pode ser feito independentemente

| Item | Independência |
|---|---|
| `promoteCase` | Sim; não depende de prisão rápida |
| `mz_police_case_links` | Sim; aditivo |
| `mz_police_case_vehicles` + resolução | Sim; aditivo |
| `IssuePrisonUniform` / `RemovePrisonUniform` | Sim; testável com harness de inventário |
| Sentença multi-acusação | Depende do formato de charges do P7.4, mas pode ser desenhada/testada antes |
| Confinamento | Depende de sentença ativa; pode ser desenvolvido em paralelo, mas só valida depois |
| NPC | Depende de P7.5/P7.6 e de runtime |

### 13.2 O que exige teste in-game

- radial/menu/target e foco CEF;
- custódia com dois jogadores e veículo de rede;
- intake e sentença;
- confinamento/bucket/reconnect/restart/morte;
- uniforme/projeção visual;
- NPC/entidades;
- economia/MySQL real;
- performance e culling.

Nenhum desses itens pode ser declarado PASS apenas com harness local.

---

## 14. Respostas objetivas

### 14.1 É viável ter prisão rápida sem perder o MDT completo?

**Sim.** A prisão rápida cria um caso normal (`occurrence`) nas tabelas P6. O MDT continua
lendo o mesmo caso, com pessoas, charges, evidências, perícias e histórico. A diferença é a
orquestração: um dialog curto em vez de navegar pelo dashboard.

### 14.2 Podemos gerar casos simplificados automaticamente?

**Sim.** Com uma transação única em `mz_police`, usando `mz_police_cases`,
`mz_police_case_officers`, `mz_police_case_people`, `mz_police_case_charges` e
`mz_police_history`. Não é necessário criar tabela de “caso simplificado”. O caso é o registro
canônico; “simplificado” é o fluxo, não o schema.

### 14.3 Podemos aproveitar integralmente P1–P9?

**Quase integralmente, com lacunas específicas.**

- P1/P2: policy, custódia, revista, veículo — reutilizáveis.
- P3: estações/armaria — reutilizáveis, mas catálogo/coords ainda pendentes.
- P4: fiscalização/impound — reutilizáveis; não é pré-requisito.
- P5: evidências/cadeia de custódia — reutilizáveis.
- P6: casos/charges/multas/mandados — base da prisão rápida.
- P7/P7.1/P7.2: sentença/intake/propriedade/atividades/soltura — base, mas **single-charge**.
- P7.3: radial/MDT — base da UI.
- P8: dispatch — independente; pode notificar, mas não é necessário para prender.
- P9: perícia `weapon_identity` — existe, mas globalmente desligada; não é pré-requisito.

As lacunas são: multi-charge, multa de charge, transferência de custódia, confinamento,
uniforme, promoção de caso, vínculos de caso/veículo e histórico de sentença no MDT.

### 14.4 Quais contratos precisam ser adicionados?

Os contratos da seção 10.3. Em resumo:

1. `arrest:preview` / `arrest:quote` / `arrest:commit` em `mz_police`;
2. `RequestSentenceFromCharges` + `mz_prison_sentence_charges` em `mz_prison`;
3. `custody:transfer` em `mz_police`;
4. `IssueChargeFine` interno em `mz_police` (reutilizando citations/economy);
5. `promoteCase` em `mz_police`;
6. `mz_police_case_links` e `mz_police_case_vehicles`;
7. `PoliceResolveVehicleReference` em `mz_vehicles`;
8. `GetSentenceDetails` em `mz_prison`;
9. `mz_prison_confinement` em `mz_prison`;
10. `IssuePrisonUniform` / `RemovePrisonUniform` em `mz_inventory` + `mz_prison_uniform`.

### 14.5 Qual transporte prisional vale mais a pena?

**Manual.** É o mais realista, confiável, barato e compatível. NPC é opcional e posterior;
teleporte disfarçado não deve existir. O ganho de tempo vem da prisão rápida, não de mover o
preso sem transporte.

### 14.6 Como implementar confinamento sem quebrar o personagem?

- bucket dedicado + áreas permitidas + sweep server-side;
- persistir `previous_bucket` e restaurar **sempre** na soltura;
- reaplicar no reconnect/restart/morte/respawn;
- loop guard e kill switch;
- usar apenas exports públicos do `mz_core` (`GetDeathState`, `ResolvePlayerIdentity`,
  `GetSourceByCitizenId`);
- não alterar o lifecycle do `mz_core`;
- validar com dois jogadores, reconnect, restart, morte e soltura sem destino.

### 14.7 É viável integrar uniformes com `mz_clothing`?

**Sim.** O uniforme é um `clothing_garment` único com metadata `clothing` e marcador
`prisonUniform`. O `mz_inventory` emite/remove por export restrito; o `mz_clothing` reconcilia a
projeção visual. Não criar inventário, catálogo ou sistema de aparência paralelo. O bloqueio
atual é a ausência de um garment/drawable aprovado.

### 14.8 Quais mudanças envolvem maior risco?

1. Confinamento (bucket, teleporte, reconnect, release).
2. Sentença multi-acusação (regressão do P7).
3. Transporte NPC (entidade client-owned, perda/desconexão).
4. Uniforme (duplicação/perda de item, ordem de retenção).
5. Runtime FiveM (OneSync, CEF, MySQL) — todo o resto depende dele.

### 14.9 Quais recursos devem ser modificados?

Obrigatórios:

- `mz_police` (arrest, custódia, casos, MDT, multa);
- `mz_prison` (multi-charge, confinamento, uniforme, release);
- `mz_inventory` (uniforme restrito);
- `mz_target` (consumidor da ação `arrest`; provavelmente sem mudança de código se o registro vier do `mz_police`);
- `mz_menu` (consumidor; provavelmente sem mudança de código);
- `mz_org` (apenas capabilities, via seed/config, sem novo sistema);
- `mz_economy` (reutilizado; sem mudança);
- `mz_vehicles` (contrato de referência de veículo, se P7.8);
- `mz_clothing` (reutilizado; possível ajuste de reconcile, se necessário);
- `mz_docs` (este relatório e os relatórios de wave);
- `mz_starter` (ordem/boot, se novos arquivos/recursos não forem adicionados; não criar resource novo).

**Não modificar:** `mz_core`.

### 14.10 Qual sequência de implementação você recomenda?

**P7.4 → P7.5 → P7.6 → P7.7 → P7.8 → (P7.9 opcional).**

- P7.4 entrega o maior ganho de UX com o menor risco de runtime.
- P7.5 resolve o transporte/multi-policial antes de prender fisicamente.
- P7.6 confina, com gate in-game.
- P7.7 veste o preso, com reconciliação.
- P7.8 completa a investigação/MDT.
- P7.9 (NPC) só se ainda houver demanda e após runtime estável.

---

## 15. Decisões pendentes antes da implementação

1. **Catálogo legal de produção**: versão, crimes, penas, multas e jurisdição. O staging atual
   não é produção.
2. **Multa**: cobrar no intake ou no booking? Recomendação: intake, com citation pendente se
   não houver saldo.
3. **Capability de multa**: exigir `police.citation.issue` para charges com multa? Recomendação:
   sim; bloquear a seleção sem a capability.
4. **Limite de acusações na prisão rápida**: 3 (mz_menu) ou 8 (NUI)? Recomendação: 3 na P7.4;
   MDT para casos complexos.
5. **Transferência de custódia**: permitir que qualquer policial com `police.custody` receba,
   ou exigir hierarquia? Recomendação: capability + proximidade + duty; sem hierarquia nova.
6. **Confinamento**: bucket dedicado? Faixa reservada? Células compartilhadas ou uma por preso?
   Recomendação: bucket dedicado, faixa documentada, uma célula por preso quando houver cela
   livre.
7. **Uniforme**: garment/drawable aprovado e slot. Sem asset, manter desligado.
8. **Origem do caso**: histórico (`quick_arrest_created`) ou coluna `origin`? Recomendação:
   histórico na P7.4; coluna só com migração aditiva aprovada.
9. **NPC**: entra no escopo? Recomendação: não na P7.4/P7.5; P7.9 opcional.
10. **Runtime gate**: quem valida e em qual staging? Sem isso, nenhuma wave é “PASS”.

---

## 16. Índice de evidências

### 16.1 `mz_police`

| Evidência | Arquivo:linha |
|---|---|
| Ações atuais do radial | `mz_police/client/interactions.lua:225-235` |
| Registro do radial e submenu | `mz_police/client/interactions.lua:256-274` |
| `PendingIntake` local | `mz_police/client/interactions.lua:295-304` |
| Evento de intake | `mz_police/client/interactions.lua:306-318` |
| Resultado da sentença no cliente | `mz_police/client/interactions.lua:360-367` |
| Contexto server-side sem `arrest` | `mz_police/server/interactions.lua:66-94` |
| Ações de custódia | `mz_police/server/custody.lua:10-17` |
| Transições de custódia | `mz_police/server/custody.lua:265-341` |
| Resolução de veículo/assento | `mz_police/server/custody.lua:143-190` |
| Veículo put/remove | `mz_police/server/custody.lua:298-311` |
| Cleanup de custódia | `mz_police/server/custody.lua:413-426` |
| Policy | `mz_police/server/policy.lua:76-142` |
| Catálogo jurídico | `mz_police/server/records/service.lua:152-177` |
| Criação de caso | `mz_police/server/records/service.lua:207-233` |
| Update de caso sem `kind` | `mz_police/server/records/service.lua:261-297` |
| Pessoa do caso | `mz_police/server/records/service.lua:299-326` |
| Acusação | `mz_police/server/records/service.lua:328-360` |
| Citation | `mz_police/server/records/service.lua:362-424` |
| Warrant | `mz_police/server/records/service.lua:426-488` |
| Repositório de caso | `mz_police/server/records/repository.lua:50-89` |
| Acusação no repositório | `mz_police/server/records/repository.lua:166-179` |
| Schema de casos/charges | `mz_police/server/records/schema.lua:33-80` |
| Schema de citations | `mz_police/server/records/schema.lua:115-133` |
| Histórico | `mz_police/server/records/schema.lua:151-166` |
| Sentença single-charge | `mz_police/server/prison.lua:66-116` |
| Custódia do mesmo policial | `mz_police/server/prison.lua:77-79` |
| Eventos de registros | `mz_police/server/events.lua:131-147` |
| Eventos prisionais | `mz_police/server/events.lua:164-178` |
| NUI do MDT | `mz_police/client/mdt.lua:176-251` |
| `prepareIntake` local | `mz_police/client/mdt.lua:194-204` |
| Abrir MDT pelo cidadão | `mz_police/client/mdt.lua:284-290` |
| Seções do MDT | `mz_police/web/app.js:15-26` |
| Tela de acusações | `mz_police/web/app.js:293-317` |
| Preparar intake na UI | `mz_police/web/app.js:549-558` |
| Config de staging | `mz_police/config/server.lua:84-116` |
| P9 desligado | `mz_police/config/server.lua:121-158` |

### 16.2 `mz_prison`

| Evidência | Arquivo:linha |
|---|---|
| Intake point | `mz_prison/server/service.lua:210-232` |
| RequestSentence | `mz_prison/server/service.lua:293-378` |
| Catálogo/clock | `mz_prison/server/service.lua:323-329` |
| Lock/active sentence | `mz_prison/server/service.lua:336-358` |
| Retain/activate | `mz_prison/server/service.lua:359-374` |
| ReleasePoint | `mz_prison/server/service.lua:81-111` |
| DispatchRelease/bucket | `mz_prison/server/service.lua:124-143` |
| ReleaseSentence | `mz_prison/server/service.lua:234-291` |
| Resume | `mz_prison/server/service.lua:626-660` |
| Tick online | `mz_prison/server/service.lua:662-686` |
| Schema sentences | `mz_prison/server/prepare.lua:23-46` |
| Schema property | `mz_prison/server/prepare.lua:47-62` |
| Schema history | `mz_prison/server/prepare.lua:63-76` |
| Schema activities | `mz_prison/server/prepare.lua:77-96` |
| Schema deliveries | `mz_prison/server/prepare.lua:97-109` |
| Sem ALTER | `mz_prison/server/prepare.lua:10-19` |
| Config de staging | `mz_prison/config.lua:5-84` |
| Release client | `mz_prison/client/release.lua:1-20` |
| PlayerLoaded resume | `mz_prison/server/main.lua:47-49` |
| PlayerDropped | `mz_prison/server/main.lua:43-45` |
| Bridge de propriedade | `mz_prison/server/bridge.lua:44-60` |
| Audit | `mz_prison/server/bridge.lua:62-82` |
| Repositório current | `mz_prison/server/repository.lua:101-105` |
| Repositório create | `mz_prison/server/repository.lua:107-120` |
| Repositório activate | `mz_prison/server/repository.lua:122-147` |

### 16.3 `mz_inventory` e `mz_clothing`

| Evidência | Arquivo:linha |
|---|---|
| Item `clothing_garment` | `mz_inventory/shared/items.lua:2-14` |
| Inventory types | `mz_inventory/shared/constants.lua:6-17` |
| Retain property | `mz_inventory/server/service.lua:1979-2038` |
| Get property | `mz_inventory/server/service.lua:2040-2053` |
| Restore property | `mz_inventory/server/service.lua:2055-2160` |
| Exports prisionais restritos | `mz_inventory/server/exports.lua:185-202` |
| Add/Remove abertos | `mz_inventory/server/exports.lua:133-139` |
| Outfit save/apply | `mz_clothing/server/main.lua:432-471` |
| Exports de clothing | `mz_clothing/server/main.lua:514-520` |
| Contrato de roupa | `mz_docs/inventory/CLOTHING_EQUIPMENT.md:13-18, 48-52, 66-76` |
| Categorias de loja | `mz_clothing/shared/catalog.lua:1-27` |

### 16.4 UI, target, menu, notify

| Evidência | Arquivo:linha |
|---|---|
| Target API | `mz_docs/api/TARGET_API.md:22-34` |
| Limite radial | `mz_target/shared/config.lua:11-19` |
| `InputDialog` | `mz_menu/client/main.lua:846-865` |
| Tipos de input | `mz_menu/web/app.js:530-559` |
| Exports do menu | `mz_menu/client/main.lua:1294-1325` |
| Notify server | `mz_notify/server/main.lua:116-123` |
| Notify README | `mz_notify/README.md:56-96` |
| `mz_frontend` é demo | `mz_frontend/README.md:1-20` |

### 16.5 Organização, economia, veículos, transporte

| Evidência | Arquivo:linha |
|---|---|
| Capabilities policiais | `mz_org/shared/capabilities.lua:62-81` |
| Seed policial | `mz_org/server/domain/default_orgs.lua:304-317` |
| Settle charge | `mz_economy/server/accounts/police_charges.lua:44-141` |
| NPC provider server | `mz_transport/server/providers/population_standalone.lua:8-167` |
| NPC provider client | `mz_transport/client/providers/population_standalone.lua:65-167` |

---

## 17. Impacto arquitetural

**Architectural impact: documentação apenas; nenhum código, configuração, SQL ou migration foi
alterado.**

- Contratos públicos: nenhum alterado nesta execução; contratos propostos na seção 10.3.
- Ownership canônico: inalterado. `mz_police` continua casos/charges; `mz_prison` continua
  sentença/intake/propriedade/confinamento; `mz_inventory` continua itens/uniforme.
- Writer/authority: inalterado. Nenhum writer paralelo proposto.
- Dependências cross-resource: propostas são `mz_police` ↔ `mz_prison` ↔ `mz_inventory` +
  `mz_economy`, com `mz_vehicles` apenas para vínculo de veículo.
- Persistência: nenhuma alterada. Tabelas propostas são aditivas e sujeitas a aprovação:
  `mz_prison_sentence_charges`, `mz_prison_confinement`, `mz_prison_uniform`,
  `mz_police_case_links`, `mz_police_case_vehicles`.
- Segurança/permissões: nenhuma permissão enfraquecida; novas operações usam capabilities
  existentes.
- Novos exports/eventos/callbacks: propostos, não implementados.
- Dependências externas: nenhuma nova.
- Graphify: consultado; insuficiente; nenhuma atualização (tarefa read-only).
- Graphify synchronization: not required.
- Graphify validation: NOT APPLICABLE.
- Validação pendente: toda a validação de runtime FiveM/OneSync/CEF/MySQL.

---

## 18. Conclusão

A MZ Framework já tem a base correta para um fluxo policial simples: custódia revisionada,
catálogo jurídico server-side, casos persistentes, evidências, sentença, intake, propriedade e
soltura. O que falta não é “um sistema policial mais simples”; é uma **camada de orquestração**
que esconda a burocracia do MDT nas situações comuns, sem remover nenhuma verificação.

A arquitetura recomendada é:

- **um único backend policial** (`mz_police`) e **um único backend prisional** (`mz_prison`);
- **um único inventário** (`mz_inventory`);
- **um único MDT**, que continua sendo a interface avançada;
- **um caso normal** criado pela prisão rápida, não um caso paralelo;
- **uma sentença multi-acusação** com tabela de vínculo e validação server-side;
- **transporte manual** como padrão, NPC apenas opcional;
- **confinamento por bucket + área**, com restauração obrigatória de bucket;
- **uniforme como item**, não como sistema de roupa paralelo.

A ordem segura é **P7.4 → P7.5 → P7.6 → P7.7 → P7.8**, com P7.9 (NPC) somente se ainda houver
demanda e depois de runtime estável. Nenhuma dessas waves pode ser declarada operacional sem
teste real no FXServer.

**Aguardando aprovação explícita antes de qualquer implementação.**
