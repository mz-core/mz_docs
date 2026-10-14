# MZ Police — relatório de implementação P6

Data: 2026-10-14
Wave: P6 — registros e MDT
Estado: **gate estático aprovado; regras legais e runtime FiveM/MySQL pendentes**

## Resultado implementado

O backend policial agora oferece contratos server-side para:

- busca paginada e autorizada de cidadãos pelo contrato canônico do `mz_core`;
- ficha policial isolada por organização;
- criação, leitura, listagem e atualização concorrente de ocorrências, investigações e casos;
- vínculo de pessoas e oficiais aos casos;
- acusações com snapshot imutável da regra legal aplicada;
- multas persistentes com estados `pending` e `paid`;
- criação de mandados pendentes e revisão separada para aprovação/cancelamento;
- histórico de domínio durável e auditoria central no `mz_core`.

Não foi criado `mz_mdt`. Os eventos `mz_police:server:records:*` formam o contrato mínimo para uma futura interface dentro do próprio `mz_police`; nenhuma NUI foi antecipada.

## Ownership e dependências

- identidade e busca de personagem: `mz_core` (`SearchCharacters`, `GetCharacterByCitizenId`, `GetSourceByCitizenId`);
- organização, capability e duty: `mz_org`;
- casos, fichas, acusações, multas, mandados e histórico policial: `mz_police`;
- saldo, conta organizacional, transação e ledger: `mz_economy`;
- logs centrais: `mz_core`.

O `mz_core` não foi alterado.

## Persistência

Foram adicionadas tabelas InnoDB próprias do domínio:

- `mz_police_records`;
- `mz_police_cases`;
- `mz_police_case_officers`;
- `mz_police_case_people`;
- `mz_police_case_charges`;
- `mz_police_citations`;
- `mz_police_warrants`;
- `mz_police_history`.

Criação e alterações relacionadas usam transações. IDs públicos são determinísticos por ator/request ID; o histórico possui operation ID único e fingerprint, impedindo replay divergente. Atualizações de caso exigem `expectedRevision`.

## Multas e economia

`mz_economy` ganhou o export server-only allowlisted `SettlePlayerOrganizationCharge`. Ele:

- aceita somente o chamador `mz_police`;
- revalida o personagem online e sua identidade;
- bloqueia as contas do jogador e da organização;
- debita banco e credita a conta compartilhada na mesma transação;
- grava as duas pernas no ledger;
- persiste a chave idempotente junto com a movimentação;
- rejeita replay com fingerprint diferente.

A multa policial é criada como `pending` antes da chamada econômica. Após retorno, passa para `paid` com o recibo. Se houver queda entre a liquidação e a atualização policial, repetir o mesmo request recupera o recibo idempotente e reconcilia o estado sem nova cobrança. Alvo offline, saldo insuficiente ou dependência indisponível mantêm a multa pendente e registram o motivo.

## Configuração fail-closed

Permanecem desativados/vazios por padrão:

- `MZPoliceConfig.Records.legalRulesEnabled = false`;
- `jurisdictions = {}`;
- `charges = {}`;
- `citations = {}`;
- `warrants.enabled = false` e `warrants.rules = {}`;
- `legalVersion = 'unapproved'`.

Nenhum crime, valor de multa, pena, fundamento de mandado ou jurisdição foi inventado. As capabilities foram registradas no owner, sem concessão automática a cargos: `police.records.write`, `police.citation.issue`, `police.warrant.create`, `police.warrant.approve` e `police.arrest.request`.

## Segurança

- autorização por identidade ativa, organização, capability e duty em cada operação;
- payloads fechados e tamanhos limitados;
- rate limit por source;
- isolamento de leitura/escrita por `org_code`;
- idempotência durável e detecção de request ID conflitante;
- lock local por entidade e revisão otimista dos casos;
- valores de multa derivados exclusivamente do catálogo server-side;
- target source resolvido pelo citizen ID canônico;
- nenhum export amplo de mutação MDT para resources externos;
- nenhum bypass staff, decisão legal automática ou escrita direta em tabelas alheias.

## Testes e evidências

- suíte completa `mz_police` P1–P6: PASS;
- P6 records: 13 casos PASS;
- P6 contrato estático: 10 verificações PASS;
- suíte Lua completa `mz_economy`: PASS;
- liquidação policial econômica: 4 casos PASS;
- capability/allowlist do `mz_org`: PASS;
- sintaxe Lua dos resources tocados: PASS;
- `git diff --check`: PASS, apenas avisos de conversão LF/CRLF.

Os harnesses cobrem autorização, payload malicioso, regra legal desativada, replay, request conflitante, revisão obsoleta, cobrança única e aprovação separada de mandado. Mocks não equivalem ao runtime FiveM/MySQL.

## Pendências de runtime e configuração

- aprovar e configurar jurisdições, catálogo legal, multas, penas e fundamentos de mandado;
- atribuir capabilities somente aos cargos aprovados;
- validar criação/constraints das tabelas no MySQL real;
- validar busca, isolamento entre corporações, reconnect e restart no FXServer;
- validar saldo/ledger/conta organizacional e reconciliação após falha real;
- projetar a NUI MDT em wave própria de frontend, preservando estes contratos;
- associar a evidência física P5 a um caso P6 por extensão explícita no owner `mz_inventory`.

Nenhum runtime PASS é declarado.

## Próxima wave

P7 — criar `mz_prison` independente, consumindo somente sentenças server-side aprovadas e mantendo regras de pena desabilitadas até configuração real.
