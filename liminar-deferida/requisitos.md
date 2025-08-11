## Requisitos – Liminar Deferida (Processos não julgados)

Escopo: migrar a funcionalidade legada “Liminar Deferida” para endpoints REST. Objetivo: consultar/processos não julgados com liminar deferida por tempo de distribuição e gerar relatório (PDF). Ignorar UI/JSP.

### 1) Permissões
- Código base: 902 – “Liminar Deferida” (link legado: `ProcessoTempoVidaTipo`).
- Subpermissões observadas:
  - 9022 – LOCALIZAR
  - 9024 – NOVO (preparar/limpar contexto)
  - 9026 – CURINGA6 (abrir processo selecionado)
  - IMPRIMIR – usado no fluxo (PDF); não listado no JSON anexo, mas presente no código.

Fonte: `liminar-deferida.json` e `ProcessoTempoVidaTipoCt`.

### 2) Regras e validações
- Escopo por perfil/grupo (código legado):
  - Estatística: pode consultar por qualquer `serventia` (precisa selecionar uma). JSON de consulta específico.
  - Desembargador (2º grau): restringe à `serventia` do gabinete do usuário (usa `Id_UsuarioServentia`).
  - Demais usuários: restringe à `serventia` do usuário.
- Filtro “Tempo de Distribuição” (limiar em dias): opções discretas [Até 20; >20; >30; … >360]. Valor padrão (DTO): 100.
- Seleção de registro: ação especial abre o processo (CURINGA6 → navegação para módulo de “Busca de Processo”). No REST, retornar `idProcesso`/link navegável.
- Validações:
  - `serventiaId` obrigatório para grupo Estatística; ignorado (derivado) para demais grupos.
  - `limiarDias` deve ser um dos valores suportados [20,30,40,50,60,70,80,90,100,110,120,130,140,150,180,240,360] ou “ATE_20”.
  - Paginação obrigatória para listagem.
- Auditoria: incluir `idUsuarioLog` e `ipComputadorLog` nas operações relevantes (consulta/relatório).

### 3) Modelos de dados (DTO)
- ProcessoTempoVidaTipo (baseado no DTO legado):
  - `serventiaId` (string)
  - `serventia` (string)
  - `processoTipoId` (string, reservado; não exigido no fluxo atual)
  - `processoTipo` (string)
  - `periodo` (string; padrão "100" no legado)
  - `processoId` (string)
  - `numeroProcesso` (string)
  - `dataUltimaMovimentacao` (string)
  - `complementoMovimentacao` (string)

- Item de resultado da consulta (derivado dos cabeçalhos da lista):
  - `processoId` (string)
  - `numeroProcesso` (string)
  - `aceite` (string data/hora)
  - `serventia` (string)
  - `gabinete` (string)
  - `nomeDesembargador` (string)
  - `dias` (inteiro)

Observação: campos “aceite”, “gabinete”, “nomeDesembargador” e “dias” vêm do JSON de negócio (`consultarProcessosTempoVidaPorTipo*`).

### 4) Endpoints REST propostos

- Preparar contexto (equiv. NOVO/abrir tela)
  - Método: GET
  - Caminho: `/liminar-deferida/preparar`
  - Permissão: 902 (NOVO)
  - Query: —
  - Ação/regra: retorna valores padrão e opções de filtro (lista fixa de “Tempo de Distribuição” e `serventia` atual do usuário; para Estatística, sinaliza que `serventia` deve ser informada).
  - Retorno 200 (JSON):
    - `limiarDiasOpcoes`: array estático com rótulo/valor
    - `padrao`: `{ limiarDias: 100, escopo: 'MAIOR_QUE' }`
    - `serventiaAtual`: `{ id, nome } | null`
    - `perfil`: `ESTATISTICA | DESEMBARGADOR | PADRAO`

- Consultar processos com liminar deferida não julgados (equiv. LOCALIZAR)
  - Método: GET
  - Caminho: `/liminar-deferida`
  - Permissão: 9022 (LOCALIZAR)
  - Query:
    - `serventiaId` (string, obrigatório para perfil Estatística; ignorado nos demais, pois derivado do usuário)
    - `limiarDias` (int; ver validação; default 100)
    - `escopo` (string: `ATE` para “Até 20 dias” ou `MAIOR_QUE` para as demais; default `MAIOR_QUE`)
    - `page`, `pageSize`
  - Ação/regra: aplica escopo por perfil; consulta dados conforme o legado (`consultarProcessosTempoVidaPorTipo*`).
  - Retorno 200 (JSON):
    - `items`: array de itens do modelo “resultado da consulta” acima
    - `page`, `pageSize`, `total`
  - Erros: 400 (parâmetros inválidos), 403 (sem permissão), 404 (sem resultados quando aplicável).

- Gerar relatório PDF (equiv. IMPRIMIR)
  - Método: GET
  - Caminho: `/liminar-deferida/relatorio.pdf`
  - Permissão: 902 + subpermissão de imprimir (quando aplicável)
  - Query:
    - `serventiaId` (regras como no “Consultar”)
    - `limiarDias` (int)
    - `escopo` (`ATE|MAIOR_QUE`)
  - Ação/regra: chama serviço de relatório (`relLiminaresDeferidas`), com escopo por perfil (Estatística: livre por serventia; Desembargador: usa `Id_UsuarioServentia`; demais: `Id_Serventia`).
  - Retorno 200: `application/pdf` (bytes)
  - Erros: 400, 403, 404.

- Ação especial – abrir processo (equiv. CURINGA6)
  - Observação: no legado, redireciona para “BuscaProcesso”. No REST, a lista deve expor `processoId` para que o frontend navegue ao recurso de processo (fora do escopo deste módulo): ex. `GET /processos/{processoId}`.

- Endpoints auxiliares (combos/consultas)
  - `GET /serventias?nome=` (auto‑complete para perfil Estatística)
  - (Opcional) `GET /processo-tipos?descricao=` (presente no controlador, não exigido no fluxo atual)

### 5) Respostas e códigos de status
- 200 OK: sucesso nas consultas e geração do PDF
- 201 Created: não aplicável
- 400 Bad Request: parâmetros inválidos (ex.: `limiarDias` fora da lista)
- 403 Forbidden: sem permissão/subpermissão requerida ou acesso a serventia não permitida
- 404 Not Found: nenhum registro (quando aplicável) ou serventia inexistente
- 409 Conflict: conflito de negócio (se aplicável pelo serviço de domínio)

### 6) Observações de segurança
- Autenticação obrigatória; autorização por grupo/perfil (Estatística, Desembargador, demais) e permissões 902/9022/IMPRIMIR.
- Escopo de dados por `serventia` conforme perfil; nunca aceitar `serventiaId` externo para perfis não‑Estatística.
- LGPD: dados de pessoas/processos podem ser sensíveis; respeitar sigilos e perfis ao retornar colunas.
- Auditoria: registrar `idUsuarioLog` e `ipComputadorLog` nas consultas e geração de relatórios.

### 7) Itens a decidir na migração
- Semântica do filtro “Até 20 dias” vs. “Mais de X dias”: padronizar via `escopo=ATE|MAIOR_QUE` e validar `limiarDias`.
- Paginação/ordenação default para consulta; campos ordenáveis (ex.: `dias`, `numeroProcesso`).
- Exposição de colunas “gabinete” e “nomeDesembargador” para 1º grau: exibir apenas quando aplicável (2º grau).
- Tratamento de ausência de subpermissão explícita para IMPRIMIR no JSON: usar política padrão do módulo.


