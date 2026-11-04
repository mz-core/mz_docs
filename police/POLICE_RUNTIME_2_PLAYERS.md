# Homologação runtime — polícia com 2–3 jogadores

Data da preparação: 2026-10-18. Status atual de todos os casos abaixo: **NOT RUN**, pois este ambiente não dispõe de FXServer/OneSync, MySQL de staging e clientes FiveM simultâneos.

Use A = policial principal em duty, B = civil, C = segundo policial apto. Grave vídeo/timestamps e IDs de log/caso/sentença. Só altere o status para PASS após observar o comportamento.

## Preparação

- migration P10 aplicada duas vezes em cópia de teste;
- `schema_ready=true`;
- duty/capabilities do runbook conferidos;
- processing point, facility, intake, cela, área e buckets medidos no MLO;
- catálogo de staging espelhado; multi-charge/uniforme ligados somente no cenário específico;
- console `mz_prison_status` sem erro de configuração.
- `MZPoliceConfig.Debug.targetStatus=true` somente durante a homologação; reverta para `false` ao terminar.

## Casos

| ID | Execução e evidência esperada | Status |
|---|---|---|
| R01 | A abre F3 em duty: uma raiz policial. B não vê a raiz. `/mz_police_target_status` registra `registered/authorized`. | NOT RUN |
| R02 | A mira B: identificar/algemar aparecem; nenhum `Prender` de rua. Evento forjado de processing longe retorna `processing_point_required`. | NOT RUN |
| R03 | A algema e revista B; ambos os clientes concordam e audit registra actor/target/revision. | NOT RUN |
| R04 | A escolta B, coloca em banco traseiro livre, dirige, retira e retoma escolta; motorista nunca é removido. | NOT RUN |
| R05 | C próximo assume custódia; B permanece algemado. Civil/sem duty/bucket distinto não assume. | NOT RUN |
| R06 | No ponto real, `Processar prisão` abre MDT com o citizenid de B e mantém seleção ao trocar abas. | NOT RUN |
| R07 | A seleciona uma charge e relato válido, recebe quote server-side e confirma uma vez; caso/charge/history aparecem sem fechar MDT. | NOT RUN |
| R08 | Double-click, retry e reopen não duplicam caso nem citation; antes do intake o estado visível é `Pendente de intake`. | NOT RUN |
| R09 | Citation pendente aparece. A solicita pagamento; B recusa e nada debita. Depois B confirma; recibo existe e repetição não debita novamente. Testar saldo insuficiente/falha transiente. | NOT RUN |
| R10 | Intake fora do ponto falha. No ponto, com A e B presentes, ativa uma sentença e envia B à cela/bucket; ACK fica confirmado. | NOT RUN |
| R11 | B tenta sair da área e trocar bucket: sweep corrige uma vez, respeita cooldown e não entra em loop. | NOT RUN |
| R12 | B reconecta, troca timing de playerLoaded, morre/respawna e ocorre restart de `mz_prison`; sessão velha é recusada e a cela é recomposta. | NOT RUN |
| R13 | Fim da pena e soltura manual autorizada removem uniforme, restauram UIDs/metadata/quantidade e retornam ao bucket anterior. Sem release point, não fica na dimensão privada. | NOT RUN |
| R14 | Inventário cheio, arma/roupa equipada e falha parcial: nenhuma duplicação/perda; retry converge. | NOT RUN |
| R15 | Com policy staging: 2–3 charges somam penas, respeitam teto e gravam snapshots. Charge repetida/versão divergente/sentença ativa falham. | NOT RUN |
| R16 | Dispatch mantém chamados após toast, atualiza lista, aceita/fecha sem duplicar notificação; reopen/reconnect recupera snapshot. | NOT RUN |
| R17 | Reiniciar isoladamente `mz_target`, `mz_police`, `mz_prison` e `mz_dispatch`: sem raiz duplicada, callback órfão, sentença/item duplicado ou ped preso. | NOT RUN |
| R18 | B em bucket diferente, sem personagem ou morto: ações falham fechadas com mensagem curta. | NOT RUN |

## Comandos e evidências

Antes/depois de intake:

```text
mz_prison_status <source de B>
```

Esperado após intake: `status=active`, facility/cell reais, bucket privado, `property=retained`, `confinement=confirmed`, `last_error=` vazio.

Esperado após soltura: nenhuma sentença corrente em nova consulta e bucket público/anterior no runtime. Confirme no banco apenas por SELECT; não corrija estado manualmente durante o teste.

No cliente A:

```text
/mz_police_target_status
```

Capture handle, registry, reason, access, nearest, cache, actions e revision. Não use o comando como prova de autorização final: a mutação server-side precisa também retornar sucesso.

## Registro do resultado

Para cada caso, anote build/revisão, horário, sources/sessões, operationId/caseId/sentenceId, resultado observado e link da evidência. Em falha, preserve logs dos owners envolvidos e marque FAIL; não substitua por inferência de harness ou browser.

