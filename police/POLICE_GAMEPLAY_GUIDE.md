# MZ Police — Guia de gameplay operacional P7.3

Data: 16/10/2026

Estado: implementação estática concluída; runtime FiveM **NOT RUN**

## Pré-requisitos

O policial precisa possuir personagem e sessão ativos, membership em uma organização autorizada,
estar em duty e ter as capabilities correspondentes. A decisão é sempre refeita no servidor; a
visibilidade do radial no cliente é apenas conveniência.

Ordem de inicialização relevante:

1. `oxmysql`, `ox_lib`;
2. `mz_core`, `mz_economy`;
3. `mz_menu`, `mz_interact`, `mz_vehicles`;
4. `mz_org`, `mz_inventory`;
5. `mz_target`, `mz_radialmenu`, `mz_fuel`, `mz_garagem`;
6. `mz_prison`;
7. `mz_police`.

O arquivo de referência distribuído é `mz_starter/cfg/resources.cfg`. O `mz_prison` continua uma
integração fail-closed: deve iniciar antes de `mz_police` para o fluxo completo, mas não é uma
dependência rígida do manifesto policial.

## Fluxo do policial

1. Entre em duty pelo fluxo real da organização.
2. Para agir sobre um cidadão, use o target (`ALT ESQ`) diretamente sobre o personagem. No radial
   (`F3`), **Selecionar cidadão** apenas inicia esse modo de mira explícita.
3. Abra **Ações policiais** e escolha **Identificar cidadão** ou **Algemar**.
4. Com o cidadão algemado, ficam disponíveis **Revistar**, **Escoltar** e **Colocar na viatura**.
5. Na revista, a leitura exige `police.search`. A apreensão só aparece para quem também possui
   `police.seize`; quantidade, grant e transferência são revalidados pelo servidor e por
   `mz_inventory`.
6. Use **Parar escolta**, **Colocar na viatura** ou **Retirar da viatura** conforme o estado real.
7. Use **Abrir ficha no MDT** no target do cidadão. O radial independente abre o MDT geral, sem
   inferir um cidadão próximo. Isso mantém o cliente não autoritativo.
   cliente autoritativo.
8. No MDT, abra ou crie a ocorrência adequada, selecione a ficha do cidadão, abra **Acusações** e
   carregue o catálogo jurídico aprovado.
9. Adicione uma acusação canônica ao caso. O browser envia somente `ruleCode`; nome, multa e duração
   vêm da configuração do servidor.
10. Na mesma tela, escolha uma acusação com pena e use **Preparar intake**.
11. Leve o custodiado ao ponto real de intake e use a interação espacial do `mz_interact`. A sentença
    só começa após nova validação server-side de sessão, organização, duty, capability, custódia,
    proximidade, routing bucket, caso, acusação e catálogo prisional.
12. Após o intake, a custódia policial é encerrada. O `mz_prison` controla tempo, atividades
    configuradas, propriedade retida e soltura.
13. Ao final legítimo da pena, a devolução de propriedade e a soltura são idempotentes. Teleporte só
    ocorre se houver exatamente um destino válido e aprovado para a organização.

### Algemar, item e Staff

- **Policial em serviço** algema qualquer alvo válido pelo target. A custódia não exige
  `police.custody` e não exige que o alvo tenha ou não sentença prisional ativa.
- **Qualquer jogador com o item `handcuffs`** pode usar o item no inventário. O servidor escolhe
  o jogador mais próximo no mesmo bucket, dentro de 2,5 m, e delega ao fluxo físico de custódia.
- O item físico exige UID e serial gerados pelo `mz_inventory`; a algema é consumida e uma chave
  vinculada é emitida.
- **Staff** usa `/staffalgemar <id> [motivo]` e `/staffdesalgemar <id>`, com
  `staff.restraints.manage`, hierarquia Staff e distância de 2,5 m.
- O target continua exigindo organização policial autorizada e duty. Busca, identificação, MDT,
  prisão e perícia mantêm suas capabilities próprias.

