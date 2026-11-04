# MZ Police — relatório P8 Dispatch

Data: 16/10/2026

Estado: gates estáticos aprovados; runtime FiveM **NOT RUN**.

## Decisão arquitetural

A inspeção encontrou produtores e consumidores reais, mas nenhum owner de ocorrência compartilhada:

- `mz_phone` possuía chamadas pessoa-a-pessoa, sem fila de emergência;
- `mz_medical` aceitava pedido de ajuda e confirmava apenas a disponibilidade de coordenadas;
- `mz_police` possuía MDT, organização e autorização, mas nenhum chamado operacional;
- `mz_org` já possui as organizações `police` e `ambulance`, duty e capabilities canônicas;
- `mz_notify` é somente apresentação, sem persistência;
- `mz_target` permanece owner de interação contextual, não de comunicação.

Como polícia e ambulância consomem o mesmo ciclo de vida, foi criado `mz_dispatch` independente. Ele é
o único escritor de chamadas, unidades e histórico. Não foi criado segundo sistema de identidade,
organizações, permissões, notificações ou logs.

## Implementado

- abertura pelo app **Emergência** do `mz_phone`;
- abertura automática e não bloqueante pelo pedido de socorro do `mz_medical`;
- localização e routing bucket lidos pelo servidor;
- fila persistente por serviço;
- atribuição idempotente de múltiplas unidades;
- transições `open` → `accepted` → `closed`;
- encerramento restrito a unidade atribuída;
- integração da fila policial no MDT;
- fila compartilhada por `/mz_dispatch` para polícia e ambulância autorizadas;
- notificações pelo `mz_notify` quando disponível;
- blip temporário somente após aceite;
- histórico próprio e log detalhado no `mz_core`;
- payload fechado, rate limit, limite de texto e rejeição de coordenadas client-side;
- schema aditivo, não executado nesta entrega;
- flag `enabled = false` por padrão.

## Autoridade

- `mz_core.ResolvePlayerIdentity`: conta/personagem/sessão;
- `mz_org.GetPlayerOrgContext`, `GetOrganizationReadiness` e `CanOrg`: organização, duty e capability;
- polícia: `police` + `mdt.open`;
- atendimento médico: `ambulance` + `ambulance.tablet.open`;
- `mz_dispatch`: fila, atribuições, estados e histórico;
- cliente/NUI: apenas intenção, mensagem e IDs públicos.

## Arquivos

Novo resource `mz_dispatch`: manifesto, config, bridge, repository, service, eventos, blips/menu,
schema, testes e API. Adaptadores adicionados em `mz_phone`, `mz_police` e `mz_medical`. O guia de
staging está em `mz_docs/police/POLICE_STAGING_OPERATIONAL_CONFIGURATION.md`.

`mz_core` não foi modificado. A RC P7.3 RC2 não foi modificada nem reconstruída.

## Contratos validados

O ciclo automatizado cobre:

1. criação com identidade canônica, mensagem limitada e coordenadas/routing bucket do servidor;
2. rejeição de coordenada enviada pelo cliente e de payload com campos extras;
3. notificação e evento apenas para responder autorizado; personagem fora de duty não recebe;
4. listagem paginada limitada e isolada por serviço;
5. negação de encerramento por unidade não atribuída;
6. aceite, atribuição única, replay idempotente e mudança para `accepted`;
7. encerramento pela unidade atribuída, replay idempotente e mudança para `closed`;
8. blip criado somente após atribuição e removido no encerramento/TTL;
9. dispatch desabilitado por padrão e falhas de dependência tratadas sem autoridade no cliente.
10. envelope público fechado, `requestId` limitado/validado e antispam comum a todas as ações NUI.

O MDT usa o mesmo evento/serviço de `mz_dispatch`; não há backend paralelo. O app de emergência do
telefone envia apenas `serviceCode`, mensagem e chave de idempotência. `mz_medical` chama o export
canônico e mantém seu pedido de ajuda funcional caso o dispatch esteja ausente.

## Validação executada

- `p8_service_harness.lua`: **14/14 PASS**;
- `p8_events_harness.lua`: **5/5 PASS**;
- `p8_static_harness.lua`: **17/17 PASS**;
- `p8_nui_contract.test.js`: **8/8 PASS**;
- sintaxe Lua dos arquivos alterados: **PASS**;
- `node --check` dos adaptadores/NUI alterados: **PASS**;
- builder da distribuição: **1/1 PASS**;
- runner integrado `covered`: **51/51 PASS**;
- `git diff --check` nos repositórios tocados: **PASS**; avisos existentes de LF/CRLF não são erro;
- schema P8: somente `CREATE TABLE IF NOT EXISTS`, índices e FKs; nenhuma instrução destrutiva;
- QA responsivo em navegador: MDT e telefone em 1366×768, 1920×1080, 2560×1440 e 3440×1440,
  sem clipping da ação principal e com raiz transparente;
- estados visuais exercitados: fila vazia/erro de conexão no MDT, seleção, validação vazia e envio no
  telefone. Navegador não comprova foco/ESC, CEF, blips, OneSync ou restart no FiveM.

Correções comprovadas durante o gate:

- fixtures do `mz_phone` alinhadas ao catálogo real de apps e assets PNG intencionais;
- acessibilidade do app Emergência: `aria-pressed`, nome/autocomplete do campo e foco/hover visíveis;
- builder passou a detectar resources locais sem Git antes de consultar o VCS;
- evento público passou a limitar listagem/aceite/encerramento, validar o envelope e limpar a janela
  por sessão no disconnect;
- pacote P8 inclui owner, adaptadores, dependências internas, schema e documentação sem credenciais.

## Ordem de boot e ativação

Ordem relevante validada para staging:

`oxmysql`/`ox_lib` → `mz_notify` → `mz_core` → `mz_economy` → `mz_progress`/`mz_menu`/
`mz_interact`/`mz_world_editor` → `mz_vehicles` → `mz_org` → `mz_inventory` → `mz_target` →
`mz_fuel` → `mz_garagem` → `mz_prison` → `mz_dispatch` → `mz_police` → `mz_medical` → `mz_phone`.

O `resources.cfg` ativo não foi alterado. A ativação exige decisão humana: aplicar e verificar o
schema em staging, adicionar `ensure mz_dispatch` na posição documentada e só então mudar
`MZDispatchConfig.enabled` para `true`.

## Release candidate

Nome versionado final: `mz-police-p8-dispatch-rc2-20261016`.

O pacote captura o estado atual dos working trees necessários, inclui checksums por arquivo e não
depende apenas dos HEADs. Resources locais sem repositório Git são marcados com `vcsAvailable=false`
e continuam verificáveis pelo `SHA256SUMS`. A RC1 gerada antes do hardening final foi preservada; a
RC2 é o artefato recomendado. As RCs anteriores não foram sobrescritas.

Não houve deploy, execução SQL, migration, alteração destrutiva ou teste in-game. `mz_core` não foi
modificado por esta etapa e P9 não foi iniciado.

## Limitações de runtime

Pendências exclusivas de runtime:

1. OneSync para `GetPlayerPed`/`GetEntityCoords` server-side;
2. oxmysql e tabelas P8 reais;
3. sessão, duty e capabilities de personagens reais;
4. NUI do telefone/MDT, `mz_menu`, notify e blips dentro do CEF/FiveM;
5. concorrência e restart/reconnect com MariaDB real.

**Runtime FiveM: NOT RUN. P9: NOT STARTED.**
