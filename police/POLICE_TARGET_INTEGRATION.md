# MZ Police — target e radial independentes

## Canais

- `mz_target`: ao mirar explicitamente um player, mostra **Ações policiais** com as ações de custódia, revista, viatura, identificação e MDT já existentes.
- `mz_radialmenu`: mostra **Polícia** apenas para policial autorizado e em duty. Contém ações sem alvo (`Abrir MDT`, sistemas avançados habilitados) e `Selecionar cidadão`, que abre o modo de mira do target.

O radial nunca escolhe o jogador mais próximo e não usa entity handle armazenado. Desligar target mantém MDT/sistemas independentes no radial; desligar radial mantém todas as ações por mira no target.

## Autoridade

O estado cliente de acesso controla somente visibilidade. Cada operação existente continua passando pelos handlers e por `MZPolice.Policy.authorize`, com revalidação de identidade/sessão, organização, duty, capability, distância, routing bucket, custódia e revisão conforme a ação.

Processing/intake e integrações prisionais desativadas continuam desativadas. A integração visual não cria evento de negócio nem backend novo.

## Lifecycle

Os dois registros usam IDs namespaced e idempotentes. `mz_police` re-registra quando target/radial iniciam e também atende `mz_radialmenu:client:ready`. Cada registry limpa as opções do provider quando ele para.

Ordem recomendada, sem dependência rígida entre os canais:

```cfg
ensure mz_target
ensure mz_radialmenu
ensure mz_police
```

## Teste runtime com dois clientes

1. Policial on-duty abre F3 e vê **Polícia**; civil/off-duty não vê.
2. **Abrir MDT** funciona sem target.
3. **Selecionar cidadão** fecha o radial, ativa o target e exige mirar o outro jogador.
4. No player mirado, abrir **Ações policiais** e testar as ações válidas.
5. Alvo ausente, distante, em bucket diferente ou desconectado é rejeitado pelo servidor.
6. Reiniciar cada resource isoladamente não duplica nem deixa opções órfãs.

Runtime FiveM desta entrega: **NOT RUN**.
