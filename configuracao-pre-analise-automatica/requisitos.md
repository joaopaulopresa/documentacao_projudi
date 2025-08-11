## Requisitos – Configuração Pré-Análise Automática

Escopo: migrar a funcionalidade de parametrização de pré-análise automática (por classificador) para endpoints REST. Ignorar telas/JSP; focar em permissões, regras/validações, modelos de dados e endpoints.

### 1) Permissões
- Código base: 108 – "Configuração Pré-Análise Automática"
- Subpermissões (do legado):
  - 1084 – NOVO (preparação de contexto)
  - 1085 – SALVAR (confirmação/gravação)
  - 1080 – EXCLUIR (remoção)
  - 1086 – CURINGA6 (operação especial: gerar pré-análises)

Regras de autorização sugeridas para REST:
- Preparar (NOVO): requer 1084
- Validar/Confirmar/Salvar: requer 1085
- Excluir: requer 1080
- Gerar pré‑análises: requer 1086

### 2) Regras e validações
- Escopo por serventia: toda configuração é vinculada à serventia do usuário logado.
- Unicidade por classificador na serventia: ao escolher um `classificadorId`, a configuração existente deve ser carregada/atualizada (uma por classificador/serventia).
- Campos obrigatórios para salvar:
  - `classificadorId`
  - `modeloId`
  - `movimentacaoTipoId`
  - Pelo menos 1 item em `pendenciasGerar`
- Consistências de negócio (legado: `valideConfiguracao`):
  - `movimentacaoTipoId` deve pertencer à lista configurada para o usuário/grupo.
  - Pendências a gerar compatíveis com o tipo/subtipo selecionado e seus destinatários.
  - Quando informado, `classificadorNovoId` deve ser válido.
  - Campos opcionais tratados: `nomeArquivo`, `complementoMovimentacao`, `guardadoAssinar`.
- Perfis/grupos com condições:
  - Campos "Novo Assistente Responsável" (`serventiaCargoNovoId`) e "Novo Serventia Grupo" (`serventiaGrupoId`) somente para usuário em Gabinete/GabineteFluxoUPJ.
- Auditoria (sempre): `idUsuarioLog`, `ipComputadorLog`.

### 3) Modelos de dados (DTO)

Recurso principal: Configuração de Pré‑Análise Automática
```
ConfigPreAnaliseAuto {
  id: string
  serventiaId: string
  serventia: string
  classificadorId: string
  classificador: string
  modeloId: string
  modelo: string
  nomeArquivo?: string
  movimentacaoTipoId: string
  movimentacaoTipo: string
  complementoMovimentacao?: string
  pendenciasGerar: Array<PendenciaGerar>
  classificadorNovoId?: string
  classificadorNovo?: string
  serventiaCargoNovoId?: string   // restrito a Gabinete/UPJ
  serventiaCargoNovo?: string
  serventiaGrupoId?: string       // restrito a Gabinete UPJ
  serventiaGrupo?: string
  guardadoAssinar: boolean
  // auditoria
  idUsuarioLog: string
  ipComputadorLog: string
}

PendenciaGerar {
  pendenciaTipoCodigo: string
  pendenciaSubTipoId?: string
  destinatarios?: Array<Destinatario> // quando aplicável
  parametros?: object                  // opções específicas (ex.: expedição automática)
}

Destinatario {
  tipo: string   // ex.: parte, advogado, MP, defensor, assistente, etc.
  id?: string
}
```

Listas auxiliares (combos) expostas por endpoints próprios:
- `listaPendenciaTipos` – tipos/subtipos de pendência disponíveis ao usuário
- `listaTiposMovimentacaoConfigurado` – tipos de movimentação permitidos ao usuário/grupo

### 4) Endpoints REST propostos

Consultas e preparação
- GET `/configuracoes-pre-analises-automaticas`
  - Permissão: 108
  - Filtros: `classificadorId` (opcional). Se informado, retorna a configuração da serventia atual para o classificador.
  - Retorno: `ConfigPreAnaliseAuto | null`

