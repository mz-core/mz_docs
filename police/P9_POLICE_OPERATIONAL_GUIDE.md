# MZ Police P9 — guia operacional

Status: FLIR e rastreamento de viaturas habilitados seletivamente; FiveM/OneSync/CEF **NOT RUN**.

## Pré-requisitos do operador

O master switch P9 está habilitado, mas somente FLIR e rastreamento de viaturas estão ativos. Antes do staging, o proprietário deve:

1. confirmar que `mz_police` criou/validou `mz_police_forensic_results` no boot (a tabela é automática);
   `resources/mz_police/sql/records_schema_additive.sql` é apenas fallback quando o usuário MySQL não tem `CREATE`;
2. conceder explicitamente as capabilities `police.air.support`, `police.tracker.use` e `police.tracker.view` aos cargos aprovados;
3. confirmar os modelos permitidos em `Advanced.Tracker.vehicleModels` e o modelo `polmav` em `Advanced.Air.helicopterModels`;
4. confirmar `Advanced.Air.cameraOffset` no servidor de staging; o default agora reproduz o helicam do `qb-policejob` (`{ x = 0.0, y = 0.0, z = -1.5 }`) e pode ser recalibrado para modelos customizados;
5. manter câmeras/CCTV e perícia desativadas até aprovar seus dados e produtores;
6. manter `mz_navigation` iniciado antes de `mz_police` (declarado como dependência) para fornecer configuração/projeção cartográfica;
7. reiniciar `mz_police` depois de seus owners e dependências.

A configuração inclui modelos de viaturas policiais vanilla e um offset provisório para o `polmav`. Nenhuma capability foi concedida, nem foram cadastrados modelos customizados, equipamentos, leis ou coordenadas de CCTV.

## Permissões

- `police.camera.use`: ativar bodycam e dashcam.
- `police.camera.view`: consultar estados de câmera e CCTV autorizado.
- `police.air.support`: usar a câmera aérea/FLIR.
- `police.tracker.use`: transmitir a posição da viatura policial ocupada.
- `police.tracker.view`: consultar unidades transmissoras no mesmo routing bucket.
- `police.forensics.request`: executar método pericial aprovado sobre evidência vinculada.

Todas exigem sessão e personagem válidos, organização policial ativa e duty. Cadastrar a capability no catálogo não concede permissão a nenhum cargo automaticamente.

## Acesso no jogo

Use o radial do `mz_radialmenu` (tecla configurada pelo servidor) e selecione **Polícia > Sistemas policiais**. O mesmo painel pode ser aberto por `/mz_police_advanced`.

Dentro de um helicóptero autorizado, o radial também mostra **Polícia > Helicóptero** com câmera aérea, visão, refletor e rapel. O atalho `/mz_police_heli` abre/fecha a câmera como fallback. Os controles de voo (E, RMB, scroll, espaço, H e X) seguem o helicam do `qb-policejob`.

O item radial só aparece quando P9 está habilitada. Cada ação é revalidada no servidor; o menu do cliente não concede autoridade.

### Bodycam (desativada)

A seção `Advanced.Cameras` está desativada na configuração atual; este fluxo ainda não pode ser ativado no jogo.

1. Entre em serviço.
2. Abra **Sistemas policiais**.
3. Selecione **Ativar bodycam**.
4. Confirme o indicador `BODYCAM` na tela.
5. Desative pelo mesmo painel ao concluir o serviço.

O sistema registra ativação/desativação, personagem e organização. Ele **não grava vídeo** e não anuncia uma gravação inexistente.

### Dashcam (desativada)

A seção `Advanced.Cameras` está desativada e a allowlist de dashcam está vazia.

1. Entre em uma viatura cujo modelo foi aprovado na configuração.
2. Selecione **Ativar dashcam**.
3. Confirme o indicador com a placa derivada pelo servidor.

A viatura é resolvida pelo ped do policial no servidor; o cliente não envia entity ID, net ID ou placa. Sair/trocar de viatura, perder duty, desconectar ou deixar o heartbeat expirar encerra o estado. O painel autorizado lista estados ativos, mas não simula streaming remoto.

### CCTV (desativado)

A seção `Advanced.Cameras` está desativada e não há câmeras cadastradas.

