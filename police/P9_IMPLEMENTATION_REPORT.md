# MZ Police — relatório de implementação P9

Data: 2026-10-16  
Escopo: P9A–P9D  
Runtime FiveM/OneSync/CEF/MySQL: **NOT RUN**

## Resultado

A implementação estática P9 foi adicionada sem alterar `mz_core`, sem criar resource paralelo e sem modificar as RCs anteriores. Este relatório registra a baseline fail-closed anterior à ativação seletiva; consulte o [guia operacional atual](./P9_POLICE_OPERATIONAL_GUIDE.md) para o estado de FLIR e tracker.

### P9A — câmeras

- Bodycam: estado temporário server-authoritative, associação a sessão/personagem/organização, duty/capability, indicador local, auditoria e cleanup.
- Dashcam: viatura ocupada derivada pelo `mz_vehicles`, modelo allowlisted, placa server-side, heartbeat, consulta de estado por painel autorizado e cleanup.
- CCTV: catálogo vazio e configurável, listagem protegida, definição escolhida revalidada no servidor, câmera nativa local e limpeza de render/foco.
- Painel: radial pessoal do `mz_target`, `mz_menu` e comando `/mz_police_advanced`.
- Não existe gravação ou transmissão persistente de vídeo.

### P9B — apoio aéreo e tracker

- FLIR: câmera local anexada à aeronave autorizada, visão térmica, zoom e rotação; grant curto revalidado por heartbeat.
- Tracker: transmissão opt-in apenas de viaturas policiais allowlisted, snapshot server-side, mesmo routing bucket, blips temporários e expiração.
- `mz_dispatch` continua owner de chamados, atribuições e blips de ocorrência. Como P8 não expõe contrato de telemetria de unidades, P9 não alterou seu banco nem criou atribuição paralela.
- Vigilância de civis ou veículos não policiais não foi implementada.

### P9C — investigação e perícia

- O `mz_inventory` continua owner do item físico e passou a expor ao contrato policial somente `kind` e `serial` sanitizados de armas em custódia.
- O `mz_police` persiste resultado `weapon_identity` ligado a caso e evidência, com idempotência, histórico e proveniência `mz_inventory.metadata.serial`.
- O MDT mostra resultados e só oferece a ação quando o item atual é uma arma serializada e o método está habilitado.
- GSR, digitais, cápsulas, amostras e balística foram deliberadamente bloqueados: não há produtor server-authoritative aprovado no código atual. Nenhum resultado fictício foi criado.

## Fronteiras e contratos

- Identidade/sessão: exports existentes do `mz_core`, sem mudança.
- Organização/duty/capabilities: `mz_org` (`CanOrg`/contexto existente).
- Veículo: novo export server-only `PoliceResolveOccupiedVehicle`; aceita apenas `actorSource` fornecido pelo owner policial e deriva entidade, placa, modelo, seat e bucket.
- Item físico: `mz_inventory` preserva a custódia e expõe cópia sanitizada.
- Caso, histórico e resultado investigativo: `mz_police` é o único writer.
- Interação: `mz_target`/`mz_menu`; MDT preserva o callback existente e acrescenta uma ação allowlisted.

## Segurança

- Payloads fechados e action allowlist.
- Sessão/personagem, organização, duty e capability em todas as ações privilegiadas.
- Modelos e câmeras em allowlists vazias por padrão.
- Nenhum entity ID, net ID, placa, posição ou resultado pericial enviado pelo cliente é autoritativo.
- Replay guard nos toggles; resultado pericial idempotente pelo histórico durável.
- Rate limiting, TTL curto, heartbeat fail-closed e cleanup em disconnect/resource stop.
- Tracker isolado por routing bucket e somente para unidades policiais opt-in.
- Loops por frame existem apenas enquanto uma câmera renderizada ou indicador está ativo; snapshots usam intervalo mínimo de 2 segundos.

## Limitações reais do FiveM

