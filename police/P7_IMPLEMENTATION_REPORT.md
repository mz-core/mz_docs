# MZ Police — relatório de implementação P7

Data: 2026-10-14
Wave: P7 — sistema prisional
Estado: **gate estático aprovado; configuração legal, coordenadas e runtime FiveM/MySQL pendentes**

## Resultado implementado

Foi criado o resource independente `mz_prison`, proprietário de:

- recebimento allowlisted de sentenças solicitadas por `mz_police`;
- sentença persistente e histórico idempotente;
- intake com validação server-side de coordenada, raio e routing bucket;
- retenção e devolução reconciliável da propriedade pelo owner `mz_inventory`;
- relógio de pena somente durante sessão online;
- redução configurável e soltura manual configurável;
- recuperação de estados `intake_pending`, `active` e `release_pending` após reconnect/restart;
- auditoria central por `mz_core`.

O cliente nunca informa duração, saldo de pena ou condição de soltura. `mz_police` revalida identidade, organização, capability, duty, custódia, distância e routing bucket antes de chamar o owner prisional.

## Ownership e contratos

- identidade, sessão, personagem e logs: `mz_core`;
- organização, capability e duty: `mz_org`;
- custódia e autorização operacional: `mz_police`;
- itens, UID, metadata, slots, locks e transações: `mz_inventory`;
- sentença, relógio, intake, histórico e soltura: `mz_prison`.

Exports server-side de `mz_prison`, restritos ao caller exato `mz_police`:

- `RequestSentence(actorSource, targetSource, request)`;
- `GetSentence(citizenid)`;
- `ReduceSentence(actorSource, citizenid, request)`;
- `ReleasePrisoner(actorSource, citizenid, request)`.

Exports server-side adicionados ao owner `mz_inventory`, restritos ao caller exato `mz_prison`:

- `RetainPrisonerProperty(targetSource, sentenceId)`;
- `GetPrisonerProperty(targetSource, sentenceId)`;
- `RestorePrisonerProperty(targetSource, sentenceId)`.

Não foram criados eventos públicos de mutação em `mz_prison`, bypass administrativo ou lógica prisional no `mz_core`.

## Persistência e recuperação

O resource prepara somente tabelas próprias com `CREATE TABLE IF NOT EXISTS`:

- `mz_prison_sentences`;
- `mz_prison_property`;
- `mz_prison_history`.

As operações têm operation ID e fingerprint persistentes. Replays equivalentes recuperam o resultado; reuso divergente é rejeitado. Locks por cidadão evitam concorrência local e as transições persistentes usam transações. A propriedade permanece fisicamente no owner `mz_inventory`; `mz_prison_property` registra apenas a reconciliação da sentença.

Quando o alvo está offline, o relógio não avança e uma soltura pendente não perde a propriedade. Ao voltar, o tick resolve novamente o `source` pelo `citizenid` canônico e retoma a transição. Devoluções do inventário são idempotentes.

## Configuração fail-closed

Permanecem desativados/vazios por padrão:

- `intakeEnabled = false` e `intakePoints = {}`;
- `sentenceRulesEnabled = false`, `jurisdictions = {}` e `sentenceRules = {}`;
- `reductionRulesEnabled = false` e `reductionRules = {}`;
- `manualReleaseEnabled = false` e `manualReleaseReasons = {}`;
- `jobsEnabled = false` e `jobs = {}`;
- integração `MZPoliceConfig.Prison.enabled = false`.

Para informar coordenadas reais de intake, configurar no servidor, após medição e aprovação:

```lua
MZPrisonConfig.intakeEnabled = true
MZPrisonConfig.intakePoints = {
  booking = {
    coords = { x = COORD_X_REAL, y = COORD_Y_REAL, z = COORD_Z_REAL },
    radius = RAIO_APROVADO,
    routingBucket = BUCKET_APROVADO -- opcional
  }
}
```

Os placeholders acima não são valores executáveis. Nenhuma coordenada foi inventada. `releasePoints` permanece reservado e sem teleporte automático; a posição de soltura precisa de decisão/configuração real antes de qualquer implementação de deslocamento. Trabalhos prisionais também permanecem sem fluxo fictício até existirem definições aprovadas.

## Inicialização e distribuição

O `mz_starter` foi preparado para a ordem:

`mz_core → mz_economy → mz_menu/mz_interact → mz_vehicles → mz_org → mz_inventory → mz_fuel → mz_garagem → mz_police → mz_prison`

`mz_prison` não possui repositório público configurado. Os instaladores Windows/Linux o tratam como resource de distribuição local e falham antes de alterar dependências quando `MZ_RESOURCE_SOURCE_ROOT/mz_prison/fxmanifest.lua` não existe. O preflight de estrutura também exige seu manifest.

## Segurança

- payloads fechados e tokens limitados;
- autorização policial por sessão ativa, organização, capability e duty;
- sentence duration derivada apenas de regra server-side versionada;
- alvo sob custódia do mesmo policial, próximo e no mesmo routing bucket;
- intake confirmado novamente pelo owner prisional em ponto configurado;
- uma sentença ativa por cidadão;
- redução e soltura limitadas à organização da sentença;
- rate limiting no adapter policial;
- idempotência, proteção contra replay divergente e locks;
- itens movidos atomicamente com UID e metadata preservados;
- falha fechada quando Core, Inventory, Prison, MySQL, configuração ou alvo online não estão disponíveis.

## Testes e evidências

- suíte completa `mz_police` P1–P7: PASS;
- P7 adapter prisional: 9 casos PASS;
- P7 contrato estático policial: 8 verificações PASS;
- `mz_prison` service harness: 13 casos PASS;
- `mz_prison` static harness: 11 verificações PASS;
- `mz_inventory` propriedade prisional: 8 verificações PASS;
- regressões policiais do `mz_inventory`: 22 casos + contrato estático PASS;
- capability policial do `mz_org`: PASS;
- manifesto/ordem/distribuição do `mz_starter`: 4 testes PASS;
- sintaxe Lua dos arquivos/resources tocados: PASS;
- `git diff --check`: PASS, apenas avisos de conversão LF/CRLF.

Os harnesses cobrem configuração desativada, intake fora do ponto, payload com duração injetada, sentença duplicada, replay, redução desativada, relógio offline, retomada online, allowlists e retenção/devolução de propriedade. Mocks não equivalem ao runtime FiveM/MySQL.

## Pendências de runtime e produto

- aprovar jurisdições, regras de pena, reduções e motivos de soltura;
- medir/configurar intake e decidir o comportamento real de release;
- atribuir capabilities somente aos cargos aprovados;
- validar constraints, transações, restart e reconciliação no MySQL real;
- validar OneSync natives, routing buckets, reconnect e relógio online no FXServer;
- validar retenção/devolução com inventários reais cheios e equipamentos ativos;
- definir trabalhos prisionais antes de implementar atividades/reduções associadas;
- executar o gate operacional completo dentro do FiveM.

Nenhum runtime PASS é declarado. P8 e P9 não foram iniciadas.

## Checkpoints

O checkpoint isolado de `mz_police` deve conter apenas o adapter P7 e seus testes. `mz_prison` recebe seu próprio commit inicial, e este relatório recebe commit separado em `mz_docs`.

As extensões indispensáveis em `mz_inventory`, `mz_org`, `mz_economy` e `mz_starter` permanecem no working tree porque esses repositórios já continham alterações preexistentes misturadas. Não devem ser incluídas em commits amplos sem separar/revisar a autoria existente.
