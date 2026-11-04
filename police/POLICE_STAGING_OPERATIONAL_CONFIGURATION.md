# MZ Police — configuração operacional de staging

Data: 16/10/2026

Estado: guia de preparação; nenhuma configuração ativa foi alterada; runtime FiveM **NOT RUN**.

## 1. O que pode ser derivado e o que exige decisão humana

| Área | Derivado dos contratos existentes | Decisão humana obrigatória |
|---|---|---|
| Organização | códigos `police` e `ambulance`; membership, grade, duty e capabilities pertencem ao `mz_org` | membros, cargos efetivos e ponto real de duty |
| Permissões policiais | capabilities listadas abaixo e avaliadas pelo `mz_org` | quais grades recebem cada capability além dos defaults já persistidos |
| Delegacia | schema de `MZPoliceConfig.Stations` e limites de distância | coordenadas, rótulo, catálogo de itens e grade mínima |
| Viaturas | seleção de entidade/assento e autoridade de `mz_vehicles`/`mz_garagem` | modelos liberados, garagens e capabilities `vehicle.*` |
| Jurídico | identificadores e igualdade obrigatória de `legalVersion`/`ruleCode` | versão, jurisdição, leis, nomes, multas e penas |
| Presídio | schemas, limites e fluxo idempotente de intake/soltura | coordenadas, bucket, duração, redução, limites e destino |
| Dispatch P8 | serviços `police`/`medical`, orgs/capabilities e localização server-side | habilitar a flag após aplicar/verificar o schema |

Não copie números dos harnesses: são fixtures e não catálogo aprovado.

## 2. Ordem de boot proposta

Não altere `mz_starter/cfg/resources.cfg` antes de aprovar a ativação. Na configuração de staging,
insira `ensure mz_dispatch` depois de `mz_org`/`mz_menu` e antes dos consumidores:

```cfg
ensure oxmysql
ensure ox_lib

ensure mz_notify
ensure mz_core
ensure mz_economy
ensure mz_status
ensure mz_progress
ensure mz_menu
ensure mz_interact
ensure mz_world_editor
ensure mz_vehicles
ensure mz_org
ensure mz_inventory
ensure mz_target
ensure mz_fuel
ensure mz_garagem
ensure mz_prison
ensure mz_dispatch
ensure mz_police
ensure mz_medical
ensure mz_bank
ensure mz_phone
```

O `mz_dispatch` depende de `oxmysql`, `mz_core`, `mz_org` e `mz_menu`. `mz_notify` é integração
opcional. O `mz_prison` deve permanecer antes de `mz_police`; `mz_dispatch` deve iniciar antes de
`mz_police`, `mz_medical` e `mz_phone` para que todos os adaptadores encontrem o owner.

## 3. Duty e permissões policiais

O ponto de duty é persistido no config da organização pelo `mz_org`. O payload aceito pela mutação
canônica contém somente:

```lua
dutyPoint = {
  enabled = true,
  coords = {
    x = <COORDENADA_X_REAL>,
    y = <COORDENADA_Y_REAL>,
    z = <COORDENADA_Z_REAL>
  },
  radius = <VALOR_ENTRE_1_E_5>,
  drawDistance = <VALOR_ENTRE_5_E_50_E_MAIOR_OU_IGUAL_A_RADIUS>,
  label = '<ROTULO_APROVADO_ENTRE_2_E_64_CARACTERES>'
}
```

Use o fluxo staff existente do `mz_org`; não edite membership, duty ou permissões no cliente. Antes
do teste, verifique no banco/gestão organizacional que os cargos reais possuem:

- fluxo básico: `mdt.open`, `police.custody`, `police.search`;
- apreensão: `police.seize`;
- arsenal: `armory.basic` e, quando usado, a capability de cada item;
- registros: `police.records.write`;
- multas: `police.citation.issue`;
- mandados: `police.warrant.create` e `police.warrant.approve` conforme segregação aprovada;
- prisão: `police.arrest.request`;
- revisão/liberação manual, se aprovada: `reports.approve`;
- dispatch policial: `mdt.open`.

Os defaults do código não provam que uma base legada já foi reconciliada.

## 4. Delegacia, arsenal e viaturas

