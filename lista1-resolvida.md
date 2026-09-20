# ASRs, GRASP e SOLID — Sistema de Bicicletas no Campus
Luíza Diapp - GRR20221252


## Requisitos:
O sistema deverá permitir que o usuário:
1. consulte as bicicletas disponíveis em cada estação;
2. reserve uma bicicleta por até 10 minutos;
3. retire uma bicicleta previamente reservada;
4. devolva uma bicicleta em qualquer estação;
5. consulte seu histórico de viagens;
6. informe problemas encontrados em uma bicicleta.
Além disso, o sistema deverá:
7. impedir que a mesma bicicleta seja reservada simultaneamente por dois usuários;
1
8. liberar automaticamente uma reserva caso a bicicleta não seja retirada em até 10
minutos;
9. responder às consultas de bicicletas disponíveis em até 2 segundos;
10. suportar pelo menos 3.000 usuários simultâneos nos horários de maior movimento;
11. continuar permitindo devoluções mesmo quando o serviço responsável pelo envio de
notificações estiver indisponível;
12. permitir futuramente diferentes mecanismos de identificação, tais como QR Code, cartão
universitário e NFC;
13. permitir diferentes formas de cobrança no futuro;
14. manter registro das retiradas e devoluções realizadas;
15. permitir que somente usuários com autorização administrativa alterem informações
sobre bicicletas e estações.
## Parte 1 — ASRs

O primeiro passo foi separar o que é funcionalidade comum do que realmente mexe na arquitetura. Consultar bicicleta, fazer retirada, registrar um problema: isso tudo resolve com CRUD e um pouco de regra de negócio, e não muda o desenho do sistema. O que muda é outra coisa.

| #  | Requisito | ASR? | Atributo | Impacto no design |
| -- | --------- | ---- | -------- | ----------------- |
| 7  | Não reservar em duplicidade | Sim | Integridade | Reserva precisa ser atômica |
| 8  | Liberar reserva em 10 min | Sim | Confiabilidade | Precisa existir controle de expiração |
| 9  | Resposta em até 2 s | Sim | Desempenho | Consulta precisa ser rápida |
| 10 | 3.000 usuários simultâneos | Sim | Escalabilidade | Não dá para depender de lock em memória |
| 11 | Devolução sem depender da notificação | Sim | Disponibilidade | Notificação fica fora do caminho crítico |
| 12 | Formas de identificação variáveis | Sim | Modificabilidade | Abstração e polimorfismo |
| 13 | Cobrança variável | Sim | Modificabilidade | Regra de tarifação separada |
| 14 | Registrar retirada/devolução | Parcial | Auditabilidade | Eventos e histórico |
| 15 | Só admin altera dados administrativos | Sim | Segurança | Controle de acesso separado |

Fora da tabela, há ainda a possibilidade de outros campi no futuro. Não é requisito, mas deixar `Campus` no modelo desde o começo custa quase nada agora e evita uma migração bem chata depois.

### Prioridade
7, 11, 15, 10 
7 — impedir reserva simultânea. É o único que afeta a corretude do sistema. Se duas pessoas reservam a mesma bicicleta, o usuário chega na estação e ela não está lá. A solução também define onde o estado da bicicleta pode ficar, o que limita todo o resto do design.

15 — só administradores alteram dados. Vale para todas as telas e APIs, inclusive as futuras. Como diz o que o sistema não pode permitir, precisa ser garantido pela estrutura, não por cuidado do programador. Deixar para depois obriga a mexer em todas as camadas.

11 — devolver mesmo com notificação fora do ar. Define como os serviços conversam. Se a devolução esperar pela notificação, cai junto com ela — e o usuário fica preso à bicicleta e à cobrança. Mudar de síncrono para assíncrono depois altera o contrato entre os módulos.

10 — 3.000 usuários no pico. Fica por último porque cache e réplicas podem ser ajustados depois. O que não dá para adiar é manter os serviços sem estado, senão a escala horizontal fica inviável.

---

## Parte 2 — Reserva concorrente

O problema aqui é uma race condition clássica. Duas pessoas olham a mesma bicicleta:

```text
Ana    -> vê DISPONÍVEL -> reserva
Bruno  -> vê DISPONÍVEL -> reserva
```

Se as duas verificações acontecem antes de qualquer escrita, as duas passam. E o código mais óbvio do mundo não resolve:

```java
if (bicicleta.isDisponivel()) {
    bicicleta.reservar();
}
```
Responda:

