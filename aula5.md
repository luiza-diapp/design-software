# Aula 5 — GRASP: General Responsibility Assignment Software Patterns

**CI1163 · Profª. Simone Dominico · 20/08/2026**

## Pergunta da aula

A aula anterior disse *que* responsabilidades devem ser distribuídas. GRASP responde **quem deve saber e quem deve fazer o quê**.

**Anti-exemplo:** o "Sistema-Deus" — uma classe que agenda, cobra, valida, notifica e integra. Alto acoplamento, baixa coesão, difícil de testar e evoluir.

**Caso de referência:** sistema de telemedicina, fluxo *Agendar Consulta*.

## Responsabilidade: knowing e doing

- **Knowing** — dados privados, dados relacionados, estado interno, regras deriváveis.
- **Doing** — criar objetos, iniciar ações, calcular, validar, coordenar colaborações.

## Os nove padrões, em quatro grupos

| Grupo | Padrões | Papel |
|---|---|---|
| Atribuição | Information Expert, Creator, Controller | **atribuem** a responsabilidade |
| Critérios de qualidade | High Cohesion, Low Coupling | **avaliam** a atribuição |
| Variação e extensão | Polymorphism, Protected Variations | tratam o que **muda** |
| Estruturais | Indirection, Pure Fabrication | resolvem o que a atribuição direta **não resolve** |

---

## Grupo 1 — Atribuindo

### Information Expert
> Quem possui a informação necessária para cumprir esta responsabilidade?

- A responsabilidade fica próxima dos dados que ela usa.
- Reduz "caça a dados" entre objetos e facilita testes.
- *Ex.:* conflito de horário → `AgendaMedico`/`Slot`, não `Paciente`.

### Creator
> Quem deve criar este objeto?

Três critérios: quem **agrega ou contém** · quem **usa intensamente** · quem tem os **dados para inicializar**.

*Ex.:* `AgendaMedico` cria `Consulta` — atua como Creator e Expert ao mesmo tempo.

### Controller
> Quem recebe o evento do sistema?

- Recebe a operação e **coordena**; delega regras de domínio para os Experts.
- **Sinal de problema:** o Controller começa a validar, calcular, criar, persistir e notificar sozinho.
- **Facade Controller** representa o sistema inteiro; **Use-Case Controller** representa um caso de uso. Se o facade cresce demais, separe por casos de uso.

### Sequência de Agendar Consulta

```
Paciente → Controller: agendarConsulta(req)
Controller → AgendaMedico: verificarDisponibilidade(inicio, fim)
AgendaMedico → Controller: disponivel()
Controller → AgendaMedico: agendar(...)
AgendaMedico → Consulta: new Consulta(...)
Controller → Paciente: confirmacao(idConsulta)
[alt] horário indisponível → falhaAgendamento(motivo)
```

---

## Grupo 2 — Avaliando

**Duas alternativas comparadas na aula:**

| A | B |
|---|---|
| Controller verifica, cria, persiste, formata mensagem e envia | Controller coordena · `AgendaMedico` verifica · serviço aplica a regra · componente de notificação envia |

### High Cohesion
> As responsabilidades desta classe pertencem ao mesmo propósito?

Sinais de baixa coesão: classe longa, muitos motivos para mudar, métodos que não se relacionam.

### Low Coupling
> Quantas outras classes precisam ser conhecidas para executar esta responsabilidade?

Evite dependências diretas desnecessárias e objetos que "sabem demais". Prefira colaborações pequenas, explícitas e estáveis.

> **Cuidado com o excesso:** baixo acoplamento não significa interface para tudo; alta coesão não significa uma classe por método. Decisão equilibrada = regras no Expert, Controller leve, dependências justificadas.

---

## Grupo 3 — Quando o comportamento varia

**O problema do if/switch:**

```typescript
switch (meio) {
  case "pix": return pixApi.pagar(valor);
  case "cartao": return gateway.autorizar(valor);
  case "boleto": return emissor.gerar(valor);
}
```

