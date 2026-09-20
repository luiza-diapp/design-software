# Aula 7 — SOLID

## Ideia geral

- SOLID reúne **5 princípios de projeto orientado a objetos**.
- Objetivo: facilitar manutenção e evitar que uma mudança se espalhe pelo sistema.

## S — Single Responsibility Principle (SRP)

**Ideia:** uma classe deve ter uma **razão principal para mudar**.

### Problema

Uma classe mistura responsabilidades diferentes:

- regra de negócio;
- banco de dados;
- notificações;
- geração de arquivos.

### Pergunta para identificar

> Quais mudanças futuras poderiam fazer essa classe mudar?

Se existem **motivos independentes**, provavelmente as responsabilidades deveriam ser separadas.

### Exemplo

Em vez de `PedidoService` fazer tudo:

- `Pedido` → regras do pedido;
- `PedidoRepository` → persistência;
- `Notificador` → comunicação;
- `PedidoService` → coordena o processo.

---

## O — Open/Closed Principle (OCP)

**Ideia:** permitir adicionar novos comportamentos sem modificar constantemente código que já funciona.

### Problema

```java
if (tipo == PREMIUM)
    ...
else if (tipo == FUNCIONARIO)
    ...
else if (tipo == PARCEIRO)
    ...
```

Cada novo tipo exige alterar o método existente.

### Solução

Criar um **ponto de extensão**:

```java
interface PoliticaDesconto {
    double calcular(double valor);
}
```

Cada desconto implementa essa interface.

Assim, para adicionar um novo desconto:

- cria-se uma nova implementação;
- código antigo não precisa ser alterado.

### Importante

- Não criar abstrações pensando em qualquer possibilidade futura.
- OCP é mais útil quando:
  - existem várias versões de um comportamento;
  - novas variações aparecem frequentemente;
  - código estável precisa ser alterado constantemente.

---

## L — Liskov Substitution Principle (LSP)

**Ideia:** um subtipo deve poder substituir o tipo base sem quebrar o comportamento esperado.

### Pergunta

> Se eu trocar o tipo base por qualquer subtipo, o código continua funcionando?

### Sinais de problema

Uma subclasse:

- lança exceções inesperadas;
- ignora operações;
- exige condições extras;
- muda o comportamento esperado.

### Exemplo ruim

```java
class PedidoDigital extends Pedido {
    double calcularFrete() {
        throw new UnsupportedOperationException();
    }
}
```

`PedidoDigital` não deveria ser obrigado a calcular frete.

### Melhor solução

```java
interface Entregavel {
    double calcularFrete();
}
```

Somente pedidos que realmente possuem entrega implementam `Entregavel`.

---

## I — Interface Segregation Principle (ISP)

**Ideia:** uma classe não deveria depender de operações que não utiliza.

### Problema

Uma interface muito grande:

```java
interface GatewayPagamento {
    void pagar();
    void parcelar();
    void gerarBoleto();
    void cancelarBoleto();
}
```

PIX não precisa de várias dessas operações.

### Solução

Criar interfaces menores:

```java
interface Pagamento {
    void pagar();
}

interface PagamentoParcelado {
    void parcelar();
}

interface OperacoesBoleto {
    void gerarBoleto();
    void cancelarBoleto();
}
```

Cada implementação assume apenas os contratos que fazem sentido.

---

## LSP x ISP

### LSP

- Foco: **comportamento**.
- Pergunta:
  > O subtipo consegue substituir corretamente o tipo base?

### ISP

- Foco: **dependências/interfaces**.
- Pergunta:
  > A classe depende apenas das operações que realmente precisa?

---

## D — Dependency Inversion Principle (DIP)

**Ideia:** regras importantes do sistema não devem ficar presas diretamente a tecnologias específicas.

### Problema

```text
PedidoService → MySQL
PedidoService → EmailSender
```

Trocar banco ou mecanismo de comunicação exige alterar a regra de negócio.

### Melhor

```text
PedidoService → PedidoRepository ← MySQL
PedidoService → Notificador      ← Email
```

O código principal depende de **abstrações**, não de implementações específicas.

### Exemplo

```java
interface PedidoRepository {
    void salvar(Pedido pedido);
}

interface Notificador {
    void enviar(Pedido pedido);
}
```

Depois podemos ter:

- `MySQLPedidoRepository`;
- `PostgreSQLPedidoRepository`;
- `EmailNotificador`;
- `WhatsAppNotificador`.

---

## DIP x Injeção de Dependência

- **DIP:** princípio de projeto.
- **Injeção de dependência:** técnica para fornecer dependências externamente.
>  DIP diz que a classe deve depender de abstrações. A injeção de dependência fornece uma implementação dessa abstração para a classe.

Exemplo:

```java
PedidoService(
    PedidoRepository repository,
    Notificador notificador
)
```

Usar injeção de dependência **não garante**, sozinho, que o sistema segue DIP.

---

## As 5 perguntas do SOLID

- **S:** essa classe possui razões independentes para mudar?
- **O:** adicionar uma nova variação exige modificar código estável?
- **L:** qualquer subtipo consegue substituir corretamente o tipo base?
- **I:** algum cliente depende de operações que não utiliza?
- **D:** regras importantes estão presas a detalhes concretos?

---

## Sinais de um design problemático

- Um requisito novo modifica muitas classes.
- Uma classe muda por motivos muito diferentes.
- Um `if/switch` cresce a cada nova variação.
- Subclasses possuem métodos "não suportados".
- Implementações possuem vários métodos vazios.
- Testar regras exige banco, rede ou serviços externos.
- Trocar uma tecnologia afeta regras de negócio.

Esses sinais indicam que o design deve ser analisado, mas não significam automaticamente que está errado.

---

## Cuidado com SOLID em excesso

Abstrações demais podem causar:

- muitas classes pequenas;
- dificuldade para navegar pelo código;
- interfaces desnecessárias;
- maior complexidade.

> Uma abstração deve existir para resolver uma mudança ou variação relevante do sistema.

---

## Resumo para decorar

- **S — Responsibility:** separar motivos diferentes para mudança.
- **O — Extension:** adicionar comportamento sem modificar código estável.
- **L — Substitution:** subclasses devem respeitar o contrato da classe base.
- **I — Interfaces:** depender apenas das operações necessárias.
- **D — Dependency:** regras de negócio dependem de abstrações, não de tecnologias.

## Objetivo final do SOLID

Criar código com:

- responsabilidades claras;
- pontos de extensão explícitos;
- contratos coerentes;
- dependências controladas;
- menor impacto quando o sistema muda.