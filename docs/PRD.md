### PRD: Order Management System (OMS) Webhooks de Notificação de Pedidos

Versão: 0.1 (rascunho)
Data: 2026-11-17
Responsável: Larissa (Tech Lead)

---

### Resumo e contexto

Os clientes B2B da plataforma hoje descobrem mudanças de status dos seus pedidos fazendo polling periódico no GET /orders, o que torna a integração lenta e cara para eles. Esta feature cria um sistema de webhooks outbound: sempre que o status de um pedido muda, a plataforma notifica automaticamente os endpoints cadastrados pelo cliente, em menos de 10 segundos, com payload assinado via HMAC-SHA256. A entrega é assíncrona (padrão outbox no MySQL e worker em processo separado), com retry, fila de falhas (DLQ) e reprocessamento manual por administradores.

Onde essa feature será implantada
- Sistema existente: o Order Management System (order-management-api, em Node.js, TypeScript, Express, Prisma e MySQL), que hoje não tem nenhum mecanismo de notificação externa, eventos ou filas. A feature entra como um novo módulo em src/modules/webhooks, seguindo o padrão de módulos do projeto, mais uma nova entry point de worker (src/worker.ts) rodando como processo separado da API e conectada ao mesmo banco.

---

### Problema e motivação

Problemas priorizados
- Clientes B2B fazem polling contínuo no GET /orders para detectar mudanças de status, o que deixa a integração lenta e cara do lado deles e gera carga desnecessária na nossa API. Impacto: experiência de integração ruim para os clientes. Prioridade: alta.
- Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) fizeram pedido formal de notificação em tempo real, e a Atlas sinalizou que pode migrar para um concorrente se a feature não for entregue até o fim do trimestre. Impacto: risco direto de perda de cliente e receita. Prioridade: alta.
- Disparar a notificação de forma síncrona dentro da transação de mudança de status não é viável: a transação já atualiza orders, insere em order_status_history e decrementa estoque, e uma chamada HTTP para um cliente lento ou fora do ar travaria a mudança de status de outros pedidos. Impacto: degradação do fluxo central de pedidos. Prioridade: alta.

---

### Público-alvo e cenários de uso

Público-alvo
- Clientes B2B que integram seus sistemas à plataforma via API (inicialmente Atlas Comercial, MaxDistribuição e Nova Cargo), representados por usuários autenticados no nosso sistema.
- Administradores internos da plataforma (role ADMIN), responsáveis por reprocessar entregas que falharam definitivamente.

Cenários de uso chave
- Cliente cadastra um endpoint de webhook informando a URL HTTPS e a lista de status que quer receber (ex: só SHIPPED e DELIVERED) e recebe a secret gerada pela plataforma.
- Um pedido do cliente muda de status e, em menos de 10 segundos, o endpoint do cliente recebe um POST assinado com os dados do evento.
- O endpoint do cliente fica fora do ar por algumas horas (ex: manutenção planejada) e a plataforma retenta automaticamente com backoff até conseguir entregar.
- Cliente consulta o histórico das últimas entregas de um webhook para depurar falhas na integração.
- Cliente suspeita de vazamento da secret e solicita rotação, mantendo a secret antiga válida por 24 horas enquanto migra.
- Administrador reprocessa manualmente um evento que esgotou as tentativas e foi para a DLQ.

---

### Objetivos e métricas

| Objetivo                                                               | Métrica                                                         | Meta                      |
| ---------------------------------------------------------------------- | --------------------------------------------------------------- | ------------------------- |
| Notificar o cliente em tempo real sobre mudanças de status de pedido | Latência entre o commit da mudança de status e a entrega bem sucedida do webhook (primeira tentativa) | Abaixo de 10 segundos |
| Cobrir indisponibilidades temporárias do endpoint do cliente sem intervenção manual | Janela de retry automático entre a primeira falha e a última tentativa | Cerca de 15 horas (5 tentativas: 1m, 5m, 30m, 2h, 12h) |
| Não perder notificações de mudança de status | Eventos que não terminam nem entregues nem registrados na DLQ | Zero |

---

### Escopo

