1. PRD da feature

Produza o arquivo docs/PRD.md cobrindo a feature de Sistema de Webhooks de Notificação de Pedidos. O PRD deve seguir o formato apresentado no curso e incluir, no mínimo, as seguintes seções:

    Resumo e contexto da feature
    Problema e motivação
    Público-alvo e cenários de uso
    Objetivos e métricas de sucesso
    Escopo (incluso e fora de escopo)
    Requisitos funcionais
    Requisitos não funcionais
    Decisões e trade-offs principais
    Dependências
    Riscos e mitigação
    Critérios de aceitação
    Estratégia de testes e validação

A seção "Fora de escopo" deve listar explicitamente pelo menos 2 itens descartados ou adiados durante a reunião.
2. RFC da feature

Produza o arquivo docs/RFC.md com a proposta técnica da solução, no formato de um documento submetido à equipe para revisão. O RFC opera em nível de arquitetura: apresenta a abordagem escolhida, as alternativas que foram colocadas na mesa e as questões deixadas em aberto. É um documento conciso (2 a 4 páginas); o detalhamento de implementação fica no FDD. Deve seguir o formato apresentado no curso e incluir, no mínimo:

    Metadados (autor, status, data, revisores); use os participantes da reunião como revisores
    Resumo executivo (TL;DR) da proposta
    Contexto e problema
    Proposta técnica (visão geral da solução, sem descer ao detalhe de implementação do FDD)
    Alternativas consideradas (pelo menos 2 alternativas reais discutidas e descartadas na reunião, cada uma com o trade-off que levou ao descarte)
    Questões em aberto (pelo menos 2 pontos levantados na reunião e não decididos ou adiados)
    Impacto e riscos
    Decisões relacionadas (links para os ADRs correspondentes)

O RFC não deve duplicar o detalhamento do FDD. Ele responde "o que propomos e por quê"; o "como construir" em detalhe fica no FDD.
3. FDD da feature

Produza o arquivo docs/FDD.md detalhando o "como implementar" da feature. O FDD é o documento mais técnico e precisa estar acionável o suficiente para um desenvolvedor pegar e começar a codar. Deve seguir o formato apresentado no curso e incluir, no mínimo:

    Contexto e motivação técnica
    Objetivos técnicos
    Escopo e exclusões
    Fluxos detalhados (criação do evento na outbox, processamento pelo worker, retry, DLQ)
    Contratos públicos (endpoints HTTP com payloads de exemplo, headers, status codes, semântica)
    Matriz de erros previstos com códigos no padrão WEBHOOK_*
    Estratégias de resiliência (timeouts, retries, backoff, fallback)
    Observabilidade (métricas, logs, tracing)
    Dependências e compatibilidade
    Critérios de aceite técnicos
    Riscos e mitigação

Seção obrigatória adicional, específica deste desafio: "Integração com o sistema existente". Esta seção deve nomear pelo menos 4 caminhos de arquivo reais do código base e descrever como o módulo de webhooks vai se integrar com cada um (por exemplo, como o método changeStatus será estendido, como as classes de erro existentes serão reutilizadas).
4. ADRs

Produza entre 5 e 8 ADRs em arquivos separados dentro de docs/adrs/, nomeados no formato ADR-NNN-titulo-em-kebab-case.md (ex: ADR-001-outbox-no-mysql.md).

Cada ADR deve seguir o formato MADR (ou variante padrão) com no mínimo as seções: Status, Contexto, Decisão, Alternativas Consideradas (pelo menos 1 alternativa real discutida ou plausível), Consequências (positivas e negativas, com trade-off explícito).

Pelo menos 1 ADR deve referenciar explicitamente arquivos, módulos ou padrões do código existente.

O conjunto de ADRs deve cobrir, no mínimo, 5 das 6 decisões principais discutidas na reunião:

    Padrão Outbox no MySQL
    Política de retry com backoff e DLQ
    Autenticação HMAC-SHA256 com secret por endpoint
    Garantia at-least-once com X-Event-Id
    Worker em processo separado em polling
    Reuso dos padrões existentes do projeto

Decisões técnicas secundárias (formato de payload, timeouts, headers, entre outras) podem virar ADRs adicionais ou ficar apenas no FDD, conforme você considerar mais adequado.
5. Tracker de Rastreabilidade

Produza o arquivo docs/TRACKER.md, uma tabela markdown que mapeia cada item registrado nos seus documentos à origem na transcrição ou no código. O tracker funciona como uma referência cruzada: permite que qualquer leitor entenda de onde veio cada decisão, requisito ou restrição, e garante que a documentação está alinhada com o que foi efetivamente discutido e com o que existe no código.

O tracker não é um conceito padrão do mercado nem é um documento abordado diretamente no curso. É uma exigência específica deste desafio que ajuda a manter a integridade da documentação contra alucinações da IA.

Formato obrigatório da tabela:
ID	Documento	Tipo	Conteúdo (resumo)	Fonte	Localização
 	 	 	 	 	 

