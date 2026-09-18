# Aula 1 — Design de Software, Arquitetura e Projeto de Sistemas

## Ideia central

Engenharia de Requisitos pergunta *o que* o sistema deve fazer. Design pergunta **como transformar isso em uma solução estruturada**. Projeto começa quando uma necessidade vira escolhas técnicas avaliáveis.

## Problema percebido não é requisito

- *"O sistema de matrícula sempre trava"* não diz o que é travar, quantos usuários havia, por quanto tempo, qual operação nem qual comportamento seria aceitável.
- Versão analisável: *no primeiro dia de matrícula, com até 15 mil usuários simultâneos, as consultas devem responder em até 2 s e o sistema ficar disponível em 99,9% do período.*
- Cadeia da aula: **relato → cenário → driver → decisão**.

## Quatro níveis

| Nível | Do que trata |
|---|---|
| Projeto / design | Atividade ampla de conceber a solução |
| Arquitetura | Módulos, comunicação, dados, implantação, qualidades sistêmicas |
| Design detalhado | Classes, responsabilidades, interfaces, padrões |
| Implementação | Código, bibliotecas, configuração, banco, testes |

Quanto mais ampla a decisão, maior o custo de mudá-la depois.

## A fronteira depende do impacto

Escolher um banco pode ser decisão local num sistema pequeno e arquitetural numa plataforma com replicação. É arquitetural quando:

- afeta várias partes do sistema;
- influencia atributos de qualidade importantes;
- restringe decisões futuras;
- tem alto custo de alteração;
- exige coordenação entre equipes ou infraestrutura.

**ASR (requisito arquiteturalmente significativo):** aquele que influencia estrutura, comunicação, dados, implantação, tecnologias, riscos ou custo de evolução.

## Drivers arquiteturais

Fatores que orientam e limitam as decisões mais importantes. Três origens:

1. **Funcionalidades críticas** — operações centrais que estruturam o sistema.
2. **Atributos de qualidade** — desempenho, disponibilidade, segurança, modificabilidade, testabilidade.
3. **Restrições** — tecnologia, orçamento, prazo, legislação, infraestrutura, sistemas existentes.

**Requisito × restrição:** requisito é um resultado ou qualidade esperada ("confirmar matrícula em até 3 s"); restrição limita o espaço de soluções ("usar o banco institucional existente").

## Toda decisão tem custo

Processar matrículas de forma assíncrona: ganha absorção de picos, controle de carga e tolerância a falhas; paga com confirmação não imediata, mais complexidade, idempotência e tratamento de duplicatas.

> Pergunta central: o benefício justifica as consequências introduzidas?

Soluções precipitadas ("vamos usar microsserviços", "é só aumentar o servidor") são escolhas tecnológicas soltas sem essa análise.

## ADR — Architecture Decision Record

Documento curto que registra **por que** uma alternativa foi escolhida — o código mostra apenas o que foi implementado.

Estrutura: **título · status · contexto · alternativas · decisão · consequências**.

Uma decisão por registro, guardado junto ao projeto, substituído por novo ADR quando muda. Não é ata de reunião, lista de tecnologias nem justificativa criada depois da implementação.

## Do requisito ao código

Relato → requisito analisável → drivers (desempenho, disponibilidade) → decisão arquitetural (separar leitura de confirmação) → design detalhado (interfaces de consulta, reserva, confirmação) → implementação (índices, logs, testes de carga).

## Questão da aula

**Diferença entre decisão de implementação e arquitetural?** A de implementação tem efeito local e é barata de reverter; a arquitetural afeta várias partes, condiciona atributos de qualidade e decisões futuras, e é cara ou arriscada de mudar depois.