Incluso
- CRUD de configuração de webhook por cliente (criar, editar, remover e listar), com filtro de quais status o endpoint quer receber.
- Geração da secret pela plataforma na criação do webhook e endpoint de rotação de secret com grace period de 24 horas.
- Publicação do evento na tabela webhook_outbox dentro da mesma transação de mudança de status em order.service.changeStatus.
- Worker em processo separado (src/worker.ts, script npm run worker) que lê a outbox por polling a cada 2 segundos e envia os webhooks.
- Assinatura HMAC-SHA256 do corpo da requisição e headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id e Content-Type application/json.
- Retry com backoff exponencial (5 tentativas: 1m, 5m, 30m, 2h, 12h) e DLQ persistida na tabela webhook_dead_letter.
- Endpoint admin de replay de eventos da DLQ, restrito à role ADMIN e com log de auditoria.
- Endpoint de histórico de entregas por webhook (últimos 100 envios).

Fora de escopo
- Notificação por email ao cliente quando o webhook falha repetidamente: adiado para uma próxima fase, depois de medir o impacto da feature.
- Dashboard visual para o cliente acompanhar seus webhooks: fora desta entrega, nesta fase existem apenas endpoints; o painel é um projeto separado do time de frontend.
- Rate limiting de envio para o cliente: não será implementado agora, fica como ponto em aberto para observar e decidir depois se virar problema.
- Garantia de ordering global e execução com múltiplos workers em paralelo (particionamento por order_id ou lock pessimista): adiado; nesta fase roda um único worker.
- Arquivamento ou limpeza de eventos já entregues da outbox (ex: após 30 dias): fora do escopo desta feature.
- Webhooks inbound (clientes enviando eventos para a plataforma): não faz parte da feature, o fluxo é apenas da plataforma para o cliente.

---

### Requisitos funcionais

#### FR-001 Cadastrar webhook
O sistema deve permitir que um usuário autenticado cadastre um endpoint de webhook para um cliente, informando a URL e os status de pedido que deseja receber, e devolver a secret gerada pela plataforma.

**Fluxo principal**
- Usuário autenticado envia POST com customer_id (no body ou no path), url e lista de status desejados.
- Sistema valida o payload com schema Zod, incluindo URL obrigatoriamente HTTPS e status restritos aos valores do enum OrderStatus (PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED).
- Sistema confirma que o customer existe e gera uma secret única para o endpoint.
- Sistema persiste a configuração (url, secret, customer_id, status filtrados, estado ativo) com id UUID e a retorna junto com a secret.

**Fluxos alternativos e exceções**
- O customer_id não vem do JWT: o token carrega apenas id, email e role do usuário (ADMIN ou OPERATOR), e não existe vínculo entre usuário e customer no modelo de dados atual.

**Erros previstos**
- WEBHOOK_INVALID_URL: URL ausente, malformada ou com esquema http.
- NOT_FOUND: customer inexistente, reutilizando o NotFoundError('Customer'), como já faz a criação de pedidos.
- VALIDATION_ERROR: demais falhas de schema, tratadas pelo validate middleware e pelo error middleware existentes.

Observação: hoje o validate middleware converte toda falha de schema Zod em VALIDATION_ERROR, então emitir WEBHOOK_INVALID_URL exige uma checagem específica para a URL; o FDD deve definir onde essa checagem fica.

**Prioridade:** alta

---

#### FR-002 Editar webhook
O sistema deve permitir alterar a URL, a lista de status filtrados e o estado ativo de um webhook existente.

**Fluxo principal**
- Usuário autenticado envia PATCH para o webhook com os campos a alterar.
- Sistema valida os campos com schema Zod, incluindo a regra de URL HTTPS.
- Sistema persiste as alterações e retorna a configuração atualizada.

**Fluxos alternativos e exceções**
- Desativar o webhook (estado ativo falso) faz com que novos eventos deixem de ser inseridos na outbox para ele.

**Erros previstos**
- WEBHOOK_NOT_FOUND: webhook inexistente.
- WEBHOOK_INVALID_URL: nova URL inválida ou não HTTPS.

**Prioridade:** alta

---

#### FR-003 Remover webhook
O sistema deve permitir remover a configuração de um webhook de um cliente.

**Fluxo principal**
- Usuário autenticado envia DELETE para o webhook.
- Sistema remove a configuração e deixa de gerar eventos para esse endpoint.

**Fluxos alternativos e exceções**
- Nenhum fluxo alternativo foi definido na reunião.

**Erros previstos**
- WEBHOOK_NOT_FOUND: webhook inexistente.

**Prioridade:** alta

---

#### FR-004 Listar webhooks do cliente
O sistema deve listar todas as configurações de webhook de um customer.