O fluxo completo está em [`P15_HANDCUFFS_AND_CUSTODY.md`](./P15_HANDCUFFS_AND_CUSTODY.md).

### Prisão rápida (patrulha)

Com o cidadão algemado e sob sua custódia, o target oferece **Prisão rápida**. O formulário permite
selecionar até 3 acusações do catálogo `MZ-RP-STAGING-2`, calcular a pena e a multa no servidor e
confirmar. O `mz_police` cria um caso `occurrence`, as acusações e as citações pendentes; o
`mz_prison` continua responsável pela sentença e pelo intake.

As multas não são debitadas na prisão. O cidadão paga as citações pelo fluxo de pagamento. O intake
prisional só começa em um ponto real configurado e validado no servidor. O fluxo completo está em
[`P13_PENAL_CODE_AND_PATROL.md`](./P13_PENAL_CODE_AND_PATROL.md).

O comando `/mz_police_mdt` permanece disponível para abrir o MDT sem contexto de alvo. Nesse modo, a
ficha pode ser pesquisada, mas o encaminhamento ao intake exige que o MDT tenha sido aberto pelo alvo
no radial, pois o `targetSource` online precisa ser conhecido e será revalidado pelo servidor.

## Capabilities usadas

| Operação | Capability server-side |
|---|---|
| Algemar, desalgemar, escoltar e viatura | Organização autorizada + duty (target) |
| Usar item `handcuffs` | Item físico + alvo próximo (qualquer jogador) |
| Staff algemar/desalgemar | `staff.restraints.manage` + hierarquia Staff |
| Revistar | `police.search` |
| Apreender item | `police.seize` |
| Identificar, consultar ficha e sentença | `mdt.open` |
| Alterar ficha, caso e acusação | `police.records.write` |
| Iniciar sentença | `police.arrest.request` |
| Redução ou soltura manual, quando habilitada | `reports.approve` |

Além da capability, a policy exige organização autorizada e duty. Ausência de resource, sessão,
posição ou configuração resulta em negação fechada com feedback ao operador.

## Configuração obrigatória

O catálogo fictício **`MZ-RP-STAGING-2`** está habilitado para staging com 40 acusações, 11 multas
administrativas/de trânsito e 40 regras prisionais espelhadas. O intake usa a facility de staging
`bolingbroke`; as coordenadas e o bucket precisam ser confirmados no MLO real antes de produção.

### Catálogo policial

Em `mz_police/config/penal_code.lua` (complementando `MZPoliceConfig.Records`):

- `Records.legalRulesEnabled = true` e `Records.legalVersion = 'MZ-RP-STAGING-2'`;
- `Records.jurisdictions[orgCode].enabled` deve permanecer habilitado apenas para a jurisdição
  aprovada;
- `Records.charges[ruleCode]` usa `enabled`, `name`, `description`, `category`, `fineAmount` e
  `sentenceSeconds`;
- `Records.citations[ruleCode]` usa `enabled`, `name`, `description` e `fineAmount`;
- `Arrest.quickEnabled = true`, `Arrest.multiChargeEnabled = true`, `maximumCharges = 3` e
  `maximumSentenceSeconds = 7200`;
- o catálogo deve permanecer espelhado em `mz_prison/config.lua` (`sentenceRules`). O harness
  `legal_catalog_parity_harness.lua` valida a paridade.

### Organização, duty, delegacia, armaria e viaturas

- `MZPoliceConfig.AuthorizedOrganizations` define quais organizações podem usar a policy policial;
  membership, cargo, duty e capabilities continuam pertencendo ao `mz_org`.
- O ponto de duty fica em `organization.config.dutyPoint`, no owner `mz_org`. O contrato real aceita
  `enabled`, `coords = { x, y, z }`, `radius`, `drawDistance` e `label`. O radius permitido é de 1 a 5,
  o draw distance é de 5 a 50 e não pode ser menor que o radius. Informe somente coordenadas reais.
- `MZPoliceConfig.Stations[stationId]` recebe `enabled`, `label`, `orgCode` e
  `armory = { enabled, coords = { x, y, z }, interactDistance }`. A estação só é exposta se a
  organização estiver autorizada e as coordenadas forem válidas.
