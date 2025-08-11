## Requisitos – Dados do Usuário

Breve descrição: Migração da funcionalidade de gestão/consulta de usuários para endpoints REST. Abrange consulta com filtros (grupo, serventia, nome), exibição de detalhes, ativação/desativação, limpeza de senha (reset), alteração de senha do usuário logado, consultas auxiliares (grupos e serventias) e emissão de relatório PDF. Ignorar fluxos/telas JSP.

### 1) Permissões
- **Código base**: 326 — “Dados Usuário” (origem do método `Permissao()` do controlador)
- **Subpermissões relevantes** (do JSON):
  - 3266 — “Dados Usuário CURINGA6” — Altera Senha do Usuário Logado

Observação: Operações específicas (ativar, desativar, limpar senha, imprimir) devem exigir a permissão base 326 e, quando aplicável, subpermissões específicas definidas pela política do módulo (ex.: alterar senha do logado sob 3266).

### 2) Regras e validações
- **Consulta de usuários**: filtros opcionais por `grupoId`, `serventiaId`, `nome`. Retorna lista resumida (id, nome, login, grupo, documento, e‑mail, situação/ativo).
- **Ativar usuário**: requer confirmação prévia; exige `id` válido; retorno de sucesso altera status para ativo.
- **Desativar usuário**: requer confirmação prévia; exige `id` válido; retorno de sucesso altera status para inativo.
- **Limpar (resetar) senha de usuário**: requer confirmação prévia; no legado define senha padrão “12345”. Na migração, substituir por senha temporária randômica e exigir troca no próximo login.
- **Alterar senha do usuário logado (CURINGA6)**:
  - Se sessão estiver em modo “loginToken”: não exigir senha atual; validar apenas nova e confirmação.
  - Do contrário: exigir `senhaAtual`, validar `senhaNova` e confirmação.
  - Política de senha forte (conforme orientação em tela): mínimo 8 caracteres, ao menos 1 maiúscula, 1 minúscula, 1 número e 1 símbolo; nova senha = confirmação.
  - Pode haver fluxo de “troca obrigatória” (flag de sessão `codigoTemp = "trocarSenha"`).
- **Consultas auxiliares**:
  - Grupos: pesquisa por nome com escopo/visão restrita ao grupo do usuário logado.
  - Serventias: pesquisa por nome; suportar paginação conforme necessário.
- **Auditoria (todas as operações mutáveis)**: capturar `idUsuarioLog` e `ipComputadorLog` do contexto de autenticação.

### 3) Modelos de dados (DTO)
- **Usuario** (detalhe):
  - `id` (string)
  - `nome` (string)
  - `usuario` (login) (string)
  - `rg` (string)
  - `cpf` (string)
  - `dataNascimento` (string | date)
  - `matriculaTjGo` (string) — quando aplicável (não para advogado)
  - `advogado` (boolean)
  - `ativo` (boolean)
  - `telefone` (string)
  - `email` (string)
  - `grupoId` (string), `grupo` (string), `grupoCodigo` (string)
  - `serventiaId` (string), `serventia` (string)
  - `endereco`:
    - `logradouro`, `numero`, `complemento`, `bairro`, `cidade`, `uf`, `cep`
  - Auditoria: `idUsuarioLog`, `ipComputadorLog` (preenchidos via contexto)

- **UsuarioListaItem** (lista):
  - `id`, `nome`, `usuario`, `grupo`, `doc`, `email`, `ativo`

- **FiltrosConsultaUsuarios**:
  - `grupoId?` (string), `serventiaId?` (string), `nome?` (string), `pagina?` (string|number), `tamanho?` (number)

- **AlteracaoSenhaLogadoPayload**:
  - `senhaAtual?` (string) — obrigatório quando não estiver em loginToken
  - `senhaNova` (string)
  - `senhaNovaConfirmacao` (string)

### 4) Endpoints REST propostos

- Consultas
  - GET `/usuarios`
    - Permissão: 326
    - Query: `grupoId?`, `serventiaId?`, `nome?`, `pagina?`, `tamanho?`
    - Ação: consulta lista de usuários com filtros
    - Retorno: `200` `[UsuarioListaItem]`

  - GET `/usuarios/{id}`
    - Permissão: 326
    - Ação: consulta detalhada do usuário
    - Retorno: `200` `Usuario`