**1. Que propriedade o sistema precisa garantir nessa situação?**

Exclusividade e Atomicidade

**2. O trecho de código apresentado é suficiente para garantir que apenas um usuário consiga reservar a bicicleta? Justifique.**

Não. Check-then-act não atômico. 
```
Ana:   isDisponivel() → true
Bruno: isDisponivel() → true     ← ainda ninguém reservou
Ana:   reservar(ana)
Bruno: reservar(bruno)           ← sobrescreve
``` 
Mesmo colocando um bloqueio em memória, ele só protege dentro de uma instância. Se o sistema usa redundância e balanceamento de carga, Ana e Bruno podem cair em instâncias diferentes, cada uma com seu próprio bloqueio — e o problema volta. A garantia precisa estar num ponto compartilhado, tipicamente o banco.

**3. Qual requisito apresentado anteriormente está diretamente relacionado a esse problema?**

Requisito 7 

**4. Qual atributo de qualidade está sendo afetado?**

Integridade
Consistência

**5. Qual componente ou objeto deveria possuir a responsabilidade de decidir se uma bicicleta pode ser reservada?**

Por Information Expert: quem tem a informação necessária. O estado DISPONIVEL pertence à Bicicleta, então a decisão deve ficar junto dele

**6. Que decisão de design poderia ser adotada para evitar duas reservas da mesma bicicleta?**

Uma atualização condicional, deixando o próprio banco decidir quem chegou primeiro:

```sql
UPDATE veiculo
SET status = 'RESERVADO'
WHERE id = 42
  AND status = 'DISPONIVEL';
```
**7. Essa decisão pode produzir algum trade-off relacionado a desempenho ou escalabilidade?Explique.**

Sim. O UPDATE condicional serializa o acesso à linha e obriga a reserva a ir ao banco primário, sem cache nem réplica — o que conflita com os requisitos de desempenho e escala. Mas o escopo é estreito: só concorrem requisições pela mesma bicicleta, e a transação dura milissegundos


## Parte 3 — Distribuição de responsabilidades

Considere inicialmente os seguintes conceitos do domínio:

- `Usuario`
- `Bicicleta`
- `BicicletaEletrica`
- `Estacao`
- `Reserva`
- `Viagem`
- `Pagamento`
- `Notificacao`

Considere também as seguintes responsabilidades:

- consultar bicicletas disponíveis;
- verificar disponibilidade;
- reservar uma bicicleta;
- cancelar uma reserva;
- controlar o tempo de validade da reserva;
- registrar a retirada;
- registrar a devolução;
- registrar uma viagem;
- calcular o valor de utilização;
- identificar um usuário;
- enviar uma confirmação ao usuário.

**1. Distribua essas responsabilidades entre os objetos ou componentes que considerar adequados.**


| Responsabilidade | O que você colocou | Avaliação |
|---|---|---|
| Consultar bicicletas disponíveis | `Estacao` |  |
| Verificar disponibilidade | `Bicicleta` |  |
| Reservar uma bicicleta | `Bicicleta` | Instancia Reserva |
| Cancelar uma reserva | `Reserva` |  |
| Controlar tempo de validade | `Reserva` | |
| Registrar retirada | `Viagem` |  |
| Registrar devolução | `Viagem` |  |
| Registrar uma viagem | `RepositorioViagem` |  |
| Calcular valor de utilização | `CalculoCusto` | ✅ Boa ideia. Eu só usaria um nome como `CalculadoraTarifa`. |
| Identificar usuário | `Usuario` | ⚠️ O usuário pode guardar sua identidade, mas não deveria necessariamente ser responsável pelo mecanismo de identificação. |
| Enviar confirmação | `Notificacao` | ✅ Faz bastante sentido. |

**2. Escolha pelo menos **quatro decisões tomadas pelo grupo** e justifique-as utilizando princípios **GRASP**.**

Considere, quando aplicável:

- **Information Expert**
- **Creator**
- **Controller**
- **Low Coupling**
- **High Cohesion**
- **Polymorphism**
- **Indirection**
- **Protected Variations**

1 - bicicleta information expert verifica disponibilidade
2 - bicicleta inicia um reserva creator
3 - high cohesion calculocusto calcula o que deve e nao faz mais nada
4 - protected variations vao existir varias formas de pagamento - pagamento

## Parte 4 — Quem cria a reserva?
#### 1. Quem deveria ser responsável pela criação de uma `Reserva`?

