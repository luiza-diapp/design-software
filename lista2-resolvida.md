## Lista 2

Questões 1 a 11, com enunciados e respostas curtas. As respostas apresentam propostas justificadas; outras escolhas podem ser adequadas ao cenário.

**ASRs:** requisitos arquiteturalmente significativos. **Trade-offs:** benefícios e custos de uma decisão.

---

### 1. Plataforma de streaming de vídeos

Uma plataforma permite que usuários enviem vídeos. Após o upload, cada arquivo deve passar por validação, extração de metadados, transcodificação para diferentes resoluções, geração de miniatura e publicação. O processamento pode levar vários minutos e, em horários de pico, centenas de vídeos podem chegar quase simultaneamente.

**a) Qual estilo arquitetural seria adequado para organizar as etapas de processamento? Justifique.**
**Resposta:** **Pipes and Filters**, pois divide o processamento em etapas independentes, facilitando manutenção e testes.

**b) O processamento deveria ser síncrono ou assíncrono? Explique.**
**Resposta:** **Assíncrono**, porque o processamento demora e o usuário não precisa esperar para continuar usando a plataforma.

**c) Filas de mensagens ajudariam nesse cenário? Qual seria seu papel?**
**Resposta:** Sim. As filas guardam tarefas até os trabalhadores conseguirem processá-las, ajudando a absorver picos de uploads.

**d) Compare a solução escolhida com uma implementação em que um único componente executa todas as etapas.**
**Resposta:** Separar as etapas facilita manutenção e escala individual, mas exige coordenação. Um componente único é mais simples, porém menos flexível.

**e) Indique dois ASRs relevantes para essa decisão.**
**Resposta:** **Escalabilidade:** suportar muitos uploads simultâneos. **Recuperabilidade:** retomar tarefas após falhas.

---

### 2. Plataforma de comércio eletrônico em crescimento

Uma loja virtual possui catálogo, carrinho, pedidos, pagamentos, promoções e logística. O sistema começou pequeno e hoje é utilizado por uma única equipe de desenvolvimento, que realiza aproximadamente duas implantações por semana. O volume de acesso está crescendo, mas ainda não existe necessidade de escalar cada funcionalidade separadamente.

**a) Você escolheria monólito, monólito modular ou microsserviços? Justifique.**
**Resposta:** **Monólito modular**, pois organiza as funcionalidades em módulos sem introduzir a complexidade dos microsserviços.

**b) Quais características do cenário mais influenciam sua decisão?**
**Resposta:** Uma única equipe, duas implantações semanais e nenhuma necessidade atual de escalar funcionalidades separadamente.

**c) Quais custos surgiriam se microsserviços fossem adotados imediatamente?**
**Resposta:** Comunicação pela rede, múltiplas implantações, monitoramento distribuído e maior dificuldade para manter dados consistentes.

**d) Em que situação futura você reconsideraria a arquitetura escolhida?**
**Resposta:** Quando módulos precisarem de escala própria ou equipes precisarem implantar funcionalidades independentemente.

---

### 3. Confirmação de pagamento e atividades secundárias

Após a confirmação de um pagamento, o sistema precisa registrar a transação, gerar um comprovante, enviar uma notificação, atualizar indicadores, alimentar um mecanismo antifraude e registrar informações para auditoria. A indisponibilidade temporária do sistema de indicadores não deve impedir a confirmação do pagamento.

**a) Quais atividades deveriam permanecer no caminho crítico da confirmação?**
**Resposta:** Validar a confirmação e registrar a transação de forma durável. Verificações obrigatórias para autorizar o pagamento também ficam no caminho crítico.

**b) Quais poderiam ser executadas assincronamente?**
**Resposta:** Notificações, indicadores, comprovantes e alimentação posterior do antifraude. O processamento de auditoria pode ser posterior, desde que seu registro essencial esteja garantido.

**c) Uma arquitetura orientada a eventos seria adequada? Justifique.**
**Resposta:** Sim. Um evento `PagamentoConfirmado` permite que outros componentes executem suas tarefas sem bloquear a confirmação.