**Fluxo principal**
- Usuário autenticado envia GET informando o customer.
- Sistema retorna a lista de webhooks com url, status filtrados e estado ativo, no formato paginado padrão da API (data e pagination).

**Fluxos alternativos e exceções**
- Customer sem webhooks cadastrados: retorna lista vazia.

**Erros previstos**
- NOT_FOUND: customer inexistente, reutilizando o NotFoundError('Customer').

**Prioridade:** alta

---

#### FR-005 Publicar evento de mudança de status na outbox
Toda mudança de status de pedido deve registrar o evento de webhook na tabela webhook_outbox dentro da mesma transação SQL da mudança de status.

**Fluxo principal**
- order.service.changeStatus executa a transação atual (update em orders, insert em order_status_history, atualização de estoque).
- Dentro da mesma transação, chama publishWebhookEvent(tx, order, fromStatus, toStatus) recebendo o tx client atual.
- A função identifica os webhooks ativos do customer que filtram o novo status.
- Para cada webhook interessado, insere uma linha na outbox com id UUID, status pendente e o payload já renderizado (snapshot do pedido no momento da mudança), incluindo event_id UUID.

**Fluxos alternativos e exceções**
- Nenhum webhook do customer quer aquele status: nenhuma linha é inserida na outbox.
- A mudança de status é recusada antes de chegar à outbox (InvalidStatusTransitionError, ConflictError por status repetido ou InsufficientStockError no débito de estoque): a transação sofre rollback e nenhum evento é gerado.
- A criação do pedido (status inicial PENDING) acontece em OrderService.create, fora do changeStatus, e por isso não dispara webhook.
- Payload renderizado acima de 64KB: o evento não é enviado e gera erro, sem truncamento.

**Erros previstos**
- Falha ao inserir na outbox causa rollback de toda a transação de mudança de status; não pode existir status alterado sem evento registrado.
- Payload acima de 64KB gera erro explícito, com código de prefixo WEBHOOK_ a definir no FDD.

**Prioridade:** alta

---

#### FR-006 Processar e entregar eventos pendentes
Um worker em processo separado deve ler os eventos pendentes da outbox e entregá-los aos endpoints dos clientes.

**Fluxo principal**
- Worker (src/worker.ts, com PrismaClient próprio e mesma DATABASE_URL) busca a cada 2 segundos um lote pequeno dos eventos pendentes mais antigos, ordenados por created_at.
- Marca o evento como processando e envia POST para a URL do webhook com o payload JSON.
- Inclui os headers X-Event-Id, X-Signature (HMAC-SHA256 do corpo com a secret do endpoint), X-Timestamp, X-Webhook-Id e Content-Type application/json.
- Resposta de sucesso dentro do timeout de 10 segundos: evento marcado como entregue e a tentativa registrada no histórico de entregas.

**Fluxos alternativos e exceções**
- Cliente não responde em 10 segundos: tratado como falha e encaminhado para retry (FR-007).
- O mesmo evento pode ser entregue mais de uma vez (garantia at-least-once); o cliente deduplica pelo X-Event-Id.
- Ordering garantida apenas por order_id e enquanto houver um único worker.

**Erros previstos**
- Timeout, erro de rede ou resposta de erro do cliente: falha de entrega registrada e retry agendado.

**Prioridade:** alta

---

#### FR-007 Retry com backoff e encaminhamento para DLQ
Entregas que falharem devem ser retentadas com backoff exponencial e, após esgotar as tentativas, movidas para a DLQ.

**Fluxo principal**
- Após uma falha, o worker agenda nova tentativa seguindo a progressão 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas.
- Cada tentativa é registrada no histórico de entregas com resultado, resposta e tempo de resposta.
- Após a 5ª tentativa sem sucesso, o evento é removido do fluxo da outbox e gravado na tabela webhook_dead_letter com payload, motivo da falha e timestamp.

**Fluxos alternativos e exceções**
- Sucesso em qualquer tentativa intermediária: evento marcado como entregue e retries restantes cancelados.

**Erros previstos**
- Falha permanente após 5 tentativas (cerca de 15 horas entre a primeira falha e a última tentativa): evento vai para a DLQ.

**Prioridade:** alta

---

#### FR-008 Reprocessar evento da DLQ
Um administrador deve conseguir reenviar manualmente um evento que está na DLQ.

