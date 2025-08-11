## Requisitos – Movimentação de Arquivo

Escopo: migrar a funcionalidade legada de Movimentação de Arquivo (perm. 195) para endpoints REST. O módulo cobre validação/alteração de visibilidade e publicação de arquivos de movimentações processuais, listagem de arquivos por processo/movimentação e fluxos de conversão entre processo digital e híbrido (com adição de peças), com auditoria e regras por perfil.

### 1) Permissões
- Código base: 195 – Movimentação Arquivo
- Subpermissões (conforme `doc/movimentacao-arquivo/movimentacao-arquivo.json`):
  - 1951 – Publicar/Despublicar Arquivo
  - 1952 – Localizar Movimentação Arquivo (listagem/consulta)
  - 1953 – LocalizarDWR Movimentação Arquivo (operações com lista de arquivos – conversão híbrido/digital e adição de peças)
  - 1956 – CURINGA6 (Validar Arquivos de Movimentação)
  - 1957 – CURINGA7 (Invalidar/alterar visibilidade de Arquivos de Movimentação)
  - 1958 – Listar Arquivos Movimentação JSON (lista por processo/movimentação)

### 2) Regras e validações
- Alteração de visibilidade e validação/invalidade:
  - Só permitida quando `VerificarAlteracaoVisibilidadeAquivo` autorizar. Arquivos com visibilidade de nível “Magistrado” só podem ser alterados por Magistrado (mensagem de erro conforme legado).
  - Para invalidar/alterar visibilidade é obrigatório informar `tipoAcesso` (mapa: `Publico`, `Adv`, `Delegacia`, `Mp`, `Cartorio`, `Juiz`, `Global`).
- Publicar/Despublicar:
  - Ação obrigatória em `{acao}` ∈ {`Publicar`, `Despublicar`}. Qualquer outro valor deve retornar 400 (legado lança exceção).
- Listar arquivos da movimentação (JSON):
  - Requer processo válido. No legado, falha quando `processoDt` não está na sessão; na API, exigir `idProcesso` e retornar 404 se não encontrado.
- Conferir publicação:
  - Requer `id` do arquivo de movimentação; retorna booleano.
- Fluxos híbrido/digital (operações por lista de arquivos):
  - Converter híbrido → digital; Adicionar peças em processo que já foi híbrido; Retornar digital → híbrido (com confirmação opcional).
  - Quando “retornar digital → híbrido” no legado: usa preparo (SALVAR) para confirmação e efetivação (SALVARRESULTADO). Na API, separar em `/preparar` e `/confirmar`.
- Auditoria: todos os endpoints que modificam estado devem registrar `idUsuarioLog` e `ipComputadorLog`.
- Perfis/grupos: respeitar escopo por perfil (Magistrado, Cartório, MP, Delegacia, Advogados), refletindo o nível de `AcessoArquivo`/`AcessoUsuario`.

### 3) Modelos de dados (DTO base)
- MovimentacaoArquivo
  - Identificação e vínculos: `id`, `idArquivo`, `idMovimentacao`, `movimentacaoTipo`
  - Arquivo e tipo: `nomeArquivo`, `arquivoTipoCodigo`, `arquivoTipo`, `hash`, `recibo`
  - Assinatura: `usuarioAssinador` (entregue formatado no JSON – CN do certificado; contempla coassinaturas)
  - Validação/estado: `valido` (boolean), `codigoTemp` (ex.: OBJECT_STORAGE, PJE_MIGRADO)
  - Acesso: `acessoArquivo`, `acessoUsuario`, `idMovimentacaoArquivoAcesso`
  - Derivados/flags: `isFisico`, `isHistoricoFisico`, `isMidiaDigitalUpload`
- Constantes relevantes (legado):
  - Estados: `NORMAL`, `BLOQUEADO_POR_VIRUS`, `RESTRICAO_DOWNLOAD`, `OBJECT_STORAGE`, `FILE_SYSTEM`, `PJE_MIGRADO`
  - Acessos: `ACESSO_PUBLICO`, `ACESSO_NORMAL`, `ACESSO_SOMENTE_*`, `ACESSO_BLOQUEADO`, `ACESSO_BLOQUEADO_VIRUS`, `ACESSO_BLOQUEADO_ERRO_MIGRACAO`
  - Fluxos: `FLUXO_PUBLICAR`, `FLUXO_DESPUBLICAR`

### 4) Endpoints REST propostos

1) Consultas/Listagens
- GET `/movimentacao-arquivos`
  - Permissão: 1952
  - Query: `descricao` (string, opcional), `pagina` (long), `tamanho` (long)
  - Ação: listar por descrição com paginação
  - Retorno: lista paginada de `MovimentacaoArquivo`

- GET `/processos/{idProcesso}/movimentacoes/{idMovimentacao}/arquivos`
  - Permissão: 1958
  - Ação: listar arquivos da movimentação em JSON (equivalente ao `consultarArquivosMovimentacaoHashJSON`)
  - Retorno: array de itens com campos: `id`, `id_arquivo`, `nome_arquivo`, `id_arquivo_tipo`, `arquivo_tipo`, `usuario_assinador`, `acessoUsuario`, `acessoArquivo`, `valido`, `hash`, `recibo`, `codigo_temp`

- GET `/movimentacao-arquivos/{id}`
  - Permissão: 195
  - Ação: obter detalhes de um arquivo de movimentação
  - Retorno: objeto `MovimentacaoArquivo`

