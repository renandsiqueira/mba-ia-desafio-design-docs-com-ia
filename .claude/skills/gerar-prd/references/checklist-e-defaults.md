# Checagens de consistência e defaults inteligentes

## Checagens de consistência antes de finalizar

Antes de gerar o PRD final, confirme que:

- Cada objetivo tem métrica e meta alvo.
- Todo requisito funcional tem nome, descrição, fluxo principal e
  prioridade.
- Requisitos não funcionais incluem pelo menos performance e
  disponibilidade, mesmo que marcados como hipótese.
- Fora de escopo não contradiz o que está incluso.
- A arquitetura proposta suporta os requisitos não funcionais declarados.
- Toda decisão técnica relevante tem justificativa e trade-off.
- Cada dependência está clara e específica.
- Cada risco tem probabilidade, impacto, mitigação (podendo ter vários
  subitens) e plano de contingência.
- A checklist de critérios de aceitação está objetiva e verificável.
- Os tipos de teste obrigatórios estão definidos.

Se alguma checagem falhar, volte e pergunte o que falta antes de renderizar
o documento final — não preencha lacunas de requisitos reais sem perguntar.

## Checagens mínimas quando o PRD vem de uma transcrição de reunião

Estas checagens adicionais só valem quando a entrevista partiu de um arquivo
de contexto (`contexto=`) que é uma transcrição de reunião ou notas de
descoberta formais — ou seja, quando plausivelmente existe profundidade
suficiente no material de origem para extrair vários itens distintos. Para
uma entrevista do zero sobre uma feature pequena (sem `contexto=`), não
aplique estes mínimos artificialmente: é melhor um PRD com poucos itens
reais do que um PRD com itens inventados só para bater uma contagem.

Quando aplicável, antes de finalizar confirme que o PRD reúne:

- Pelo menos 8 requisitos funcionais distintos extraídos da transcrição.
- Pelo menos 2 itens na seção "Fora de escopo" explicitamente descartados
  ou adiados durante a reunião (não apenas inferidos).
- Pelo menos 2 riscos na seção "Riscos e mitigação".

Se a transcrição claramente não sustenta esses mínimos (reunião curta,
poucos requisitos discutidos), não invente itens para completar a
contagem. Em vez disso, avise o usuário explicitamente que o mínimo não foi
atingido, releia a transcrição em busca de requisitos/riscos/itens de
escopo que possam ter passado despercebidos, e só então pergunte ao usuário
como prosseguir se ainda faltar.

## Defaults inteligentes

Use defaults apenas se o usuário não souber responder. Marque explicitamente
como hipótese no texto (ex: "(hipótese)").

- Latência p95 de APIs síncronas menor que 150 ms.
- Disponibilidade alvo 99.9% para sistemas voltados ao cliente externo e
  99.5% para sistemas internos.
- Observabilidade mínima: logs estruturados, métricas de erro por endpoint,
  tracing distribuído ponta a ponta.
- Segurança mínima: autenticação, autorização por papel, auditoria de
  alterações sensíveis.
- Atualizações críticas (por exemplo estoque) devem ser transacionais.