1. Selecione **Consultar CCTV**.
2. Escolha uma câmera previamente configurada e autorizada.
3. Use `Backspace`/`ESC` para sair da visualização.

O servidor envia somente a definição allowlisted escolhida. A lista vazia exibe um estado indisponível. Não há câmeras ou posições padrão.

### Helicóptero e FLIR

1. Entre no `polmav`, o primeiro modelo allowlisted, em serviço e no ar (mais de 1,5 m do solo).
2. Pressione `E` ou use **Polícia > Helicóptero > Câmera aérea (FLIR)**.
3. Rotacione com o mouse/analógico direito e use o scroll para zoom.
4. `RMB` alterna visão normal, noturna e térmica.
5. Aponte para um veículo por cerca de 1 segundo para escanear e travar. `Espaço` destrava.
6. `H` alterna o refletor (piloto ou copiloto). `X` executa o rapel nos assentos laterais.
7. Pressione `E` novamente, use o radial/painel ou saia da aeronave para encerrar.

O offset default `{ x = 0.0, y = 0.0, z = -1.5 }` reproduz o helicam do `qb-policejob`; recalibre apenas se o modelo customizado exigir. A câmera, zoom, visão, scaleform, leitura do veículo mirado e rapel são executados localmente. O servidor concede um estado curto e revalida aeronave, sessão, duty e capability no heartbeat. O refletor usa o evento `mz_police:server:heli:spotlight`, validado no servidor por capability, modelo da aeronave, assento de piloto/copiloto e routing bucket, com auditoria.

O scan exibe modelo, placa, velocidade e rua do veículo apontado; ele não consulta identidade de ocupantes nem cria um segundo fluxo de dados no servidor. A operação real de câmera, térmica, escala e rapel permanece pendente de validação in-game.

### Rastreamento de unidades

O transmissor é opt-in e só pode ser ativado dentro de uma viatura configurada. Para publicar a unidade, selecione **Ativar transmissor da unidade**. No MDT, abra **Viaturas** para consultar o mapa e as unidades autorizadas; a tela solicita um snapshot a cada intervalo configurado enquanto permanece aberta.

As posições vêm de entidades resolvidas pelo servidor. O `mz_police` mantém a propriedade dos snapshots; o `mz_navigation` fornece ao MDT a cartografia e sua projeção pelo export `GetMapConfig()`. O cliente não publica coordenadas. O snapshot exige `police.tracker.view`, inclui apenas unidades transmissoras no mesmo routing bucket e expira sem heartbeat. A lista inicial de modelos cobre viaturas policiais vanilla; inclua os modelos customizados somente após aprovação. O sistema não rastreia civis nem veículos não allowlisted. Os blips de chamados continuam pertencendo ao `mz_dispatch`; P9 não cria um segundo fluxo de chamados.

### Identificação pericial de arma (desativada)

A seção `Advanced.Forensics` está desativada na configuração atual.

1. apreenda o item pelo fluxo P2/P5;
2. vincule a evidência a um caso no MDT;
3. abra **Evidências** e consulte o intake;
4. quando o método estiver habilitado e o item for uma arma serializada, selecione **Identificar arma**.

O resultado registra apenas item e serial já existentes no `mz_inventory`, com proveniência explícita. GSR, impressão digital, cápsulas, vestígios e correspondência balística permanecem indisponíveis até existir um produtor server-authoritative aprovado.

## Checklist de staging

- Confirmar ordem de boot e estado `started` dos owners, incluindo `mz_navigation`.
- Validar cada capability com um cargo permitido e um negado.
- Testar perda de duty, troca de personagem, desconexão, troca/destruição de viatura e restart.
- Testar policiais em routing buckets diferentes.
- Validar modelos e offset do FLIR no mapa real; CCTV permanece desativado.
- Testar `E`, RMB, scroll, espaço, `H` e `X` no helicam; confirmar que o refletor não liga em assento não autorizado e que o rapel só funciona nos assentos laterais e no ar.
- Confirmar limpeza de câmeras, térmica, foco, scaleform, refletor e blips no stop/restart.
- Confirmar que o registro pericial referencia caso/evidência corretos após restart.
- Medir `resmon` com FLIR e tracker ativos.

Somente evidência no FXServer real pode alterar o estado de runtime de **NOT RUN**.
