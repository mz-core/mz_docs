---
title: MZ Framework — Sincronizacao visual das armas no corpo
status: CANONICAL
authority: LEVEL_3_CONTRACT
owner: mz_inventory
last_reviewed: 2026-10-17
applies_to: [mz_inventory]
related_contracts: [CLOTHING_EQUIPMENT]
---

# Sincronizacao visual das armas no corpo

## Ownership

- `mz_inventory` continua sendo o unico owner/writer da custodia, do equipamento e da arma sacada.
- O cliente nao escreve o estado visual. Ele apenas le o state bag publicado pelo servidor e reutiliza `Config.Weapons.bodyAttachments` para modelo, preset, osso, offset e rotacao.
- Nenhuma tabela, export ou callback novo foi criado.

## Contrato publico

Player state bag replicado:

```lua
Player(source).state['mz_inventory_body_weapons'] = {
  v = 1,
  session = 3,            -- incrementa em clear/troca de sessao/resource stop
  revision = 12,          -- monotonico por sessao; reinicia na limpeza
  updated_at = 1760000000,
  drawn_slot = 'primary', -- nil quando nenhuma arma esta sacada
  slots = {
    { key = 'primary', slot = 15, item = 'weapon_carbinerifle' },
    { key = 'sidearm', slot = 16, item = 'weapon_pistol' }
  },
  reason = 'equipment_changed'
}
```

- `key` e o `equipment_slot` (`primary`, `sidearm`, `melee`, `special`).
- `drawn_slot` remove o prop correspondente do corpo; a arma em maos continua no fluxo nativo do ped.
- O payload e limpo em desconexao, fechamento/troca de sessao de personagem, stop do resource e quando o jogador deixa de estar carregado.

## Reconciliacao no cliente

1. `AddStateBagChangeHandler` usa o `value` recebido; nao rele o bag durante o callback.
2. Um loop de 1 s resolve `serverId = GetPlayerServerId(playerIndex)` para cada entrada de `GetActivePlayers()` e le `Player(serverId).state`. Player index local nunca e usado como server ID.
3. O ped remoto e resolvido por `GetPlayerFromServerId(serverId)` + `GetPlayerPed(playerIndex)`; a identidade e conferida com `GetPlayerServerId(playerIndex) == serverId`.
4. Props sao chaveados por `equipment_slot` e comparados por assinatura (`item|model|preset`), impedindo duplicacao.
5. `session` diferente para maior reseta `revision`; sessao menor so e aceita quando `updated_at` e mais novo (restart do resource), nunca por leitura atrasada.
6. Ausencia temporaria do bag preserva os props por `REMOTE_BODY_MISSING_GRACE_MS`; estado vazio valido (`slots = {}`) e autoritativo e remove imediatamente.
7. Props antigos sao deletados quando o ped muda, o item sai do estado, a arma e sacada ou o jogador sai do alcance.
8. `onResourceStop` remove todos os props locais e remotos.

## Autoridade

O payload e derivado de `getEquipmentContext(source)` e de `EquippedWeaponsBySource[source]`. Nenhum campo enviado por cliente influencia o estado publicado.
