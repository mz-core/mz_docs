---
title: MZ Framework — Gate operacional P1 do sistema policial
status: TEST_REPORT
authority: LEVEL_4_REFERENCE
owner: MZ Police
last_reviewed: 2026-10-14
applies_to: [mz_police, mz_core, mz_org, mz_starter]
related_contracts: [MZ-IDENTITY-v2, MZ-DOCS-v1]
---

# Gate operacional P1 em staging

> **Documento hist��rico:** o harness `server/p1_gate.lua` e o comando
> `mz_police_p1_gate` foram removidos do runtime na limpeza estrutural. As instru����es
> abaixo registram a valida��ǜo P1 da ��poca e nǜo se aplicam ao resource atual.

## 1. Resultado atual

Estado do gate: **PRONTO PARA EXECUÇÃO EM STAGING; RUNTIME NÃO EXECUTADO**.

Checklist condensado para o operador: [P1_FXSERVER_CHECKLIST.md](P1_FXSERVER_CHECKLIST.md).

A revisão de fonte e os testes locais confirmam que o harness está preparado e que os contratos esperados são compatíveis. Isso não comprova integração no FiveM. O proprietário autorizou em 2026-10-14 a continuidade estática de P2–P7 sem esperar este runtime; P1 continua sem aprovação operacional e esta matriz permanece obrigatória antes de homologação.

Nenhum evento de rede, export público, callback, bypass administrativo, capability, persistência de domínio ou coordenada foi criado. O harness é server-side, somente console, desabilitado por padrão e restrito ao ambiente declarado como staging.

Evidência local de preparação em 2026-10-14:

| Verificação | Resultado |
|---|---|
| `luac -p` em `server/*.lua` e `tests/*.lua` | PASS |
| `lua tests/p1_harness.lua` | PASS, 23 casos |
| suíte Node do `mz_starter` | PASS, 9 testes |
| `bash -n` nos scripts do starter via Git Bash | PASS |
| scan de registros `RegisterNetEvent`/exports públicos em `mz_police` | PASS, nenhum encontrado |
| `git diff --check` em `mz_starter` e `mz_docs` | PASS, somente avisos LF/CRLF |
| Graphify restrito a `mz_police` | atualizado para 43 nodes e 58 edges |
| processo FXServer/FiveM local | NÃO ENCONTRADO |
| `mz_starter/resources/[mz]/mz_police` local | NÃO INSTALADO |

Os dois últimos itens impedem execução runtime nesta estação; não são falhas da policy.

## 2. Contratos revisados

| Owner | Export real | Retorno consumido pela P1 | Comportamento da policy |
|---|---|---|---|
| `mz_core` | `ResolvePlayerIdentity(source)` | sucesso: `{ ok=true, source, accountId, characterId, citizenid, sessionId, active }`; falhas incluem `session_not_found`, `character_not_active` e `player_not_loaded` | exige source correspondente, IDs positivos, `citizenid` não vazio e `active == true` |
| `mz_core` | `CreateDetailedLog(scope, action, payload)` | ID retornado por `MySQL.insert.await`, ou falha do contrato | registra start, negações resolvidas e probes do gate |
| `mz_org` | `GetOrganizationReadiness()` | `{ ready, terminal, reason }`; reasons reais: `organization_owner_preparing`, `organization_schema_prepare_failed`, `organization_seed_pending`, `organization_seed_failed`, `ready` | só autoriza com `ready == true`; o bridge preserva o objeto no terceiro retorno para diagnóstico |
| `mz_org` | `GetPlayerOrgContext(source)` | array com `orgId`, `code`, `type`, `grade`, `status`, `active`, `config`, `duty`, `permissions` e `capabilities` | seleciona somente organização ativa autorizada e usa duty autoritativo |
| `mz_org` | `CanOrg(source, orgCode, capability)` | boolean | exige `true`; `staff.*` é recusada pela policy e também pelo owner |

Conclusão da revisão: os nomes, a convenção method-style e os tipos usados por `mz_police` correspondem às implementações atuais. Não foi encontrada necessidade de alteração em `mz_core` ou `mz_org`.

## 3. Inicialização, instalação e restart