Amanhã entra Wallet. Depois débito. Depois outro gateway.

### Polymorphism
> O comportamento muda de acordo com o tipo?

Interface comum para a variação; cada classe concreta implementa o seu; o cliente conhece o contrato, não os detalhes. *Ex.:* `PaymentMethod` com `Pix`, `Cartao`, `Boleto` — e `PaymentFactory` centralizando a criação.

### Protected Variations
> Qual parte do sistema provavelmente vai mudar?

Identifique pontos de mudança (pagamento, mensageria, agenda externa, autenticação) e proteja com contratos estáveis. Realização comum: **Polymorphism + Indirection**.

---

## Grupo 4 — Quando é preciso inventar classes

**O problema:** quem envia a confirmação da consulta? `Consulta`? `Paciente`? `AgendaMedico`? O controller? **Nenhuma é Expert natural para envio de mensagens.**

### Pure Fabrication
> Não existe classe natural do domínio para essa responsabilidade?

Crie uma classe de software para preservar coesão e reduzir acoplamento — ela não precisa representar um conceito do domínio. *Ex.:* `Repository`, `NotificationService`, `Mapper`, `Logger`.

### Indirection
> Um intermediário reduziria dependências entre partes voláteis?

Camada intermediária quando dois componentes não deveriam se conhecer diretamente.

```typescript
interface NotificationPort { send(msg): void; }
interface NotificationPolicy { chooseChannel(ctx): "email" | "whatsapp"; }
```

- **Porta:** contrato estável · **Adapters:** Email, WhatsApp · **Política:** decide o canal.

### Antes e depois de Agendar + Notificar

| Antes | Depois |
|---|---|
| Controller valida, calcula, cria, persiste e notifica | `AgendamentoController` orquestra |
| `NotificationHandler` decide canal, monta template e chama SDK | `ServicoAgendamento` aplica a regra |
| | `NotificationPolicy` decide canal |
| | `NotificationMessageFactory` monta mensagem |
| | `NotificationPort` contrata o envio |

O serviço publica `AppointmentScheduled` num `EventBus`; o handler reage de forma assíncrona. O paciente recebe `201 Created` sem esperar a notificação.

---

## GRASP como sequência de perguntas

| Pergunta de design | Padrão |
|---|---|
| Quem possui a informação necessária? | Information Expert |
| Quem deve criar este objeto? | Creator |
| Quem recebe o evento do sistema? | Controller |
| As responsabilidades têm o mesmo propósito? | High Cohesion |
| Estamos criando dependências desnecessárias? | Low Coupling |
| O comportamento varia por tipo? | Polymorphism |
| Este ponto tende a mudar? | Protected Variations |
| Um intermediário reduziria dependências? | Indirection |
| Não existe classe natural para assumir isso? | Pure Fabrication |

## Método: do domínio ao design

1. Identifique conceitos do domínio.
2. Liste responsabilidades de conhecer e fazer.
3. Atribua com Expert, Creator e Controller.
4. Avalie com coesão e acoplamento.
5. Proteja pontos de mudança com contratos, polimorfismo e intermediários.

## Estudo de caso

*Cancelar Consulta* ou *Fechar Pedido*: tabela **decisão → padrão → justificativa**, mínimo de 5 decisões, mais uma violação proposital e sua refatoração.

| Decisão | GRASP | Justificativa |
|---|---|---|
| Verificar conflito em `AgendaMedico` | Expert | A agenda possui slots e regras |
| Criar `Consulta` a partir da agenda | Creator | Tem os dados para inicializar |
| Receber `cancelarConsulta()` no controller | Controller | Recebe o evento e delega |
| Isolar envio em porta | Indirection | Domínio não conhece SMTP nem SDKs |
| Escolher canal por política | Protected Variations | O canal tende a mudar |

## Ponte para a próxima aula

GRASP decide responsabilidades. SOLID analisa dependências, abstrações e extensibilidade. A pergunta que atravessa as duas: **quando uma abstração vale a pena?**