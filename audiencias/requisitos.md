## Requisitos – Audiências

Escopo: migração da funcionalidade de Audiências (consulta, preparação, agendamento manual/automático, público, impressão e consultas auxiliares), hoje implementada em servlets/JSP, para endpoints REST. Ignorar detalhes de UI.

### 1) Permissões
- Código base: 116 (Audiências)
- Subpermissões relevantes (extraídas de `audiencias.json`):
  - 1161: Audiências IMPRIMIR (geração de PDF/ODT)
  - 1162: Localizar Audiências (pendentes/para hoje/movimentadas)
  - 1163: LocalizarDWR Audiências (consultar livres ao movimentar)
  - 1164: Nova Audiências (preparar contexto)
  - 1166: Curinga6 Audiências (consultar agendas livres e confirmar agendamento manual)
  - 1167: Curinga7 Audiências (agendamento manual)
  - 1168: Curinga8 Audiências (agendamento automático)
  - 1169: Curinga9 Audiências (agendamento via carta precatória/CEJUSC)
- Menus por tipo (exemplos): Conciliação, Instrução, Una, Todas, CEJUSC, cada qual com ações Filtro/Para Hoje/Pendentes/Movimentadas Hoje (vide `audiencias.json`).

### 2) Regras e validações
- Perfis/grupos e movimentação (controle visual e de permissão):
  - Advogado/MP/assessores desses perfis não podem movimentar; magistrado/conciliador podem.
  - Para movimentar, a `Id_Serventia` da audiência deve coincidir com a do usuário.
- Preparação de contexto (NOVO): limpa `AudienciaDt` e sua lista de `AudienciaProcessoDt`.
- Consulta por fluxo (Localizar):
  - fluxo=0: audiências livres para agendamento manual
  - fluxo=1: para hoje
  - fluxo=2: pendentes
  - fluxo=3: movimentadas hoje
  - fluxo=4: todas (para troca de responsável)
  - default: por filtro
- Agendamento manual (Curinga6 → confirmação; Curinga7 → efetiva):
  - Validar com `validarAudienciaAgendamento` antes de confirmar/persistir
  - Conflito de horário: erro “horário já foi utilizado”
- Agendamento automático (Curinga8):
  - Validação prévia; se não existir agenda disponível, retornar conflito
- Agendamento via carta precatória/CEJUSC (Curinga9):
  - Quando acessado de outra serventia, marcar `acessoOutraServentia` com tipo de pendência
  - Bloqueio por tipo de processo (ex.: cálculo de liquidação de pena → erro “Processo físico!”)
  - Escolha entre manual (redireciona para manual) e automático (tenta alocar)
- Regras de CEJUSC/mediação/preliminar:
  - Se o tipo de audiência for um dos códigos CEJUSC/mediação/preliminar e o usuário não for de serventia preprocessual, é obrigatório remapear `Id_Serventia` para a serventia preprocessual relacionada; se inexistente, erro.
- Consulta pública:
  - Título “AUDIÊNCIAS/SESSÕES DO DIA”; por padrão 1º grau
  - 1º grau: exige informar Comarca e Data; 2º grau: exige Data (Comarca fixa definida no backend)
  - Retorna audiências públicas (1º grau) ou sessões (2º grau)
- Impressão:
  - PDF via `relListagemAudiencias` quando solicitado (passoEditar=1); caso contrário ODT via `listagemAudienciasODT`
- Auditoria: sempre preencher `idUsuarioLog` e `ipComputadorLog` nos DTOs.