- `mz_police/fxmanifest.lua` declara dependências explícitas de `mz_core` e `mz_org`.
- `mz_core` registra identidade antes dos exports de player e registra logs antes do bootstrap final.
- `mz_org` registra os exports de domínio antes de `server/main.lua`; schema e seed ficam prontos de forma assíncrona e são refletidos por `GetOrganizationReadiness`.
- `mz_starter/cfg/resources.cfg` inicia `mz_core`, depois `mz_org`, depois `mz_police`.
- `mz_police` é `distribution-only`. O instalador copia a distribuição apenas quando o destino ainda não existe; uma staging já instalada deve receber a revisão pelo processo normal de deploy.
- Se `mz_org` ainda estiver preparando schema/seed quando `mz_police` iniciar, o resource permanece carregado, mas toda autorização falha fechada até readiness. O comando `status` deve ser usado para confirmar a transição para `ready`.
- `restart mz_police` não altera sessão, membership ou duty, que continuam pertencendo aos owners. O restart limpa somente o throttle local de logs de negação.

## 4. Ativação temporária do harness

Adicionar somente à configuração de staging, antes de `ensure mz_police`:

```cfg
set mz_police_environment staging
set mz_police_p1_gate_enabled 1
```

Após publicar os arquivos, iniciar o servidor ou executar:

```text
restart mz_police
mz_police_p1_gate status
```

Resultado mínimo esperado do status:

```text
[mz_police][P1_GATE] STATUS core=started org=started ready=true terminal=false reason=ready error=
```

O comando é registrado com ACL restrita e rejeita qualquer `source` diferente de `0`; portanto deve ser executado no console do servidor. Ele não altera membership, capability ou duty.

## 5. Matriz operacional

Usar jogadores reais com personagem ativo. Substituir `<source>` pelo server ID atual; nunca reutilizar evidência após reconnect sem confirmar a nova sessão.

| ID | Cenário e comando | Resultado esperado |
|---|---|---|
| G01 | boot completo e `mz_police_p1_gate status` | states `started`, `ready=true`, `reason=ready`; log `police.resource.started` presente |
| G02 | `mz_police_p1_gate probe <policial_em_duty> org.view police` | `identity=ok`, membership `found`, `duty=true`, `can_org=true`, `policy=authorized`, audit numérico |
| G03 | `mz_police_p1_gate probe <policial_fora_de_duty> org.view police` | `duty=false`, `can_org=true`, `policy=duty_required` |
| G04 | `mz_police_p1_gate probe <nao_policial> org.view police` | membership `missing`, `policy=police_organization_not_found` |
| G05 | `mz_police_p1_gate probe <policial_em_duty> police.p1.gate.denied police` | `can_org=false`, `policy=capability_denied`; nenhuma capability é criada |
| G06 | `mz_police_p1_gate probe <policial_em_duty> staff.logs.view police` | `can_org=false`, `policy=staff_capability_forbidden` |
| G07 | `mz_police_p1_gate probe <source_offline> org.view police` | `identity=session_not_found` ou erro canônico equivalente, policy negada |
| G08 | durante janela controlada: `stop mz_org`, depois probe de jogador válido | `org_context=organization_unavailable`, `policy=organization_unavailable`; restaurar com `ensure mz_org` e aguardar `status` ready |
| G09 | em janela isolada e sem jogadores: parar `mz_police`, parar `mz_core`, tentar iniciar `mz_police` | o manifest deve bloquear o start pela dependência; se o runtime mantiver `mz_police` ativo após a perda do Core, o probe deve retornar `core_unavailable` |
| G10 | restaurar a ordem canônica e executar `restart mz_police`, `status` e o probe G02 | um novo start auditado e probe autorizado, sem comando duplicado ou erro de export |

Não automatizar `stop mz_core`: isso afeta identidade e sessão do servidor inteiro. G09 exige janela isolada, jogadores desconectados e procedimento de restauração já aprovado para staging.

## 6. Duty point

O seed policial não contém `config.dutyPoint`. O probe informa somente `duty_point=missing`, `disabled` ou `enabled_unverified`; ele nunca imprime ou inventa coordenadas. `enabled_unverified` confirma apenas a presença da configuração: validade e proximidade continuam sendo verificadas pelo fluxo real do owner.

