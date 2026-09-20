# Resumo — ASRs, GRASP e SOLID

## 1. ASRs (requisitos importantes para a arquitetura)

- **Req. 7 — impedir reserva duplicada:** exige operação atômica no banco.
- **Req. 8 — reserva expira em 10 min:** exige controle automático de expiração.
- **Req. 9 — resposta em até 2 s:** exige consultas rápidas, cache ou réplica.
- **Req. 10 — 3.000 usuários simultâneos:** exige serviços sem estado e escala horizontal.
- **Req. 11 — devolver mesmo sem notificação:** notificação deve ser assíncrona.
- **Req. 12 — QR Code, cartão e NFC:** exige interface e polimorfismo.
- **Req. 13 — diferentes cobranças:** exige política de tarifa separada.
- **Req. 14 — registrar operações:** exige histórico e auditoria.
- **Req. 15 — somente administradores alteram dados:** exige autorização centralizada.

Prioridade principal: **7 → 15 → 11 → 10**.

## 2. Reserva concorrente

- Problema: dois usuários podem consultar a bicicleta ao mesmo tempo e ambos tentarem reservá-la.
- A propriedade necessária é **atomicidade/exclusividade**.
- `if (disponivel) reservar()` não funciona, pois é um **check-then-act não atômico**.
- Lock em memória não basta quando existem várias instâncias do sistema.
- A garantia deve ficar em um recurso compartilhado, normalmente o banco.

```sql
UPDATE veiculo
SET status = 'RESERVADO'
WHERE id = ? AND status = 'DISPONIVEL';
```

- Uma linha alterada: reserva realizada.
- Zero linhas alteradas: outro usuário reservou primeiro.
- Trade-off: a reserva precisa usar o banco principal, mas o bloqueio é curto e apenas naquela bicicleta.

## 3. Distribuição de responsabilidades

- `Estacao`: consulta veículos disponíveis.
- `Veiculo`: verifica se pode ser retirado.
- `Reserva`: controla validade e cancelamento.
- `Viagem`: registra retirada, devolução e trajeto.
- `RepositorioViagens`: salva viagens.
- `CalculadoraTarifa`: calcula o valor.
- `MeioDeIdentificacao`: identifica por QR Code, cartão ou NFC.
- `Notificador`: envia confirmações.
- `ReservaService`: coordena o caso de uso de reserva.

## 4. GRASP aplicado

- **Information Expert:** a responsabilidade fica com quem possui os dados necessários.
  - Ex.: `Veiculo` verifica sua disponibilidade.
- **Creator:** quem possui ou usa diretamente um objeto pode criá-lo.
  - Ex.: `Bicicleta` poderia criar uma `Reserva`.
- **Controller:** um serviço recebe a solicitação e coordena o caso de uso.
  - Ex.: `ReservaService` coordena a criação da reserva.
- **High Cohesion:** cada classe possui responsabilidades relacionadas.
  - Ex.: `CalculadoraTarifa` apenas calcula tarifas.
- **Low Coupling:** classes conhecem o mínimo possível umas das outras.
- **Polymorphism:** comportamentos diferentes são implementados por tipos diferentes.
- **Indirection:** um intermediário reduz a dependência direta.
  - Ex.: fila de eventos entre devolução e notificação.
- **Protected Variations:** interfaces protegem o sistema de mudanças futuras.

## 5. Formas de identificação

- Evitar vários `if/else` para QR Code, cartão e NFC.
- Criar a interface `MetodoIdentificacao` e uma implementação para cada tecnologia.
- Para adicionar biometria, cria-se outra implementação sem alterar as existentes.

Princípios envolvidos:

- GRASP: **Polymorphism, Low Coupling e Protected Variations**.
- SOLID: **OCP e DIP**.

## 6. Bicicletas convencionais e elétricas

- Não usar uma flag `eletrica` com vários condicionais.
- Usar uma classe base e subclasses:
  - `BicicletaConvencional`.
  - `BicicletaEletrica`: só pode ser retirada com bateria igual ou superior a 10%.
- Cada subtipo implementa `podeSerRetirada()` conforme sua regra.

Princípios:

