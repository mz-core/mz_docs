# MZ Prison P14 — cela, intake, rotina e eventos

Status: preset de **staging** aplicado e validado por harnesses locais. Runtime
FiveM/OneSync/MLO/CEF/MySQL: **NOT RUN**.

## 1. O que foi criado

O `mz_prison` agora tem uma facility de staging completa em Bolingbroke:

- ponto de **intake/booking**;
- uma **cela** (`cell_a`);
- duas **áreas permitidas** (`cell_block` e `yard`);
- ponto de **rotina** no pátio, com redução de pena;
- **release point** habilitado na saída;
- **HUD** com status, cela e tempo restante;
- evento server → client de status da sentença.

As coordenadas são **candidatas de staging**, retiradas da referência vRP e dos relatórios
anteriores. Elas precisam ser confirmadas no MLO/bucket real antes de produção.

## 2. Configuração aplicada

Em `mz_prison/config.lua`:

```lua
confinement = {
  enabled = true,
  correctionCooldownSeconds = 10,
  resumeGraceSeconds = 15
},
facilities = {
  bolingbroke = {
    enabled = true,
    orgCode = 'police',
    routingBucket = 0,
    intakePoints = {
      booking = {
        enabled = true,
        coords = { x = 1847.14, y = 2585.90, z = 45.68 },
        radius = 3.0,
        routingBucket = 0
      }
    },
    cells = {
      cell_a = {
        enabled = true,
        coords = { x = 1680.10, y = 2513.00, z = 45.50 },
        heading = 0.0
      }
    },
    allowedAreas = {
      cell_block = { enabled = true, coords = { x = 1680.10, y = 2513.00, z = 45.50 }, radius = 30.0 },
      yard = { enabled = true, coords = { x = 1691.65, y = 2564.11, z = 45.56 }, radius = 30.0 }
    }
  }
}
```

Em `mz_police/config/server.lua`, o processamento do MDT usa o mesmo ponto:

```lua
Prison = {
  processingEnabled = true,
  processingPoints = {
    bolingbroke_booking = {
      enabled = true,
      coords = { x = 1847.14, y = 2585.90, z = 45.68 },
      radius = 3.0,
      routingBucket = 0
    }
  }
}
```

### Aviso de bucket

`routingBucket = 0` é o preset de staging mais simples: a cela fica no mundo principal e o
servidor controla a área permitida. Para produção, use um **bucket prisional dedicado** e
documente a faixa para não colidir com casas, interiores ou outros recursos. Troque o bucket em
`facility.routingBucket`, no intake point, nas áreas e no release point.

## 3. Como funciona o fluxo

### 3.1 Prisão rápida / caso

1. Policial em duty algema o cidadão.
2. No target, seleciona **Prisão rápida**.
3. Escolhe até 3 acusações e confirma.
4. O `mz_police` cria o caso, as acusações e as citações pendentes.
5. O cliente guarda o `PendingIntake` com `targetSource`, `caseId` e `chargeId`.

### 3.2 Intake

1. Leve o custodiado ao ponto `1847.14, 2585.90, 45.68`.
2. Use a interação `[E] Recebimento prisional`.
3. O `mz_interact` dispara `mz_police:client:prison:intake`.
4. O cliente envia `mz_police:server:prison:requestSentence` com o caso/charge.
5. O `mz_police` valida custódia, proximidade, bucket, caso e catálogo.
6. O `mz_police` chama o export `RequestSentence` ou `RequestSentenceFromCharges` do `mz_prison`.
7. O `mz_prison` valida a regra, retém a propriedade, ativa a sentença e despacha o confinamento.

### 3.3 Cela e confinamento

O servidor:

1. define `facilityId = bolingbroke`, `cellId = cell_a`;
2. aplica `SetPlayerRoutingBucket(source, 0)`;
3. envia `mz_prison:client:confinement` com as coordenadas da cela;
4. o cliente teleporta e responde `mz_prison:server:confinement:ack`;
5. o servidor confirma a posição/bucket e grava `confinement_status = confirmed`;
6. a cada tick, verifica se o preso continua em `cell_block` ou `yard`;
7. se sair, após 10s de cooldown, teleporta de volta e registra `confinement_corrected`.

### 3.4 Rotina

No pátio, o ponto `[E] Rotina prisional` inicia a atividade `rotina`:

- duração: 45s;
- redução: 30s;
- máximo: 4 conclusões por sentença;
- teto de redução: 120s;
- cooldown: 20s.

O cliente só envia `jobCode`/`pointId`; o servidor calcula duração, redução e limites.

