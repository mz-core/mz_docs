# MZ Police — Relatório P7.3: integração operacional completa

Data: 16/10/2026

Estado: gates estáticos aprovados; runtime FiveM **NOT RUN**

## Resultado

A P7.3 conecta os contratos já existentes sem criar owners paralelos:

- `mz_target` apresenta o submenu/radial contextual **Ações policiais**;
- `mz_police` filtra ações no servidor e mantém autoridade sobre custódia, revista e registros;
- `mz_menu` apresenta revista, quantidade de apreensão e feedback;
- o MDT P6 expõe catálogo jurídico server-side, acusação canônica, situação prisional e preparação de
  intake;
- `mz_interact` registra somente pontos de intake e atividades explicitamente habilitados;
- `mz_prison` mantém a autoridade sobre sentença, atividades, propriedade e soltura.

O fluxo coberto é: abordagem → algema → revista → escolta → viatura → MDT → acusação → sentença →
intake → atividades configuradas → soltura.

## Defeitos comprovados corrigidos durante o gate

1. O boot explícito iniciava `mz_police` antes de `mz_prison` e `mz_target`. A ordem foi corrigida em
   `mz_starter/cfg/resources.cfg`.
2. O builder não incluía `mz_target` e registrava `mz_police` antes de `mz_prison`. O resource e a ordem
   correta agora fazem parte do manifesto e do teste do builder.
3. Uma dependência rígida de `mz_police` para `mz_prison` contrariava o contrato P7 fail-closed. Ela foi
   removida; o boot completo continua ordenando `mz_prison` antes de `mz_police`.
4. No radial rápido, uma entidade não-player podia impedir o fallback para o jogador próximo. O
   adaptador agora prioriza um ped de player e, caso contrário, usa `nearby_player_server_id`.
5. O seletor novo de acusação recebeu o mesmo tratamento de autocomplete dos demais controles do MDT.

## Autoridade e integridade

- Todas as mutações reutilizam eventos existentes e revalidam no servidor identidade/sessão,
  organização, duty e capability.
- O contexto de ação também valida proximidade e routing bucket. A custódia revalida revision e estado;
  a revista usa grant temporário do inventário; viatura e intake revalidam estado real.
- O cliente nunca envia multa, nome da regra, duração de sentença ou redução. A acusação persiste o
  snapshot do catálogo canônico; o adapter prisional compara esse snapshot ao catálogo prisional.
- O `mz_inventory` continua único owner de itens físicos. O `mz_prison` continua único owner da
  sentença e da devolução. O `mz_police` mantém somente o caso, acusação e contexto operacional.
- Replay, locks, request IDs, auditoria e retomada após restart/reconnect permanecem nos owners
  canônicos.
- Configurações jurídicas, intake, jobs, soltura e liberação manual continuam desativadas por padrão.

## Arquivos P7.3

`mz_police`:

- `fxmanifest.lua`;
- `client/custody.lua`, `client/mdt.lua`, `client/interactions.lua`;
- `server/events.lua`, `server/main.lua`, `server/records/service.lua`, `server/interactions.lua`;
- `web/app.js`;
- `tests/p6_records_harness.lua`, `tests/p7_3_interactions_harness.lua`,
  `tests/p7_3_operational_contract.test.js`.

`mz_prison`:

- `fxmanifest.lua`, `client/interactions.lua`, `server/service.lua`,
  `tests/p7_service_harness.lua`.

`mz_starter`:

- `cfg/resources.cfg`;
- `tools/build_police_release.ps1`;
- `tests/police_release_builder.test.js`.

`mz_docs`:

- `police/POLICE_GAMEPLAY_GUIDE.md`;
- `police/P7_3_OPERATIONAL_GAMEPLAY_REPORT.md`.

Alterações preexistentes nos mesmos working trees foram preservadas. `mz_core` não foi modificado.

## Validação executada

| Gate | Resultado |
|---|---|
| Todos os harnesses Lua de `mz_police` | **16/16 arquivos PASS**; P7.3 = 6 casos |
| Contratos Node do MDT/P7.3 | **5/5 PASS** |
| Todos os harnesses Lua de `mz_prison` | **5/5 arquivos PASS**; 54 casos declarados |
| Harnesses Lua de `mz_target` | **4/4 PASS** |
| Contrato NUI de `mz_target` | **1/1 PASS** |
| Runner global `mz_testkit -Suite covered` | **51/51 PASS** |
| Teste do builder da release | **1/1 PASS** |
| `luac -p` nos Lua alterados e `node --check web/app.js` | **PASS** |
| `git diff --check` nos repositórios relevantes | **PASS**; apenas avisos LF/CRLF |
| QA local do MDT em 1366×768 e 1920×1080 | **PASS** para layout/estados vazios/ESC; zero erros/warnings no console |

A QA do browser não equivale a CEF dentro do FiveM.

## Configuração e distribuição

- A ordem autoritativa de boot está no guia e no manifesto da RC.
- Nenhuma coordenada, pena, multa, redução ou regra jurídica foi inventada.
- O builder inclui o estado atual dos working trees necessários, registra commit/branch/dirty state,
  bloqueia nomes privados e procura assinaturas comuns de credenciais.
- A release candidate anterior não foi alterada.
- Nenhum deploy, migration, reset, clean ou push foi executado.

### Release candidate P7.3 final

- Diretório: `release-artifacts/mz-police-p7-3-operational-rc2-20261016`;
- ZIP: `release-artifacts/mz-police-p7-3-operational-rc2-20261016.zip`;
- SHA-256 do ZIP: `862b76639a3c6d223f4edd6a19d4c07ed62465f7ba6a4432dc9e130e88f06cf8`;
- SHA-256 do manifesto: `fc17ced0e9593fc58b99f47ede068f9d50bbc5dae521ea044f32db1af37a8891`;
- SHA-256 de `SHA256SUMS`: `4489b588583abc45e304e3301ad016c5c2ed99124c0548d12fc7d1c12c0d6fb5`;
- `SHA256SUMS`: **2.342/2.342 PASS**;
- ZIP: 2.343 arquivos, incluindo manifesto, checksums, `mz_target`, adapters operacionais de
  `mz_police`/`mz_prison` e o guia de gameplay.

As RCs anteriores foram reconferidas e preservaram seus hashes:

- `mz-police-p1-p7-mdt-rc-20261015.zip`:
  `1779af652d2eefde691cf80a9b8a66c1d4d8a723ae24059c2608dbef27b56761`;
- `mz-police-p5-1-p7-1-p7-2-mdt-rc-20261015.zip`:
  `a29c7e077dcc74fe1c8f3847faccf57870c86b025aa54215abbf7884fefb0a0c`.

A primeira montagem P7.3, `mz-police-p7-3-operational-rc-20261016`, também foi preservada sem
sobrescrita, mas foi substituída pela `rc2` acima após o guia receber o contrato exato de duty point,
delegacia, armaria e viaturas. Ela não é a distribuição recomendada.

## Pendências exclusivas de runtime

1. Inicialização real de FXServer, oxmysql e MySQL/MariaDB.
2. Sessões/personagens reais, duty e capabilities do `mz_org`.
3. Target/radial, animações, foco NUI, proximidade OneSync e routing buckets reais.
4. Fluxo integral com dois jogadores, veículo de rede, intake configurado e restart/reconnect.
5. Catálogo jurídico e prisional aprovado, coordenadas reais, jobs e destino de soltura.
6. Persistência, concorrência, auditoria e devolução de propriedade em staging.

**Runtime FiveM: NOT RUN.** P8/P9 não foram iniciadas.
