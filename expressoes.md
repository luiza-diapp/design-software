# Expressões — glossário da disciplina

Termos que apareceram nas aulas e que quero ter claros.
*Última atualização: aula 3.*

---

## Redundância

**Manter mais de uma cópia de algo em operação, para que a queda de uma não derrube o sistema.**

Na prática, é rodar várias instâncias da aplicação ao mesmo tempo em vez de uma só. Se uma instância para de responder, as outras continuam atendendo.

- **Qualidade que favorece:** disponibilidade.
- **Custo:** custo financeiro, implantação e operação — mais máquinas para provisionar, monitorar e manter.
- **Costuma vir acompanhada de:** balanceamento de carga, verificação de saúde, logs, alertas e procedimento de recuperação.

**Cenário da aula:** uma instância do servidor de aplicação deixa de responder durante o período de matrícula; o tráfego é redirecionado para instâncias saudáveis e novas solicitações voltam a ser aceitas em até 1 minuto.

> Cuidado: redundância **não** melhora modificabilidade. O código continua o mesmo — ele só passa a rodar em mais lugares.

---

## Balanceamento de carga

**Distribuir as requisições que chegam entre várias instâncias do sistema.**

O termo correto é *balanceamento de carga* (load balancing), não "de código". Quem faz isso é um componente chamado **balanceador**: ele fica na frente das instâncias e decide para qual delas mandar cada requisição.

Serve para duas coisas:

1. **Espalhar o trabalho** — nenhuma instância recebe todo o pico sozinha.
2. **Desviar de instâncias doentes** — junto com a verificação de saúde, o balanceador para de mandar tráfego para quem não está respondendo.

Redundância e balanceamento andam juntos: ter várias instâncias só ajuda se alguém souber distribuir as requisições entre elas.

- **Qualidades que favorece:** disponibilidade e desempenho sob carga.
- **Custo:** mais um componente de infraestrutura para configurar e monitorar.

---

## Termos relacionados que já apareceram

Anotações curtas, para expandir depois se precisar.

- **Verificação de saúde (health check)** — checagem periódica que pergunta a cada instância se ela está funcionando; é o que permite ao balanceador tirar uma instância doente da rotação.
- **Instância** — uma cópia da aplicação em execução (num servidor, contêiner ou máquina virtual).
- **Fila** — estrutura onde as solicitações ficam guardadas para serem processadas depois, em vez de na hora. Absorve picos de carga, mas a confirmação deixa de ser imediata.
- **Cache** — guardar o resultado de uma consulta para responder mais rápido nas próximas. Melhora desempenho, mas exige política de invalidação e cria risco de dados desatualizados.
- **Idempotência** — propriedade de uma operação que pode ser repetida sem gerar efeito duplicado. Importante quando há filas e retentativas: o estudante clicar duas vezes não pode gerar duas matrículas.
- **Consistência eventual** — o dado não fica correto imediatamente em todos os lugares, mas converge depois de um tempo. É a contrapartida comum de processamento assíncrono.

---

## Como continuar

Para cada termo novo, vale registrar:

1. o que significa em uma frase;
2. qual qualidade ele favorece;
3. qual é o custo ou risco que ele introduz;
4. em que aula ou exemplo apareceu.
5. 