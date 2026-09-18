# Aula 4 — Responsabilidades, Coesão e Acoplamento

### Ideia principal

**Como dividir o sistema em partes com responsabilidades coerentes?**

* Uma classe não deve concentrar responsabilidades que **mudam por motivos diferentes**.
* Ex.: `MatriculaService` não deveria cuidar ao mesmo tempo de regra acadêmica, banco, notificação e auditoria.

### Responsabilidade

É aquilo que uma parte deve:

* **Saber** → dados, estado, regras
* **Decidir** → políticas e validações
* **Fazer** → executar, coordenar ou comunicar

**Pergunta-chave:** quem tem a informação ou autoridade para fazer isso?

### Abstração x Encapsulamento

* **Abstração** → pensar no nível certo.

  * Negócio: "O aluno pode se matricular?"
  * Técnico: "Qual tabela consultar?"
* **Encapsulamento** → esconder os detalhes internos.

```java id="v5a1f8"
// Evitar regra espalhada
if (turma.vagas > 0 && aluno.cr >= 0.7) { ... }

// Melhor
politicaMatricula.avaliar(aluno, turma);
```

### Coesão

**Olha para dentro da classe.**

* **Alta coesão** → responsabilidades relacionadas e propósito claro.
* **Baixa coesão** → classe faz várias coisas diferentes.

**Teste:** consigo explicar a classe em uma frase sem usar **"e também"**?

### Acoplamento

**Olha para fora da classe.**

* Mede quanto uma parte depende dos **detalhes de outra**.
* Alto acoplamento: conhecer SQL, SMTP, JSON externo, classes concretas etc.
* Quanto menos detalhes externos precisar conhecer, **menor o acoplamento**.

### Objetivo

**Alta coesão + baixo acoplamento**

* Caso de uso → **coordena**
* Política → **decide**
* Repositório → **persiste**
* Notificador → **comunica**
* Auditoria → **registra**

### Cuidado com separação excessiva

Mais classes **não significa** melhor design.

Separar quando ajuda a:

* esclarecer uma responsabilidade
* proteger uma mudança
* reduzir uma dependência

### Relação com qualidade

* **Modificabilidade** → baixo acoplamento
* **Testabilidade** → isolamento
* **Desempenho** → evitar indireções desnecessárias
* **Observabilidade** → logs bem definidos

### Checklist

* Classe tem **um propósito claro**? → coesão
* Muda por **um motivo principal**? → responsabilidade
* Conhece detalhes demais de outras partes? → acoplamento
* É fácil testar isoladamente? → bom isolamento
* Mudança pequena afeta muitos lugares? → design frágil

### Ponte para GRASP

GRASP ajuda a decidir:

* **Quem deve saber?**
* **Quem deve criar?**
* **Quem deve coordenar?**

### Para decorar
- **Responsabilidade** → quem deve fazer?
- **Coesão** → olha para **dentro**
- **Acoplamento** → olha para **fora**
- **Meta** → **alta coesão + baixo acoplamento**
