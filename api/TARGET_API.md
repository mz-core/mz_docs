---
title: mz_target — Contextual option API
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: mz_target
last_reviewed: 2026-10-19
applies_to: [mz_target, client_consumers]
related_contracts: [MZ-TARGET-v1]
---

# Target API

`mz_target` é o owner da interação por mira. Ele detecta o alvo e apresenta opções somente em cards retangulares. Ações rápidas circulares pertencem ao `mz_radialmenu`; pontos/zonas pertencem ao registry espacial do `mz_interact`.

## Registry cliente

```lua
local ok, handle = exports['mz_target']:RegisterOption({
  id = 'inspect',
  label = 'Inspecionar',
  contexts = { 'player', 'ped', 'vehicle', 'object' },
  models = { 'police' }, -- opcional
  canInteract = function(context) return context.entity ~= nil end,
  onSelect = function(context) end
})

exports['mz_target']:RemoveOption(handle)
local metadata = exports['mz_target']:GetOption(handle)
```

Campos: `id`, `label`, `description`, `icon`, `kind`, `contexts`, `entity`/`entities`, `model`/`models`, `children`, `disabled`, `canInteract(context)` e `onSelect(context)`. O owner vem de `GetInvokingResource()` e seus registros são removidos quando o provider para.

Contexts: `player`, `ped`, `vehicle`, `object`, `entity`, `model`, `drop`, `trunk` e `glovebox`. Player, NPC, veículo e objeto preservam tipos distintos; `entity` aceita qualquer entidade mirada e `model` requer um modelo resolvido. O contexto de player inclui `target.player_server_id`.

## Seleção explícita

`exports['mz_target']:StartTargeting()` ativa o modo de mira sem depender do radial. É a integração usada pelo atalho policial para pedir que o operador selecione um cidadão explicitamente.

`OpenPlayerMenu()` continua como ponte de migração e delega a abertura para `mz_radialmenu` quando disponível. Os campos legados `quick` e `presentation` não alteram mais a interface do target.

## Segurança e lifecycle

A NUI recebe apenas dados serializáveis e responde somente com `option_id`. Lua resolve o ID dentro do menu aberto, revalida alvo e `canInteract`, e então executa o callback do provider. Autoridade e mutações sensíveis permanecem no servidor.

- `ESC` fecha e libera foco;
- `Backspace`, botão direito ou `Voltar` sobe um nível;
- alvo inválido fecha o menu;
- probes assíncronos persistem entre ticks e são invalidados no fechamento;
- resource stop limpa foco, entidades locais e registros do owner.