- `MZPoliceConfig.Armory.catalog[catalogKey]` recebe `item`, `label`, `capability` e `minimumGrade`.
  Os itens precisam existir no catálogo canônico do inventário; não crie IDs apenas no resource
  policial.
- A custódia não possui catálogo próprio de viaturas: ela seleciona no servidor um veículo de rede
  válido próximo e um assento livre. Propriedade, garagem e catálogo de veículos permanecem nos
  owners `mz_vehicles`/`mz_garagem`, com capabilities `vehicle.*` concedidas pelo `mz_org`.

### Intake e catálogo prisional

O `mz_prison` usa a facility de staging **`bolingbroke`** com booking, cela, áreas permitidas,
rotina e release point. O fluxo detalhado está em
[`P14_PRISON_CELLS_AND_EVENTS.md`](./P14_PRISON_CELLS_AND_EVENTS.md).

Em `mz_prison/config.lua`:

- `confinement.enabled = true` e `facilities.bolingbroke` contém `intakePoints.booking`,
  `cells.cell_a`, `allowedAreas` e `routingBucket = 0` (staging);
- cada ponto precisa de `enabled`, `coords.x/y/z`, `radius`, `routingBucket` e, opcionalmente,
  `label`, `drawDistance`, `marker` e `blip`;
- `sentenceRulesEnabled = true` e `sentenceRules` contém as 40 regras espelhadas do catálogo
  `MZ-RP-STAGING-2`;
- `multiCharge.enabled = true`, `maximumCharges = 3`, `maximumSentenceSeconds = 7200` e
  `aggregation = 'sum'`;
- `legalVersion`, jurisdição, `ruleCode`, nome e duração precisam corresponder exatamente ao snapshot
  persistido pelo catálogo policial;
- `MZPoliceConfig.Prison.processingEnabled = true` com o ponto `bolingbroke_booking`;
- confirme coordenadas, piso, colisão e bucket no MLO real antes de produção. Para produção, prefira
  um bucket prisional dedicado em vez de `0`.

### Atividades e soltura

- `jobsEnabled = true` e `jobs.rotina` está habilitada no pátio (`bolingbroke_yard`): 45s de
  duração, 30s de redução, 4 conclusões e teto de 120s por sentença.
- `releasePoints.bolingbroke_exit` está habilitado em staging. Cada destino requer `enabled`,
  `orgCode`, coordenadas, raio, routing bucket e heading opcional.
- sem destino válido, a soltura e a devolução podem concluir sem teleporte; com múltiplos destinos
  válidos, nenhum é escolhido arbitrariamente.
- `manualReleaseEnabled = true` com o motivo `staging_release` para testes autorizados.

## Checklist de staging no FXServer

1. Fazer backup recuperável dos resources e do banco.
2. Conferir o hash da RC e todos os arquivos de `SHA256SUMS`.
3. Revisar de forma não destrutiva os schemas de `mz_police` e `mz_prison`, incluindo
   `charge_public_id`; não executar `DROP`, limpeza ou migration automática.
4. Publicar todos os resources da mesma RC e confirmar a ordem de boot acima.
5. Validar as flags de staging, o catálogo `MZ-RP-STAGING-2` e as coordenadas prisionais candidatas
   no MLO/bucket real antes de produção.
6. Reiniciar o FXServer e confirmar ausência de erro de `oxmysql`, dependência ou schema.
7. Testar com dois personagens reais: capability negada, fora de duty, alvo distante, bucket distinto,
   algoritmo completo de custódia, revista somente leitura e apreensão autorizada.
8. Testar MDT, acusação, intake, restart/reconnect, atividade, término de pena, devolução e replay.
9. Confirmar foco/ESC no CEF e feedbacks em `mz_menu`.
10. Registrar console, IDs públicos, auditoria e resultado de cada etapa.

Nenhum item deste checklist foi executado no FXServer nesta entrega. **Runtime FiveM: NOT RUN.**
