---
title: MZ Framework — Checklist FXServer para validação P1 policial
status: GUIDE
authority: LEVEL_4_REFERENCE
owner: MZ Police
last_reviewed: 2026-10-14
applies_to: [mz_police, mz_core, mz_org, mz_starter]
related_contracts: [MZ-IDENTITY-v2, MZ-DOCS-v1]
---

# Checklist FXServer — P1 policial

Resultado atual: **pronto para staging; runtime ainda não aprovado**.

> **Atualização da limpeza estrutural:** o diagnóstico `server/p1_gate.lua` e o comando
> `mz_police_p1_gate` foram removidos do runtime. As seções 4–6 deste documento permanecem
> como registro histórico do gate P1 e não devem ser executadas no resource atual. Use os
> logs de boot, a auditoria de `mz_police` e os comandos operacionais vigentes.

## 1. Publicar e conferir na VPS

`mz_police` é `distribution-only`. Em uma instalação nova, `MZ_RESOURCE_SOURCE_ROOT` fornece o resource ao instalador. Se a VPS já possui `resources/[mz]/mz_police`, o instalador preserva essa pasta e o deploy normal deve atualizá-la.

Arquivos mínimos que precisam chegar juntos:

```text
mz_police/fxmanifest.lua
mz_police/config/shared.lua
mz_police/config/server.lua
mz_police/config/penal_code.lua
mz_police/server/bridge.lua
mz_police/server/audit.lua
mz_police/server/policy.lua
mz_police/server/main.lua
```

No shell da VPS, a partir do `server-data`:

```sh
for file in fxmanifest.lua config/shared.lua config/server.lua config/penal_code.lua server/bridge.lua server/audit.lua server/policy.lua server/main.lua; do
  test -f "resources/[mz]/mz_police/$file" || echo "MISSING: $file"
done
```

Nenhuma saída significa que o conjunto mínimo está presente. `tests/` foi removido da
distribuição; QA fica em cópia externa e não é carregado pelo manifesto.

## 2. Ordem de boot

`server.cfg` já executa `cfg/resources.cfg`. A sequência necessária e já configurada é:

```cfg
ensure mz_core
# dependências intermediárias do mz_org permanecem na ordem existente
ensure mz_org
ensure mz_police
```

Não mover `mz_police` antes de `mz_org` ou `mz_core`.

## 3. Informar o dutyPoint real

Não editar banco, seed ou `config_json` manualmente.

1. Entrar com uma conta owner ou staff que possua `staff.orgs.duty`.
2. Posicionar o personagem exatamente no ponto real de entrada/saída de serviço.
3. Executar `/stafforg`.
4. Abrir **Organizações → Polícia Militar (`police`) → Ponto de serviço**.
5. Escolher **Criar ponto no local atual** ou **Reposicionar ponto no local atual**.
6. Informar label, raio, distância de desenho e motivo; revisar as coordenadas exibidas e confirmar.

O menu captura `GetEntityCoords(PlayerPedId())`. O contrato persistido pelo owner é:

```lua
dutyPoint = {
  enabled = true,
  coords = { x = REAL_X, y = REAL_Y, z = REAL_Z },
  radius = REAL_ENTRE_1_E_5,
  drawDistance = REAL_ENTRE_5_E_50,
  label = 'TEXTO_DE_2_A_64_CARACTERES'
}
```

`drawDistance` deve ser maior ou igual ao `radius`; o motivo deve ter entre 3 e 255 caracteres. Os placeholders acima documentam o formato e não são valores para copiar.

Após salvar, um policial real deve ir ao ponto e usar a interação do `mz_org` para alternar duty. Não usar `SetOrgMemberDuty`, SQL ou bypass staff para produzir os estados de G02/G03.

## 4. Ativar temporariamente o harness

No console FXServer:

```text
set mz_police_environment staging
set mz_police_p1_gate_enabled 1
restart mz_police
```

Sem esses comandos, o harness continua desativado por padrão. Ele aceita somente o console do servidor.

Anotar três IDs reais com `status`:

- `<POLICE_ON>`: policial com personagem ativo e em duty;
- `<POLICE_OFF>`: policial com personagem ativo e fora de duty;
- `<NON_POLICE>`: personagem ativo sem vínculo policial.

## 5. Executar G01–G10

```text
# G01 — boot/readiness
mz_police_p1_gate status

# G02 — organização + capability + duty válidos
mz_police_p1_gate probe <POLICE_ON> org.view police

# G03 — negação fora de duty
mz_police_p1_gate probe <POLICE_OFF> org.view police

# G04 — negação sem organização policial
mz_police_p1_gate probe <NON_POLICE> org.view police

# G05 — capability inexistente/negada
mz_police_p1_gate probe <POLICE_ON> police.p1.gate.denied police

# G06 — separação entre staff e polícia
mz_police_p1_gate probe <POLICE_ON> staff.logs.view police

# G07 — após desconectar um jogador, sem reutilizar o source
mz_police_p1_gate probe <OFFLINE_SOURCE> org.view police

# G08 — perda do owner de organizações
stop mz_org
mz_police_p1_gate probe <POLICE_ON> org.view police
ensure mz_org
ensure mz_police
mz_police_p1_gate status

# G09 — somente em janela isolada e sem jogadores
stop mz_police
stop mz_core
ensure mz_police
ensure mz_core
stop mz_org
ensure mz_org
ensure mz_police
mz_police_p1_gate status

# G10 — restart do consumidor
restart mz_police
mz_police_p1_gate status
mz_police_p1_gate probe <POLICE_ON> org.view police
```

Em G08/G09, se o FXServer interromper também os dependentes, registrar a cascata como evidência e restaurar sempre na ordem `mz_core` → `mz_org` → `mz_police`. Após restaurar `mz_org`, repetir `mz_police_p1_gate status` até `ready=true`; não avançar enquanto o seed estiver pendente. Não executar G09 com jogadores conectados.

Resultados esperados: G02 `authorized`; G03 `duty_required`; G04 `police_organization_not_found`; G05 `capability_denied`; G06 `staff_capability_forbidden`; G07 negação de sessão; G08 `organization_unavailable` ou cascata documentada; G09 start bloqueado pela dependência; G10 readiness e autorização restauradas.

## 6. Encerrar

Capturar as linhas `[mz_police][P1_GATE]` e os registros `police.p1_gate.probe`. Depois, no console:

```text
set mz_police_p1_gate_enabled 0
restart mz_police
```

Confirmar que `mz_police_p1_gate` deixou de existir. Não iniciar P2 até G01–G10 terem evidência real e aprovação.