### 3) Modelos de dados (DTO)
- Audiência (`AudienciaDt`):
  - Identificação: `id`, `id_AudienciaTipo`, `audienciaTipo`, `audienciaTipoCodigo`
  - Localização: `id_Serventia`, `serventia`, `id_Comarca`, `comarca`, `grauJusticaConsultaPublica`
  - Datas: `dataAgendada`, `dataMovimentacao`, `dataInicialConsulta`, `dataFinalConsulta`, `prazoAgendamentoAudiencia`
  - Flags: `reservada`, `virtual`, `sessaoIniciada`, `acessoOutraServentia`
  - Observações e metadados: `observacoes`, `informacoesGuia`, `id_ArquivoFinalizacaoSessao`
  - Auditoria: `id_UsuarioLog`, `ipComputadorLog`
  - Relações: `listaAudienciaProcessoDt` (contém 1..N processos; no 1º grau geralmente 1)
- AudiênciaProcesso (campos observados em setters):
  - `id`, `id_Audiencia`, `id_AudienciaProcessoStatus`, `audienciaProcessoStatusCodigo`, `audienciaProcessoStatus`
  - `id_ServentiaCargo`, `serventiaCargo`
  - Processo: `processoNumero` e/ou `processoDt` (no físico: `processoNumeroFisico`)
  - Auditoria: `id_UsuarioLog`, `ipComputadorLog`
- Observações específicas:
  - Para CEJUSC/mediação/preliminar: pode haver remapeamento de serventia
  - Consulta pública: usar `grauJusticaConsultaPublica` para alternar entre 1º e 2º grau

### 4) Endpoints REST propostos

4.1 Consultas (lista)
- GET /audiencias
  - Permissão: 1162 (Localizar) ou equivalente por perfil
  - Query: `fluxo` [0|1|2|3|4], `audienciaTipoCodigo`, `idServentia`, `idComarca`, `dataInicial`, `dataFinal`, `reservada`, paginação (`page`, `size`)
  - Ação: delega para o método de consulta conforme `fluxo`
  - Retorno: lista de `Audiencia` com paginação

- GET /audiencias/publicas
  - Permissão: acesso público
  - Query: `grau` [1|2], `idComarca` (obrigatório em grau=1), `data`, paginação
  - Ação: valida parâmetros; retorna audiências (1º grau) ou sessões (2º grau)
  - Retorno: lista pública (campos não sensíveis)

- GET /audiencias/agendas-livres
  - Permissão: 1166 (Curinga6) ou 1168/1169 conforme origem
  - Query: `idProcesso` ou `numeroProcesso`, `idServentia`, `audienciaTipoCodigo|idAudienciaTipo`, `prazoDiasMin`, `dataInicial`, `dataFinal`, paginação
  - Ação: consultar audiências livres para agendamento manual
  - Retorno: slots disponíveis (audiências livres)

4.2 Preparação/Confirmação/Agendamento
- POST /audiencias/agendamentos/manual/preparar
  - Permissão: 1166
  - Body: `{ idProcesso|numeroProcesso, idServentia, audienciaTipoCodigo|idAudienciaTipo, prazoDiasMin?, dataInicial?, dataFinal? }`
  - Ação: validar (`validarAudienciaAgendamento`) e retornar contexto/slots (não persiste)
  - Retorno: 200 com slots ou 400 com mensagem de validação

- POST /audiencias/agendamentos/manual/confirmar
  - Permissão: 1166
  - Body: `{ idAudiencia (slot), idProcesso|numeroProcesso }`
  - Ação: confirmação visual/lógica (pré-efetivação)
  - Retorno: 200 com resumo do agendamento a efetivar

- POST /audiencias/agendamentos/manual
  - Permissão: 1167
  - Body: `{ idAudiencia (slot), idProcesso|numeroProcesso }`
  - Ação: efetivar agendamento manual
  - Retorno: 201 em sucesso; 409 se horário já utilizado; 400 em validação

- POST /audiencias/agendamentos/automatico
  - Permissão: 1168
  - Body: `{ idProcesso|numeroProcesso, idServentia, audienciaTipoCodigo|idAudienciaTipo, prazoDiasMin? }`
  - Ação: tenta alocar automaticamente; usa validação prévia
  - Retorno: 201 com audiência marcada; 409 se não há agenda disponível