O `ReservaService` deveria ser responsável por coordenar a criação da `Reserva`.

#### 2. Justificativa

Essa decisão pode ser justificada pelo princípio **Controller**, pois o `ReservaService` recebe a solicitação de reserva e coordena as etapas necessárias, como verificar a bicicleta, criar a reserva e registrá-la.

#### 3. Análise da decisão

* **Information Expert:** a `Bicicleta` continua responsável por informar se está disponível, pois conhece seu próprio estado.
* **Coesão:** o `ReservaService` concentra as operações relacionadas ao processo de reserva.
* **Acoplamento:** evita que `Usuario`, `Bicicleta` ou `Estacao` precisem conhecer detalhes de criação, persistência e coordenação da reserva.

#### 4. Outra solução possível

Uma alternativa seria a própria `Bicicleta` criar a `Reserva`.

O principal **trade-off** é que a `Bicicleta` deixa a lógica mais próxima do domínio e aplica bem **Creator**, enquanto o `ReservaService` facilita a coordenação de várias etapas e reduz responsabilidades das entidades, mas adiciona um componente intermediário ao sistema.

---

## Parte 5 — Identificação

O sistema precisa aceitar formas diferentes de identificação, e o caminho fácil é este aqui:

```java
if (tipo.equals("QRCODE")) {
    ...
} else if (tipo.equals("CARTAO")) {
    ...
} else if (tipo.equals("NFC")) {
    ...
}
```

Funciona até a terceira tecnologia. Depois disso, toda tecnologia nova exige editar esse bloco, e o pior é que esses condicionais tendem a se reproduzir: um dia o mesmo `if` aparece em três lugares.

Strategy resolve o problema:

```java
public interface MetodoIdentificacao {

    TipoCredencial tipoSuportado();

    Optional<Usuario> identificar(Credencial credencial);
}
```

Cada tecnologia vira uma implementação:

```text
MetodoIdentificacao
       |
   +---+---+
   |   |   |
  QR Cartão NFC
```

O `IdentificacaoService` recebe as estratégias e escolhe a certa pelo tipo. Para adicionar biometria depois, basta criar mais uma implementação e registrar — as anteriores não são tocadas.

Os princípios que aparecem aqui são basicamente Polymorphism (cada estratégia sabe se comportar do jeito dela), Protected Variations (o domínio fica protegido de mudança de tecnologia), Low Coupling (QR Code não precisa nem saber que NFC existe), OCP (extensão em vez de alteração) e DIP (o serviço depende da interface, não das implementações). A tecnologia fica na infraestrutura; o domínio não precisa saber como um leitor NFC funciona por dentro.

---

## Parte 6 — Convencionais e elétricas


## Domínio de Bicicletas

#### Requisitos de Atributos

###### Bicicletas Convencionais
Possuem os seguintes atributos:
- `identificador`
- `status`
- `estacaoAtual`

###### Bicicletas Elétricas
Possuem adicionalmente:
- `nivelBateria`
- `autonomia`

> **Regra de Negócio:** Uma bicicleta elétrica não pode ser liberada quando o nível de bateria estiver abaixo de 10%.

---

#### Sugestão de Implementação

Um desenvolvedor sugere a seguinte estrutura em Java:

```java
public class Bicicleta {
    private boolean eletrica;
    private int nivelBateria;

    public boolean podeSerRetirada() {
        if (eletrica) {
            return nivelBateria >= 10;
        }
        return true;
    }
}
``` 
### 1. Avaliação da solução

Funciona, mas tem problemas estruturais:

- Bicicletas convencionais carregam `nivelBateria`, campo que não se aplica a elas. Permite estados inválidos (`eletrica = false` com `nivelBateria = 45`).
- A flag `eletrica` é um tipo disfarçado de booleano — decidir comportamento por `if` de tipo é o que o polimorfismo existe para evitar.
- O `if` tende a se repetir em todo método com comportamento variável (tarifa, manutenção, exibição).

### 2. Problemas ao acrescentar novos tipos

- Cada tipo novo exige **modificar** `Bicicleta`, classe que já funcionava.
- O `if` vira `if/else if/else if`, replicado em vários métodos.
- A classe acumula campos de todos os tipos, e cada instância usa poucos.
- Um `boolean` deixa de bastar; troca-se por `enum` com `switch`, o que só adia o problema.

### 3. Alternativa proposta

Cada tipo responde pela própria regra:

