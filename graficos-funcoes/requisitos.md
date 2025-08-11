## Requisitos – Gráficos (Processos por Comarca/Serventia/Item de Produtividade)

Objetivo: Migrar a funcionalidade legada de geração de gráficos de processos (por Comarca, por Serventia e por Itens de Estatística de Produtividade) para endpoints REST. A saída primária é PDF. Ignorar detalhes de UI/JSP.

### 1) Permissões
- Código base: 533 – "Gráficos Funções"
- Subpermissões mapeadas (graficos-funcoes.json):
  - 5334 – NOVO: preparar contexto para novo gráfico
  - 5331 – IMPRIMIR: gerar/entregar PDF do gráfico
  - 5332 – LOCALIZAR: consultas auxiliares (auto‑complete) para Comarca, Serventia e Itens de Produtividade
  - 5336 – CURINGA6: preparar gráfico por Comarca
  - 5337 – CURINGA7: imprimir gráfico por Comarca (PDF)
  - 5338 – CURINGA8: preparar gráfico por Serventia
  - 5339 – CURINGA9: imprimir gráfico por Serventia (PDF)

Observação: a geração por Item de Produtividade utiliza NOVO/IMPRIMIR; Comarca/Serventia utilizam CURINGA6/7 e CURINGA8/9, respectivamente.

### 2) Regras e validações
- Datas padrão: ao preparar (NOVO/CURINGA6/CURINGA8), definir mês/ano inicial e final para o mês/ano corrente.
- Gráfico por Item de Produtividade (IMPRIMIR):
  - Obrigatório informar ao menos 1 Item de Produtividade.
  - Para usuários do grupo Estatística (`GrupoDt.ESTATISTICA`): obrigatório informar Comarca ou Serventia. Caso ambos vazios, retornar 400.
  - Para demais usuários: a consulta é automaticamente limitada à Comarca/Serventia do usuário logado; valores de entrada, se fornecidos, devem ser ignorados para escopo além do usuário.
- Gráfico por Comarca (CURINGA7 – imprimir): Comarca é obrigatória (400 se ausente).
- Gráfico por Serventia (CURINGA9 – imprimir):
  - Grupo Estatística: Serventia é obrigatória (400 se ausente).
  - Demais usuários: sempre usar a Serventia do usuário logado; ignorar valores de entrada para escopo além do usuário.
- Formato/validação de período: `mes` ∈ [1..12] e `ano` numérico; validar coerência do intervalo (mês/ano inicial ≤ mês/ano final), retornando 400 quando inválido.
- Localizar (auto‑complete):
  - Serventia: permite filtrar por Comarca (param opcional `idComarca`). Se `idComarca` requerido no fluxo, retornar 400 quando ausente.
  - Comarca e Itens de Produtividade: pesquisa por texto com paginação.
- Auditoria: registrar `idUsuarioLog` e `ipComputadorLog` em todas as operações de geração/consulta.
- Autorização: validar subpermissão requerida por endpoint (ver seção 5) e restringir escopo conforme regras de grupo acima.

### 3) Modelos de dados (DTO)
- GraficoProcessoFiltro
  - `mesInicial` (string|int)
  - `anoInicial` (string|int)
  - `mesFinal` (string|int)
  - `anoFinal` (string|int)
  - `idComarca` (string, opcional)
  - `comarca` (string, opcional – exibição)
  - `idServentia` (string, opcional)
  - `serventia` (string, opcional – exibição)
  - `itensProdutividade` (array de { id: string, descricao: string }, obrigatório apenas para o gráfico por Item)
  - Observação: para usuários fora do grupo Estatística, `idComarca/idServentia` devem ser preenchidos pelo backend com os valores do usuário logado, ignorando sobreposição do cliente.

- Resultado de consultas auxiliares (lista padrão)
  - `items`: array de { id: string, descricao: string }
  - `page`: número da página atual
  - `pageSize`: tamanho da página
  - `total`: total de registros

### 4) Endpoints REST propostos

Preparação (define período padrão; não persiste estado no servidor):
- GET /graficos-processos/item-produtividade/preparar
  - Permissão: 5334 (NOVO)
  - Ação: retorna período padrão corrente
  - Retorno 200: `{ mesInicial, anoInicial, mesFinal, anoFinal }`

