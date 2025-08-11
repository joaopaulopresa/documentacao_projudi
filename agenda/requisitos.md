## Requisitos – Agenda de Audiências (Geração e Gestão de Agendas Livres)

Escopo: migrar a funcionalidade de geração e gestão de agendas livres de audiências para endpoints REST, preservando regras de negócio, permissões e validações. Telas/JSP não fazem parte do escopo.

### 1) Permissões
- **Código base**: 222 (Agenda)
- **Subpermissões (conforme agenda.json)**
  - 2224 – Nova Agenda (Novo)
  - 2225 – Salvar Agenda (Salvar)
  - 2220 – Excluir Agenda (Excluir)
  - 2222 – Localizar Agenda (Localizar)
  - 2223 – LocalizarDWR Agenda (LocalizarDWR)
  - 2226 – Curinga6 Agenda (Confirmação de reserva/exclusão de agendas livres)
  - 2227 – Curinga7 Agenda (Reservar agendas livres)
- Observação: o fluxo legado utiliza também “Curinga8” (liberação de reservas). Não consta no JSON fornecido, mas é suportado no controlador.

### 2) Regras e validações
- **Validação de geração de agendas**: `validarDadosGeracaoAgendasAudiencias(audienciaAgendaDt)` obrigatória antes de persistir.
- **Período máximo**: intervalo entre `dataInicial` e `dataFinal` não pode superar 90 dias.
- **Quantidade simultânea**: `quantidadeAudienciasSimultaneas` numérica positiva; máximo padrão 15 (se não informado, aplicar padrão).
- **Tipo de audiência**: obrigatório para consultas e geração. Ausência gera erro.
- **Cargo da serventia**:
  - Se grupo = Conciliadores de Vara, o cargo é fixado via sessão (não selecionável).
  - Demais grupos podem selecionar `serventiaCargo`.
- **Reserva/liberação/exclusão em lote**: operações aceitam múltiplos IDs; quando nenhum selecionado, retornar erro de validação.
- **Auditoria**: sempre registrar `idUsuarioLog` e `ipComputadorLog` em todas as operações de gravação (gerar, reservar, liberar, excluir).

### 3) Modelos de dados (DTO baseados no legado)
- AudienciaAgenda
  - idAudienciaTipo, audienciaTipoCodigo, audienciaTipo
  - idServentia, serventia
  - quantidadeAudienciasSimultaneas, quantidadeMaximaAudienciasSimultaneas
  - dataInicial, dataFinal
  - horariosDuracao[]: vetor de 21 posições (por dia: inicial, final, duração; 7 dias)
  - dataAgendada (quando aplicável), reservada (flag)
  - auditoria: idUsuarioLog, ipComputadorLog
  - aninhado: AudienciaProcesso
    - id, idAudiencia, idAudienciaProcessoStatus, audienciaProcessoStatusCodigo, audienciaProcessoStatus
    - idServentiaCargo, serventiaCargo
- Registro de Agenda Livre (consulta)
  - id, dataHora, tipoAudiencia, cargoServentia, reservada (boolean)

### 4) Endpoints REST propostos

- Consultas
  - GET `/audiencias/agendas-livres`
    - Permissão: 2222 (Localizar) ou 2223 (LocalizarDWR)
    - Query: `audienciaTipoId` (obrigatório), `serventiaCargoId` (condicional por perfil), `serventiaId` (da sessão), `page`, `pageSize`
    - Ação: retorna agendas livres paginadas
    - Retorno: `{ itens: [ {id, dataHora, tipoAudiencia, cargoServentia, reservada} ], pagina, totalPaginas }`
  - GET `/audiencias/tipos`
    - Permissão: relacionada a busca auxiliar (ver `AudienciaTipo` localizar)
    - Query: `q`
    - Retorno: lista de tipos de audiência para autocomplete
  - GET `/serventias/cargos-audiencia`
    - Permissão: relacionada a `ServentiaCargo` localizar
    - Query: `q`, `serventiaId` (da sessão), `subTipo` (quando aplicável)
    - Retorno: lista de cargos aptos a agenda

- Preparação/validação da geração
  - POST `/audiencias/agendas/preparar`
    - Permissão: 2224 (Novo) e/ou 2225 (Salvar)
    - Body: AudienciaAgenda (campos necessários para validação; inclui `audienciaTipoId`, `serventiaId`, `serventiaCargoId` quando aplicável, `dataInicial`, `dataFinal`, `quantidadeAudienciasSimultaneas`, `horariosDuracao[]`)
    - Ação: valida dados e retorna mensagem de confirmação/pre-visualização
    - Retorno: `{ ok: true, mensagem: "Clique para confirmar a geração de agendas" }`

- Confirmação (persistência) da geração
  - POST `/audiencias/agendas/confirmar`
    - Permissão: 2225 (Salvar)
    - Body: AudienciaAgenda (mesmo payload da preparação)
    - Ação: efetiva a geração de agendas conforme parâmetros
    - Retorno: 201, `{ mensagem: "Agenda(s) salva(s) com sucesso." }`

- Operações especiais (lote)
  - POST `/audiencias/agendas/reservas`
    - Permissão: 2227 (Curinga7 – Reservar)
    - Body: `{ agendasIds: string[] }`
    - Ação: reserva agendas livres selecionadas
    - Retorno: `{ mensagem: "Agenda(s) reservada(s) com sucesso." }`
  - POST `/audiencias/agendas/reservas/liberar`
    - Permissão: Curinga8 (liberar) – não listado no JSON, mas requerido pelo fluxo
    - Body: `{ agendasIds: string[] }`
    - Ação: libera reservas
    - Retorno: `{ mensagem: "Agenda(s) liberada(s) com sucesso." }`
  - DELETE `/audiencias/agendas`
    - Permissão: 2220 (Excluir Agenda) ou 2226 (Curinga6 – confirmação conforme fluxo)
    - Body: `{ agendasIds: string[] }`
    - Ação: exclui agendas livres selecionadas
    - Retorno: `{ mensagem: "Agenda(s) excluída(s) com sucesso." }`

### 5) Respostas e códigos de status
- 200 OK: consultas e operações concluídas sem criação
- 201 Created: geração de agendas confirmada
- 400 Bad Request: validações de negócio (ex.: período > 90 dias; tipo de audiência ausente; nenhum ID informado)
- 403 Forbidden: usuário sem permissão
- 404 Not Found: IDs de agendas inexistentes
- 409 Conflict: conflitos de reserva/existência/concorrência

### 6) Observações de segurança
- Autenticação e autorização obrigatórias para todas as rotas.
- Auditoria: registrar `idUsuarioLog` e `ipComputadorLog` em gravações.
- Dados sensíveis: tratar IP como dado pessoal (LGPD); não expor informações desnecessárias.
- Concorrência: reservas e exclusões devem ser transacionais e idempotentes quando possível.

### 7) Itens a decidir na migração
- Padrão de paginação (page/pageSize) e limites.
- Formato de data/hora e timezone (legado usa HH:mm e LATIN1; sugerir ISO-8601 UTF-8).
- Validação/estrutura de `horariosDuracao[]` (21 posições) e política para dias não atendidos.
- Granularidade de permissões para liberar (Curinga8) ausente no JSON – definir código e fluxo.
- Comportamento quando `Conciliadores de Vara` (cargo fixo) – refletir regra no backend via claims/perfil.