Preencha `MZPoliceConfig.Stations` somente com locais medidos no servidor:

```lua
Stations = {
  <STATION_ID> = {
    enabled = true,
    label = '<NOME_REAL_DA_UNIDADE>',
    orgCode = 'police',
    armory = {
      enabled = true,
      coords = { x = <X_REAL>, y = <Y_REAL>, z = <Z_REAL> },
      interactDistance = <VALOR_APROVADO_ENTRE_1_E_5>,
      drawDistance = <DISTANCIA_VISUAL_APROVADA>,
      marker = { enabled = true, type = <TIPO_APROVADO> },
      blip = { enabled = false },
      label = '<ROTULO_APROVADO>'
    }
  }
}
```

O `Armory.catalog` referencia apenas itens já existentes no catálogo do `mz_inventory`:

```lua
catalog = {
  <CATALOG_KEY> = {
    item = '<ITEM_CANONICO_EXISTENTE>',
    label = '<ROTULO_APROVADO>',
    capability = '<CAPABILITY_ORGANIZACIONAL_EXISTENTE>',
    minimumGrade = <NIVEL_APROVADO>
  }
}
```

Não há lista de viaturas em `mz_police`: catálogo, propriedade e garagem continuam em
`mz_vehicles`/`mz_garagem`. O policial usa `vehicle.*` do `mz_org`; a custódia escolhe no servidor um
veículo de rede próximo e um assento livre.

## 5. Catálogo jurídico versionado

Mantenha `MZPoliceConfig.Records.legalRulesEnabled = false` até a aprovação integral. Depois, publique
uma versão imutável e use exatamente os mesmos IDs no policial e no presídio:

```lua
Records = {
  legalRulesEnabled = true,
  legalVersion = '<VERSAO_JURIDICA_APROVADA>',
  jurisdictions = {
    police = { enabled = true }
  },
  charges = {
    <RULE_CODE_APROVADO> = {
      enabled = true,
      name = '<NOME_LEGAL_APROVADO>',
      fineAmount = <MULTA_APROVADA>,
      sentenceSeconds = <PENA_APROVADA_EM_SEGUNDOS>
    }
  },
  citations = {
    -- somente regras aprovadas para cobrança
  },
  warrants = { enabled = false, rules = {} }
}
```

O cliente envia apenas `ruleCode`. Nome, multa, duração e versão são resolvidos e persistidos pelo
servidor. Não reutilize um `legalVersion` depois de alterar qualquer regra.

## 6. Intake, sentença, atividades e soltura

Ative `MZPoliceConfig.Prison.enabled`, `MZPrisonConfig.intakeEnabled` e
`MZPrisonConfig.sentenceRulesEnabled` somente no mesmo gate. O catálogo prisional deve corresponder
à versão, jurisdição, `ruleCode`, nome e duração do catálogo policial.

```lua
MZPrisonConfig.legalVersion = '<MESMA_VERSAO_JURIDICA_APROVADA>'
MZPrisonConfig.jurisdictions = { police = { enabled = true } }
MZPrisonConfig.sentenceRules = {
  <MESMO_RULE_CODE> = {
    enabled = true,
    name = '<MESMO_NOME_APROVADO>',
    durationSeconds = <MESMA_PENA_APROVADA>
  }
}

MZPrisonConfig.intakePoints = {
  <POINT_ID> = {
    enabled = true,
    coords = { x = <X_REAL>, y = <Y_REAL>, z = <Z_REAL> },
    radius = <RAIO_APROVADO_ATE_25>,
    routingBucket = <BUCKET_REAL>,
    label = '<ROTULO_APROVADO>'
  }
}
```

Atividades permanecem desativadas até todas as regras e posições serem aprovadas:

```lua
MZPrisonConfig.activityRateLimit = {
  maximum = <MAXIMO_APROVADO>,
  windowMs = <JANELA_APROVADA_EM_MS>
}
MZPrisonConfig.jobs = {
  <JOB_CODE> = {
    enabled = true,
    durationSeconds = <DURACAO_APROVADA>,
    reductionSeconds = <REDUCAO_APROVADA>,
    maximumCompletionsPerSentence = <LIMITE_APROVADO>,
    maximumReductionSecondsPerSentence = <TETO_APROVADO>,
    cooldownSeconds = <COOLDOWN_APROVADO>,
    points = {
      <POINT_ID> = {
        enabled = true,
        coords = { x = <X_REAL>, y = <Y_REAL>, z = <Z_REAL> },
        radius = <RAIO_APROVADO_ATE_25>,
        routingBucket = <BUCKET_REAL>
      }
    }
  }
}
```

