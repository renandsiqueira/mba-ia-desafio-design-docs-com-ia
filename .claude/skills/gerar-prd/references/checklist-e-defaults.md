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
