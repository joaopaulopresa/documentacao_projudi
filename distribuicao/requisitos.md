## Requisitos – Relatório de Distribuição por Serventia

Escopo: migrar a funcionalidade legacy (ProcessoDistribuidoPorServentia) para endpoints REST. Ignorar UI/JSP; focar em permissões, regras/validações, modelos de dados e endpoints para consulta auxiliar e geração de PDF (sintético/analítico).

### 1) Permissões
- **Código base**: `732` – Distribuição (menu: “Relatório de Distribuição por Área”)
- **Subpermissões relevantes (legado → REST)**:
  - `IMPRIMIR` (gera PDF do relatório)
  - `LOCALIZAR` de entidades auxiliares (área de distribuição, serventia, usuário)
- Observação: há item relacionado no menu (`631 – Quantidade Processos no Ano`), fora do escopo deste documento.

### 2) Regras e validações
- **Obrigatórios para gerar PDF**: `idAreaDistribuicao`, `dataInicial`, `dataFinal`, `tipoRelatorio`.
- **Tipos de relatório**:
  - `1` → Sintético (totais por serventia e variações de responsabilidade/compensação/correção)
  - `2` → Analítico (linhas com processos: número, classe, assunto, responsável, data de recebimento, tipo de distribuição)
- **Consulta de áreas de distribuição**:
  - Se o usuário puder consultar outras áreas (`podeConsultarOutrasAreasDistribuicao`): listar por texto livre.
  - Caso contrário: restringir à(s) área(s) vinculada(s) à `serventia` do usuário logado.
- **Consulta de serventias**:
  - Requer `idAreaDistribuicao` previamente selecionado; caso ausente, retornar erro de validação.
- **Consulta de usuários (servidores)**: filtra por nome de servidor do Judiciário.
- **Restrições por grupo/perfil** (efeito no backend):
  - Grupo Estatística: pode localizar/consultar livremente.
  - Distribuidor 2º Grau: permitido quando a `serventia` do usuário for a Divisão de Distribuição do Tribunal; caso contrário restringir/negação.
- **Sem resultados**: retornar “não encontrado” para a combinação de filtros.
- **Auditoria**: registrar `idUsuarioLog` e `ipComputadorLog` em chamadas que geram arquivos.

### 3) Modelos de dados (DTO)
- **Filtro principal (entrada)**
  - `idAreaDistribuicao` (string)
  - `areaDistribuicao` (string)
  - `idServentia` (string, opcional)
  - `serventia` (string, opcional)
  - `idUsuario` (string, opcional)
  - `usuario` (string, opcional – nome)
  - `dataInicial` (string data)
  - `dataFinal` (string data)
  - `tipoRelatorio` ("1" ou "2")
  - `tipoArquivo` (string, opcional; padrão PDF)

- **Dados de saída – Sintético (exemplo de campos por linha/agregação)**
  - `nomeAreaDistribuicao`, `nomeServentia`, `nomeUsuario`
  - `distribuicao`, `redistribuicao`
  - `ganhoResponsabilidade`, `perdaResponsabilidade`
  - `ganhoCompensacao`, `perdaCompensacao`, `compensacao`
  - `ganhoCorrecao`, `perdaCorrecao`, `correcao`

- **Dados de saída – Analítico (exemplo de campos por processo)**
  - `nomeAreaDistribuicao`, `nomeServentia`, `nomeUsuario`
  - `distribuicaoTipo`
  - `numeroProcesso`
  - `dataRecebimento`
  - `nomeClasse`, `nomeAssunto`
  - `nomeResponsavel`

Observação: no legado o resultado é entregue como PDF; os campos acima orientam estrutura e auditoria do relatório.

### 4) Endpoints REST propostos

- Preparação
  - **GET** `/relatorios/distribuicao-por-serventia/preparar`
    - Permissão: `732`
    - Ação: retorna metadados/flags do contexto do usuário (ex.: podeConsultarOutrasAreasDistribuicao, opções de `tipoRelatorio`)
    - Retorno 200 com JSON

- Consultas auxiliares (combos/listagens)
  - **GET** `/relatorios/distribuicao-por-serventia/areas-distribuicao?query=&page=&pageSize=`
    - Permissão: `732`
    - Regra: respeitar `podeConsultarOutrasAreasDistribuicao`; caso falso, restringir por `idServentia` do usuário
    - Retornos: 200 (lista), 403 (sem permissão de ampliar escopo)

  - **GET** `/relatorios/distribuicao-por-serventia/serventias?query=&idAreaDistribuicao=&page=&pageSize=`
    - Permissão: `732`
    - Regra: `idAreaDistribuicao` é obrigatório; se ausente, 400
    - Retornos: 200 (lista), 400 (validação)

  - **GET** `/relatorios/distribuicao-por-serventia/usuarios?query=&page=&pageSize=`
    - Permissão: `732`
    - Regra: retorna apenas servidores judiciais
    - Retornos: 200 (lista)

- Geração de relatório (arquivo)
  - **POST** `/relatorios/distribuicao-por-serventia/pdf`
    - Permissão: `IMPRIMIR` de `732`
    - Body (JSON):
      - `idAreaDistribuicao` (obrigatório)
      - `idServentia` (opcional)
      - `idUsuario` (opcional)
      - `dataInicial` (obrigatório)
      - `dataFinal` (obrigatório)
      - `tipoRelatorio` ("1"|"2", obrigatório – sintético/analítico)
      - `tipoArquivo` (opcional; default "PDF")
      - `idUsuarioLog`, `ipComputadorLog` (auditoria)
    - Ação: gera e retorna o PDF correspondente
    - Retornos: 200 `application/pdf`; 400 (validação faltante); 404 (sem dados para os filtros); 403 (sem permissão)

Observações:
- Endpoints de consulta devem suportar paginação (`page`, `pageSize`) e filtro textual (`query`).

### 5) Respostas e códigos de status
- 200: sucesso (JSON ou PDF conforme endpoint)
- 400: erro de validação (campos obrigatórios, formato inválido, ausência de `idAreaDistribuicao` na busca de serventia)
- 403: acesso negado por perfil/grupo/escopo
- 404: nenhuma informação encontrada para os filtros informados

### 6) Observações de segurança
- Autenticação obrigatória; autorizações por grupo/perfil (Estatística; Distribuidor 2º Grau com serventia específica) aplicadas nos endpoints.
- Dados pessoais restritos aos necessários ao relatório; observar LGPD ao expor nomes de usuários/servidores.
- Entrega de arquivos via `application/pdf`; considerar proteção contra acesso direto por URL.

### 7) Itens a decidir na migração
- Padrão de data no REST: ISO 8601 (`YYYY-MM-DD`) em vez de strings livres do legado.
- GET vs. POST para geração de PDF; aqui definido como POST devido ao corpo com filtros e auditoria.
- Paginação e limite padrão das listas auxiliares.
- Internacionalização de mensagens (ex.: erros de validação).
- Timeouts e tamanho máximo do relatório analítico.