```java
public abstract class Bicicleta {
    protected String identificador;
    protected StatusBicicleta status;
    protected Estacao estacaoAtual;

    public boolean podeSerRetirada() {
        return status == StatusBicicleta.DISPONIVEL;
    }
}

public class BicicletaConvencional extends Bicicleta { }

public class BicicletaEletrica extends Bicicleta {
    private int nivelBateria;
    private int autonomia;

    @Override
    public boolean podeSerRetirada() {
        return super.podeSerRetirada() && nivelBateria >= 10;
    }
}
```

O cliente não conhece o tipo:

```java
if (bicicleta.podeSerRetirada()) { ... }
```

O único `if` de tipo fica no repositório, ao criar o objeto a partir da linha do banco. Adicionar um tipo novo é criar uma classe, sem tocar no código existente.

### 4. Princípio GRASP

**Polimorfismo** — quando o comportamento varia por tipo, use operações polimórficas em vez de condicionais.

**Information Expert** (secundário) — a regra dos 10% depende de `nivelBateria`, então a decisão fica onde o dado está.

### 5. Princípios SOLID

- **Open/Closed** (principal): tipo novo passa a exigir extensão, não modificação.
- **Single Responsibility**: cada classe muda por um motivo só.
- **Liskov**: o cliente usa qualquer subtipo sem saber qual é.
- **Interface Segregation**: justifica não expor `getNivelBateria()` na classe base.

### 6. Herança garante o LSP?

**Não.** Herança é mecanismo da linguagem; LSP é regra sobre como usá-la. O compilador aceita qualquer hierarquia, mas o LSP exige que a subclasse se comporte como a base do ponto de vista de quem a usa.

Violação que compila normalmente:

```java
public class BicicletaEletrica extends Bicicleta {
    @Override
    public void retirar() {
        if (nivelBateria < 10) throw new BateriaInsuficienteException();
        super.retirar();
    }
}
```

O cliente que chamava `retirar()` com segurança agora recebe exceção inesperada e passa a precisar de `instanceof` — o `if` que queríamos eliminar volta.

A solução da questão 3 não viola porque a regra está numa **consulta** (`podeSerRetirada()`), cujo contrato sempre foi "responda se pode ou não". Muda *quando* retorna falso, não o que o método promete.

Regras do LSP: a subclasse não pode exigir mais (pré-condição), entregar menos (pós-condição), nem lançar exceções que a base não previa.

---

## Parte 7 — Novo requisito: patinetes elétricos

### Ponto de partida

Patinete não é bicicleta — colocá-lo como subclasse de `Bicicleta` violaria o LSP. O conceito raiz estava errado: o sistema gerencia **veículos compartilhados**, não bicicletas.

```java
public abstract class Veiculo {
    protected String identificador;
    protected StatusVeiculo status;
    protected Estacao estacaoAtual;

    public boolean podeSerRetirado() {
        return status == StatusVeiculo.DISPONIVEL;
    }
}

public abstract class Bicicleta extends Veiculo { }
public class BicicletaConvencional extends Bicicleta { }
public class BicicletaEletrica extends Bicicleta { ... }

public class PatineteEletrico extends Veiculo {
    private int nivelBateria;
    private int autonomia;

    @Override
    public boolean podeSerRetirado() {
        return super.podeSerRetirado() && nivelBateria >= 20;
    }
}
```

### 1. O que precisa ser modificado

- Hierarquia: introduzir `Veiculo` como raiz, `Bicicleta` vira um ramo
- Repositório/fábrica: um caso novo no mapeamento tipo → classe
- Banco: coluna de tipo na tabela de veículos
- Nomenclatura: `RepositorioBicicletas` → `RepositorioVeiculos`

Tudo isso é renomeação e reorganização. **Nenhuma regra de negócio existente muda.**

### 2. O que permanece inalterado

`ReservaService`, `RetiradaService`, `DevolucaoService`, o mecanismo de exclusão mútua (req. 7), a expiração de reserva (req. 8), `Reserva`, `Viagem`, `Usuario`, `Estacao` e `Notificador`.

A regra de autorização do patinete mora só na classe dele — nenhum serviço precisa saber que ela existe.

### 3. Serão necessários novos condicionais?

Não. Só a fábrica ganha um ramo:

```java
return switch (linha.tipo) {
    case "CONVENCIONAL" -> new BicicletaConvencional(...);
    case "ELETRICA"     -> new BicicletaEletrica(...);
    case "PATINETE"     -> new PatineteEletrico(...);   // única linha nova
};
```