- POST /audiencias/agendamentos/externo
  - Permissão: 1169
  - Body: `{ idProcesso, origem: [carta-precatoria|cejusc-...], tipoAgendamento: [manual|automatico], audienciaTipoCodigo|idAudienciaTipo }`
  - Ação: trata acesso de outra serventia e segue para manual/automático
  - Retorno: conforme manual/automático

4.3 Impressão/relatórios
- GET /audiencias/relatorio
  - Permissão: 1161
  - Query: filtros de listagem + `formato` [pdf|odt]
  - Ação: gerar relatório conforme formato
  - Retorno: 200 (arquivo binário)

4.4 Consultas auxiliares (combos/listagens)
- GET /audiencias/tipos
  - Permissão: 1162
  - Query: `q`, paginação
  - Retorno: tipos de audiência

- GET /audiencias/status
  - Permissão: 1162
  - Query: `q`, `serventiaTipoCodigo` (influencia lista), paginação
  - Retorno: status possíveis de audiência de processo

- GET /serventias
  - Permissão: 1162
  - Query: `q`, `dataAgenda?`, `grau` (1º/2º) – obriga `dataAgenda` nos fluxos públicos
  - Retorno: serventias/varas/gabinetes ativas para o fluxo

- GET /comarcas
  - Permissão: 1162
  - Query: `q`, paginação

- GET /serventias/cargos
  - Permissão: 1162
  - Query: `q`, `idServentiaUsuario`, `serventiaSubtipoCodigo`, paginação
  - Retorno: cargos da serventia para agendas de audiência

4.5 Movimentação (Processo físico, atrelado a audiências)
- POST /audiencias/processos/fisicos/movimentacoes/preparar
  - Permissão: 116 (base) por perfil apto a movimentar
  - Body: `{ idAudienciaProcesso, tipoMovimentacao, fluxo? }`
  - Ação: carregar audiência/processo e status possíveis
  - Retorno: 200 com contexto/combos

- POST /audiencias/processos/fisicos/movimentacoes
  - Permissão: 116 (base) por perfil apto a movimentar
  - Body: `{ idAudienciaProcesso, audienciaStatusCodigo, audienciaStatus, acordo?, valorAcordo?, textoEditor? }`
  - Ação: validar e salvar movimentação; limpar contexto
  - Retorno: 200 em sucesso; 400 em validação

4.6 Observações sobre lote
- Sempre que aplicável, aceitar vetores: por exemplo, efetivar múltiplos agendamentos manuais ou excluir/reservar várias agendas (coerente com módulo Agenda).

### 5) Respostas e códigos de status
- 200 OK: consultas, preparação, confirmações e movimentações salvas
- 201 Created: criação de agendamento (manual/automático) bem-sucedida
- 400 Bad Request: validações de negócio/formato (ex.: parâmetros obrigatórios ausentes; horário inválido)
- 403 Forbidden: perfil sem permissão (ex.: advogado/MP tentar movimentar)
- 404 Not Found: audiência/processo/slot inexistente
- 409 Conflict: sem agenda disponível; horário já utilizado; conflito de estado

### 6) Segurança
- Autenticação obrigatória para endpoints não públicos; autorização por grupo/tipo (magistrado, conciliador, etc.)
- Garantir que apenas usuários da mesma `serventia` movimentem audiências dessa serventia
- Dados sensíveis: restringir campos de pessoas/partes no endpoint público; logs/auditoria em todas as operações
- LGPD: evitar expor dados pessoais no endpoint público; anonimizar quando necessário

### 7) Itens a decidir na migração
- Representação única de slots de agenda (ID do slot vs. audiência livre) e bloqueio otimista de concorrência
- Paginação e ordenação padrão das listagens (por data, horário, tipo, serventia)
- Política de time zone e formatação de datas (ISO 8601)
- Contratos dos endpoints auxiliares compartilhados com outros módulos (serventias, comarcas, cargos)
- Padronização das causas de erro para impressão (PDF/ODT)