- Operações com preparar/confirmar (separar confirmação do efetivo)
  - POST `/usuarios/{id}/ativar/preparar`
    - Permissão: 326
    - Ação: retorna mensagem de confirmação (“Clique para Ativar...”)
    - Retorno: `200` `{ mensagem }`
  - POST `/usuarios/{id}/ativar/confirmar`
    - Permissão: 326
    - Ação: efetiva ativação
    - Retorno: `200` `{ mensagem: "Usuário Ativado com sucesso." }`

  - POST `/usuarios/{id}/desativar/preparar`
    - Permissão: 326
    - Ação: retorna mensagem de confirmação (“Clique para Desativar...”) 
    - Retorno: `200` `{ mensagem }`
  - POST `/usuarios/{id}/desativar/confirmar`
    - Permissão: 326
    - Ação: efetiva desativação
    - Retorno: `200` `{ mensagem: "Usuário Desativado com sucesso." }`

  - POST `/usuarios/{id}/senha/limpar/preparar`
    - Permissão: 326
    - Ação: retorna mensagem de confirmação para reset de senha
    - Retorno: `200` `{ mensagem }`
  - POST `/usuarios/{id}/senha/limpar/confirmar`
    - Permissão: 326
    - Ação: efetiva reset de senha; gerar senha temporária randômica e marcar “troca obrigatória no próximo login”
    - Retorno: `200` `{ mensagem: "Senha resetada com sucesso.", senhaTemporaria?: string }` (avaliar omitir a senha e enviá‑la por canal seguro)

- Operação especial (CURINGA6)
  - POST `/usuarios/senha/alterar`
    - Permissão: 3266
    - Body: `AlteracaoSenhaLogadoPayload`
    - Ação: valida e altera senha do usuário logado; se `loginToken=true` no contexto, `senhaAtual` é ignorada
    - Retorno: `200` `{ mensagem: "Senha Alterada com Sucesso." }`

- Consultas auxiliares (combos)
  - GET `/usuarios/grupos`
    - Permissão: 326
    - Query: `nome?`, `pagina?`, `tamanho?`
    - Ação: lista grupos permitidos ao usuário logado (escopo por `grupoCodigo` do perfil)
    - Retorno: `200` `[{ id, grupo, codigo, tipoServentia }]`

  - GET `/usuarios/serventias`
    - Permissão: 326
    - Query: `nome?`, `pagina?`, `tamanho?`
    - Ação: lista serventias por nome, com paginação
    - Retorno: `200` `[{ id, serventia, estado }]`

- Relatórios
  - POST `/usuarios/relatorios/lista`
    - Permissão: 326
    - Body: `FiltrosConsultaUsuarios`
    - Ação: gera PDF da listagem de usuários conforme filtros
    - Retorno: `200` binário `application/pdf` (download)

Observação: Avaliar suporte a operações em lote para ativar/desativar/resetar senha — ex.: `POST /usuarios/ativar/confirmar` com `{ ids: [...] }`.

### 5) Respostas e códigos de status
- `200` sucesso em consultas/operações
- `201` criação (apenas se for definido fluxo de inclusão, não presente no legado atual)
- `400` validação (ex.: senha fraca, confirmação divergente, parâmetros inválidos)
- `403` permissão insuficiente
- `404` recurso não encontrado (usuário, grupo, serventia)
- `409` conflito de negócio (ex.: ativar já ativo; desativar já inativo)

### 6) Observações de segurança
- Não utilizar senha padrão fixa em reset; gerar credencial temporária randômica e exigir troca.
- Armazenar senha com hash seguro e salt (ex.: bcrypt/argon2), nunca em claro.
- Não retornar senhas em respostas; se necessário, entregar token/URL de redefinição por canal seguro.
- Registrar auditoria (`idUsuarioLog`, `ipComputadorLog`, timestamp, ação, alvo, sucesso/erro).
- Rate limiting e proteção contra brute‑force em endpoints de senha.

### 7) Itens a decidir na migração
- Política de quem pode resetar senha de terceiros e ativar/desativar (perfil/cargo).
- Se a senha temporária deve ser exibida na resposta ou enviada por e‑mail/SMS.
- Padrão de paginação (cursor vs. página/tamanho) para consultas de grupos/serventias/usuários.
- Mapeamento de campos `doc` (CPF/RG) e estrutura de `endereco` na saída pública.
- Suporte (ou não) a operações em lote para ativação/desativação/reset de senha.