Sem um ponto configurado no owner `mz_org`, `SetSelfOrgDutyAtPoint` retorna `duty_point_not_configured`. Portanto:

- é possível validar a negação para um membership que já esteja fora de duty;
- é possível validar autorização se a staging já possuir membership legitimamente em duty;
- não é possível validar o ciclo completo entrar/sair de serviço pelo fluxo real até o produto configurar o ponto no `mz_org`.

Se não existirem estados reais adequados em staging, G02 ou G03 devem ficar **BLOQUEADOS**, não simulados por `SetOrgMemberDuty`, SQL ou bypass staff.

## 7. Evidências obrigatórias

Para cada caso, registrar: timestamp com timezone, build/commit implantado, artifact do FXServer, OneSync, comando exato, source usado, sessão/personagem retornados, linha completa `[P1_GATE]`, resultado e operador.

Consultar logs de forma somente leitura:

```sql
SELECT id, scope, action, actor, target, org_code, audit_id, data_json, created_at
FROM mz_logs
WHERE scope = 'police'
  AND action IN (
    'police.resource.started',
    'police.authorization.denied',
    'police.p1_gate.probe'
  )
ORDER BY id DESC
LIMIT 100;
```

Modelo de resultado:

| ID | Data/hora | Build | Comando | Resultado | Evidência console/log | Observação |
|---|---|---|---|---|---|---|
| G01–G10 | pendente | pendente | pendente | NÃO EXECUTADO | pendente | preencher em staging |

## 8. Desativação

Após coletar as evidências:

```cfg
set mz_police_p1_gate_enabled 0
```

Executar `restart mz_police` e confirmar que `mz_police_p1_gate` não está registrado. Manter `server/p1_gate.lua` somente até a aprovação do gate; sua remoção deve ser um diff isolado posterior, sem ser confundida com P2.

## 9. Critério de aprovação

O gate só pode ser aprovado quando:

1. G01–G10 estiverem executados com evidência, ou bloqueios forem explicitamente aceitos pelo responsável.
2. G02, G03, G04, G05, G06 e G10 apresentarem exatamente a decisão esperada.
3. Nenhuma autorização sobreviver à perda de dependency, sessão, membership, capability ou duty.
4. O comportamento após restart estiver demonstrado.
5. O harness estiver novamente desabilitado.
6. A pendência de `dutyPoint` estiver resolvida ou aceita como bloqueio formal antes de P2.

Até isso ocorrer, o resultado é **NÃO EXECUTADO NO FIVEM**. A continuidade de desenvolvimento não deve ser confundida com aprovação deste gate.

## 10. Checkpoint Git recomendado após aprovação

Os diretórios são repositórios independentes e contêm alterações preexistentes. Não criar um commit agregado na raiz. Após aprovação, preparar checkpoints separados e revisar o staged diff:

```text
git -C mz_police add .gitignore fxmanifest.lua server tests
git -C mz_police diff --cached --check

git -C mz_starter add -p cfg/resources.cfg tools/install_deps.bat tools/install_deps.sh tools/check_structure.bat docs/install.md
git -C mz_starter diff --cached --check

git -C mz_docs add police/P0_AUDIT_AND_PLAN.md police/P1_IMPLEMENTATION_REPORT.md police/P1_STAGING_VALIDATION_GATE.md
git -C mz_docs add -p INDEX.md
git -C mz_docs diff --cached --check
```

Usar `git add -p` nos arquivos com trabalho preexistente para não incluir hunks não relacionados. O checkpoint deve ser apenas recomendado até o gate runtime ser satisfatório; este documento não autoriza commit automático.

## Impacto arquitetural

- Contratos públicos: nenhum novo.
- Ownership/writers: inalterados.
- Superfície de teste: comando temporário, server-side, console-only, restrito por convars de staging e desabilitado por padrão.
- Persistência: somente auditoria canônica em `mz_logs`; nenhuma tabela ou mutação de domínio nova.
- Segurança: sem evento de rede, export, callback, capability, ACE bypass, duty bypass ou canary alterado.
- P2: desenvolvimento estático autorizado separadamente; não altera o gate runtime P1.
