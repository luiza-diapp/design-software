# Aula 2 — Atributos de qualidade

## Funcionar não é suficiente

- Requisito: *"o estudante deve poder solicitar matrícula."*
- O que ele não diz:
  - em quanto tempo?
  - com quantos usuários?
  - com que segurança?
  - e se falhar?
  - como evoluir depois?
- Arquitetura é moldada pelas **qualidades esperadas**, não só pelas funções.

## Atributos de qualidade

- **Como** o sistema se comporta sob certas condições.
- Lista: desempenho · disponibilidade · segurança · confiabilidade · modificabilidade · testabilidade · usabilidade · interoperabilidade · manutenibilidade.

## Do relato ao diagnóstico

| Relato | Atributo | Pergunta técnica |
|---|---|---|
| "Demora muito" | Desempenho | Que tempo é aceitável? |
| "Cai no horário crítico" | Disponibilidade | Que interrupção é tolerável? |
| "Difícil alterar uma regra" | Modificabilidade | Quanto esforço a mudança exige? |
| "Ninguém sabe onde falhou" | Observabilidade | Que dados permitem diagnosticar? |
| "Erram sempre a mesma etapa" | Usabilidade | A interface induz o erro? |

> Qualidade precisa virar **cenário verificável**.

## Cenário

- Seis elementos: **fonte · estímulo · ambiente · artefato · resposta · medida**.
- Desempenho → 95% das consultas em até 2 s com 15 mil usuários.
- Disponibilidade → solicitações voltam a ser aceitas em até 1 min.
- Modificabilidade → nova regra implantada em até 2 dias.
- Segurança → 100% das tentativas não autorizadas bloqueadas e registradas.

## Qualidade puxa estrutura

- Desempenho → cache, índices, consultas otimizadas.
- Disponibilidade → redundância, balanceamento, recuperação.
- Modificabilidade → módulos, interfaces, baixo acoplamento.
- Segurança → autorização, auditoria, proteção de dados.

## O que orienta as decisões - Drivers

- Funcionalidades críticas.
- Atributos de qualidade.
- Restrições.

> Alguns chamam isso de *architectural drivers*; aqui: **fatores que orientam as decisões arquiteturais**.

## Síntese

- Funcional = **o que** faz. Qualidade = **como** se comporta.
- Termos vagos viram cenários verificáveis.
- Qualidade influencia arquitetura e design.
- Toda decisão melhora algo, custa algo e cria riscos novos.

## Questão da aula

> "A decisão de ___ foi escolhida porque prioriza ___, mas pode prejudicar ou exigir ___."