**d) Explique onde podem surgir consistência eventual e necessidade de idempotência.**
**Resposta:** **Consistência eventual:** indicadores podem atualizar depois. **Idempotência:** um evento repetido não deve duplicar efeitos.

**e) Cite dois trade-offs introduzidos pela solução.**
**Resposta:** Menor espera e maior isolamento de falhas, mas atualizações atrasadas e maior complexidade para acompanhar mensagens e tratar falhas.

---

### 4. Editor de documentos extensível

Uma empresa desenvolve um editor de documentos. O produto básico deve permanecer pequeno e estável, mas clientes podem instalar funcionalidades adicionais, como exportação para novos formatos, correção especializada, integração com serviços externos, recursos de IA e ferramentas de análise. Novas extensões devem poder ser desenvolvidas sem modificar o núcleo da aplicação.

**a) Qual estilo arquitetural é especialmente adequado ao problema?**
**Resposta:** **Microkernel**, pois permite adicionar funcionalidades por plugins sem modificar o núcleo.

**b) O que deveria fazer parte do núcleo e o que deveria ficar fora dele?**
**Resposta:** O núcleo contém edição básica e gerenciamento de extensões. Exportadores, integrações e recursos opcionais ficam nos plugins.

**c) Como o núcleo e as extensões deveriam se comunicar?**
**Resposta:** Por interfaces ou APIs de extensão com contratos bem definidos.

**d) Quais são dois benefícios e dois custos dessa escolha?**
**Resposta:** **Benefícios:** extensibilidade e personalização. **Custos:** manter compatibilidade e controlar segurança e falhas dos plugins.

---

### 5. Sistema de reconhecimento automático de fala

Um sistema recebe áudio e produz uma transcrição. Diferentes componentes especializados contribuem para a solução: análise acústica, reconhecimento de fonemas, modelo linguístico, análise sintática e reconhecimento de nomes próprios. Nenhum componente sozinho consegue determinar a resposta final. Hipóteses produzidas por um componente podem ser utilizadas e refinadas por outros.

**a) Qual estilo arquitetural se encaixa melhor nesse cenário? Justifique.**
**Resposta:** **Blackboard**, pois diferentes componentes colaboram e refinam hipóteses até produzir uma transcrição.

**b) Qual é o papel do estado compartilhado?**
**Resposta:** Armazenar hipóteses e resultados parciais que os componentes consultam e atualizam.

**c) Compare essa solução com Pipes and Filters.**
**Resposta:** Pipes and Filters organiza etapas de transformação. Blackboard permite colaboração e revisão de hipóteses sobre um estado compartilhado.

**d) Por que uma sequência fixa de processamento pode ser inadequada?**
**Resposta:** Porque uma descoberta posterior pode exigir revisar resultados anteriores.

**e) Qual é um risco arquitetural importante da solução escolhida?**
**Resposta:** A dificuldade de coordenar componentes e evitar atualizações conflitantes no estado compartilhado.

---

### 6. Integração de sistemas corporativos

Uma organização possui sistemas independentes de vendas, faturamento, clientes, logística e contratos. Esses sistemas foram construídos em tecnologias diferentes e em momentos distintos. Vários deles precisam acessar capacidades semelhantes, como consultar clientes, calcular preços, emitir documentos e consultar informações financeiras.

**a) SOA seria adequada para esse cenário? Justifique.**
**Resposta:** Sim. **SOA** permite compartilhar capacidades de negócio entre sistemas de tecnologias diferentes.

**b) Que tipos de capacidades poderiam ser disponibilizados como serviços?**
**Resposta:** Consulta de clientes, cálculo de preços, emissão de documentos e consulta financeira.

**c) Qual é a importância dos contratos de serviço?**
**Resposta:** Definem operações, entradas, respostas e erros, permitindo integração sem conhecer a implementação interna.