**Fluxo principal**
- Usuário com role ADMIN envia POST /admin/webhooks/dead-letter/:id/replay.
- Sistema valida a role com o requireRole('ADMIN') existente, no mesmo formato já usado na rota de usuários.
- Sistema recoloca o evento na outbox com status pendente para ser processado pelo worker.
- Sistema registra em log de auditoria quem executou o replay e quando.

**Fluxos alternativos e exceções**
- Usuário autenticado sem role ADMIN: acesso negado.

**Erros previstos**
- WEBHOOK_NOT_FOUND: id inexistente na DLQ.
- FORBIDDEN (HTTP 403): usuário sem role ADMIN, erro já emitido pelo requireRole.
- UNAUTHORIZED (HTTP 401): requisição sem token válido, erro já emitido pelo authenticate.

**Prioridade:** alta

---

#### FR-009 Consultar histórico de entregas
O cliente deve conseguir consultar o histórico das entregas de um webhook para depurar a integração.

**Fluxo principal**
- Usuário autenticado envia GET /webhooks/:id/deliveries.
- Sistema retorna os últimos 100 envios do webhook com resultado (sucesso ou falha), payload enviado, resposta recebida e tempo de resposta.

**Fluxos alternativos e exceções**
- Webhook sem envios: retorna lista vazia.

**Erros previstos**
- WEBHOOK_NOT_FOUND: webhook inexistente.

**Prioridade:** media

---

#### FR-010 Rotacionar secret do webhook
O cliente deve conseguir solicitar uma nova secret pela API, com a secret anterior permanecendo válida por 24 horas.

**Fluxo principal**
- Usuário autenticado solicita a rotação da secret de um webhook.
- Sistema gera uma nova secret única e a retorna.
- Durante 24 horas a secret anterior continua válida em paralelo, para o cliente migrar seus sistemas.
- Após 24 horas a secret anterior é invalidada.

**Fluxos alternativos e exceções**
- Nenhum fluxo alternativo foi definido na reunião; o mecanismo de assinatura durante o grace period fica para o FDD.

**Erros previstos**
- WEBHOOK_NOT_FOUND: webhook inexistente.

**Prioridade:** media

---

### Requisitos não funcionais

Performance
- Latência de entrega abaixo de 10 segundos entre a mudança de status e a primeira tentativa de envio; o polling de 2 segundos define a latência mínima no pior caso.
- Timeout de 10 segundos para a chamada HTTP do worker ao endpoint do cliente.
- A mudança de status do pedido não pode ficar mais lenta por causa do envio do webhook: nenhuma chamada HTTP acontece dentro da transação.
- Worker lê a outbox em lotes pequenos, apoiado em índices nos campos status e created_at.

Disponibilidade
- Worker roda como processo separado da API, para que reinícios da API não interrompam o processamento.
- Indisponibilidade do endpoint do cliente não afeta a API: a falha fica isolada no worker e é absorvida pelo retry.
- Nenhuma meta numérica de disponibilidade (SLA) foi definida na reunião.

Segurança e autorização
- Assinatura HMAC-SHA256 sobre o corpo da requisição, enviada no header X-Signature.
- Secret única por endpoint de webhook, nunca global; rotacionável pelo cliente com grace period de 24 horas.
- TLS obrigatório: somente URLs https são aceitas no cadastro, validado no schema Zod.
- Endpoints de configuração exigem autenticação via JWT (qualquer role autenticada nesta fase).
- Replay de DLQ exige role ADMIN, reaproveitando o requireRole existente.
- Header X-Timestamp enviado para permitir que o cliente detecte replay attack.
- Limite de 64KB para o payload, com erro em caso de excesso.

Observabilidade
- Logs estruturados com o logger Pino já usado no projeto (timestamp ISO e redaction de campos sensíveis), sem introduzir novas bibliotecas.
- Requisições à API de webhooks já recebem requestId e tempo de resposta pelo request logger existente.
- Histórico de cada tentativa de entrega com resultado, resposta e tempo de resposta.
- O projeto não tem hoje biblioteca de métricas nem de tracing; a reunião não definiu nenhuma, e a definição fica para o FDD.