- GET /graficos-processos/comarcas/preparar
  - Permissão: 5336 (CURINGA6)
  - Ação: retorna período padrão corrente
  - Retorno 200: `{ mesInicial, anoInicial, mesFinal, anoFinal }`

- GET /graficos-processos/serventias/preparar
  - Permissão: 5338 (CURINGA8)
  - Ação: retorna período padrão corrente
  - Retorno 200: `{ mesInicial, anoInicial, mesFinal, anoFinal }`

Geração/Impressão (retorna PDF):
- POST /graficos-processos/item-produtividade/imprimir
  - Permissão: 5331 (IMPRIMIR)
  - Body: GraficoProcessoFiltro com `itensProdutividade` obrigatório; para grupo Estatística, exigir `idComarca` ou `idServentia`.
  - Ação: gera PDF de processos por Itens de Produtividade
  - Retorno 200: `application/pdf` (bytes)
  - Erros: 400 validação; 403 permissão; 409 conflito de negócio

- POST /graficos-processos/comarcas/imprimir
  - Permissão: 5337 (CURINGA7)
  - Body: GraficoProcessoFiltro com `idComarca` obrigatório
  - Ação: gera PDF de processos por Comarca
  - Retorno 200: `application/pdf`
  - Erros: 400 validação; 403 permissão

- POST /graficos-processos/serventias/imprimir
  - Permissão: 5339 (CURINGA9)
  - Body: GraficoProcessoFiltro
    - Grupo Estatística: `idServentia` obrigatório
    - Demais usuários: ignorar `idServentia` enviado e usar a do usuário logado
  - Ação: gera PDF de processos por Serventia
  - Retorno 200: `application/pdf`
  - Erros: 400 validação; 403 permissão

Consultas auxiliares (LOCALIZAR):
- GET /comarcas?query=texto&page=1&pageSize=20
  - Permissão: 5332 (LOCALIZAR) ou permissão do domínio Comarca
  - Ação: auto‑complete de comarcas
  - Retorno 200: lista paginada padrão

- GET /serventias?query=texto&idComarca=123&page=1&pageSize=20
  - Permissão: 5332 (LOCALIZAR) ou permissão do domínio Serventia
  - Ação: auto‑complete de serventias; aceita filtro por Comarca
  - Retorno 200: lista paginada padrão
  - Erros: 400 quando o fluxo exigir `idComarca` e não for informado

- GET /estatistica-itens?query=texto&page=1&pageSize=20
  - Permissão: 5332 (LOCALIZAR) ou permissão do domínio Estatística de Produtividade
  - Ação: auto‑complete de Itens de Produtividade
  - Retorno 200: lista paginada padrão

### 5) Respostas e códigos de status
- 200: sucesso (JSON nos preparar/consultas, PDF nos imprimir)
- 201: não aplicável (não há criação de recurso)
- 400: validação (período inválido; filtros obrigatórios ausentes; nenhum item selecionado)
- 403: acesso negado (sem subpermissão ou fora do escopo autorizado)
- 404: não encontrado (consultas auxiliares sem correspondência podem retornar 200 com lista vazia)
- 409: conflito de negócio (se aplicável)

### 6) Observações de segurança
- Autenticação e autorização obrigatórias para todos os endpoints.
- Restringir escopo conforme grupo/perfil do usuário: apenas `GrupoDt.ESTATISTICA` pode arbitrar livremente Comarca/Serventia; demais perfis são limitados à Comarca/Serventia do usuário logado.
- PDFs gerados não devem expor dados sensíveis além do necessário ao relatório.

### 7) Itens a decidir na migração
- Definição de paginação padrão nas consultas auxiliares (ex.: `pageSize` default = 20).
- Tolerância de intervalo de datas (anos aceitos, limite máximo de meses).
- Padronização de domínio de permissões para consultas auxiliares (reutilizar permissões próprias de Comarca/Serventia/Itens ou centralizar em 5332).
- Localização por Serventia: exigir sempre `idComarca` como filtro ou mantê-lo opcional.


