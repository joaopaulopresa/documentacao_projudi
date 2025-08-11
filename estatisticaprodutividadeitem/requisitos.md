## Requisitos – Itens do Relatório de Estatística de Produtividade

Breve: Migração da funcionalidade legada de cadastro/consulta e geração de relatório de itens de estatística de produtividade para endpoints REST. Ignorar UI JSP; focar em permissões, regras, modelos e operações expostas.

### Permissões
- **Código base**: 511 (`EstatisticaProdutividadeItem`)
- **Subpermissões** (do JSON):
  - 5112: `LOCALIZAR`
  - 5113: `LOCALIZARDWR`

Observação: geração de relatório (impressão) ocorre sob a permissão base 511.

### Regras e validações
- **Obrigatórios**:
  - `estatisticaProdutividadeItem` (descrição)
- **Formato/limites**:
  - `dadoCodigo`: numérico, até 11 dígitos (aceita vazio). No legado há máscara de dígitos e limite de tamanho.
- **Validação de negócio**:
  - Executada por `Verificar(...)` antes de persistir. Em caso de falha, retornar 400 com mensagem de validação.
- **Paginação**:
  - Consultas retornam paginação conforme tamanho padrão (parametrizável por `page`/`pageSize`).
- **Auditoria**:
  - Preencher `idUsuarioLog` e `ipComputadorLog` a partir do usuário autenticado em todas as operações de alteração.

### Modelos de dados (DTO)
- **EstatisticaProdutividadeItem**
  - `id` (string)
  - `estatisticaProdutividadeItem` (string): descrição do item. Obrigatório.
  - `dadoCodigo` (string): código numérico opcional (até 11 dígitos).
  - `codigoTemp` (string): uso interno/transitório.
  - Campos de auditoria herdados (ex.: `idUsuarioLog`, `ipComputadorLog`).

Observações: `codigoTemp` é interno e não precisa ser aceito em criação/edição pública; pode ser somente leitura quando retornar.

### Endpoints REST propostos

- Consulta/listagem
  - GET `/estatisticas-produtividade-itens`
    - Permissão: 511 ou 5112
    - Query: `descricao` (string, opcional), `page` (int, opcional), `pageSize` (int, opcional)
    - Ação: consultar por descrição com paginação
    - Retorno: 200 com `{ items: EstatisticaProdutividadeItem[], page, pageSize, total }`

- Localização (modo leve/auto-complete – legado `LOCALIZARDWR`/JSON)
  - GET `/estatisticas-produtividade-itens/localizar`
    - Permissão: 511 ou 5113
    - Query: `termo` (string), `page` (int, opcional)
    - Ação: consulta enxuta para auto-complete
    - Retorno: 200 com `{ items: [{ id, descricao }], page, total }`

- Recuperar por id
  - GET `/estatisticas-produtividade-itens/{id}`
    - Permissão: 511
    - Ação: consultar registro por identificador
    - Retorno: 200 com DTO completo; 404 se não encontrado

- Preparar (equivalente ao NOVO do legado)
  - POST `/estatisticas-produtividade-itens/preparar`
    - Permissão: 511
    - Corpo: vazio
    - Ação: fornecer DTO padrão com valores iniciais (limpo)
    - Retorno: 200 com DTO vazio (para preenchimento do cliente)

- Criar
  - POST `/estatisticas-produtividade-itens`
    - Permissão: 511
    - Corpo: `{ estatisticaProdutividadeItem, dadoCodigo? }`
    - Ação: valida e persiste novo registro
    - Retorno: 201 com DTO criado; 400 se falha de validação

- Atualizar
  - PUT `/estatisticas-produtividade-itens/{id}`
    - Permissão: 511
    - Corpo: `{ estatisticaProdutividadeItem, dadoCodigo? }`
    - Ação: valida e persiste alterações
    - Retorno: 200 com DTO atualizado; 400 se falha de validação; 404 se não encontrado

- Excluir
  - DELETE `/estatisticas-produtividade-itens/{id}`
    - Permissão: 511
    - Ação: excluir registro
    - Retorno: 204; 404 se não encontrado; 409 se bloqueio de negócio

- Relatório (Imprimir)
  - GET `/estatisticas-produtividade-itens/relatorio`
    - Permissão: 511
    - Query: `descricao` (string, opcional), `tipoArquivo` (int: `1`=PDF [padrão], `2`=TXT)
    - Ação: gera e retorna arquivo (filtrado por descrição, quando enviado)
    - Retorno: 200 com `application/pdf` ou `text/plain`

### Respostas e códigos de status
- 200: sucesso (consulta, recuperar, preparar, atualizar)
- 201: criado (criação)
- 204: sem conteúdo (exclusão)
- 400: erro de validação (mensagem do `Verificar(...)`)
- 403: sem permissão (base/subpermissões)
- 404: não encontrado (id inexistente)
- 409: conflito de negócio (exclusão/atualização impedida por regra)

### Observações de segurança
- Autenticação obrigatória; autorização por código de permissão 511 e subpermissões listadas.
- Auditoria: registrar `idUsuarioLog` e `ipComputadorLog` em operações de escrita.
- Relatórios devem ser entregues com nome/tipo de conteúdo corretos; evitar vazamento de informações além do escopo solicitado.

### Itens a decidir na migração
- Regras exatas de validação do `Verificar(...)` (ex.: unicidade da descrição; ranges aceitos para `dadoCodigo`).
- Ordenação padrão nas consultas (descrição ascendente?).
- Padrão de paginação (valores default de `pageSize`).
- Nome do arquivo e metadados do relatório (PDF/TXT), inclusive charset do TXT.
- Se `codigoTemp` deve ou não ser exposto no contrato (sugestão: ocultar).


