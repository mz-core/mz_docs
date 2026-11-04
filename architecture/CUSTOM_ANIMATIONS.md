---
title: MZ Framework — Animações customizadas
status: CANONICAL
authority: LEVEL_2_ARCHITECTURE
owner: mz_animations
last_reviewed: 2026-09-24
applies_to: [mz_animations, mz_parkour, animation_consumers]
related_contracts: [MZ-CUSTOM-ANIMATIONS-v1, MZ-R011-MZ-R023]
---

# Animações customizadas — MZ-CUSTOM-ANIMATIONS-v1

## Ownership e direção

`mz_animations` é owner de YCDs/dictionaries customizados, nomenclatura de clips,
registry, validação e ferramentas de parsing/retarget/export. O source de
desenvolvimento fica local e ignorado pelo Git em seu workspace, fora de `stream/`.
Somente assets GTA V validados são liberados aos consumidores e distribuídos.
Um YCD experimental com estrutura conferida pode estar em `stream/` para teste
local opt-in; isso não libera o registry nem a distribuição de produção.

`mz_parkour` é owner de detecção, raycasts, validação de espaço, controles, state
machine, regras de traversal e alinhamento do player/ledge no mundo. O estado
canônico do player continua pertencendo ao `mz_core`; parkour consome esse contrato.

Dependência atual: `mz_parkour -> mz_animations -> mz_core`, com dependência direta
de parkour no core. `mz_animations -> mz_parkour` é proibida. Ler uma state key de
restrição não transfere ownership de gameplay. Futuros consumidores (phone, inventory,
police, jobs, medical) usam chaves do owner em vez de duplicar strings de assets.

## Contrato client atual

Exports aditivos, síncronos, locais, somente leitura e sem eventos de rede:

```lua
local definition, reason = exports['mz_animations']:GetAnimation('parkour.hangIdle')
local info, infoReason = exports['mz_animations']:GetAnimationInfo('parkour.hangIdle')
```

- `GetAnimation(key: string) -> table | nil, reason?`: retorna definição somente se
  `available=true` e `status='DONE'`. Erros: `invalid_key`, `not_found`,
  `asset_not_validated`. Não carrega dict, toca clip nem altera estado do ped.
- `GetAnimationInfo(key: string) -> table | nil, reason?`: snapshot de metadados,
  inclusive reservas indisponíveis. Erros: `invalid_key`, `not_found`.
- Lookup exato, case-sensitive. A tabela retornada é cópia com campos escalares;
  alterá-la não habilita um asset. A disponibilidade é decisão de release do owner,
  não evidência de que o cliente já carregou o dictionary.
- Campos atuais: `key`, `owner`, `dict`, `anim`, `targetModel`, `sourceClip`, `status`,
  `available`, `reason`, `rootMotion`. `status` representa validação do asset.

Reserva atual: `parkour.hangIdle -> mz_parkour_climb / climb_idle`, target
`mp_m_freemode_01`, **PARTIAL**, `available=false`, root motion `IN_PLACE_EXPERIMENTAL`.
Existe YCD experimental em stream, com round-trip estrutural e poses conferidos.
Falta validação visual/FiveM. `GetAnimation` deve retornar
`nil, 'asset_not_validated'` neste estado.

`MZEmotes.Catalog/ByName` e exports existentes `Play`, `PlayByName`, `Stop`,
`IsPlaying`, `GetCurrent`, `RegisterEmote`, `GetCatalog` permanecem compatíveis.
O registry de assets não registra automaticamente um emote. O executor atual bloqueia
emotes durante `mz:parkour`; eventual playback de traversal exige contrato próprio
após o MVP, sem contornar essa restrição com `ignoreRestrictions`.

## CURRENT e TARGET

CURRENT: parkour usa `TaskClimb` nativo e já cancela emotes pelo owner. Nenhum clip
customizado é consumido. Pipeline GTA IV tem inventário/hash/parser, diagnóstico de
canais no Blender, skeleton GTA IV real (90 bones), target masculino (128 bones),
bake experimental de um clip e YCD real. Teste isolado `/mztestanim` só é registrado
com `setr mz_animations_devtest 1` antes de iniciar/reiniciar o resource. Ele usa
metadados de `GetInfo`, nunca habilita o asset e respeita as restrições do ped.
Ver protocolo e limitações no tooling; teste mock não comprova FiveM.

TARGET: validar um único clip primeiro. Disponibilidade exige source/target e bone
map verificados, pose/duração/root validados, round-trip de YCD e prova FiveM sem
deformações graves. Criação de arquivo, registro no catálogo ou sucesso de mock não
atendem esse critério. Gameplay continua responsável pelo alinhamento/deslocamento;
a política de root/pelvis do asset precisa de calibração explícita.

Pipeline e evidências: [workspace GTA IV](../../mz_animations/tools/gta4_climb/README.md).
Ferramentas, rigs e source não entram no manifest nem em pacotes de runtime.
