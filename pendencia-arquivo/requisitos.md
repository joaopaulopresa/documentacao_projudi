## Requisitos – Pendência Arquivo

Escopo: migrar a funcionalidade legada de gestão de arquivos associados a pendências (vínculo, consulta, exclusão, download e alteração de status) para endpoints REST. Ignorar detalhes de UI/JSP.

### 1) Permissões
- Código base: 114 – "Pendência Arquivo"
- Subpermissões relevantes (do JSON):
  - 1142 – "Localizar Pendência Arquivo"
  - 1143 – "LocalizarDWR Pendência Arquivo" (auto‑complete/consulta leve)
  - 1146 – "Pendência Arquivo Baixar Arquivo"

Observação: o controlador também referencia consultas auxiliares com permissões de `Arquivo` e `Pendência` para compor buscas externas.

### 2) Regras e validações
- Regras gerais
  - A operação de salvar executa validação de negócio via `Verificar(...)`; só persiste quando a mensagem de validação vier vazia.
  - Exclusão retorna mensagem de sucesso quando concluída.
  - Consulta suporta paginação baseada em posição e quantidade de páginas informadas pela camada de negócio.
  - Alteração de status exige `Id_PendenciaArquivo` válido e um `Status` aceitável.
- Download/entrega
  - Download protegido (`CURINGA6`) exige validação de hash: `UsuarioSessao.VerificarCodigoHash(id, hash)`; em caso de falha, negar acesso.
  - Quando `CodigoVerificacao=true`, gerar e retornar PDF de verificação da publicação.
  - Parâmetro `finalizado=true` ou quando a NE identificar movimento concluído (`isPendenciaArquivoMovido`) indica que o arquivo é tratado como finalizado.
  - Há endpoint público de publicação (`CURINGA7`).
- Perfis/grupos
  - Aplicar a permissão base 114 para operações de CRUD/consulta internas; subpermissão 1146 para download.
  - Consultas auxiliares podem requerer permissões dos módulos `Arquivo` e `Pendência`.
- Auditoria
  - Enviar sempre `idUsuarioLog` e `ipComputadorLog` nas operações de escrita.

### 3) Modelos de dados (DTO)
- PendenciaArquivo
  - id: string (Id_PendenciaArquivo)
  - idArquivo: string (Id_Arquivo)
  - nomeArquivo: string (NomeArquivo)
  - idPendencia: string (Id_Pendencia)
  - pendenciaTipo: string (PendenciaTipo)
  - resposta: string lógico "true"/"false" (Resposta)
  - codigoTemp: string (CodigoTemp)
  - hash: string (para links protegidos)
  - valido: boolean (controle interno/negócio)
  - assistenteResponsavel: string
  - juizResponsavel: string
  - multiplo: boolean
  - processoNumero: string
  - dataPreAnalise: string (data/hora)
  - idPendenciaRelacionada: string
  - auditoria: { idUsuarioLog: string, ipComputadorLog: string }
- Constantes de status vistas no módulo
  - NORMAL, BLOQUEADO_POR_VIRUS, RESTRICAO_DOWNLOAD, AGUARDANDO_ASSINATURA (para política de download/visibilidade)

### 4) Endpoints REST propostos

- Consultas
  - GET `/pendencia-arquivos`
    - Permissão: 1142
    - Query: `q` (texto), `idPendencia`, `idArquivo`, `pagina`, `tamanho`
    - Ação: listar/vinculações por filtro, com paginação
    - Retorno: `{ itens: PendenciaArquivo[], pagina, quantidadePaginas }`
  - GET `/pendencia-arquivos/{id}`
    - Permissão: 114
    - Ação: detalhar um vínculo arquivo×pendência
    - Retorno: `PendenciaArquivo`
  - GET `/pendencia-arquivos/localizar-dwr`
    - Permissão: 1143
    - Query: `q`, `limite`
    - Ação: consulta leve (auto‑complete)
    - Retorno: lista resumida

- Criação/alteração
  - POST `/pendencia-arquivos`
    - Permissão: 114
    - Body: `{ idArquivo, nomeArquivo?, idPendencia, pendenciaTipo?, resposta?, codigoTemp?, auditoria }`
    - Ação: validar (`Verificar`) e persistir vínculo
    - Retorno: 201 com recurso criado ou 400 com mensagens de validação
  - PUT `/pendencia-arquivos/{id}/status`
    - Permissão: 114
    - Body: `{ status, auditoria }`
    - Ação: alterar status do arquivo vinculado à pendência
    - Retorno: 200 com mensagem de sucesso
  - DELETE `/pendencia-arquivos/{id}`
    - Permissão: 114
    - Body: `{ auditoria }`
    - Ação: excluir vínculo arquivo×pendência
    - Retorno: 200 com confirmação

- Download/entrega
  - GET `/pendencia-arquivos/{id}/download`
    - Permissão: 1146
    - Query: `hash` (obrigatório), `finalizado` (bool), `recibo` (bool)
    - Ação: valida hash; quando `CodigoVerificacao` não informado, baixar conteúdo do arquivo da pendência (considerando finalização)
    - Retorno: arquivo binário; 403 em hash inválido
  - GET `/pendencia-arquivos/{id}/codigo-verificacao`
    - Permissão: 1146
    - Query: `hash=true`
    - Ação: gerar e retornar PDF de verificação da publicação
    - Retorno: `application/pdf`
  - GET `/pendencia-arquivos/{id}/publicacao`
    - Permissão: público (conforme política atual do legado para publicação pública)
    - Ação: baixar publicação pública vinculada
    - Retorno: arquivo binário

- Endpoints auxiliares (combos/consultas)
  - GET `/arquivos`
    - Permissão: do módulo Arquivo
    - Query: `q`, `pagina`, `tamanho`
    - Ação: localizar arquivos para associação
  - GET `/pendencias`
    - Permissão: do módulo Pendência
    - Query: `q`, `pagina`, `tamanho`
    - Ação: localizar pendências para associação

### 5) Respostas e códigos de status
- 200 OK: consultas, alteração de status, exclusão
- 201 Created: criação de vínculo
- 400 Bad Request: validações de negócio (`Verificar`) e formato
- 403 Forbidden: hash inválido para download protegido
- 404 Not Found: recurso inexistente
- 409 Conflict: conflito de estado (ex.: status incompatível)

### 6) Observações de segurança
- Autenticação obrigatória para operações internas (CRUD/consulta); avaliar se publicação pública mantém acesso anônimo.
- Download protegido por hash; nunca expor identificadores sensíveis sem verificação.
- Auditoria: sempre registrar `idUsuarioLog` e `ipComputadorLog` em gravações.
- LGPD: arquivos podem conter dados pessoais; respeitar escopos de acesso e registrar consentimento quando aplicável.

### 7) Itens a decidir na migração
- Enumeração final dos valores permitidos de `status` em `/status` (sugeridos: `VALIDO`, `INVALIDO`, `RESTRICAO_DOWNLOAD`, `BLOQUEADO_POR_VIRUS`, `AGUARDANDO_ASSINATURA`).
- Política de acesso ao endpoint `/publicacao` (público vs. autenticado) e expiração/renovação de `hash`.
- Formato do campo `resposta` (padronizar como boolean) e renomear campos para `camelCase` no JSON.
- Estratégia de paginação (offset/limit vs. página/tamanho) e limites.
- Tamanho máximo de arquivos, verificação antivírus e bloqueios.


