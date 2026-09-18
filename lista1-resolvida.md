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
| 4  | Devolver em qualquer estação | Sim | Consistência | A bicicleta não pode pertencer permanentemente a uma estação |
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

A ordem seria 7 → 9 e 10 → 11 → 12 e 13.

O 7 vem primeiro porque é o único que, se falhar, deixa o sistema simplesmente errado. Não há muito o que negociar aí: se duas pessoas conseguem reservar a mesma bicicleta, o resto não importa muito.

Logo depois vêm o 9 e o 10, que entram meio em rota de colisão com o 7. Resolver a consistência colocando lock em tudo leva a escalabilidade junto. Por isso os três precisam ser decididos no mesmo momento, e não um de cada vez.

O 11 é o que mais mexe na estrutura de dependências. Separar devolução de notificação no começo é fácil; descobrir seis meses depois que a operação inteira trava porque o servidor de e-mail caiu é bem menos.

O 12 e o 13 entram por último, mas entram. O próprio enunciado avisa que essas duas coisas vão variar, então não é generalização especulativa — a mudança já está prevista, só não se sabe ainda em que formato.

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

A busca de bicicletas disponíveis pode mostrar uma bike que já foi reservada segundos antes:

```text
10:00:00.000 -> usuário abre o mapa
10:00:00.100 -> outra pessoa reserva a bike
10:00:00.200 -> a primeira pessoa ainda vê a bike disponível
```

Isso não é grave. A consulta existe para mostrar opções rápido; quem decide de verdade é o banco na hora da reserva. Se a pessoa perder a corrida, o sistema responde algo como "essa bicicleta acabou de ser reservada, escolha outra" e a vida segue. Tentar deixar toda a leitura fortemente consistente sairia caro demais para o requisito de 2 segundos.


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
| Consultar bicicletas disponíveis | `Bicicleta` | ⚠️ Melhor seria `Estacao` ou um serviço de consulta, porque a consulta envolve um conjunto de bicicletas. |
| Verificar disponibilidade | `Bicicleta` | ✅ Certo. A bicicleta conhece seu próprio status. |
| Reservar uma bicicleta | `Reserva` | ⚠️ Discutível. A `Reserva` ainda nem existe antes da operação. Melhor `Bicicleta` ou um coordenador de reservas. |
| Cancelar uma reserva | `Reserva` | ✅ Muito bom. |
| Controlar tempo de validade | `Reserva` | ✅ Faz sentido, embora o disparo automático possa ficar em um componente externo. |
| Registrar retirada | `Viagem` | ✅ Faz sentido, principalmente se a retirada inicia a viagem. |
| Registrar devolução | `Viagem` | ✅ Faz sentido, pois encerra a viagem. |
| Registrar uma viagem | `Viagem` | ⚠️ Se "registrar" significar persistir no banco, seria melhor um `RepositorioViagens`. Se significar manter os dados da viagem, `Viagem` está ok. |
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

*1 - ReservaService cria a Reserva — Controller*

O ReservaService recebe a solicitação e coordena as etapas necessárias para realizar a reserva.

*2- Bicicleta verifica disponibilidade — Information Expert*

A própria Bicicleta conhece seu estado e sabe se está disponível ou não.

*3- Notificação é feita por um componente específico — Indirection / Low Coupling*

O domínio não precisa conhecer diretamente como a notificação é enviada, reduzindo o acoplamento.

*4- Cálculo do custo fica em CalculadoraTarifa — High Cohesion / Protected Variations*

A lógica de cálculo fica concentrada em um único componente e pode ser alterada sem afetar o restante do sistema.


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

A tentação inicial é resolver com uma flag:

```java
boolean eletrica;
```

Começa simples e envelhece mal. Bicicleta convencional tem bateria? O que significa `0` — sem carga ou veículo sem bateria? E quando entrar patinete, vira isso:

```java
if (eletrica) {
    ...
}

if (patinete) {
    ...
}
```

Duas flags já dão quatro combinações, e nem todas fazem sentido.