Confiabilidade e integridade de dados
- Inserção na outbox atômica com a mudança de status: se a transação principal sofrer rollback, o evento também é descartado.
- Garantia de entrega at-least-once, com X-Event-Id (UUID gerado na inserção na outbox) para deduplicação no cliente.
- Payload gravado como snapshot no momento da mudança de status, refletindo o estado do pedido naquele instante.
- Ordering garantida apenas por order_id e enquanto houver um único worker (limitação conhecida).
- Nenhum evento é perdido silenciosamente: termina entregue ou registrado na DLQ.

Compatibilidade e portabilidade
- Payload JSON com event_id, event_type (order.status_changed), timestamp ISO 8601, order_id, order_number, from_status, to_status, customer_id e campos básicos do pedido como total_cents, sem os itens do pedido.
- Mesmo banco MySQL e mesma stack (Node.js, Prisma) da API existente; worker com instância própria de PrismaClient.
- Os endpoints citados na reunião (ex: GET /webhooks/:id/deliveries e POST /admin/webhooks/dead-letter/:id/replay) ficam sob o prefixo /api/v1 que a API já usa.
- Identificadores em UUID, seguindo o padrão do restante do projeto.

Compliance
- Log de auditoria de todo replay de DLQ, registrando quem executou e quando.
- DLQ preservada como evidência para debug e reprocessamento, com payload, motivo da falha e timestamp.

Acessibilidade no frontend consumidor
- Não aplicável nesta fase: a feature expõe apenas API, sem interface visual (o dashboard está fora de escopo).

---

### Arquitetura e abordagem

Abordagem
- Padrão outbox no MySQL existente: o evento é gravado na mesma transação da mudança de status e um worker assíncrono, em processo separado, faz a entrega HTTP aos clientes por polling.

Componentes
- Módulo src/modules/webhooks com controller, service, repository, routes e schemas, seguindo o padrão de módulos do projeto.
- Tabela de configuração de webhooks (url, secret, customer_id, status filtrados, estado ativo).
- Tabela webhook_outbox com status (pendente, processando, falhou, entregue), índices em status e created_at e payload renderizado.
- Tabela webhook_dead_letter com payload, motivo da falha e timestamp.
- Registro do histórico de entregas por webhook, consultado pelo endpoint de deliveries.
- Função publishWebhookEvent(tx, order, fromStatus, toStatus), chamada dentro da transação de order.service.changeStatus.
- Worker em src/worker.ts (script npm run worker), com a lógica de processamento em src/modules/webhooks (webhook.worker.ts ou webhook.processor.ts).
- Reuso de AppError, Pino, error middleware centralizado, schemas Zod e requireRole já existentes.

Integrações
- order.service.changeStatus passa a chamar publishWebhookEvent dentro da transação existente.
- Chamadas HTTPS outbound do worker para os endpoints cadastrados pelos clientes B2B.
- Portal do desenvolvedor atualizado pelo time de produto com as instruções de integração, validação de assinatura e deduplicação por X-Event-Id.

### Decisões e trade-offs

#### Decisão: Padrão outbox no MySQL existente em vez de fila dedicada (ex: Redis Streams)
- **Justificativa:** garante atomicidade entre a mudança de status e o registro do evento, sem subir infraestrutura nova para um time pequeno.
- **Trade-off:** a entrega depende de polling da tabela e a outbox cresce no banco principal, exigindo índices e arquivamento futuro.

#### Decisão: Worker em processo separado com polling de 2 segundos em vez de trigger de banco
- **Justificativa:** o MySQL não tem mecanismo nativo de notificação a processos externos como o LISTEN/NOTIFY do Postgres, e 2 segundos atende a meta de menos de 10 segundos. Em processo separado, reinícios da API não derrubam o worker.
- **Trade-off:** latência mínima de até 2 segundos e um processo adicional para operar e monitorar.

#### Decisão: Worker único, com ordering garantida apenas por order_id
- **Justificativa:** evita a complexidade de particionamento ou lock pessimista agora; os clientes não pediram ordering global.
- **Trade-off:** escalar para vários workers exigirá particionar por order_id antes, senão a ordering se perde.

#### Decisão: 5 tentativas com backoff exponencial 1m/5m/30m/2h/12h e DLQ em tabela separada
- **Justificativa:** cobre indisponibilidades longas do cliente (já houve caso real de 2 horas de manutenção); 3 tentativas mataria o evento em cerca de 30 minutos e retry infinito deixaria eventos pendurados para sempre. A DLQ separada mantém a outbox limpa e preserva evidência para debug e replay.
- **Trade-off:** um evento pode levar cerca de 15 horas até ser declarado falho, e a falha definitiva exige ação manual de um admin.

