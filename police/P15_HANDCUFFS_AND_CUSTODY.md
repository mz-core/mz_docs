# MZ Police P15 — algemas, item usável e custódia simplificada

Status: implementação estática concluída; runtime FiveM **NOT RUN**.

## 1. Regras atuais

- **Não existe gate de situação prisional para algemar.** Um cidadão com sentença ativa
  continua podendo ser algemado; sentença e algema são fluxos separados.
- **Policial em serviço pode algemar qualquer alvo válido** pelo target, sem exigir
  `police.custody` para a ação de custódia. A organização e o duty continuam sendo
  validados no servidor.
- **Qualquer jogador com o item `handcuffs` pode algemar outro jogador** usando o item
  pelo inventário. O servidor escolhe o jogador mais próximo no mesmo bucket e delega
  ao fluxo `physical` existente.
- **Staff** pode usar `/staffalgemar <id> [motivo]` e `/staffdesalgemar <id>`, com
  distância máxima de 2,5 m, permissão `staff.restraints.manage` e hierarquia Staff.
- Busca, identificação, MDT, prisão e perícia continuam com suas capabilities próprias.
  A simplificação é do alvo/custódia, não do MDT avançado.

## 2. Fluxos

### 2.1 Policial pelo target

1. Policial em serviço, organização autorizada.
2. Target no cidadão → **Algemar**.
3. Servidor valida:
   - identidade do policial e do alvo;
   - organização + duty;
   - alvo vivo;
   - mesmo routing bucket;
   - distância de até 2,5 m;
   - alvo não algemado;
   - sem operação pendente/cooldown.
4. Servidor aplica custódia e animação.
5. Escolta, viatura, transferência e desalgemar seguem o mesmo owner de custódia.

### 2.2 Qualquer jogador com o item

1. O jogador usa o item **Algemas** no inventário.
2. O `mz_inventory` chama o handler registrado pelo `mz_police`.
3. O servidor procura o jogador mais próximo:
   - mesmo routing bucket;
   - até 2,5 m;
   - não é o próprio usuário.
4. O `mz_police` chama `MZPolice.Restraints.request(..., 'physical')` com o
   `preferredUid` do item usado.
5. A algema é consumida no fluxo server-side e uma chave vinculada é emitida.
6. O alvo não precisa estar rendido/downed: `civiliansRequireSurrenderedTarget = false`.

O item `handcuffs` agora é `usable = true` e `closeOnUse = true` no catálogo do
`mz_inventory`.

### 2.3 Staff por comando

```text
/staffalgemar <id> [motivo]
/staffdesalgemar <id>
```

Requisitos:

- `enableStaffCommand = true`;
- permissão `staff.restraints.manage`;
- hierarquia de staff (`mz_core:CanStaffActOnPlayer`);
- alvo e staff no mesmo bucket, dentro de 2,5 m.

## 3. Segurança preservada

Mesmo com a simplificação:

- o cliente não define custódia, item, duração ou autorização;
- o servidor resolve identidade, item, UID/serial, distância, bucket e estado;
- o item físico exige `instance_uid` e `metadata.serial` do `mz_inventory`;
- uma chave vinculada é emitida e validada no desalgemar;
- a operação tem rate limit, cooldown, lock por alvo e auditoria;
- o servidor não confia em `DoesEntityExist` para posição de ped de jogador
  (o check pode falhar em OneSync); usa `GetPlayerPed` + `GetEntityCoords`.

## 4. Arquivos

- `mz_police/server/restraints/service.lua` — fluxo de algema, item físico e validações.
- `mz_police/server/custody.lua` — custódia, escolta, viatura e transferência.
- `mz_police/server/items.lua` — handler de uso do item `handcuffs`.
- `mz_police/server/interactions.lua` — contexto do target.
- `mz_police/server/commands.lua` — `/algemar`, `/desalgemar`, `/staffalgemar`,
  `/staffdesalgemar`.
- `mz_police/config/shared.lua` — distância, item e flags de restrição.
- `mz_inventory/shared/items.lua` — `handcuffs.usable = true`.

## 5. Testes

Os harnesses foram removidos da distribuição do resource e ficam em QA externo/Git.
Os comandos abaixo pressupõem essa cópia de QA.

- `lua tests/p10_restraints_harness.lua .`
- `lua tests/p2a_custody_harness.lua .`
- `lua tests/p2b_vehicle_harness.lua .`
- `lua tests/p7_3_interactions_harness.lua .`
- `lua tests/p15_handcuffs_item_harness.lua .`
- `lua tests/p10_static_harness.lua .`
- `lua tests/p13_patrol_contract.test.js` / testes Node do `mz_police`.

Runtime do target, animação, inventário, item físico, comandos Staff, OneSync e
chave vinculada continuam **NOT RUN**.