- GET `/movimentacao-arquivos/{id}/publicado`
  - Permissão: 1951
  - Ação: conferir se o arquivo de movimentação está publicado
  - Retorno: `{ "publicado": boolean }`

2) CRUD básico (escopo do módulo genérico)
- POST `/movimentacao-arquivos`
  - Permissão: 195
  - Body: campos do DTO base necessários (`idArquivo`, `idMovimentacao`, `nomeArquivo`, `arquivoTipoCodigo`, ...)
  - Ação: criar registro de arquivo de movimentação
  - Retorno: 201 com objeto criado

- PUT `/movimentacao-arquivos/{id}`
  - Permissão: 195
  - Body: campos editáveis do DTO base
  - Ação: atualizar registro
  - Retorno: 200 com objeto atualizado

- DELETE `/movimentacao-arquivos/{id}`
  - Permissão: 195
  - Ação: excluir registro de arquivo de movimentação
  - Retorno: 200 com mensagem de sucesso

3) Preparação/Confirmação (retorno de digital → híbrido)
- POST `/processos/{idProcesso}/retorno-hibrido/preparar`
  - Permissão: 1953
  - Body: `{ "proximaAcao": "retorno-hibrido", "mensagem": string? }`
  - Ação: preparar confirmação do retorno de processo digital para híbrido (mensagem de confirmação)
  - Retorno: `{ "mensagem": "Clique para confirmar...", "tokenConfirmacao": string }`

- POST `/processos/{idProcesso}/retorno-hibrido/confirmar`
  - Permissão: 1953
  - Body: `{ "tokenConfirmacao": string }`
  - Ação: efetivar retorno digital → híbrido (sem movimentação no processo)
  - Retorno: 200 com mensagem de sucesso

4) Operações especiais sobre arquivos de movimentação
- POST `/movimentacao-arquivos/{id}/validar`
  - Permissão: 1956
  - Body: `{ }`
  - Ação: validar arquivo da movimentação (requer autorização para alterar visibilidade)
  - Retorno: 200 com mensagem de sucesso

- POST `/movimentacao-arquivos/{id}/visibilidade`
  - Permissão: 1957
  - Body: `{ "tipoAcesso": "Publico|Adv|Delegacia|Mp|Cartorio|Juiz|Global" }`
  - Ação: invalidar/alterar visibilidade do arquivo de movimentação para o nível informado
  - Retorno: 200 com mensagem de sucesso

- POST `/movimentacao-arquivos/{id}/publicacao`
  - Permissão: 1951
  - Body: `{ "acao": "Publicar" | "Despublicar" }`
  - Ação: publicar/despublicar o arquivo de movimentação
  - Retorno: `{ "status": "sucesso" }`

5) Conversão e adição de peças (híbrido/digital)
- POST `/processos/{idProcesso}/arquivos/hibrido/converter-digital`
  - Permissão: 1953
  - Body: `{ "arquivos": [ { ... } ] }`
  - Ação: converter processo híbrido para digital, atualizando a lista de arquivos
  - Retorno: 200 com mensagem de sucesso

- POST `/processos/{idProcesso}/arquivos/hibrido/adicionar-pecas`
  - Permissão: 1953
  - Body: `{ "arquivos": [ { ... } ] }`
  - Ação: adicionar novas peças a processo digital que já foi híbrido
  - Retorno: 200 com mensagem de sucesso

- POST `/processos/{idProcesso}/arquivos/hibrido/retornar`
  - Permissão: 1953
  - Body: `{ "arquivos": [ { ... } ], "registrarMovimentacao": boolean }`
  - Ação: retornar processo digital para híbrido; quando `registrarMovimentacao = true`, registrar movimentação correspondente
  - Retorno: 200 com mensagem de sucesso

- POST `/processos/{idProcesso}/arquivos/sincronizar-storage` (opcional – avaliar uso)
  - Permissão: 1953
  - Ação: sincronizar com o Object Storage de digitalização
  - Observação: trecho classificado como possivelmente morto no legado; só migrar se houver consumidor

### 5) Respostas e códigos de status
- 200 OK: operações concluídas; consultas retornam JSON conforme descrito
- 201 Created: criação via POST `/movimentacao-arquivos`
- 400 Bad Request: parâmetros inválidos (ex.: ação de publicação diferente de `Publicar`/`Despublicar`; `tipoAcesso` ausente)
- 403 Forbidden: sem permissão para a operação ou para alterar visibilidade (regra de Magistrado)
- 404 Not Found: processo/arquivo inexistente
- 409 Conflict: conflitos de negócio (ex.: estado incompatível para mudança de visibilidade/publicação)

### 6) Observações de segurança
- Autenticação/Autorização obrigatórias; aplicar checagem de perfil por subpermissão.
- Dados sensíveis: arquivos e metadados de assinatura; respeitar níveis de visibilidade e LGPD.
- Operações de publicação/visibilidade devem ser auditadas com `idUsuarioLog` e `ipComputadorLog`.

### 7) Itens a decidir na migração
- Modelo de paginação e filtros da listagem geral (`/movimentacao-arquivos`).
- Estrutura do payload de `arquivos` nas operações de conversão/adição (metadados mínimos e validações).
- Estratégia de confirmação para “retorno digital → híbrido” (token, idempotência, expiração).
- Manutenção (ou descontinuação) do endpoint de sincronização de storage.


