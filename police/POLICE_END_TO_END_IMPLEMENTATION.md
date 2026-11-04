# Polícia end-to-end — implementação

Revisado em 2026-10-18. Este documento descreve o código atual; não declara homologação em FXServer.

## Resultado

O fluxo foi organizado em três etapas físicas e uma etapa jurídica, sem criar outro owner:

1. `mz_target` registra uma única raiz `police_player_actions`; `mz_police` autoriza algema, revista, escolta, viatura, identificação, MDT e transferência de custódia.
2. `Processar prisão` só aparece quando o policial detém a custódia e ambos estão no mesmo `processingPoint`, no mesmo bucket.
3. O MDT chama `arrest:preview`, `quote` e `commit` em modo `processing`. O servidor deriva artigos, versão, pena e multa, exige relatório e grava ocorrência, pessoa, charges, histórico e citations no writer policial existente.
4. O commit apenas deixa o encaminhamento `Pendente de intake`. A sentença só é criada/ativada quando policial e custodiado chegam ao intake configurado e acionam a entrada física.
5. `mz_prison` retém propriedade via `mz_inventory`, aplica uniforme somente se a flag e garments reais estiverem configurados, ativa a sentença, define bucket/cela no servidor e exige ACK validado.
6. Reconnect/restart recompõe a cela por `citizenid`. Um sweep limitado corrige saída de bucket/área com cooldown. Na soltura, uniforme é removido, propriedade é restituída e o bucket anterior é restaurado mesmo sem coordenada de saída.

`mz_core` não foi alterado. Transporte NPC, fiança e lei real continuam fora do escopo/sem política aprovada.

## Antes e depois

| Antes | Depois |
|---|---|
| prisão rápida podia ser a conclusão operacional | backend preservado para compatibilidade, flag real desligada e nenhuma ação `Prender` na rua |
| registry policial era diagnosticado apenas pelo handle local | `GetOption` real, raiz única, retry/restart e harness cruzando `mz_target` + `mz_police` |
| não havia transferência explícita do custodiante | transferência valida novo policial, distância, bucket, policy, revisão e auditoria |
| MDT não conduzia preview/quote/commit no balcão | processamento abre o cidadão correto, exige relato, calcula no servidor e termina como `Pendente de intake` |
| intake candidato não continha o preso | facility/cela/bucket/ACK/reconnect/sweep existem; defaults sem coordenadas reais falham fechados |
| uniforme prisional não tinha contrato restrito | apply/remove usam equipment, UID e metadata canônicos do `mz_inventory`, com flag/garments desligados |

## Owners e contratos

| Owner | Responsabilidade usada |
|---|---|
| `mz_police` | policy, custódia, caso, charge, citation, MDT, histórico e preparação de intake |
| `mz_target` | registry e apresentação target/F3; `GetOption` permite diagnóstico sanitizado pelo owner |
| `mz_org` | membership, duty, grade e capabilities |
| `mz_prison` | intake, sentença, facility/cela/bucket, confinamento, atividade e soltura |
| `mz_inventory` | itens/UID/metadata, propriedade retida e `clothing_garment` prisional |
| `mz_economy` | liquidação idempotente de citation, sempre após confirmação do cidadão |
| `mz_dispatch` | fila persistente de chamados; MDT apenas consulta/aceita/fecha o owner |

Contratos adicionados/estendidos:

- `mz_target:GetOption(handle)`: retorna somente handle, id, owner, quick e IDs dos filhos do próprio resource.
- custody action `transfer`, com `newOfficerSource`, revisão otimista, proximity/bucket e auditoria old/new.
- arrest payload `mode = processing|quick`; `quick` continua compatível, mas `quickEnabled=false` no config real.
- `mz_inventory:ApplyPrisonUniform` e `RemovePrisonUniform`, server-only e restritos a `mz_prison`.
- eventos `mz_prison:client:confinement` e `mz_prison:server:confinement:ack`.
- solicitação de pagamento de citation: o policial solicita; o cidadão confirma e chama o pagamento canônico em seu próprio source.

## Matriz requisito → implementação → prova automatizada