**d) Compare SOA com uma solução em que cada sistema acessa diretamente o banco de dados de outro sistema.**
**Resposta:** SOA expõe operações controladas. Acesso direto ao banco cria dependência das tabelas e pode contornar regras de negócio.

**e) Cite dois ASRs e dois trade-offs relacionados à decisão.**
**Resposta:** **ASRs:** interoperabilidade e facilidade de evolução. **Trade-offs:** reutilização versus dependência da disponibilidade dos serviços; isolamento das implementações versus manutenção de contratos.

---

### 7. Ferramentas científicas sobre dados compartilhados

Um laboratório possui várias ferramentas que trabalham sobre o mesmo conjunto de dados: importação, limpeza, análise estatística, visualização, geração de relatórios e exportação. Todas precisam acessar uma representação consistente do mesmo conjunto de dados.

**a) Uma arquitetura centralizada em dados seria adequada? Explique.**
**Resposta:** Sim. As ferramentas precisam acessar uma representação comum e consistente dos dados.

**b) Como uma Repository Architecture poderia organizar esse sistema?**
**Resposta:** Um repositório central armazena os dados, enquanto ferramentas de importação, análise e visualização os consultam ou atualizam.

**c) Quais benefícios surgem ao existir uma fonte central de dados?**
**Resposta:** Menos duplicação, maior consistência e centralização de validações e controle de acesso.

**d) Quais problemas podem aparecer quando o esquema de dados precisa mudar?**
**Resposta:** Mudanças no esquema podem exigir adaptações em várias ferramentas.

**e) Compare essa solução com uma arquitetura em que cada ferramenta mantém sua própria cópia dos dados.**
**Resposta:** O repositório compartilhado reduz divergências. Cópias individuais aumentam a autonomia, mas exigem sincronização.

---

### 8. API de catálogo de filmes

Uma empresa pretende disponibilizar seu catálogo para uma aplicação Web, um aplicativo mobile, televisores inteligentes e parceiros externos. Os clientes precisam consultar filmes, categorias, atores e disponibilidade. O número de consumidores pode crescer bastante e várias consultas retornam informações que mudam com pouca frequência.

**a) REST seria adequado como estilo de interação? Justifique.**
**Resposta:** Sim. **REST** permite disponibilizar recursos por uma interface uniforme para diferentes clientes.

**b) Quais elementos poderiam ser tratados como recursos?**
**Resposta:** Filmes, categorias, atores e disponibilidade dos filmes.

**c) Quais características de REST favorecem escalabilidade nesse cenário?**
**Resposta:** Requisições stateless facilitam distribuir o atendimento entre servidores. Cache reduz consultas e processamento repetidos.

**d) Onde cache poderia ajudar e qual trade-off ele introduz?**
**Resposta:** Cache ajuda em informações pouco alteradas, como sinopses e categorias. O custo é poder apresentar dados desatualizados.

**e) Explique por que uma API HTTP que retorna JSON não é necessariamente RESTful.**
**Resposta:** HTTP e JSON são protocolo e formato. REST também exige restrições como interface uniforme, stateless e definição da possibilidade de cache.

---

### 9. Plataforma com diferentes interfaces e tecnologias

Uma plataforma possui cadastro, busca, publicação de conteúdo e administração. Atualmente existe apenas uma aplicação Web, mas estão previstos um aplicativo mobile, uma CLI administrativa e integrações com parceiros. As regras centrais do sistema devem permanecer independentes da interface do usuário, do banco de dados e dos frameworks utilizados.

**a) Hexagonal, Onion ou Clean Architecture poderiam ajudar? Explique.**
**Resposta:** Sim. Essas arquiteturas mantêm as regras centrais independentes das interfaces, do banco e dos frameworks.

**b) Escolha uma das três e justifique a escolha.**
**Resposta:** Escolheria **Hexagonal**, pois portas e adaptadores permitem conectar Web, CLI e integrações ao mesmo núcleo.