Onde:

    ID: identificador único do item (ex: PRD-FR-01, RFC-ALT-02, FDD-CONTRATO-03, ADR-002)
    Documento: arquivo onde o item aparece (docs/PRD.md, docs/RFC.md, docs/FDD.md, docs/adrs/ADR-002-...md)
    Tipo: Requisito Funcional, Requisito Não Funcional, Decisão, Restrição, Trade-off, entre outros
    Conteúdo (resumo): descrição de uma linha do item
    Fonte: TRANSCRICAO ou CODIGO
    Localização: para TRANSCRICAO, timestamp + nome do falante (ex: [09:17] Diego). Para CODIGO, caminho do arquivo (ex: src/modules/orders/order.service.ts).

Cobertura mínima: pelo menos 80% dos itens identificáveis nos seus documentos devem ter linha correspondente no tracker.
6. README com o processo

O README.md na raiz do repositório base contém este enunciado. Substitua o conteúdo dele pela documentação do seu processo de produção. Você pode manter um link ou seção fazendo referência ao enunciado original se quiser, mas o foco do novo conteúdo é descrever sua jornada.

Estrutura obrigatória do novo README:

    Sobre o desafio: 1 a 2 parágrafos descrevendo a tarefa em suas palavras
    Ferramentas de IA utilizadas: lista das ferramentas que você usou, com breve nota sobre o papel de cada uma
    Workflow adotado: como você organizou o trabalho. Em que ordem produziu os documentos, como organizou a interação com a IA
    Prompts customizados: pelo menos 2 prompts relevantes que você escreveu ou adaptou, mostrados em blocos de código
    Iterações e ajustes: descreva os principais momentos em que a IA gerou algo errado ou superficial e você teve que corrigir. Quantas iterações principais até chegar ao resultado final
    Como navegar a entrega: caminho dos arquivos entregues e ordem sugerida de leitura

Critérios de Aceite

A entrega é avaliada contra os critérios abaixo. Todos são obrigatórios.
PRD (docs/PRD.md)

    ☐ Arquivo existe e está em Markdown
    ☐ Contém todas as seções obrigatórias listadas no requisito 1
    ☐ Identifica no mínimo 8 requisitos funcionais discutidos na reunião
    ☐ Inclui pelo menos 1 objetivo com métrica e meta quantitativa
    ☐ Seção "Fora de escopo" lista pelo menos 2 itens explicitamente descartados ou adiados na reunião
    ☐ Seção "Riscos" inclui pelo menos 2 riscos com probabilidade, impacto e mitigação

RFC (docs/RFC.md)

    ☐ Arquivo existe e está em Markdown
    ☐ Contém todas as seções obrigatórias listadas no requisito 2
    ☐ Seção "Alternativas consideradas" lista pelo menos 2 alternativas descartadas na reunião, cada uma com o trade-off que motivou o descarte
    ☐ Seção "Questões em aberto" lista pelo menos 2 pontos adiados ou não decididos na reunião
    ☐ Referencia, com link, pelo menos 2 ADRs do pacote

FDD (docs/FDD.md)

    ☐ Arquivo existe e está em Markdown
    ☐ Contém todas as seções obrigatórias listadas no requisito 3
    ☐ Seção "Contratos públicos" inclui pelo menos 4 endpoints HTTP com payload de exemplo (request e response) e status codes
    ☐ Matriz de erros usa códigos com prefixo WEBHOOK_
    ☐ Seção "Integração com o sistema existente" referencia pelo menos 4 caminhos de arquivo reais do código base
    ☐ Seção "Observabilidade" cita métricas, logs e tracing

ADRs (docs/adrs/ADR-NNN-*.md)

    ☐ Pasta docs/adrs/ contém entre 5 e 8 arquivos no formato ADR-NNN-titulo-em-kebab-case.md
    ☐ Cada ADR contém as seções Status, Contexto, Decisão, Alternativas Consideradas, Consequências
    ☐ O conjunto cobre pelo menos 5 das 6 decisões principais listadas no requisito 4
    ☐ Pelo menos 1 ADR referencia explicitamente arquivos, módulos ou classes do código base

Tracker (docs/TRACKER.md)

    ☐ Arquivo existe e segue o formato de tabela definido no requisito 5
    ☐ Pelo menos 80% dos itens identificáveis dos documentos têm linha correspondente
    ☐ Pelo menos 70% das linhas têm Fonte = TRANSCRICAO com timestamp válido no formato [hh:mm] Nome
    ☐ Pelo menos 5 linhas têm Fonte = CODIGO com caminho de arquivo real

README (README.md)

    ☐ Contém todas as seções obrigatórias listadas no requisito 6
    ☐ Lista pelo menos 1 ferramenta de IA utilizada
    ☐ Mostra pelo menos 2 prompts customizados em blocos de código
    ☐ Descreve pelo menos 2 iterações ou ajustes concretos feitos durante a produção

Consistência geral

    ☐ Nenhum requisito, decisão ou restrição registrada nos documentos contradiz a transcrição ou o código
    ☐ Nenhum arquivo de código mencionado nos documentos é inexistente no repositório
