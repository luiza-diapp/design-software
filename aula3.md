# Aula 3 — Atributos de qualidade, cenários e trade-offs

## Ideia central

"Funcional", "de qualidade" e "arquitetural" não são a mesma classificação. São **dois eixos independentes**, e todo requisito responde aos dois.

| Pergunta | Respostas |
|---|---|
| Que tipo de necessidade é esta? | funcional · qualidade · restrição |
| Qual o alcance do impacto? | implementação · design detalhado · arquitetura |

- **Funcional** — o que o sistema oferece. *Permitir solicitar matrícula.*
- **Qualidade** — quão bem responde numa situação. *Responder em até 2 s sob carga.*
- **Restrição** — limita as soluções possíveis. *Usar a autenticação institucional.*

## As combinações que quebram a intuição

- **Funcional pode ser arquitetural:** processar pagamentos por vários provedores, com idempotência e tolerando queda de um provedor, afeta integração, falhas, persistência e observabilidade.
- **Qualidade pode ser local:** carregar 50 itens numa tela em 2 s resolve-se com uma consulta; 15 mil usuários simultâneos exige cache, replicação e monitoramento. Mesmo atributo, alcances diferentes.

## Três perguntas para decidir o alcance

1. Quantas partes do sistema serão afetadas?
2. A decisão altera comunicação, dados, implantação ou fronteiras entre módulos?
3. Mudar depois seria caro, arriscado ou exigiria coordenação ampla?

## Cenário de atributo de qualidade

Seis elementos: **fonte · estímulo · ambiente · artefato · resposta · medida**.

Exemplos (sistema de matrícula):

- **Desempenho** — 15 mil usuários no 1º dia; 95% das consultas em até 2 s.
- **Disponibilidade** — instância cai; novas solicitações aceitas em até 1 minuto.
- **Modificabilidade** — nova regra de prioridade sem tocar em interface, autenticação ou persistência; implantada em até 2 dias.
- **Segurança** — tentativa não autorizada negada e registrada; 100% bloqueadas e auditadas.

O cenário existe para permitir comparar alternativas concretas.

## O mesmo requisito em vários níveis

*"Nova regra de prioridade implantada em até 2 dias"*:

- **Arquitetura** — isolar regras em módulo com fronteira clara.
- **Design detalhado** — interfaces, responsabilidades, Strategy ou composição.
- **Implementação** — classes concretas, testes.

## Trade-offs

| Se priorizo... | Posso comprometer... |
|---|---|
| Alta disponibilidade | Consistência imediata, simplicidade |
| Alto desempenho | Segurança, precisão, manutenibilidade |
| Segurança forte | Usabilidade e desempenho |
| Modificabilidade | Desempenho, simplicidade inicial |
| Consistência forte | Disponibilidade, tempo de resposta |
| Observabilidade | Custo, armazenamento, exposição de dados |

**Disponibilidade × consistência imediata**
- Continuar respondendo com algo caído exige várias cópias dos dados.
- Exigir que todas concordem antes de responder → a queda de uma trava tudo.
- Aceitar responder com a cópia viva → o usuário pode ler dado desatualizado.
- Matrícula por fila: fica disponível no pico, mas a confirmação não é imediata (consistência eventual) e surge fila para monitorar.

**Desempenho × segurança, precisão e manutenibilidade**
- Segurança: criptografia, validações e auditoria custam processamento.
- Precisão: cache responde rápido porque responde com dado guardado, possivelmente velho.
- Manutenibilidade: otimizar gera código menos direto — consulta manual, pré-cálculo, atalhos.
  
**Segurança × usabilidade e desempenho**
- Todo controle de acesso é uma etapa a mais entre o usuário e o objetivo.
- Extremo: proteção tão pesada que o usuário a contorna — segurança nominal sobe, real cai.
- Modificabilidade × desempenho e simplicidade inicial
- Modificabilidade se compra com interfaces e pontos de extensão, ou seja, com indireção.
- Cada indireção é camada a mais no caminho crítico e classe a mais para entender.
- O serviço que faz tudo é mais rápido de escrever; o custo aparece na terceira mudança.

**Consistência forte × disponibilidade e tempo de resposta**
- Garantir o mesmo dado para todos exige coordenação: bloqueios, transações, confirmação entre réplicas.
- Enquanto isso acontece, outras operações esperam.
- Impedir que dois alunos peguem a última vaga significa serializar o acesso àquela turma.
  
**Observabilidade × custo, armazenamento e exposição de dados**
- Diagnosticar bem exige registrar muito: ocupa disco, consome banda, custa dinheiro.
- Log detalhado o bastante para explicar a falha do aluno X contém dados do aluno X.
- Cria um novo lugar com informação sensível e controle de acesso mais frouxo que o do banco.
- Não é troca absoluta — é tensão.
 
**Decisão frágil:** *"microsserviços porque é mais moderno."*
**Decisão justificada:** *"separar notificações porque tem volume variável, precisa evoluir independentemente e pode ser assíncrono."*

## Caso guiado: notificações no AVA

Notificações chegam tarde → não é funcionalidade ausente, é qualidade insuficiente.

**Cenário:** ao publicar atividade avaliativa, notificar ≥95% dos matriculados em até 5 minutos, registrando falhas e permitindo reenvio.

| Alternativa | Benefício | Risco |
|---|---|---|
| Envio síncrono | Simples | Falha bloqueia a publicação |
| Fila | Absorve falhas, permite reenvio | Monitoramento, duplicidade |
| Resumo diário | Menos mensagens | Tarde para urgências |

→ **ADR-002:** usar fila, com registro, reenvio e monitoramento.

## Síntese

- Natureza (funcional/qualidade/restrição) ≠ impacto (arquiteturalmente significativo).
- Um requisito funcional pode ser arquiteturalmente significativo.
- Um atributo de qualidade pode gerar decisões arquiteturais e locais.
- O contexto determina alcance e trade-offs; a classificação é hipótese, não rótulo.