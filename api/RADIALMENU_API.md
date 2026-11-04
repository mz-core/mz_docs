---
title: mz_radialmenu — Quick action API
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: mz_radialmenu
last_reviewed: 2026-10-19
applies_to: [mz_radialmenu, client_consumers]
---

# Radial menu API

`mz_radialmenu` é o owner da apresentação circular de ações rápidas. Ele não usa raycast, entidade mirada ou cache do `mz_target`.

```lua
local ok, handle = exports['mz_radialmenu']:RegisterOption({
  id = 'operations',
  label = 'Operações',
  icon = 'shield',
  canOpen = function(context) return context.in_vehicle == false end,
  children = {
    { id = 'action', label = 'Executar', onSelect = function(context) return true end }
  }
})

exports['mz_radialmenu']:RemoveOption(handle)
exports['mz_radialmenu']:GetOption(handle)
exports['mz_radialmenu']:Open()
exports['mz_radialmenu']:Close()
exports['mz_radialmenu']:IsOpen()
```

IDs são determinísticos (`owner:id`), registro repetido substitui a raiz anterior e resource stop remove opções do provider. Re-registre ao receber `mz_radialmenu:client:ready` ou no `onClientResourceStart` do radial.

`canOpen` controla visibilidade, nunca autoridade. A seleção revalida `canOpen`, fecha e libera o foco antes de chamar `onSelect`. Abertura é recusada quando outra NUI já possui foco.

F3 é o padrão remapeável via FiveM. Sem `ox:locale`, o fallback é pt-BR; o owner pode definir `setr ox:locale en` para inglês (en-US) ou `pt-br` explicitamente.