Configure exatamente um destino válido por organização quando desejar teleporte automático:

```lua
MZPrisonConfig.releasePoints = {
  <RELEASE_ID> = {
    enabled = true,
    orgCode = 'police',
    coords = { x = <X_REAL>, y = <Y_REAL>, z = <Z_REAL> },
    heading = <HEADING_REAL>,
    radius = <RAIO_APROVADO_ATE_25>,
    routingBucket = <BUCKET_REAL>
  }
}
```

Sem destino único válido, a liberação não escolhe coordenadas arbitrariamente. `manualReleaseEnabled`
deve continuar `false` até os motivos serem aprovados.

## 7. Ativação segura do dispatch P8

1. Faça backup recuperável do banco e dos resources.
2. Revise `mz_dispatch/sql/mz_dispatch.sql`; ele contém apenas `CREATE TABLE IF NOT EXISTS`, índices e
   FKs, sem `DROP`, `DELETE` ou dados fictícios.
3. Adicione `ensure mz_dispatch` na ordem proposta.
4. Com `MZDispatchConfig.enabled = true`, reinicie `mz_dispatch`. O `server/prepare.lua` aguarda o
   MySQL, cria as tabelas ausentes a partir do SQL canônico e valida colunas/índices/FKs.
5. Confirme as três tabelas (`mz_dispatch_calls`, `mz_dispatch_assignments`, `mz_dispatch_history`)
   e os logs `database connection ready`, `schema initialization started`, `schema validated`,
   `schema_ready=true enabled=true`.
6. Se o usuário MySQL não possuir `CREATE`/`REFERENCES`, conceda as permissões ou aplique
   `mz_dispatch/sql/mz_dispatch.sql` manualmente (fallback). O serviço permanece fail-closed até validar.
7. Verifique `police` com `mdt.open` e `ambulance` com `ambulance.tablet.open`, ambos em duty.

O telefone abre chamados em **Emergência**. Policiais operam pela tela **Dispatch** do MDT.
Ambulância e polícia também podem abrir a fila compartilhada com `/mz_dispatch`; o servidor nega a
fila que não corresponda à organização/capability/duty do personagem. Aceitar atribui a unidade e
cria blip temporário; encerrar exige que o mesmo personagem esteja atribuído.

## 8. Fluxo normal de prisão, sem comando administrativo

1. policial entra em duty no ponto real do `mz_org`;
2. usa `mz_target` (`ALT ESQ`) ou radial rápido (`F3`) no cidadão;
3. identifica, algema, revista, apreende quando autorizado, escolta e coloca na viatura;
4. abre a ficha pelo radial, cria/seleciona caso e adiciona acusação do catálogo aprovado;
5. seleciona **Preparar intake**, conduz o custodiado ao ponto configurado e usa a interação espacial;
6. `mz_prison` retém propriedade, inicia a sentença e oferece apenas atividades habilitadas;
7. término/redução legítimos entram no fluxo idempotente de devolução e soltura;
8. o destino é aplicado somente quando houver exatamente um ponto válido aprovado.

O comando `/mz_police_mdt` é conveniência de consulta, não é necessário no fluxo normal e não cria
contexto de intake. Nenhum comando administrativo participa do gameplay acima.

## 9. Gate operacional mínimo

- schema e dependências sem erro após restart;
- negação fora de duty e sem capability;
- duty point, arsenal, intake, jobs e release em posição/bucket reais;
- catálogo policial/prisional idêntico;
- fluxo com dois jogadores, veículo de rede, reconnect e restart;
- telefone → fila → aceite → blip → encerramento para polícia e ambulância;
- MDT e `/mz_dispatch` sem vazamento entre organizações;
- logs centrais e histórico do dispatch reconciliados.

Nenhum item foi executado dentro do FXServer nesta entrega. **Runtime FiveM: NOT RUN.**
