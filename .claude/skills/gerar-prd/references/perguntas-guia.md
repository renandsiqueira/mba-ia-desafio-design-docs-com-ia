# Perguntas-guia por etapa

Use como base para a entrevista. Faça sempre uma pergunta por vez. Quando um
arquivo de contexto já respondeu a pergunta (ver "Modo rápido" em SKILL.md),
pule-a e vá para a próxima lacuna.

## 1. Contexto e visão geral

- Qual é o produto ou sistema em que essa feature entra.
- Essa feature pertence a um sistema que já existe ou faz parte de um novo
  sistema.
- Quem é o público-alvo.
- Em duas ou três frases, qual é o objetivo de negócio desta feature.

## 2. Problema e oportunidade

- O que está acontecendo hoje que torna essa feature necessária.
- Dê um exemplo real recente com números aproximados (custo, tempo perdido,
  erro operacional, impacto no cliente).
- O que já foi tentado e não funcionou.

## 3. Objetivos e métricas de sucesso

- Que resultado mensurável você quer alcançar.
- Qual métrica representa esse resultado.
- Qual é a meta alvo dessa métrica.

## 4. Escopo

- O que precisa obrigatoriamente estar pronto nessa entrega.
- O que está explicitamente fora de escopo.

## 5. Requisitos funcionais

Para cada requisito funcional:

- Nome do requisito.
- Descreva em uma frase simples o que o sistema tem que fazer.
- Mostre o fluxo principal passo a passo.
- Quais variações comuns e exceções.
- Em que condições devemos bloquear ou retornar erro.
- Qual a prioridade.

## 6. Requisitos não funcionais

- Performance esperada. Exemplo: p95 menor que 150 ms.
- Disponibilidade esperada. Exemplo: 99.9 por cento.
- Segurança e controle de acesso. Exemplo: autenticação, auditoria, permissão
  por papel.
- Observabilidade. Exemplo: logs estruturados e tracing distribuído.
- Confiabilidade. Exemplo: atualização de estoque transacional.
- Compliance, acessibilidade, compatibilidade.

## 7. Arquitetura e abordagem

- Essa feature roda onde. Monólito, microsserviço, agente de IA etc.
- Teremos comunicação síncrona, assíncrona ou ambas.
- Vamos usar fila, mensageria ou streaming.
- Vamos usar cache.
- Quais integrações externas são necessárias.
- Quais componentes principais existem.
- Existem decisões técnicas já dadas. Se sim, quais e por quê. Qual o
  trade-off.

## 8. Decisões e trade-offs

- Que decisões de arquitetura já foram assumidas.
- Por que isso foi decidido.
- Qual o trade-off de cada decisão.

## 9. Dependências

- Existe algo que precisa chegar de outro time ou de outra área (design,
  política comercial, aprovação legal etc).
- Existe algo técnico que precisa estar pronto antes.

## 10. Riscos e mitigação

- Quais são os principais riscos.
- Para cada risco: probabilidade, impacto, mitigação e plano de
  contingência.
- Se houver mais de uma ação de mitigação, liste em subitens.

## 11. Critérios de aceitação

- Liste frases objetivas que definem quando a feature pode ser considerada
  pronta.
- Evite frases vagas como "funciona bem".
- Exemplo de bom critério: "Toda alteração de preço gera auditoria
  persistida com quem alterou, preço anterior e timestamp".

## 12. Testes e validação

- Quais tipos de teste são obrigatórios (unitário, integração, segurança,
  carga etc).
- Qual será a abordagem de validação (TDD, QA manual guiado por roteiro,
  validação exploratória interna).