Com a solução original da flag `eletrica`, cada método com comportamento variável precisaria de um ramo novo — e um `boolean` nem comportaria três tipos.

### 4. O design está preparado?

**Parcialmente.**

Está preparado para a variação de **comportamento**: a regra do patinete entra como classe nova, sem editar as existentes.

Não estava preparado para a variação de **conceito**: a raiz escolhida (`Bicicleta`) era concreta demais, e precisou subir para `Veiculo`.

O custo foi baixo — renomeação e um ramo na fábrica, sem tocar em regra de negócio.

### 5. Relação com os princípios

**Protected Variations** — `Veiculo` é a interface estável que protege o resto do sistema. Onde o ponto de variação foi previsto, a mudança custou uma classe; onde não foi, custou uma refatoração.

**Polymorphism** — é o mecanismo que viabiliza a proteção. O cliente chama `podeSerRetirado()` e recebe a resposta certa sem testar tipo.

**Open/Closed** — aberto: `PatineteEletrico` é classe nova. Fechado: os serviços não foram tocados. O fechamento não é total, e isso ilustra a formulação correta do princípio — você escolhe **contra qual eixo de variação** se fechar. Fechamos contra "novos tipos de veículo", não contra "novas categorias".

---

## Parte 8 — Serviço de notificações

### 1. É adequado depender diretamente do serviço de e-mail?

Não. Na sequência proposta, a devolução só termina depois que o e-mail é enviado — então se o servidor cair ou ficar lento, a devolução cai ou fica lenta junto. Uma operação **essencial** (devolver) ficou amarrada a uma **acessória** (avisar), e a disponibilidade da primeira passou a ser o produto das duas.

### 2. ASR relacionado

**Requisito 11** — devoluções continuam funcionando com o serviço de notificações indisponível.

Atributo: **disponibilidade / tolerância a falhas**, com degradação graciosa.

### 3. Tipo de acoplamento

**Acoplamento temporal**: os dois componentes precisam estar disponíveis ao mesmo tempo.

Somado ao **acoplamento de implementação**, se o serviço de devolução instancia diretamente a classe de e-mail.

### 4. Organização proposta

Inverter quem inicia a conversa. Hoje a devolução **chama** o e-mail e espera. Na proposta, ela apenas **anuncia** que algo aconteceu, e quem se interessa reage.

```
Usuário devolve → registra devolução → publica "BicicletaDevolvida"
                                                ↓
                                    (devolução já terminou)
                                                ↓
                             consumidor lê o evento e envia o e-mail
```

Três peças:

- **Serviço de devolução** — encerra a viagem, libera a bicicleta, publica o evento. Não sabe quem escuta.
- **Evento** — registro de fato consumado. É a interface estável entre os dois lados.
- **Consumidor** — roda separado, envia o e-mail, tenta de novo se falhar.

Dois cuidados: publicar o evento numa tabela dentro da mesma transação da devolução (para não sair e-mail de devolução revertida), e retentativa com espera crescente no consumidor.

### 5. A devolução pode terminar antes da notificação?

Sim, e é a intenção. Para o negócio, a devolução está concluída quando a bicicleta está registrada na estação e a viagem encerrada. O e-mail é consequência, não parte da operação. Em troca, a notificação vira eventual — mas o usuário já viu a confirmação na tela.

### 6. Vantagens

| Vantagem | Explicação |
|---|---|
| Disponibilidade | Devolução não depende mais do notificador (req. 11) |
| Latência | A resposta não espera I/O externo (reqs. 9 e 10) |
| Extensibilidade | Novos interessados no evento entram sem tocar na devolução |
| Testabilidade | Testar devolução não exige servidor de e-mail |
| Resiliência | Falha temporária vira retentativa, não erro |

### 7. Custos e novos problemas

- **Infraestrutura**: entra uma fila no sistema, mais um componente para operar e monitorar
- **Entrega duplicada**: filas garantem entrega ao menos uma vez, então o consumidor precisa ser idempotente ou o usuário recebe dois e-mails
- **Depuração**: o fluxo deixa de ser uma pilha de chamadas legível
- **Consistência eventual**: existe uma janela entre devolver e notificar
- **Falha silenciosa**: se o consumidor quebrar, ninguém percebe pela tela

Troca-se uma falha **visível e bloqueante** por uma **invisível e não bloqueante**. Para notificação é a troca certa; para cobrança exigiria garantias mais fortes.

### 8. Relação com os princípios