A hierarquia é o caminho melhor:

```java
abstract class Veiculo {

    public boolean podeSerRetirado() {
        return status == DISPONIVEL &&
               condicoesEspecificasAtendidas();
    }

    protected abstract boolean condicoesEspecificasAtendidas();
}
```

A convencional é o caso trivial:

```java
class BicicletaConvencional extends Veiculo {

    protected boolean condicoesEspecificasAtendidas() {
        return true;
    }
}
```

E o elétrico delega a regra para uma política, porque o limite de bateria também é o tipo de coisa que muda:

```java
abstract class VeiculoEletrico extends Veiculo {

    private int nivelBateria;
    private final PoliticaDeBateria politica;

    protected boolean condicoesEspecificasAtendidas() {
        return politica.suficienteParaLiberacao(nivelBateria);
    }
}
```

No fim, quem usa o objeto só chama `veiculo.podeSerRetirado()` e não pergunta se é bicicleta, patinete ou o que for.

Uma ressalva sobre LSP: herança sozinha não garante nada. O compilador aceita `Patinete extends Veiculo` sem reclamar, e isso não quer dizer que o patinete possa ser usado em todo lugar onde se espera um `Veiculo`. A forma prática de verificar isso é escrever testes de contrato para o tipo base e rodar contra todas as subclasses — se alguma quebra, o problema é de modelagem, não de teste.

---

## Parte 7 — Entrada dos patinetes

Com `Veiculo` como abstração, a chegada dos patinetes não deveria obrigar a mexer em `ReservaService`, no mecanismo de concorrência, em `Reserva`, em `Viagem`, na identificação nem na notificação.

O que precisaria ser feito: criar `PatineteEletrico`, definir as políticas específicas dele, ajustar persistência e dar suporte na UI.

Só que aparece um detalhe novo que a abstração não cobre: nem toda vaga serve para qualquer veículo. Surge uma relação do tipo

```text
Vaga <-> Tipo de Veículo
```

e isso provavelmente muda o modelo de estações. Vale reconhecer isso em vez de fingir que o design absorve tudo. Um design bom não significa mudança zero; significa que a mudança fica concentrada nos lugares onde ela faz sentido, e não espalhada por dez classes.

---

## Parte 8 — Notificações

Mandar e-mail dentro da transação de devolução é uma péssima ideia:

```text
Usuário devolve a bicicleta
        |
Sistema registra devolução
        |
Servidor de e-mail demora 5 segundos
```

A operação toda passa a esperar uma chamada de rede que não tem nada a ver com o negócio. E se o e-mail falhar, fica pior ainda: a bicicleta já está fisicamente na estação, mas o sistema responde erro para o usuário. Situação péssima de explicar e pior ainda de corrigir.

A saída é evento com Outbox:

```text
Usuário devolve
       |
Transação:
  - encerra viagem
  - estaciona veículo
  - grava evento na OUTBOX
       |
     COMMIT
       |
Usuário recebe OK
       |
Processo em background -> Fila -> Consumidores (e-mail, push)
```

O ponto é que o caminho crítico termina no commit. A notificação acontece depois, no tempo dela.

Publicar direto na fila dentro da transação seria possível, mas aí a devolução passa a depender da fila estar de pé. Publicar depois do commit também tem furo: se o processo cair entre o commit e a publicação, a notificação simplesmente some. A Outbox fecha essa lacuna porque o evento é gravado na mesma transação do dado.

Isso tem custo, e vale ser honesto sobre ele. Passam a existir tabela de outbox, processo de publicação, fila, consumidores, retentativa, idempotência e monitoramento — tudo isso é código que alguém mantém. Num sistema menor, mandar o e-mail depois do commit com um `try/catch` provavelmente resolveria por um bom tempo. A fila começa a compensar quando aparecem vários consumidores ou uma necessidade real de escala, que é o caso aqui.

---

## Parte 9 — SOLID

