## Requisitos – Classificador

Escopo: migrar o módulo legado de Classificador (cadastro/edição/exclusão e classificação/desclassificação em lote de processos) para endpoints REST. Ignorar telas/JSP.

### 1) Permissões
- Código base: 193 (Classificador)
- Subpermissões mapeadas:
  - 1930 – Excluir Classificador
  - 1932 – Localizar Classificador
  - 1933 – LocalizarDWR Classificador (autocomplete/JSON)
  - 1934 – Novo Classificador
  - 1935 – Salvar Classificador

Observação: ações de consulta a Serventia usam permissão própria de Serventia e são externas ao módulo, mas expostas aqui como “endpoints auxiliares”.

### 2) Regras e validações
- Cadastro/edição de Classificador
  - Descrição (`classificador`) obrigatória.
  - `prioridade` numérica (no legado há restrição de “só números”).
  - Serventia:
    - Administrador pode selecionar qualquer `id_serventia`.
    - Demais perfis: `id_serventia` vem da sessão e não pode ser alterado.
  - Validação de negócio servidor: `ClassificadorNe.Verificar(ClassificadorDt)` antes de persistir.
  - Opcional (recomendado): unicidade por (`classificador`, `id_serventia`).

- Exclusão
  - Checar `ClassificadorNe.validarExclusaoClassificador` e bloquear se em uso, retornando 409.

- Consulta/Localizar
  - Filtro por descrição; suporte a retorno para autocomplete (JSON) incluindo campo oculto `Prioridade`.

- Classificação/desclassificação em lote de processos
  - Critérios de filtro (mínimo 1):
    - `cpfPoloAtivo` ou `poloAtivoVazio` (boolean);
    - `cpfPoloPassivo` ou `poloPassivoVazio` (boolean);
    - `id_tipoProcesso` (com `ProcessoTipo` descritivo);
    - `id_assunto` (com `Assunto` descritivo);
    - `id_classificador` atual (opcional);
    - `valorMinimo`, `valorMaximo` (faixa de valores).
  - Ação:
    - Informando `id_classificador_alteracao` → classificar todos os processos filtrados para esse classificador.
    - Sem `id_classificador_alteracao` → desclassificar todos os processos filtrados.
  - Resultado esperado: quantidade de itens afetados e mensagem correspondente.
  - Escopo de dados restrito à `id_serventia` do usuário da sessão (no legado, passada ao serviço de negócio).

- Auditoria
  - Em todas as operações que alteram estado: `idUsuarioLog`, `ipComputadorLog`.

### 3) Modelos de dados (DTO)
- ClassificadorDto
  - `id` (string)
  - `classificador` (string)
  - `id_serventia` (string)
  - `serventia` (string)
  - `prioridade` (string)
  - `serventiaCodigo` (string)
  - Auditoria: `idUsuarioLog`, `ipComputadorLog`

- ClassificacaoProcessosFiltro
  - `cpfPoloAtivo` (string), `poloAtivoVazio` (boolean)
  - `cpfPoloPassivo` (string), `poloPassivoVazio` (boolean)
  - `id_tipoProcesso` (string)
  - `id_assunto` (string)
  - `id_classificador` (string) – filtro pelo classificador atual
  - `valorMinimo` (string), `valorMaximo` (string)
  - `id_classificador_alteracao` (string) – destino; vazio implica desclassificar

- ClassificacaoProcessosResultado
  - `acao` ("classificar" | "desclassificar")
  - `quantidadeAfetada` (number)
  - `mensagem` (string)

### 4) Endpoints REST propostos

- Consulta de classificadores
  - GET `/classificadores`
    - Permissão: 1932
    - Query: `descricao`, `serventiaId`, `prioridade`, `page`, `pageSize`, `sort`
    - Ação: consulta paginada
    - Retorno: lista de `ClassificadorDto` + paginação

  - GET `/classificadores/autocomplete`
    - Permissão: 1933
    - Query: `term` (descrição), `serventiaId`
    - Ação: retorna lista reduzida (id, descrição, prioridade) para auto-complete

- Cadastro/edição de classificador
  - POST `/classificadores/preparar`
    - Permissão: 1935 (validação prévia)
    - Body: `ClassificadorDto`
    - Ação: valida sem persistir; retorna erros de domínio/obrigatórios
    - 200 OK ou 400 validação

  - POST `/classificadores`
    - Permissão: 1935
    - Body: `ClassificadorDto`
    - Ação: criar novo classificador
    - 201 Created com objeto criado

  - PUT `/classificadores/{id}`
    - Permissão: 1935
    - Body: `ClassificadorDto`
    - Ação: atualizar existente
    - 200 OK

  - DELETE `/classificadores/{id}`
    - Permissão: 1930
    - Ação: excluir; bloquear se em uso (validação de negócio)
    - 204 No Content ou 409 Conflito

- Classificação/desclassificação em lote
  - POST `/classificadores/processos/classificar`
    - Permissão: 1935
    - Body: `ClassificacaoProcessosFiltro`
    - Ação: classificar (quando `id_classificador_alteracao` informado) ou desclassificar (quando vazio)
    - Retorno: `ClassificacaoProcessosResultado`

- Endpoints auxiliares (combos/consultas)
  - GET `/serventias?nome=...` – consulta de serventias (perm. do módulo Serventia)
  - GET `/tipos-processo?nome=...` – consulta de tipos de processo
  - GET `/assuntos?nome=...` – consulta de assuntos

### 5) Respostas e códigos de status
- 200 OK – operações de consulta/atualização
- 201 Created – criação de classificador
- 204 No Content – exclusão sem corpo
- 400 Bad Request – validação de campos/regra de negócio
- 403 Forbidden – sem permissão (código base/subpermissão)
- 404 Not Found – recurso não encontrado (id inexistente)
- 409 Conflict – vínculos que impedem exclusão; nenhuma linha afetada em classificação

### 6) Observações de segurança
- Autenticação e verificação de permissão por código base/subpermissão.
- Escopo de dados restrito à `id_serventia` da sessão para classificação em lote.
- LGPD: campos de CPF/CNPJ são dados pessoais – minimizar retornos, mascarar quando aplicável, trafegar sob TLS e registrar acesso.
- Auditoria obrigatória: `idUsuarioLog`, `ipComputadorLog` nas operações de escrita.

### 7) Itens a decidir na migração
- Paginação/ordenação padrão de `/classificadores` e de consultas auxiliares.
- Limites e timeouts para classificação em lote; necessidade de job assíncrono e acompanhamento de progresso.
- Idempotência e “dry-run” para classificação (simular sem aplicar).
- Regra de unicidade de `classificador` por `id_serventia` e normalização (trim/casefold).