- **Polymorphism:** elimina condicionais por tipo.
- **Information Expert:** a bicicleta elétrica possui o nível de bateria e aplica a regra.
- **OCP:** novos tipos são adicionados sem modificar os existentes.
- **SRP:** cada classe possui uma responsabilidade específica.
- **ISP:** bicicletas comuns não recebem operações de bateria.
- **LSP:** qualquer subtipo deve poder substituir a classe base sem quebrar o cliente.

> Herança não garante LSP. A subclasse não pode exigir mais, entregar menos ou lançar exceções inesperadas.

## 7. Inclusão de patinetes

- `PatineteEletrico` não deve herdar de `Bicicleta`.
- Criar a classe abstrata `Veiculo` como raiz.
- `Bicicleta` e `PatineteEletrico` passam a herdar de `Veiculo`.
- A regra do patinete fica em sua própria classe.
- Serviços continuam usando `Veiculo` sem testar seu tipo.

O design estava:

- preparado para novos **comportamentos**, por causa do polimorfismo;
- parcialmente preparado para novos **conceitos**, pois a raiz `Bicicleta` precisou virar `Veiculo`.

## 8. Notificação após devolução

- A devolução não deve chamar o serviço de e-mail e esperar pela resposta.
- Primeiro, o sistema conclui a devolução e publica um evento.
- Depois, um consumidor lê o evento e envia a notificação.

```text
Devolução → evento "VeiculoDevolvido" → fila → consumidor → e-mail
```

Vantagens:

- devolução funciona mesmo se o e-mail falhar;
- menor tempo de resposta;
- menor acoplamento;
- falhas podem ser tratadas com retentativas.

Custos:

- exige fila e monitoramento;
- pode haver mensagens duplicadas;
- gera consistência eventual;
- o consumidor deve ser idempotente.

Princípios: **Low Coupling, Indirection, Protected Variations, SRP e DIP**.

## 9. SOLID no sistema

- **SRP:** cada classe ou serviço tem um único motivo principal para mudar.
- **OCP:** novos veículos, identificações e tarifas entram por extensão.
- **LSP:** subclasses respeitam o contrato da classe base.
- **ISP:** interfaces são pequenas e específicas.
- **DIP:** domínio depende de interfaces, não de banco, fila ou provedor de e-mail.

## 10. Design final

### Camadas

1. **Controllers:** recebem requisições.
2. **Serviços de aplicação:** coordenam os casos de uso.
3. **Entidades e interfaces:** contêm regras de negócio.
4. **Infraestrutura:** implementa banco, fila e notificações.

> A infraestrutura depende do domínio; o domínio não depende da infraestrutura.

### Serviços principais

- `ConsultaService`: disponibilidade e histórico.
- `ReservaService`: reserva, retirada e expiração.
- `DevolucaoService`: devolução e publicação de evento.
- `AdministracaoService`: cadastro, problemas e operações administrativas.

### Decisões principais

- Reserva: `UPDATE` condicional no banco.
- Consulta: cache ou réplica; reserva: banco principal.
- Notificação: evento assíncrono.
- Tipos de veículo: hierarquia polimórfica.
- Tecnologias externas: interfaces no domínio e implementações na infraestrutura.
- Segurança: autenticação no middleware e autorização no serviço, com negação por padrão.

## 11. Decisão controversa: controle da reserva

Solução escolhida: **atualização condicional no banco**.

- É mais simples que `SELECT ... FOR UPDATE`.
- Mantém o bloqueio por poucos milissegundos.
- Evita um componente extra, como Redis.
- Reservas de veículos diferentes não se bloqueiam.
- Encapsular o SQL no repositório, por exemplo: `reservarSeDisponivel()`.

Alternativas rejeitadas:

- **Lock pessimista:** mantém a transação aberta, pode gerar deadlock e consumir conexões.
- **Lock distribuído com Redis:** adiciona complexidade e outra fonte de falhas.

## Para memorizar

- Concorrência na reserva → **atomicidade no banco**.
- Variação por tipo → **polimorfismo**.
- Mudanças futuras → **interfaces + Protected Variations/OCP**.
- Caso de uso → **Service como Controller**.
- Regra junto dos dados → **Information Expert**.
- Infraestrutura externa → **DIP**.
- Notificação não essencial → **evento assíncrono**.
- Subclasse não pode surpreender o cliente → **LSP**.