**SRP.** Está razoavelmente bem aplicado: a devolução não manda e-mail e a viagem não conhece as regras de preço. O cuidado é com os serviços, que são justamente onde a bagunça costuma se acumular. `ReservaService` é o candidato mais provável a virar classe Deus se ninguém prestar atenção.

**OCP.** É provavelmente o ponto mais forte do desenho. Método de identificação novo é implementação nova mais registro, sem tocar nas que já existem; o mesmo vale para as políticas de tarifação.

**LSP.** Não dá para garantir olhando diagrama. Precisa de teste de comportamento nos subtipos. Se uma subclasse passa a exigir algo que o contrato original não exigia, ela quebra quem depende da abstração — e isso não aparece na compilação.

**ISP.** Interfaces pequenas, e separação entre operações administrativas e operações normais do usuário. Isso também conversa com o requisito 15.

**DIP.** O domínio trabalha com abstrações: repositórios, notificadores, relógio, políticas. A infraestrutura entrega as implementações. Existe uma exceção consciente no mecanismo de concorrência, porque a solução depende de uma operação específica do banco. Nesse caso, assumir o acoplamento é melhor do que inventar uma abstração enorme só para esconder um detalhe que é justamente o que faz o requisito funcionar.

---

## Parte 10 — Arquitetura geral

```text
App Mobile / Web / Admin
           |
          API
           |
Serviços de Aplicação
           |
+------------------------+
|        DOMÍNIO         |
|  Veiculo   Usuario     |
|  Reserva   Viagem      |
+------------------------+
           |
   Interfaces / Portas
      /          \
   Banco       Eventos
                  |
                Fila
                  |
        +---------+---------+
        |         |         |
     E-mail     Push    Auditoria
```

A ideia por trás disso é simples: regra de negócio não deveria depender de banco, framework web ou provedor de e-mail. Essas coisas ficam nas bordas, onde podem ser trocadas sem drama.

As decisões principais, resumindo:

1. Reserva com atualização condicional no banco, para evitar duplicidade sem lock distribuído.
2. Notificação assíncrona, para que problema de e-mail não bloqueie devolução.
3. `Veiculo` como abstração, para evitar explosão de flags e condicionais.
4. Políticas para identificação e cobrança, para que variação entre por extensão.
5. Leitura otimizada separada da escrita: consulta rápida, decisão sempre no banco.
6. `Campus` no modelo desde o início, para não pagar uma migração grande depois.

---

## Parte 11 — A decisão mais controversa

Sem dúvida é usar `ReservaService` com UPDATE condicional por bicicleta.

A alternativa seria tratar a estação como agregado e pegar lock:

```java
Estacao estacao =
    repositorio.buscarComLock(estacaoId);

estacao.reservar(veiculoId, usuarioId, agora);
```

Do ponto de vista de modelagem isso é mais bonito — a regra fica encapsulada dentro da estação, e o domínio ganha uma fronteira clara. O problema é o tamanho dessa fronteira. Se a estação inteira fica bloqueada enquanto alguém reserva uma bicicleta:

```text
Ana   -> bike 42
Bruno -> bike 43
Carla -> bike 44
```

os três disputam o mesmo lock mesmo querendo veículos diferentes. É serializar operações que não têm relação nenhuma entre si. Com CAS por bicicleta, Ana e Bruno só disputam se pedirem a mesma bike.

A desvantagem honesta dessa escolha é que parte da garantia de exclusividade sai do domínio e vai para o banco. Para não deixar isso virar "regra em SQL", as regras de estado continuam nas entidades e o índice único entra apenas como restrição adicional. O serviço coordena, o domínio guarda as regras, e o banco só garante que a concorrência não corrompa o dado.

Se a regra fosse outra — "uma estação pode ter no máximo N reservas simultâneas", por exemplo — aí sim a estação seria exatamente a fronteira certa, e o agregado se justificaria sem discussão. Também faria sentido se a concorrência fosse baixa. Mas com milhares de usuários e disputa acontecendo bicicleta a bicicleta, a atualização condicional por veículo é mais simples e escala melhor.