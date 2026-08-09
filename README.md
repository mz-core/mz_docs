---
title: MZ Framework — Documentação oficial
status: CANONICAL
authority: LEVEL_3_GUIDE
owner: MZ Documentation
last_reviewed: 2026-08-09
applies_to: [mz_framework, developers, integrators, maintainers]
related_contracts: [MZ-DOCS-v1, MZ-RULES-v1]
---

# MZ Framework

Este diretório é o portal oficial de documentação da MZ Framework para developers, integradores, mantenedores e ferramentas de IA. Ele descreve contratos e conceitos públicos sem incorporar auditorias internas, working-tree hashes, Gate worklogs ou findings de segurança.

## Como usar

- Comece pelo [índice](INDEX.md).
- Para mudanças técnicas, leia primeiro as [regras aplicáveis](standards/FRAMEWORK_RULES.md), depois a documentação do owner e o código/testes atuais.
- Consulte [AI Context](ai/AI_CONTEXT.md) e [Reading Order](ai/READING_ORDER.md) quando selecionar contexto para automação.
- Trate documentos `HISTORICAL` e snapshots da Wave 0 como evidência temporal, nunca como estado runtime atual.

## Autoridade

`mz_docs` contém a documentação oficial consumível. `_architecture` permanece o repository de engenharia interna: ADRs, decisões, gates, baselines, findings, audits e wave reports.

O código atual prevalece para comportamento CURRENT. Regras e decisões ratificadas prevalecem para TARGET e contratos normativos. Uma lacuna entre ambos deve ser registrada, não ocultada.

## Repository

O conteúdo foi criado como diretório documental na Wave 1A. Um repository Git independente é recomendado para revisão e versionamento futuros, mas não foi inicializado sem decisão explícita do mantenedor.