- POST `/configuracoes-pre-analises-automaticas/preparar`
  - Permissão: 1084
  - Corpo: `{ classificadorId?: string }`
  - Ação: abre contexto com listas auxiliares (`listaPendenciaTipos`, `listaTiposMovimentacaoConfigurado`) e, se `classificadorId` existir, carrega a configuração.
  - Retorno: `{ config: ConfigPreAnaliseAuto, listas: { pendenciaTipos: [...], movimentacoesTipo: [...] } }`

Validação e gravação
- POST `/configuracoes-pre-analises-automaticas/confirmar`
  - Permissão: 1085
  - Corpo: `ConfigPreAnaliseAuto` (sem campos de auditoria; são inferidos pelo backend)
  - Ação: valida as regras de negócio e retorna inconsistências sem persistir.
  - Retorno: `{ ok: boolean, mensagens?: string[] }`

- POST `/configuracoes-pre-analises-automaticas`
  - Permissão: 1085
  - Corpo: `ConfigPreAnaliseAuto`
  - Ação: cria/atualiza configuração para o `classificadorId` da serventia do usuário (regra de unicidade).
  - Retorno: `201 Created` com `ConfigPreAnaliseAuto` atualizado.

- PUT `/configuracoes-pre-analises-automaticas/{id}`
  - Permissão: 1085
  - Corpo: `ConfigPreAnaliseAuto`
  - Ação: atualiza configuração existente por `id`.
  - Retorno: `200 OK` com `ConfigPreAnaliseAuto` atualizado.

Exclusão
- DELETE `/configuracoes-pre-analises-automaticas/{id}`
  - Permissão: 1080
  - Ação: exclui a configuração; mantém histórico de auditoria.
  - Retorno: `204 No Content`

Operação especial
- POST `/configuracoes-pre-analises-automaticas/gerar-pre-analises`
  - Permissão: 1086
  - Corpo: vazio (executa para a serventia do usuário)
  - Ação: dispara rotina de geração de pré‑análises com base nas configurações vigentes.
  - Retorno: `202 Accepted` (sugerido) ou `200 OK` com sumário da execução.

Endpoints auxiliares (consultas para combos)
- GET `/classificadores` – filtros: `q`, `page` – escopo: serventia atual (e UPJ quando aplicável)
- GET `/modelos` – filtros: `q`, `page` – escopo por usuário/serventia
- GET `/movimentacoes-tipo` – filtros: `q`, `page` – escopo por grupo do usuário
- GET `/serventias-cargos` – filtros: `q`, `page`, `serventiaId`, `serventiaTipo`, `serventiaSubtipo`
- GET `/serventia-grupos` – filtros: `q`, `page`, `serventiaCargoId`
- GET `/pendencias-tipo` – filtros: `q`, `page` – retorna tipos/subtipos habilitados ao usuário

### 5) Respostas e códigos de status
- 200 OK – sucesso em consultas/atualizações
- 201 Created – criação de configuração
- 202 Accepted – disparo assíncrono de geração de pré‑análises
- 204 No Content – exclusão
- 400 Bad Request – falhas de validação (campos obrigatórios/inconsistências)
- 403 Forbidden – falta de permissão (108x)
- 404 Not Found – configuração não encontrada
- 409 Conflict – violação de unicidade/estado de negócio (ex.: conflito por classificador na serventia)

### 6) Observações de segurança
- Autenticação e autorização obrigatórias em todos os endpoints; checar subpermissões por ação.
- Escopo de dados restrito à serventia do usuário; negar acesso cruzado.
- Dados sensíveis: evitar expor informações pessoais desnecessárias em combos de usuários/cargos.
- Auditoria: registrar `idUsuarioLog`, `ipComputadorLog`, data/hora e ação.

### 7) Itens a decidir na migração
- Semântica de upsert: manter POST como criação estrita e exigir PUT para atualização ou aceitar POST idempotente por `classificadorId`/serventia.
- Modelo detalhado de `pendenciasGerar` (estrutura de destinatários e parâmetros específicos por pendência/tipo de expedição).
- Execução da geração de pré‑análises: síncrona vs. assíncrona; paginação/batch e reprocessamentos.
- Padrões de paginação dos endpoints auxiliares (tamanho de página, ordenação).
- Política de versionamento do recurso e compatibilidade com regras do CNJ/estatística (TPU).


