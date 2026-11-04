---
title: MZ Framework — Clothing items e paper-doll equipment
status: CANONICAL
authority: LEVEL_3_CONTRACT
owner: mz_inventory + mz_clothing
last_reviewed: 2026-10-09
applies_to: [mz_inventory, mz_clothing, mz_creator, mz_economy]
related_contracts: [POST_CORE_ARCHITECTURAL_FREEZE]
---

# Clothing items e paper-doll equipment

## Ownership

- `mz_inventory` é o único owner/writer da custódia: a peça é o item único `clothing_garment`, identificado por `instance_uid`, e permanece no mesmo storage canônico ao mover entre `main` e `equipment`.
- `mz_clothing` é o owner do catálogo de `garment_id`, lojas, compra, outfits e resolução visual por modelo do ped.
- `mz_economy` é o único writer de saldo/ledger. Compra e compensação usam chaves idempotentes distintas.
- `mz_creator` persiste a aparência-base (rosto, corpo e cabelo). Seus componentes de roupa são apenas bootstrap seguro; ele não persiste nem mescla o equipamento corrente.
- `mz_core` não recebeu responsabilidade, tabela ou contrato novo.

## Contrato do item

O metadata autoritativo mínimo é:

```lua
{
  schema = 1,
  display_label = 'Jaqueta urbana',
  brand = 'MZ Street',
  quality = 100,
  clothing = {
    schema = 1,
    garment_id = 'mz_basic_jacket',
    slot = 'torso',
    model_family = 'freemode',
    catalog_revision = 1,
    variants = {
      mp_m_freemode_01 = { kind = 'component', native_id = 11, drawable = 15, texture = 0 },
      mp_f_freemode_01 = { kind = 'component', native_id = 11, drawable = 15, texture = 0 }
    }
  },
  purchase = { operation_id = '...', shop_id = 'clothing_1' }
}
```

O cliente recebe apenas metadata curado. Na compra, o servidor tira um snapshot sanitizado das variantes visuais no item; assim uma alteração futura do catálogo não transforma uma peça já possuída em outra. O `garment_id` permanece estável e conteúdo custom deve preferir collection-local addressing.

## Equipamento e concorrência

`equipment` é um inventory type do mesmo Character/citizenid. Equipar, retirar, swap e aplicar outfit passam pelos locks e pela transação do `mz_inventory`. O servidor valida owner, item único/equipável, slot declarado e slots reservados. Dois pedidos concorrentes não duplicam nem criam uma segunda instância.

Slots `armour` e `backpack` ficam visíveis porém desabilitados até que seus efeitos de gameplay tenham owner e regras explícitas. O paper-doll não concede armadura, capacidade, armas ou qualquer estado de gameplay.

## Compra

Uma sessão de loja exige proximidade, action gate e lifecycle do BanGuard quando disponível. O preço vem do catálogo do servidor. A operação:

1. registra `operation_id` único;
2. debita no `mz_economy` com idempotência;
3. cria uma única instância no `mz_inventory`;
4. compensa o débito com outra chave idempotente se a criação falhar;
5. opcionalmente move a mesma instância para o slot equipado.

Retry local é serializado e retry após restart procura a instância pelo `purchase.operation_id` antes de cobrar/criar novamente.

## Outfits

`mz_clothing_outfits.slots_json` guarda somente `{ [equipment_slot] = instance_uid }`. Aplicar outfit é uma única mutação atômica no inventário. Se qualquer referência não existir ou não couber, nenhuma alteração parcial é persistida.

## Contratos públicos

- `mz_inventory`: `GetPlayerEquipment`, `EquipPlayerItem`, `UnequipPlayerItem`, `ApplyPlayerEquipmentLoadout`.
- Server-only para `mz_clothing`: `GetPlayerEquipmentState`, `FindClothingPurchaseItem`.
- `mz_clothing`: `GetPlayerClothingProjection`, `ApplyPlayerOutfit`.
- Evento local de UI: `mz_inventory:client:visibilityChanged`.
- Evento de reconciliação visual: `mz_inventory:client:equipmentChanged` → `mz_clothing:server:requestReconcile` → `mz_clothing:client:applyEquipment`.

## Reset pre-production obrigatório

O schema antigo de `mz_clothing_outfits.appearance_json` é incompatível e não tem consumidor legítimo. Em staging descartável, antes de iniciar o resource novo, execute manualmente:

```sql
DROP TABLE IF EXISTS mz_clothing_outfits;
DROP TABLE IF EXISTS mz_clothing_purchase_operations;
```

O resource recria apenas o schema final. Nenhuma operação destrutiva é executada automaticamente.

## Validação runtime pendente

Validar em FiveM os dois freemode peds, drawables baseline, collection natives para packs custom, abertura/fechamento da câmera, compra/reembolso com banco real, reconexão, concorrência, drag/drop, outfit ausente e resoluções 1920×1080 e 2560×1440.
