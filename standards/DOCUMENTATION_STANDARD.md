---
title: MZ Framework — Standard de documentação
status: CANONICAL
authority: LEVEL_1_NORMATIVE
owner: MZ Documentation
last_reviewed: 2026-08-09
applies_to: [mz_docs, resource_readmes]
related_contracts: [MZ-DOCS-v1, MZ-R035, MZ-R036, MZ-R037, MZ-R038, MZ-R053]
---

# Standard de documentação

## Metadata obrigatória

Todo documento oficial declara `title`, `status`, `authority`, `owner`, `last_reviewed`, `applies_to` e `related_contracts`.

Os estados usados neste portal são:

- `CANONICAL`: fonte vigente para seu escopo;
- `REFERENCE`: inventário ou explicação verificável, subordinada ao código e às fontes normativas;
- `GUIDE`: procedimento, onboarding ou receita;
- `GENERATED`: produzido por gerador declarado;
- `HISTORICAL`: evidência temporal, nunca verdade presente;
- `TEST_REPORT`: evidência de teste com ambiente/data declarados;
- `DEPRECATED`: substituído, com replacement e política de remoção.

## Regras de autoridade

1. Uma regra, contrato ou decisão possui uma fonte canônica.
2. ADR explica a decisão; não duplica a especificação vigente.
3. Documento local do resource resume e aponta para o portal oficial.
4. Documento CURRENT cita código/teste atual e separa TARGET.
5. Documento histórico declara baseline/data observada.
6. Documento gerado declara source e generator.
7. Assinatura, evento, tabela ou garantia não verificada deve ser marcada `UNKNOWN` ou omitida.

## Freshness

`last_reviewed` muda somente após revisão real. Alterações em API, schema, authority, dependency ou failure behavior exigem revisão da documentação correspondente.