#### Decisão: Garantia at-least-once com X-Event-Id em vez de exactly-once
- **Justificativa:** padrão de mercado (Stripe, GitHub); exactly-once exigiria coordenação entre os dois lados e muito mais complexidade.
- **Trade-off:** o cliente pode receber eventos duplicados e precisa deduplicar pelo X-Event-Id, o que exige boa documentação no portal.

#### Decisão: HMAC-SHA256 com secret por endpoint e rotação com grace period de 24 horas
- **Justificativa:** permite ao cliente validar origem e integridade do payload; secret por endpoint limita o impacto de um vazamento (já houve cliente que vazou secret em log).
- **Trade-off:** mais complexidade de gestão de secrets, com duas secrets válidas em paralelo durante a rotação.

#### Decisão: Payload renderizado como snapshot no momento da inserção na outbox
- **Justificativa:** o evento reflete o estado do pedido no instante da mudança de status, evitando inconsistências se o pedido mudar antes do envio.
- **Trade-off:** mais dados armazenados por evento na outbox.

#### Decisão: Reuso máximo dos padrões existentes do projeto
- **Justificativa:** AppError, Pino, error middleware, módulos em src/modules, schemas Zod, requireRole e códigos de erro com prefixo WEBHOOK_ mantêm a consistência da base de código e evitam novas dependências.
- **Trade-off:** a feature fica limitada às capacidades atuais dessas peças, sem introduzir ferramentas especializadas.

---

### Dependências

#### Organizacional: Revisão de segurança antes do deploy
A engenheira de segurança (Sofia) precisa revisar o código de HMAC e de geração e rotação de secret antes do deploy, com pelo menos dois dias úteis reservados para isso.

#### Organizacional: Documentação no portal do desenvolvedor
O time de produto (Marcos) precisa documentar no portal do desenvolvedor como cadastrar webhooks, validar a assinatura HMAC e deduplicar eventos pelo X-Event-Id, de forma destacada, porque a garantia at-least-once transfere essa responsabilidade ao cliente.

#### Externa: Alinhamento com os clientes B2B
Os clientes Atlas Comercial, MaxDistribuição e Nova Cargo precisam implementar o lado receptor (endpoint HTTPS, validação de assinatura e deduplicação) para que a feature gere valor; o time de produto mantém o alinhamento de expectativas com eles, em especial com a Atlas.

#### Técnica: Integração na transação de order.service.changeStatus
A feature depende de alterar o método changeStatus para chamar publishWebhookEvent com o tx client da transação atual, sem quebrar o fluxo existente de atualização de pedido, histórico e estoque.

#### Técnica: Novo processo de worker
É preciso criar a entry point src/worker.ts e o script npm run worker, com instância própria de PrismaClient apontando para a mesma DATABASE_URL, e garantir que esse processo seja executado separado da API no ambiente de deploy.

---

### Riscos e mitigação

#### Vazamento da secret de um cliente
- **Probabilidade:** media
- **Impacto:** terceiros podem forjar notificações para aquele cliente, comprometendo a autenticidade dos eventos que ele recebe.
- **Mitigação:**
  - Secret única por endpoint, nunca global.
  - Endpoint de rotação de secret com grace period de 24 horas.
  - Revisão de segurança dedicada do HMAC e da geração de secret antes do deploy.
  - Incluir o campo de secret na redaction do logger Pino, que hoje cobre apenas authorization, cookie, password, passwordHash, token e accessToken.
- **Plano de contingência:** cliente rotaciona a secret imediatamente e a antiga expira ao fim do grace period.

#### Indisponibilidade prolongada do endpoint do cliente
- **Probabilidade:** media
- **Impacto:** o cliente deixa de receber atualizações de status durante a janela de indisponibilidade.
- **Mitigação:**
  - Retry com backoff exponencial em 5 tentativas, cobrindo cerca de 15 horas.
  - DLQ persistida com payload e motivo da falha.
- **Plano de contingência:** administrador reprocessa os eventos via endpoint de replay da DLQ, e o cliente pode usar o GET /orders como fallback.

#### Entregas duplicadas causando processamento repetido no cliente
- **Probabilidade:** alta
- **Impacto:** cliente que não deduplica pode processar a mesma mudança de status mais de uma vez.
- **Mitigação:**
  - X-Event-Id único por evento enviado em toda requisição.
  - Documentação destacada no portal do desenvolvedor sobre a garantia at-least-once.