**Low Coupling** — elimina o acoplamento temporal e o conhecimento ponto a ponto entre devolução e notificador.

**Indirection** — o publicador de eventos é o intermediário; em vez de A chamar B, ambos falam com um terceiro.

**Protected Variations** — o evento é a interface estável. Trocar e-mail por push, ou de provedor, não atravessa a fronteira.

**SRP** — `DevolucaoService` volta a ter um motivo só para mudar: regras de devolução. Notificar é de outra classe.

**DIP** — O domínio do sistema define as regras e interfaces de alto nível, desacoplando-se de detalhes de infraestrutura.

---

## Parte 9 — Avaliação com SOLID

### Inventário do design

**Entidades:** `Veiculo` (abstrata) → `Bicicleta`, `BicicletaEletrica`; `Estacao`; `Usuario`; `Reserva`; `Viagem`.

**Interfaces do domínio:** `RepositorioVeiculos`, `RepositorioViagens`, `PublicadorEventos`, `PoliticaTarifaria`, `MeioDeIdentificacao`.

**Serviços de aplicação:** `ConsultaService`, `ReservaService`, `DevolucaoService`, `AdministracaoService`.

**Infraestrutura:** `RepositorioJdbc`, `PublicadorNaFila`, `ConsumidorNotificacao`, `ExpiradorDeReservas`, middleware de autorização.

### Avaliação

| Princípio | Atende? | Evidência no design | Problema que evita |
|---|---|---|---|
| **SRP** | Sim | Quatro serviços, um por grupo de casos de uso; `PoliticaTarifaria` separada de `Viagem`; notificação fora de `DevolucaoService`; persistência nos repositórios | Classe que muda por vários motivos independentes |
| **OCP** | Sim | `podeSerRetirado()` polimórfico; `PoliticaTarifaria` e `MeioDeIdentificacao` como pontos de extensão (reqs. 12 e 13); novos consumidores de evento sem tocar na devolução | Editar código que já funciona a cada variante nova |
| **LSP** | Sim, no que foi especificado | A regra dos 10% fica numa consulta cujo contrato é "responda se pode"; nenhum subtipo lança exceção nova; nenhum cliente usa `instanceof` | Subclasse que surpreende o cliente e força verificação de tipo |
| **ISP** | Parcialmente | `PublicadorEventos` e `PoliticaTarifaria` têm um método cada; `nivelBateria` não exposto em `Veiculo` | Interface inchada com métodos que o cliente não usa |
| **DIP** | Sim | As cinco interfaces pertencem ao domínio; a infraestrutura importa o domínio, nunca o contrário; permitiu trocar e-mail síncrono por fila sem alterar `DevolucaoService` | Regra de negócio presa a banco, framework e rede |

### Ressalvas

**ISP** — `RepositorioVeiculos` é o ponto de risco: se acumular consultas administrativas junto com busca por identificador, o `ConsultaService` passará a depender de métodos que não usa.

**LSP** — a avaliação cobre apenas `podeSerRetirado()`. Devolução e tarifa ainda não foram especificadas por subtipo. **Não é possível avaliar completamente com as informações disponíveis.**

---

## Parte 10 — Proposta final de design

### 1. Elementos principais

**Entidades**

| Elemento | Responsabilidade |
|---|---|
| `Veiculo` (abstrata) | Identificador, status, estação; decide se pode ser retirado |
| `Bicicleta` / `BicicletaEletrica` | Regra própria de liberação (bateria ≥ 10%) |
| `Estacao` | Localização e conjunto de veículos |
| `Usuario` | Identidade e papel (comum ou administrador) |
| `Reserva` | Prazo de validade; sabe se venceu e cancela a si mesma |
| `Viagem` | Origem, destino, horários; encerra-se na devolução |

**Interfaces do domínio**

| Interface | Por que existe |
|---|---|
| `RepositorioVeiculos` / `RepositorioViagens` | Isolar o domínio do banco |
| `PublicadorEventos` | Desacoplar a devolução da notificação (req. 11) |
| `PoliticaTarifaria` | Formas de cobrança variadas (req. 13) |
| `MeioDeIdentificacao` | QR, cartão e NFC (req. 12) |

**Serviços de aplicação**

| Serviço | Requisitos |
|---|---|
| `ConsultaService` | 1, 5, 9 — disponibilidade e histórico |
| `ReservaService` | 2, 3, 7, 8 — reservar, retirar, expirar |
| `DevolucaoService` | 4, 11 — devolver e publicar evento |
| `AdministracaoService` | 6, 14, 15 — cadastro e ocorrências |