### 3.5 Soltura

Ao terminar a pena ou em soltura manual autorizada:

1. o `mz_prison` devolve a propriedade retida;
2. marca a sentença como `released`;
3. envia `mz_prison:client:release` com o release point;
4. o cliente teleporta para `1837.14, 2591.40, 45.01` com heading `175.68`;
5. o cliente confirma `mz_prison:server:release:ack`;
6. o servidor restaura o routing bucket anterior.

## 4. Eventos

| Evento | Direção | Função |
|---|---|---|
| `mz_police:client:prison:intake` | `mz_interact` → cliente | inicia o encaminhamento usando o `PendingIntake` |
| `mz_police:server:prison:requestSentence` | cliente → servidor | valida caso/charge e chama o owner prisional |
| `mz_prison:client:confinement` | servidor → cliente | teleporta para a cela e define o bucket |
| `mz_prison:server:confinement:ack` | cliente → servidor | confirma posição/bucket da cela |
| `mz_prison:client:release` | servidor → cliente | teleporta para o release point |
| `mz_prison:server:release:ack` | cliente → servidor | confirma a chegada ao release point |
| `mz_prison:client:sentence` | servidor → cliente | publica status, cela e tempo restante para o HUD |
| `mz_prison:client:activity:result` | servidor → cliente | resultado de iniciar/concluir atividade |
| `mz_prison:server:activity:start` | cliente → servidor | solicita início de atividade |
| `mz_prison:server:activity:complete` | cliente → servidor | solicita conclusão após o tempo |
| `mz_prison:server:resume` | cliente → servidor | reconstrói cela/soltura após reconnect |

Os eventos de apresentação (`confinement`, `release`, `sentence`) não concedem autoridade. O
servidor continua validando posição, bucket, sentença, regra, propriedade e release point.

## 5. HUD

`mz_prison/client/status.lua` escuta `mz_prison:client:sentence` e desenha no topo da tela:

```text
PRISÃO • bolingbroke • Cela cell_a • CUMPRINDO PENA • 04:32
```

O contador é apenas visual; o tempo canônico continua no `mz_prison`. Ao receber `released`, o HUD
mostra **LIBERTADO** por 10 segundos e desaparece.

## 6. Como testar no FiveM

1. Inicie `mz_core`, `mz_inventory`, `mz_police` e `mz_prison`.
2. Verifique no console:
   ```text
   mz_prison_status
   ```
   Deve mostrar `ready=true`, `confinement=true` e a facility carregada.
3. Entre em duty e algeme um personagem de teste.
4. Use **Prisão rápida** no target e selecione uma acusação.
5. Vá ao booking `1847.14, 2585.90, 45.68` e use **[E] Recebimento prisional**.
6. Confirme:
   - o preso foi teleportado para `1680.10, 2513.00, 45.50`;
   - o HUD mostra a cela e o tempo;
   - o bucket do preso é `0`.
7. Vá ao pátio `1691.65, 2564.11, 45.56` e use **[E] Rotina prisional**.
8. Aguarde 45s; a pena deve reduzir 30s e o limite de 4 conclusões deve ser respeitado.
9. Saia da área permitida e aguarde o cooldown de 10s; o servidor deve teleportar de volta.
10. Use soltura manual autorizada ou aguarde o fim da pena; o preso deve ir para
    `1837.14, 2591.40, 45.01`.
11. Reinicie `mz_prison` com um preso ativo e confirme que `resumeSource` reconstrói cela/HUD.

## 7. Códigos de erro comuns

| Código | Significado |
|---|---|
| `intake_point_required` | policial ou custodiado fora do booking/bucket |
| `confinement_not_configured` | facility sem bucket, cela ou área válida |
| `confinement_runtime_unavailable` | `SetPlayerRoutingBucket`/evento indisponível no runtime |
| `confinement_not_reached` | ack do cliente sem estar na cela/bucket |
| `allowed_area_left` | preso saiu da área permitida; correção aplicada |
| `release_destination_invalid` | release point ausente/inválido para a organização |
| `sentence_rule_mismatch` | catálogo policial e prisional divergentes |
| `intake_not_configured` | `mz_prison` sem facility/intake válidos |

## 8. Limitações atuais

- coordenadas e bucket são **staging**, não produção;
- uniforme prisional continua desabilitado (sem assets/garments aprovados);
- apenas uma cela e uma atividade (`rotina`) estão configuradas;
- não há lockdown, solitary, fuga, canteen ou fiança;
- o HUD é nativo, não NUI;
- runtime FiveM/MLO/OneSync continua **NOT RUN**.