- **Plano de contingência:** suporte orienta o cliente a deduplicar pelo X-Event-Id usando o histórico de entregas para identificar os eventos repetidos.

#### Crescimento da outbox degradando o worker
- **Probabilidade:** baixa
- **Impacto:** aumento da latência de entrega acima da meta de 10 segundos.
- **Mitigação:**
  - Índices em status e created_at e leitura em lotes pequenos.
  - Filtro de status na inserção, evitando linhas para eventos que nenhum webhook quer.
- **Plano de contingência:** implementar arquivamento de eventos entregues e avaliar múltiplos workers particionados por order_id.

#### Perda de ordering ao escalar para múltiplos workers
- **Probabilidade:** baixa
- **Impacto:** cliente recebe eventos do mesmo pedido fora de ordem.
- **Mitigação:**
  - Manter um único worker nesta fase.
  - Registrar a limitação como conhecida e documentá-la.
- **Plano de contingência:** antes de escalar, particionar o processamento por order_id ou usar lock pessimista.

#### Qualquer usuário autenticado consegue gerenciar webhooks de qualquer cliente
- **Probabilidade:** media
- **Impacto:** um operador pode cadastrar, alterar ou remover webhooks de outro customer e ler o histórico de entregas com payloads de pedidos, porque o modelo de dados atual não liga usuário a customer e a autorização do CRUD aceita qualquer role autenticada.
- **Mitigação:**
  - Manter o customer_id explícito em toda operação e validar sua existência.
  - Registrar em log quem executou cada operação de configuração.
- **Plano de contingência:** endurecer a autorização do CRUD numa fase seguinte, como a própria reunião previu.

---

### Critérios de aceitação
Checklist objetivo que define se a feature está pronta.

- Toda mudança de status de um pedido cujo customer tem webhook ativo filtrando aquele status insere um evento na webhook_outbox dentro da mesma transação do changeStatus.
- Se a inserção na outbox falhar, a mudança de status sofre rollback e nada é persistido.
- Mudança de status para um status que nenhum webhook do customer filtra não insere linha na outbox.
- Em condições normais, o endpoint do cliente recebe o webhook em menos de 10 segundos após a mudança de status.
- Toda requisição enviada contém os headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id e Content-Type application/json, e a X-Signature confere com o HMAC-SHA256 do corpo usando a secret do endpoint.
- Cadastro ou edição de webhook com URL http é recusado com WEBHOOK_INVALID_URL.
- Cliente que não responde em 10 segundos tem a entrega registrada como falha e retentada.
- Após 5 tentativas falhas nos intervalos 1m, 5m, 30m, 2h e 12h, o evento é gravado na webhook_dead_letter com payload, motivo da falha e timestamp.
- POST /admin/webhooks/dead-letter/:id/replay recoloca o evento como pendente na outbox apenas para role ADMIN, registrando em log quem executou.
- GET /webhooks/:id/deliveries retorna os últimos 100 envios com resultado, payload, resposta e tempo de resposta.
- Após rotação, a secret anterior continua aceita por 24 horas e deixa de valer depois disso.
- Evento com payload acima de 64KB não é enviado e gera erro.
- Os erros do módulo usam AppError com códigos de prefixo WEBHOOK_ e são tratados pelo error middleware existente.

---

### Testes e validação

Tipos de teste obrigatórios
- Testes de integração dos endpoints de webhook, no mesmo padrão dos testes atuais (Vitest e Supertest contra a API, com MySQL real).
- Testes ponta a ponta da integração com o order.service: changeStatus, inserção na outbox, processamento pelo worker e entrega a um endpoint simulado, incluindo o rollback quando a outbox falha.
- Testes de resiliência com endpoint simulado lento (timeout de 10 segundos) e indisponível, cobrindo retry, DLQ e replay.
- Testes de segurança: rejeição de URL http, validação da assinatura HMAC-SHA256, grace period da rotação de secret e bloqueio do replay para usuários sem role ADMIN.

Estratégia de validação
- Testes automatizados de integração e ponta a ponta, com o setup de testes estendido para limpar as novas tabelas entre os casos, como já é feito com pedidos e clientes.
- Revisão de segurança da Sofia antes do deploy, com pelo menos dois dias úteis, focada em HMAC e na geração e rotação de secret.
