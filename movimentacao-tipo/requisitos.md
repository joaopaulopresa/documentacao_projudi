## Requisitos – Movimentação Tipo (Parametrização por Usuário e Grupo)

Escopo: migrar a funcionalidade legada de parametrização de tipos de movimentação por usuário e grupo para endpoints REST. Foca em permissões, regras/validações, modelos de dados e endpoints. Não descreve telas/UI.

### 1) Permissões
- Código base: 684 – "Movimentação Tipo" (ref. `UsuarioMovimentacaoTipoDt.CodigoPermissao`)
- Subpermissões (ref. JSON):
  - 6846 – "Curinga6" (operações especiais de inclusão/remoção via toggle)

Observação: no legado, as operações de inclusão/remoção são executadas quando `paginaatual == Configuracao.Curinga6` com `Passo = incluir|excluir`.

### 2) Regras e validações
- Autorização obrigatória: requer permissão 684; operações de alteração requerem subpermissão 6846.
- Operações inválidas no legado: `NOVO`, `SALVAR`, `SALVARRESULTADO`, `EXCLUIR`, `EXCLUIRRESULTADO` são rejeitadas ("Operação inválida"). Na migração REST, não expor esses verbos como fluxos separados.
- Inclusão:
  - Validar existência do `movimentacaoTipoId` alvo e que está ativo/selecionável para o grupo/usuário.
  - Prevenir duplicidade de vínculo usuário×tipo de movimentação (retornar 409 em caso de conflito).
- Exclusão:
  - Validar existência prévia do vínculo; se não existir, retornar 404 (ou decidir por idempotência com 204 — ver Itens a decidir).
- Consulta de opções:
  - Retornar a lista completa de tipos de movimentação disponíveis para o grupo/usuário, marcando quais estão selecionados.
- Auditoria: todas as operações de alteração devem registrar `idUsuarioLog` e `ipComputadorLog`.

### 3) Modelos de dados (DTO)
- UsuarioMovimentacaoTipo
  - `id` (string) – identificador do vínculo (se houver)
  - `usuarioId` (string)
  - `movimentacaoTipoId` (string)
  - `movimentacaoTipoDescricao` (string)
  - `movimentacaoTipoCodigo` (string)

- MovimentacaoTipoOpcao (para consulta de opções)
  - `movimentacaoTipoId` (string)
  - `codigo` (string)
  - `descricao` (string)
  - `selecionado` (boolean)

Observações:
- O legado compõe propriedades com `id_Usuario` e `GrupoCodigo`; manter esses campos disponíveis para filtros/escopo quando necessário.

### 4) Endpoints REST propostos

- Consulta de opções (lista completa com marcação de selecionados)
  - Método: GET
  - Caminho: `/usuario-movimentacao-tipos/opcoes`
  - Permissão: 684
  - Query: `usuarioId` (obrigatório), `grupoCodigo` (opcional; se ausente, usar grupo do usuário)
  - Ação: consultar tipos disponíveis para o grupo/usuário e indicar quais já estão selecionados
  - Retorno 200: `MovimentacaoTipoOpcao[]`

- Listar vínculos existentes (apenas selecionados)
  - Método: GET
  - Caminho: `/usuarios/{usuarioId}/movimentacao-tipos`
  - Permissão: 684
  - Ação: listar vínculos atuais usuário×tipo de movimentação
  - Retorno 200: `UsuarioMovimentacaoTipo[]`

- Incluir vínculo usuário×tipo (toggle marcar)
  - Método: POST
  - Caminho: `/usuario-movimentacao-tipos`
  - Permissão: 684 + subpermissão 6846
  - Corpo (JSON): `{ "usuarioId": string, "movimentacaoTipoId": string, "idUsuarioLog": string, "ipComputadorLog": string }`
  - Regras: validar existência e evitar duplicidade
  - Retorno 201: `UsuarioMovimentacaoTipo` (Location: `/usuario-movimentacao-tipos/{usuarioId}/{movimentacaoTipoId}`)
  - Erros: 400 (validação), 403 (permissão), 404 (não encontrado), 409 (duplicidade)

- Excluir vínculo usuário×tipo (toggle desmarcar)
  - Método: DELETE
  - Caminho: `/usuario-movimentacao-tipos/{usuarioId}/{movimentacaoTipoId}`
  - Permissão: 684 + subpermissão 6846
  - Query/Corpo: `idUsuarioLog`, `ipComputadorLog` (se não vier em cabeçalhos)
  - Regras: validar existência prévia do vínculo
  - Retorno 204 sem corpo
  - Erros: 400 (validação), 403 (permissão), 404 (não encontrado)

- Operação em lote (opcional, recomendado)
  - Método: POST
  - Caminho: `/usuario-movimentacao-tipos/lote`
  - Permissão: 684 + subpermissão 6846
  - Corpo: `{ "usuarioId": string, "movimentacaoTipoIds": string[], "idUsuarioLog": string, "ipComputadorLog": string, "modo": "substituir"|"acrescentar"|"remover" }`
  - Ação: atualizar vínculos em lote conforme `modo`
  - Retorno 200: `{ atualizado: number, adicionados: number, removidos: number }`
  - Erros: 400/403/404/409

- Combos auxiliares (tipos de movimentação)
  - Método: GET
  - Caminho: `/movimentacao-tipos`
  - Permissão: 684 (ou pública para auto-complete, se aplicável)
  - Query: `grupoCodigo` (opcional), `texto` (opcional, filtro por descrição/código)
  - Retorno 200: `{ id: string, codigo: string, descricao: string }[]`

### 5) Respostas e códigos de status
- 200 OK: consultas e operações em lote
- 201 Created: inclusão de vínculo individual
- 204 No Content: exclusão de vínculo
- 400 Bad Request: validações de formato/negócio
- 403 Forbidden: ausência de permissão 684/6846
- 404 Not Found: tipo de movimentação ou vínculo inexistente
- 409 Conflict: tentativa de duplicidade de vínculo

### 6) Observações de segurança
- Requer autenticação e autorização por permissão 684 (e 6846 para alteração).
- Registrar auditoria (`idUsuarioLog`, `ipComputadorLog`) em inclusão/exclusão/lote.
- Validar escopo: alterações devem respeitar regras organizacionais (ex.: apenas perfis autorizados podem alterar vínculos de terceiros).

### 7) Itens a decidir na migração
- Exclusão idempotente: retornar 204 mesmo quando o vínculo não existe, ou 404? (recomendação: 204 idempotente).
- Exposição de `grupoCodigo` na API: obrigatório, opcional (default do usuário) ou inferido do contexto?
- Endpoint de lote: habilitar modos `substituir`/`acrescentar`/`remover` e limites de tamanho.
- Paginação/filtro no combo de tipos quando houver grande volume (parâmetro `texto`).