### 2. Relações

```
        Controllers (HTTP)
               ↓
      Serviços de aplicação        coordenam casos de uso
               ↓
   Entidades + Interfaces          regras de negócio
               ↑
        Infraestrutura             implementa as interfaces
```

A seta de baixo aponta **para cima**: a infraestrutura depende do domínio, nunca o contrário.

Entre entidades: `Estacao` 1—N `Veiculo`; `Usuario` 1—N `Reserva` e 1—N `Viagem`; `Reserva` 0..1—1 `Veiculo`; `Viagem` referencia veículo, estação de origem e de destino.

### 3. Decisões de design

#### Decisão 1 — Reserva por atualização condicional no banco

- **Decisão:** `UPDATE veiculo SET status='RESERVADO' WHERE id=? AND status='DISPONIVEL'`. Quem alterar a linha vence; zero linhas afetadas significa que perdeu.
- **Motivação:** `if (disponivel) reservar()` é check-then-act não atômico. Lock em memória não resolve, porque com balanceamento cada instância tem o seu.
- **ASR:** requisito 7, e também o 8 (expiração disputa o mesmo recurso).
- **GRASP:** Information Expert — a decisão fica onde o estado está.
- **SOLID:** não se aplica diretamente; é decisão de concorrência.
- **Trade-off:** ganha correção garantida; custa serialização por linha e obriga a reserva a ir ao primário, sem cache. Escopo estreito: só concorrem requisições pelo mesmo veículo.

#### Decisão 2 — Separar caminho de leitura e de escrita

- **Decisão:** consulta lê de cache ou réplica, com consistência eventual; reserva escreve no primário, com consistência forte.
- **Motivação:** os requisitos 9 e 10 pedem cache; o 7 pede consistência forte. Os dois puxam em direções opostas.
- **ASR:** requisitos 9, 10 e 7.
- **GRASP:** Protected Variations.
- **SOLID:** SRP — consultar e reservar mudam por motivos diferentes.
- **Trade-off:** ganha latência e escala; custa mostrar ocasionalmente um veículo já reservado — erro recuperável.

#### Decisão 3 — Devolução publica evento em vez de chamar o notificador

- **Decisão:** `DevolucaoService` encerra a viagem e publica `VeiculoDevolvido`; um consumidor separado envia o e-mail.
- **Motivação:** chamada síncrona cria acoplamento temporal. Se o e-mail falhar, a devolução falha e o usuário fica preso à bicicleta e à cobrança.
- **ASR:** requisito 11.
- **GRASP:** Low Coupling, Indirection, Protected Variations.
- **SOLID:** SRP (devolver ≠ notificar) e DIP (o domínio define `PublicadorEventos`).
- **Trade-off:** ganha disponibilidade e latência; custa uma fila a operar, exige consumidor idempotente e torna a depuração menos linear.

#### Decisão 4 — Hierarquia polimórfica de veículos

- **Decisão:** `Veiculo` abstrata com `podeSerRetirado()`; cada tipo sobrescreve. O único `if` de tipo fica na fábrica do repositório.
- **Motivação:** a flag `eletrica` espalha condicionais, permite estados inválidos e não escala além de dois tipos.
- **ASR:** não é ASR direto — qualidade interna que antecipa a extensão para patinetes.
- **GRASP:** Polymorphism e Information Expert.
- **SOLID:** OCP (tipo novo é classe nova) e LSP (a regra fica numa consulta, não numa ação).
- **Trade-off:** ganha extensibilidade; custa uma classe por tipo. Vale porque a variação já é conhecida.

#### Decisão 5 — Interfaces no domínio, implementações fora

- **Decisão:** as cinco interfaces ficam no pacote do domínio; banco, fila e provedores ficam na infraestrutura.
- **Motivação:** sem isso, a regra de negócio importa framework e banco, e testar exige subir infraestrutura.
- **ASR:** requisitos 11, 12 e 13.
- **GRASP:** Protected Variations e Indirection.
- **SOLID:** DIP (a seta aponta para dentro) e OCP (a interface é o ponto de extensão).
- **Trade-off:** ganha testabilidade e liberdade de trocar tecnologia; custa um arquivo de interface por dependência.

#### Decisão 6 — Autorização centralizada com negação por padrão

