## Aula 14 - Integração entre sistemas: REST e SOAP

### Comunicação síncrona

- O solicitante espera uma resposta para continuar o fluxo.
- Facilita respostas imediatas e identificação de erros.
- Cria acoplamento temporal: depende da disponibilidade do serviço.
- Chamadas encadeadas podem acumular latência e propagar falhas.

### REST

- Estilo arquitetural baseado em recursos e representações.
- Recurso: elemento identificado pela API, como filme ou pagamento.
- Representação: dados enviados sobre o recurso, em JSON, XML ou outro formato.
- Exemplo: `GET /filmes/42` retorna uma representação do filme.

#### Restrições

- **Cliente-servidor:** separação de responsabilidades.
- **Interface uniforme:** interações com semântica comum.
- **Stateless:** cada requisição contém o contexto necessário.
- **Cache:** respostas indicam se podem ser reutilizadas.
- **Sistema em camadas:** permite gateways, proxies e outros intermediários.
- **Code-on-Demand:** envio de código executável ao cliente; opcional.

#### Métodos HTTP

- `GET`: consultar.
- `POST`: criar recurso ou iniciar processamento.
- `PUT`: substituir o estado do recurso.
- `PATCH`: alterar parcialmente.
- `DELETE`: remover.

#### Idempotência

- Repetir a solicitação mantém o mesmo efeito pretendido.
- `GET`, `PUT` e `DELETE` são idempotentes.
- `POST` geralmente não é; `PATCH` depende da operação.
- As respostas podem variar mesmo com efeito idempotente.

#### Cuidados e trade-offs

- Usar HTTP e JSON não garante adesão a REST.
- Stateless permite persistência; dispensa contexto de sessão entre chamadas.
- A API deve representar o domínio, sem simplesmente copiar tabelas.
- Cache reduz latência e carga, mas pode apresentar dados desatualizados.
- REST favorece Web/Mobile e escalabilidade.
- Contratos, falhas e dependências síncronas ainda precisam ser gerenciados.

### SOAP

- Protocolo de mensagens estruturadas em XML.
- Permite operações explícitas, como `autorizar` e `estornar`.
- Pode utilizar HTTP ou outros transportes.

#### Estrutura da mensagem

- **Envelope:** envolve a mensagem.
- **Header:** metadados opcionais, como segurança e endereçamento.
- **Body:** conteúdo principal, como operação e dados.

#### WSDL

- Descreve formalmente o contrato do serviço.
- Define operações, tipos, mensagens, endpoints e bindings.
- Favorece interoperabilidade e governança.
- Mudanças podem exigir coordenação entre consumidores.

#### Ecossistema WS-*

- **WS-Security:** segurança na própria mensagem.
- **WS-ReliableMessaging:** entrega confiável.
- **WS-Addressing:** endereçamento.
- **WS-AtomicTransaction:** coordenação transacional.

#### Trade-offs

- Favorece integrações corporativas, legados e contratos formais.
- Pode exigir mensagens maiores, mais processamento e ferramentas complexas.
- WSDL não elimina decisões de design e versionamento.

### Escolha arquitetural

- **ASRs:** requisitos que influenciam a arquitetura.
- Exemplos: desempenho, segurança, escalabilidade e interoperabilidade.
- **Trade-off:** benefício obtido em troca de um custo ou limitação.

| Necessidade | Tende a favorecer |
|---|---|
| API Web/Mobile e cache | REST |
| Contratos formais entre organizações | SOAP |
| Segurança com padrões WS-* | SOAP |
| Integração com legado SOAP | SOAP |
| Processamento que pode ocorrer depois | Comunicação assíncrona |

- A escolha depende do contexto; não é automática.
- SOA pode utilizar REST, SOAP ou outros mecanismos.
- Microsserviços podem utilizar REST, RPC, eventos e mensagens.
- Comunicação síncrona e assíncrona podem coexistir.

## Atividade — Slide 29

