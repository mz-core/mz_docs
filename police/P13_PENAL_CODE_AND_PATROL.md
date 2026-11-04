# MZ Police P13 — catálogo penal, patrulha e multas

Status: implementação estática concluída; **runtime FiveM/OneSync/CEF/MySQL NOT RUN**.

## 1. O que foi criado

O catálogo jurídico de staging passou de uma única acusação fictícia para um código penal
fictício versionado em **`MZ-RP-STAGING-2`**:

- **40 acusações** em `MZPoliceConfig.Records.charges`;
- **11 multas administrativas/de trânsito** em `MZPoliceConfig.Records.citations`;
- **40 regras prisionais** espelhadas em `MZPrisonConfig.sentenceRules`;
- fluxo de **Prisão rápida** exposto no target do cidadão algemado;
- **multi-acusação** habilitada para até **3 acusações** por operação;
- teto de pena multi-acusação de **7200 segundos** (2 horas) nos dois owners;
- teste de paridade entre o catálogo policial e o catálogo prisional.

O código não representa legislação real. Nomes, artigos e descrições são material de RPG/staging.

### Migração de staging

Casos e acusações persistidos com `MZ-RP-STAGING-1` não são reescritos. Como o framework está em
pré-produção, o caminho recomendado é descartar/recriar os casos de teste. Uma tentativa de sentença
com uma acusação antiga falha fechado em `sentence_rule_mismatch`; não há dual-write nem migração
automática de versão legal.

## 2. Owners e contrato

| Domínio | Owner | Arquivo |
|---|---|---|
| Acusações, multas e versão legal policial | `mz_police` | `config/server.lua` |
| Regras de duração da sentença | `mz_prison` | `config.lua` |
| Caso, acusações, citações e histórico | `mz_police` | `server/records/service.lua`, `server/records/repository.lua` |
| Sentença, intake, tempo e soltura | `mz_prison` | `server/service.lua` |
| Economia das multas | `mz_economy` | via `MZPolice.Bridge.settleCitation` |

`ruleCode`, `name`, `legalVersion` e duração devem ser idênticos nos dois arquivos. O harness
de paridade (mantido em QA externo, não na distribuição) falha se houver divergência ou regra órfã.

### Conversão de tempo

O catálogo usa meses apenas como referência de conteúdo. A duração canônica é persistida em
segundos:

```text
1 mês = 60 segundos
10 meses = 600 segundos
50 meses = 3000 segundos
120 meses = 7200 segundos
```

## 3. Categorias e exemplos

O catálogo cobre:

- **Contra a vida:** homicídio doloso/culposo, latrocínio, tentativa de homicídio.
- **Contra a pessoa:** lesão corporal grave.
- **Contra o patrimônio:** roubo, furto, extorsão, assalto a banco/joalheria, caixa eletrônico,
  loja, veículo clonado, receptação e vandalismo.
- **Armas:** porte de baixo/alto calibre, tráfico de armas, tráfico de munições, itens de
  arrombamento e colete balístico irregular.
- **Entorpecentes:** tráfico e associação ao tráfico.
- **Administração pública e fé pública:** corrupção passiva/ativa, desacato, abuso de poder,
  falsidade ideológica e suborno.
- **Organização criminosa:** formação de quadrilha, organização criminosa e lavagem de dinheiro.
- **Processual:** tentativa de fuga, resistência à prisão, desobediência, perjúrio e plantar provas.
- **Trânsito e ordem pública:** direção perigosa, corrida ilegal, porte de máscara e dinheiro sujo.

Exemplos de multas:

| Código | Multa |
|---|---:|
| `CITACAO_EXCESSO_VELOCIDADE` | R$ 12.000 |
| `CITACAO_DIRECAO_PERIGOSA` | R$ 15.000 |
| `CITACAO_EMBRIAGUEZ` | R$ 30.000 |
| `CITACAO_POSSE_DROGAS_USO` | R$ 10.000 |
| `CITACAO_DANO_PATRIMONIO` | R$ 25.000 |

O arquivo `mz_police/config/penal_code.lua` é a fonte completa de nomes, descrições,
categorias, multas e durações. As capabilities e a operação de Records ficam em
`mz_police/config/server.lua`; ambos complementam `MZPoliceConfig.Records`.

## 4. Fluxo de patrulha