- **Decisão:** autenticação no middleware, autorização no serviço de aplicação. Operação sem regra explícita é negada.
- **Motivação:** o requisito 15 é negativo — diz o que o sistema não pode permitir. Cada endpoint novo seria uma chance de esquecer, e o esquecimento é silencioso.
- **ASR:** requisito 15.
- **GRASP:** Protected Variations.
- **SOLID:** SRP — autorização não se mistura com regra de negócio.
- **Trade-off:** ganha garantia estrutural; custa passar contexto de segurança até a camada de aplicação.

### O que ficou de fora, deliberadamente

- **`Pagamento` como entidade** — só quando houver cobrança real; hoje `PoliticaTarifaria` calcula e a `Viagem` guarda o valor
- **Padrão outbox** — entra quando eventos perdidos virarem problema medido
- **Serviços separados para histórico e ocorrência** — absorvidos por `ConsultaService` e `AdministracaoService`
- **CQRS formal** — a separação de leitura e escrita é de infraestrutura, não de modelo

Num projeto real, a fila e a hierarquia de veículos também seriam adiadas. Estão aqui porque os requisitos 11 e 12 as tornam obrigatórias.

---

## Parte 11 — A decisão controversa

**Decisão analisada:** como impedir que a mesma bicicleta seja reservada por dois usuários (req. 7), sem comprometer os 3.000 usuários simultâneos (req. 10).

### 1. Solução escolhida — atualização condicional no banco

Uma única instrução que testa e altera ao mesmo tempo:

```sql
UPDATE veiculo
   SET status = 'RESERVADO', reservado_por = ?, expira_em = ?
 WHERE id = ? AND status = 'DISPONIVEL'
```

Não há verificação prévia — a tentativa **é** a verificação. Quem alterar a linha vence; zero linhas afetadas significa que perdeu.

### 2. Alternativas consideradas

**A — Lock pessimista (`SELECT ... FOR UPDATE`):** abre transação, trava a linha, decide em Java, grava, fecha.

**B — Lock distribuído (Redis):** como o sistema roda em várias instâncias, uma trava de Java não serve — cada servidor teria a sua. O lock ficaria fora, no Redis, com prazo de validade para não travar para sempre se um servidor cair.

### 3. Vantagens e desvantagens

| Opção | Vantagens | Desvantagens |
|---|---|---|
| **Escolhida** | Garantia vem do banco, sem componente novo; trava dura milissegundos e só na escrita; impossível esquecer de proteger; resolve o req. 8 pelo mesmo mecanismo | Obriga a reserva a ir ao primário, sem cache; parte da regra vive no SQL |
| **A — pessimista** | Lógica em Java, mais legível; acomoda operações compostas | Trava aberta durante toda a transação; risco de deadlock; consome conexões do pool, que com 3.000 usuários vira gargalo |
| **B — distribuído** | Funciona com estado espalhado por vários bancos; alivia o primário | Componente crítico a mais (se o Redis cai, a reserva para ou fica insegura); o lock pode expirar com a operação ainda rodando, gerando dois donos; duas fontes de verdade sobre quem tem a bicicleta |

### 4. Por que o grupo escolheu

**Problema:** `if (disponivel) reservar()` é check-then-act não atômico — duas requisições passam na verificação antes de qualquer escrita. Lock em memória não resolve, porque com balanceamento cada instância tem o seu.

**Requisito:** o req. 7 é correção, não qualidade — não admite degradação. Mas os reqs. 9 e 10 impedem uma solução que serialize demais.

**Decisão:** a atualização condicional atende o req. 7 com a **menor serialização possível** — uma linha, por milissegundos, só entre requisições pelo mesmo veículo. Reservas de veículos diferentes não se bloqueiam.

O lock pessimista foi descartado por pagar custo maior (trava longa, deadlock, pressão no pool) para um problema que aqui é simples: uma linha, uma condição. O lock distribuído foi descartado por resolver um problema que não temos — o estado da bicicleta cabe inteiro numa linha de um banco, e o banco já dá essa garantia sozinho.

**Consequências aceitas:**

- A reserva sempre vai ao primário, sem cache. Aceitável porque a operação frequente é a consulta, que fica livre para usar cache.
- Parte da regra vive no SQL. Mitigado encapsulando a instrução no repositório com nome expressivo (`reservarSeDisponivel`).
- O usuário pode receber "não está mais disponível". Erro recuperável — basta escolher outro veículo.

**Gatilho para revisar:** contenção real medida na mesma linha, ou o estado da reserva deixando de caber num único banco. Nenhuma das duas existe hoje.