**Cenário:** aplicações Web/Mobile, catálogo, cobrança interna, parceiro legado e relatórios sem urgência.

### 1. Onde REST faria sentido?

- **Web/Mobile acessando o catálogo:** consultas simples e possibilidade de cache.
- **Cobrança interna:** criação e consulta de recursos de pagamento.
- Exemplos: `GET /catalogo`, `POST /pagamentos` e `GET /pagamentos/123`.

### 2. Onde SOAP poderia ser justificável?

- Na integração com o **parceiro legado**, caso ele exponha SOAP.
- Também se houver exigência de contratos formais ou padrões WS-*.
- Ser legado, sozinho, não obriga o uso de SOAP.

### 3. Onde a comunicação assíncrona seria melhor?

- Na **geração de relatórios**, pois o resultado pode ficar pronto depois.
- Uma mensagem solicita o processamento a um consumidor.
- O usuário consulta o resultado ou recebe uma notificação posteriormente.

### 4 e 5. Quais ASRs sustentam as escolhas e quais trade-offs foram aceitos?

| Decisão | ASRs atendidos | Trade-offs aceitos |
|---|---|---|
| REST no catálogo | Desempenho, escalabilidade e interoperabilidade Web/Mobile | Cache pode ficar desatualizado; exige invalidação. |
| REST na cobrança | Interoperabilidade e resposta imediata ao consumidor | Dependência da disponibilidade; exige tratar falhas e evitar cobranças duplicadas. |
| SOAP no parceiro | Compatibilidade, governança e requisitos contratuais | Maior complexidade, mensagens extensas e evolução coordenada. |
| Relatórios assíncronos | Responsividade e absorção de picos de demanda | Resultado posterior; exige acompanhar status, falhas e reprocessamentos. |



### Reveja algumas definições:
- **SOAP:** protocolo para trocar mensagens em XML, com estrutura e regras de processamento padronizadas.
- **Mensagem SOAP:** contém `Envelope`, `Body` e, opcionalmente, `Header`. Falhas podem ser comunicadas por `Fault`.
- **Exemplo:** a loja envia `ConsultarEstoque` com o código do produto e recebe a quantidade disponível.
- **WSDL:** descreve o contrato do serviço: operações, parâmetros, respostas e endereço.
- **Contrato forte não garante bom design:** ainda precisamos decidir quais operações oferecer, suas responsabilidades e seu comportamento.
- **HTTP/HTTPS:** pode transportar mensagens SOAP. **SMTP:** serve para enviar e-mails e também pode transportar SOAP.
- **WS-**: padrões que complementam SOAP, como WS-Security para segurança, WS-Addressing para endereçamento, WS-ReliableMessaging para controle de entrega e WS-AtomicTransaction para transações. Precisam ser implementados pelos participantes.
- **REST é um estilo arquitetural:** define restrições para organizar a comunicação. **SOAP é um protocolo:** padroniza as mensagens e seu processamento. Ambos influenciam a comunicação, mas em níveis diferentes.
- **Respeitar REST:** seguir cliente-servidor, requisições independentes de sessão, cache, interface uniforme e camadas. A interface uniforme inclui recursos identificados, representações, mensagens autodescritivas e links para próximas ações.
- **Escolha entre SOAP e REST:** SOAP pode atender contratos e padrões específicos ou integrar sistemas existentes. REST costuma ser mais simples em aplicações web comuns.
- **SOAP no mesmo sistema:** pode conectar serviços em processos separados. Dentro do mesmo processo, geralmente bastam chamadas de métodos.
- **Monólito:** pode conter vários serviços internos e continuar sendo implantado como uma unidade.
- **Classe:** estrutura de código. **Camada:** organização por responsabilidade. **Serviço:** funcionalidade oferecida, acessível diretamente no código ou pela rede.
- **SOA:** organização arquitetural em serviços; SOAP é uma opção de comunicação entre eles.