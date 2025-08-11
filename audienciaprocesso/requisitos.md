## Requisitos – Audiência x Processo (Troca de Responsável)

### Escopo e objetivo
Documentar a funcionalidade “AudienciaProcesso” do legado JSP/Servlet para migração a endpoints REST. Foco em operações, parâmetros, validações e permissões. Ignorar navegação/telas.

Fontes: `AudienciaProcessoCtGen.java`, `AudienciaProcessoCt.java`, DTOs `AudienciaProcessoDt.java` e `AudienciaProcessoDtGen.java`, permissões em `audienciaprocesso.json`.

### Permissões relevantes
- Código base da funcionalidade: **457 – AudienciaProcesso**
- Subpermissões:
  - **4574 – NOVO**: Início da troca de responsável (seleção em lote ou única)
  - **4575 – SALVAR**: Confirmação da troca de responsável
  - Consultas auxiliares dependem de permissões de cada entidade relacionada (ex.: `ServentiaCargo`, `Audiencia`, `Processo`, `AudienciaProcessoStatus`)

### Regras e validações principais
- Fluxo de troca em lote ou individual:
  - Carrega itens por `audienciaProcessos[]` ou `Id_AudienciaProcesso`.
  - Validação prévia: `verificarTrocaResponsavel`.
  - Persistência: `salvarTrocaResponsavel(idServentiaCargoDestino, lista, logUsuario)`.
  - Após salvar, atualizar lista com dados atuais de cada item.
- Campos mínimos requeridos para efetivar a troca:
  - `idServentiaCargo` (destino), lista de `audienciaProcessos` (ids).
- Auditoria: enviar `idUsuarioLog` e `ipComputadorLog` no contexto da operação.

### Modelo de dados (principais campos)
Baseado em `AudienciaProcessoDt`/`Gen` e parâmetros capturados:
- Identificação e status
  - `id` (Id_AudienciaProcesso), `idAudiencia`, `audienciaTipo`,
  - `idAudienciaProcessoStatus`, `audienciaProcessoStatus`, `audienciaProcessoStatusCodigo`
- Responsáveis/vínculos
  - `idServentiaCargo`, `serventiaCargo` (responsável atual), dados auxiliares (presidente, MP, redator, etc.)
- Processo
  - `idProcesso`, `processoNumero`
- Metadados/controle
  - `dataMovimentacao`, `codigoTemp`, `audienciaTipoCodigo`
  - Campos adicionais para sessão (ata iniciada/adiada), indicadores de voto/ementa (quando aplicável)

---

## Endpoints REST propostos

### 1) Troca de responsável
- POST `/audiencias-processos/troca-responsavel/preparar`
  - Permissão: 4574 (NOVO)
  - Body: `{ audienciaProcessos: [idAudienciaProcesso...] }` ou `{ idAudienciaProcesso }`
  - Ação: carrega e retorna a lista de itens (dados resumidos) que serão alvo da troca.

- POST `/audiencias-processos/troca-responsavel/confirmar`
  - Permissão: 4575 (SALVAR)
  - Body: `{ idServentiaCargoDestino, audienciaProcessos: [idAudienciaProcesso... ] }`
  - Regras: valida com `verificarTrocaResponsavel`; salva com `salvarTrocaResponsavel` (auditoria obrigatória).
  - Retorno: itens atualizados após a troca.

### 2) Consultas auxiliares
- GET `/serventia-cargos?query=&serventiaId=&subTipoCodigo=&page=`
  - Permissão: conforme entidade `ServentiaCargo`
  - Retorna cargos de serventia aptos para agenda de audiência do usuário (usa filtros de serventia/subtipo do usuário quando aplicável).

- GET `/audiencias?query=&page=`
- GET `/audiencias-processos/status?query=&page=`
- GET `/processos?query=&page=`
  - Permissão: conforme cada entidade
  - Uso: seleção/validação de entidades relacionadas no fluxo.

---

## Respostas e códigos de status (resumo)
- 200: sucesso nas consultas e operações de preparação/confirmação.
- 201: não aplicável (troca altera vínculo existente; se criar histórico, retornar 200 com payload atualizado).
- 400: validação de parâmetros (ids ausentes/invalidos, destino obrigatório).
- 403: usuário sem permissão para a ação.
- 404: entidade não localizada (audienciaProcesso, serventiaCargo destino, etc.).
- 409: conflito de negócio detectado em `verificarTrocaResponsavel`.

## Observações de segurança
- Todas as rotas exigem autenticação e avaliação de permissão (código 457 e subcódigos quando aplicável), além de checagens de escopo do usuário (serventia, subtipo, grupo).
- Registrar auditoria: `idUsuarioLog` e `ipComputadorLog`.

## Itens a decidir na migração
- Estratégia de atualização em lote transacional vs. parcial (retentativas item a item).
- Paginação/ordenação para consultas auxiliares.