Scripted cameras e visão térmica são apresentação client-side; a documentação oficial separa natives de cliente e lógica server-side. OneSync aplica culling e jogadores/entidades fora da focus zone podem não existir no cliente, além de routing buckets isolarem entidades. Portanto uma câmera nativa não constitui um stream remoto confiável. Referências: [native functions](https://docs.fivem.net/docs/scripting-manual/introduction/about-native-functions/), [OneSync e culling/buckets](https://docs.fivem.net/docs/scripting-reference/onesync/) e [state bags](https://docs.fivem.net/docs/scripting-manual/networking/state-bags/).

Consequências:

- consulta de bodycam/dashcam remota informa estado, não vídeo;
- nenhuma gravação persistente existe sem infraestrutura externa de captura, storage e retenção;
- CCTV depende de coordenadas/rotação reais e validação visual em runtime;
- FLIR não concede identidade ou metadados de entidades apontadas;
- OneSync, câmera, térmica, foco, CEF e desempenho permanecem **NOT RUN**.

## Configuração pendente

- Atribuir seis capabilities P9 nos cargos aprovados.
- Informar modelos reais de dashcam, tracker e helicóptero.
- Informar offset relativo real da câmera aérea.
- Informar câmeras CCTV reais.
- Decidir se `weapon_identity` será habilitado.
- Aplicar/verificar `mz_police/sql/records_schema_additive.sql` em staging.
- Habilitar seções e a flag global somente após o gate operacional.

## Schema aditivo

`mz_police_forensic_results` registra uma linha imutável por operação, ligada ao caso, evidence ID e instance UID. O script é idempotente (`CREATE TABLE IF NOT EXISTS`) e não foi executado nesta sessão. Rollback conservador: desabilitar P9 e restaurar os resources; não remover tabela/dados automaticamente.

## Arquivos P9

- `mz_police/server/advanced.lua`, `client/advanced.lua`, configuração, eventos, auditoria, bridge, repository/prepare, manifest e MDT.
- `mz_police/sql/records_schema_additive.sql` e harnesses P9.
- `mz_vehicles/server/domain/police.lua` e `exports.lua`.
- `mz_inventory/server/service.lua`.
- `mz_org/shared/capabilities.lua`, allowlist e harness.
- builder/teste de RC em `mz_starter`.
- este relatório e o guia operacional.

## Validação e evidência

- Harness P9: PASS, incluindo autorização, replay, payload inválido, viatura inexistente, resultado pericial derivado do owner, troca de sessão, perda de duty e cleanup de desconexão.
- Contratos Node do MDT/P7.3/P9: 9/9 PASS.
- Harness de capabilities do `mz_org`: PASS.
- Regressões policiais Lua P1–P9: PASS.
- Regressões de evidência/arsenal do `mz_inventory`: PASS.
- Regressões de domínio/polícia do `mz_vehicles`: PASS.
- Runner integrado `mz_testkit` (`covered`): 51/51 PASS.
- Teste do builder de release: 1/1 PASS.
- Sintaxe Lua e JavaScript dos arquivos alterados: PASS.
- `git diff --check` nos owners afetados: PASS; apenas avisos de normalização CRLF do Git.

O MDT foi exercitado no browser em 1366×768, 1920×1080, 2560×1440 e 3440×1440. A tela de evidências e o estado vazio de resultados periciais ficaram acessíveis, sem overflow horizontal, clipping ou erro de console reproduzível. Essa evidência valida HTML/CSS/JavaScript responsivo; não valida CEF, foco NUI ou rendering dentro do FiveM.

O Graphify foi atualizado de modo localizado: `mz_police` (497 nós/1044 arestas), `mz_inventory` (1391/2494), `mz_vehicles` (573/1414) e `mz_org` (2006/4355). As consultas posteriores localizaram `Advanced.request`, `PoliceResolveOccupiedVehicle` e `listPoliceEvidence` em seus owners reais. Os nomes de algumas comunidades ficaram provisórios porque não há backend LLM configurado para relabeling; isso não afeta o grafo de código.

Nenhum resultado desta seção equivale a runtime FiveM. OneSync, natives de câmera, FLIR, routing buckets, CEF/NUI, MySQL e integração entre resources continuam **NOT RUN**.

## Commits e preservação

Nenhum commit foi criado. Os owners já continham alterações preexistentes sobrepostas; um commit automático não seria um checkpoint isolado verificável. A RC empacota o working tree necessário e registra hashes, sem alterar RCs anteriores.