**c) O que deveria permanecer no núcleo?**
**Resposta:** Entidades, regras de negócio, casos de uso e contratos de interação.

**d) Que tipos de elementos deveriam ficar nas regiões externas?**
**Resposta:** Controladores, interfaces de usuário, acesso ao banco, integrações e detalhes dos frameworks.

**e) Qual complexidade adicional essa decisão introduz?**
**Resposta:** Mais interfaces, adaptadores e conversões de dados, aumentando o trabalho inicial.

---

### 10. Plataforma de leitura com alto volume de consultas

Uma plataforma recebe milhões de consultas por dia a conteúdos que mudam poucas vezes por hora. Atualmente, cada requisição feita pelo cliente chega ao servidor, que consulta novamente o banco de dados. A equipe estuda introduzir cache, réplicas, múltiplos servidores e CDN.

**a) Quais atributos de qualidade estão motivando essas mudanças?**
**Resposta:** Desempenho, escalabilidade e disponibilidade.

**b) O estilo cliente-servidor precisa ser abandonado? Justifique.**
**Resposta:** Não. Cliente-servidor pode utilizar vários servidores, caches e réplicas.

**c) Quais decisões arquiteturais poderiam melhorar desempenho e escalabilidade?**
**Resposta:** Cache, réplicas de leitura, balanceamento entre servidores e CDN para conteúdo adequado.

**d) Que problemas surgem com cache e replicação?**
**Resposta:** Dados desatualizados, atraso na replicação e dificuldade para invalidar caches.

**e) Explique o trade-off entre consistência, desempenho e disponibilidade nesse cenário.**
**Resposta:** Aceitar algum atraso nas atualizações favorece desempenho e disponibilidade. Exigir dados sempre atualizados pode aumentar a espera e limitar respostas durante falhas.

---

### 11. Escolha arquitetural combinada

Uma nova plataforma digital terá inicialmente cerca de 5.000 usuários, mas pode crescer rapidamente. Ela deverá atender aos seguintes requisitos:

- aplicações Web e mobile;
- disponibilidade de 99,9%;
- algumas operações devem responder em até 500 ms;
- relatórios podem ser gerados em segundo plano;
- falhas no serviço de notificações não podem impedir operações principais;
- novas funcionalidades serão adicionadas com frequência;
- a equipe inicial possui seis desenvolvedores.

Considere, entre outras, as seguintes possibilidades arquiteturais:

- arquitetura em camadas;
- monólito modular;
- microsserviços;
- arquitetura orientada a eventos;
- REST;
- Hexagonal, Onion ou Clean Architecture;
- Microkernel.

Proponha uma arquitetura para o sistema e apresente:

**1. os três ASRs que você considera mais relevantes;**
**Resposta:** Disponibilidade de 99,9%; respostas em até 500 ms nas operações especificadas; falhas nas notificações não bloquearem operações principais.

**2. os estilos/padrões arquiteturais escolhidos;**
**Resposta:** Monólito modular com organização Hexagonal, API REST e eventos/filas para tarefas em segundo plano.

**3. como os principais componentes se comunicariam;**
**Resposta:** Web e mobile acessam a API; módulos interagem por interfaces internas; trabalhadores recebem tarefas pelas filas.

**4. uma alternativa arquitetural que você descartaria;**
**Resposta:** Microsserviços desde o início, pelo custo operacional para uma equipe de seis pessoas sem necessidade comprovada de independência.

**5. os principais benefícios da sua proposta;**
**Resposta:** Organização, implantação simples, regras reutilizáveis e isolamento das tarefas secundárias.

**6. os principais custos ou riscos;**
**Resposta:** Operar filas, evitar efeitos duplicados e manter os limites dos módulos. A disponibilidade também exige redundância e monitoramento.

**7. como a arquitetura poderia evoluir caso o número de usuários aumentasse 100 vezes.**
**Resposta:** Medir gargalos, adicionar instâncias e trabalhadores, otimizar o banco e usar cache. Extrair serviços apenas onde houver necessidade.

---