| Requisito | Estado no código | Arquivos principais | Prova |
|---|---|---|---|
| raiz única no target/F3, restart e retry | implementado | `mz_target/client/main.lua`, `mz_police/client/interactions.lua` | `p10_target_registry_integration_harness.lua` |
| sem botão de prender na rua | implementado | `mz_police/client/interactions.lua` | `p7_4_quick_arrest_contract.test.js` |
| transferência de custódia | implementado | `mz_police/server/custody.lua` | `p2a_custody_harness.lua` |
| processamento restrito ao ponto | implementado, ativação bloqueada sem coordenadas | `mz_police/server/prison.lua`, `server/arrest.lua` | `p7_4_multi_and_fines_harness.lua` |
| MDT preview/quote/commit e relatório | implementado | `mz_police/client/mdt.lua`, `web/app.js` | JS syntax/contratos + harness P7.4 |
| multi-charge até 8 + soma + teto | implementado, flags e teto de produção desligados | configs e services de police/prison | harnesses P7.4 dos dois resources |
| multa com consentimento | implementado | `server/records/service.lua`, `events.lua`, `client/arrest.lua`, MDT | harness multi/fines |
| intake valida ambos | implementado | `mz_prison/server/service.lua` | `p10_confinement_harness.lua` |
| cela/bucket/ACK/reconnect/sweep | implementado, facility real ausente | `mz_prison/server/service.lua`, `client/release.lua` | `p10_confinement_harness.lua` |
| uniforme canônico | mecanismo implementado, assets/config desligados | `mz_inventory/server/service.lua`, `mz_prison/server/bridge.lua` | contract harness; runtime não executado |
| soltura e bucket anterior | implementado | `mz_prison/server/service.lua` | release harness + inspeção contratual |
| dispatch no mesmo MDT | preservado | `mz_police/client/mdt.lua`, `web/app.js` | harnesses P8 |

## Schema

`mz_prison/sql/p10_end_to_end_additive.sql` adiciona, de forma idempotente:

- `facility_id`;
- `intake_point_id`;
- `cell_id`;
- `previous_routing_bucket`;
- `confinement_status`.

A migration não foi executada. O bootstrap do resource valida o contrato e falha fechado em schema legado; aplique a migration manualmente conforme o runbook.

## Defaults deliberadamente seguros

- `MZPoliceConfig.Prison.processingEnabled = false` e `processingPoints = {}`.
- `MZPoliceConfig.Arrest.quickEnabled = false`.
- multi-charge permanece desligada; `maximumSentenceSeconds = 0` impede ativação acidental.
- `MZPrisonConfig.intakeEnabled = false`, `facilities = {}` e confinement desligado.
- uniforme desligado e sem garments.
- jobs e release points com coordenadas apenas candidatas ficam desativados; nenhuma coordenada é inventada.
- release sem ponto real restaura apenas o bucket anterior.

## Riscos e limites reais

- FiveM/OneSync, oxmysql, MLO, morte/respawn e três clientes não estavam disponíveis neste ambiente: runtime permanece `NOT RUN`.
- Facility, processamento, cela, áreas, saída e garments precisam de dados reais aprovados antes de ligar as flags.
- A migration usa `ADD COLUMN IF NOT EXISTS`; confirme compatibilidade com a versão MySQL/MariaDB da VPS em uma cópia temporária.
- O uniforme usa o item existente `clothing_garment`; drawable/componentes continuam responsabilidade da configuração aprovada de clothing.

## Validação executada

- 51 arquivos de teste dos recursos principais: `mz_police` (30), `mz_prison` (8), `mz_target` (6) e `mz_dispatch` (7): **PASS**.
- contratos adicionais: capability policial do `mz_org`, cobrança policial do `mz_economy`, propriedade prisional e rate limit de munição do `mz_inventory`: **PASS**.
- `luac -p` nos Lua tocados, `node --check web/app.js` e `git diff --check` nos cinco repositórios afetados: **PASS**.
- smoke local Playwright/Chrome em `1440x900` e `800x900`: fluxo processamento → quote → commit → pendente de intake, sem erro/warning no console e sem overflow horizontal: **PASS local**, não FiveM.
- `mz_testkit/runners/run-local-suite.ps1`: **FAIL preexistente** em harness global porque `mz_inventory/server/service.lua` chama o global FiveM `Player`, ausente no mock (`Player=nil`).
- `mz_testkit/runners/run-fivem-runtime-suite.ps1`: perfil válido; quatro suites listadas como **NOT_RUN**. Nenhum FXServer foi iniciado.

## Matriz final de entrega

| Categoria | Itens |
|---|---|
| implemented + tests PASS | registry/target, custódia e transferência, processamento MDT, multi-charge em fixture, citations com consentimento, intake/facility, confinamento, release/bucket, contratos de uniforme e dispatch |
| implemented + runtime NOT RUN | fluxo físico completo com 2–3 jogadores, OneSync, morte/respawn, restart coordenado, oxmysql e CEF FiveM |
| blocked by approved real-world config/assets | pontos de processamento/intake/cela/áreas/saída, buckets do MLO, garments/drawables, teto e catálogo jurídico reais |
| bug still open | suíte global para em `Player=nil` no mock do `mz_inventory`; reproduzir com `powershell -File .\\mz_testkit\\runners\\run-local-suite.ps1`. Dois harnesses gerais do inventário também já falham fora deste escopo: blur do frontend e controles seguros em veículo |