1. O policial algema o cidadão e mantém a custódia.
2. No target do cidadão, selecione **Prisão rápida**.
3. Escolha de 1 a 3 acusações e escreva uma observação.
4. O servidor calcula a pena e a multa com o catálogo server-side.
5. Confirme. O `mz_police` cria um caso `occurrence`, as acusações e citações pendentes.
6. O cidadão pode pagar as citações pelo fluxo de pagamento; a multa não é debitada na prisão.
7. Leve o custodiado ao ponto de intake configurado para iniciar a sentença no `mz_prison`.

A prisão rápida não substitui o MDT: o caso criado é um caso normal e pode ser aberto,
complementado e usado por evidências, perícia e mandados. O MDT continua sendo o fluxo avançado.

### Configuração ativa

```lua
MZPoliceConfig.Arrest.quickEnabled = true
MZPoliceConfig.Arrest.multiChargeEnabled = true
MZPoliceConfig.Arrest.maximumCharges = 3
MZPoliceConfig.Arrest.maximumSentenceSeconds = 7200

MZPrisonConfig.multiCharge.enabled = true
MZPrisonConfig.multiCharge.maximumCharges = 3
MZPrisonConfig.multiCharge.maximumSentenceSeconds = 7200
MZPrisonConfig.multiCharge.aggregation = 'sum'
```

O intake prisional agora usa a facility de staging `bolingbroke`, com booking, cela, áreas e
release point. As coordenadas são candidatas e precisam ser confirmadas no MLO real antes de
produção. O fluxo completo está em [`P14_PRISON_CELLS_AND_EVENTS.md`](./P14_PRISON_CELLS_AND_EVENTS.md).

## 5. Multas

As acusações já possuem `fineAmount`; ao confirmar a prisão rápida, o servidor cria **citations
pendentes** para cada acusação com multa. O pagamento usa `mz_economy` e o contrato existente de
`payCitation`.

O catálogo `Records.citations` está preenchido com 11 infrações administrativas/de trânsito. Ainda
não existe um botão de multa isolada sem prisão no NUI; esse é o próximo passo de interface, usando
o mesmo contrato e sem criar um segundo writer de multas.

## 6. Sistema prisional — referências pesquisadas

Referências de mercado consultadas:

- **qb-prison**: cela, uniforme, propriedade retida, trabalhos, canteen, prison break e lockdown.
- **rcore_prison**: trabalhos, comunidade, solitary, economia interna, canteen, gym, prisão break,
  dashboard policial e integrações.
- **Mango Prison**: intake cinematográfico, terminal de correções, head count, lockdown, visitas,
  comportamento e fugas em múltiplos estágios.
- **tabletpolicial_v2 (vRP)**: catálogo de crimes/multas por checkbox, soma de meses, fiança,
  percentual personalizado e histórico no tablet.

O `mz_prison` já possui a base canônica:

- sentença idempotente e uma sentença ativa por cidadão;
- intake por proximidade/routing bucket;
- confinamento preparado por facility/cela;
- retenção e devolução de propriedade via `mz_inventory`;
- atividades com redução de pena;
- soltura manual e por fim de pena;
- histórico e auditoria.

### Roadmap prisional recomendado

1. **Intake real**: o preset de staging `bolingbroke` já tem booking, cela, áreas e release point;
   falta validar as coordenadas/bucket no MLO real e, se necessário, trocar para um bucket dedicado.
   Veja `P14_PRISON_CELLS_AND_EVENTS.md`.
2. **Multa isolada de patrulha**: formulário no target para `Records.citations`, criando citação
   pendente com caso opcional.
3. **Fiança**: novo contrato no `mz_prison` com valor, pagamento via `mz_economy`, auditoria e
   liberação idempotente. Não reutilizar `payCitation` como se fosse fiança.
4. **Trabalhos/comunidade**: já existe estrutura de jobs no `mz_prison`; faltam pontos reais.
5. **Solitary/lockdown/escape**: adicionar somente depois de intake e confinamento validados.
6. **Canteen/economia interna**: depende de itens canônicos no `mz_inventory`; não criar item
   paralelo.

## 7. Validação

Os comandos abaixo pressupõem a cópia externa de QA; `tests/` não faz parte do resource.

- `lua tests/legal_catalog_parity_harness.lua .` — paridade dos catálogos.
- `lua tests/p7_4_quick_arrest_harness.lua .` — orquestração da prisão rápida.
- `lua tests/p8_target_access_harness.lua .` — registro/visibilidade da ação no target.
- Demais harnesses e testes Node do `mz_police`.

Runtime FiveM, target real, NUI, economia, intake e sentença continuam **NOT RUN**.
