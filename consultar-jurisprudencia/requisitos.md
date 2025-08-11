## Requisitos – Consultar Jurisprudência e Publicações (módulo público 871)

Escopo: Migração da funcionalidade legada (JSP/Servlet) para endpoints REST, cobrindo consulta pública de jurisprudência e publicações, paginação e destaques, download do inteiro teor (PDF) e geração/entrega de relatórios de intimações/publicações, além de consultas auxiliares (serventia, magistrado, tipos de ato e subtipos). Ignorar fluxos/elementos de UI.

### 1) Permissões
- Código base do módulo: `871` (Consultar Jurisprudência) — conforme `Permissao()` em `ConsultaJurisprudenciaCt` e `consultar-jurisprudencia.json`.
- Subpermissões (herdadas do padrão do módulo público, rótulos no JSON podem referenciar “Pendência Pública”, porém os códigos se aplicam a este módulo):
  - `8711` IMPRIMIR: download/entrega do PDF do inteiro teor.
  - `8712` LOCALIZAR: execução da consulta com filtros e paginação.
  - `8713` LOCALIZARDWR: consultas AJAX/assistidas (uso legado de DWR); mapear para endpoints auxiliares.
  - `8714` NOVO: preparação/limpeza do contexto de pesquisa.
  - `8715` SALVAR: não há persistência nesta funcionalidade; manter reservado (sem endpoint na migração, a menos que a preparação/armazenamento de preferências de usuário seja exigida).
  - `8716` a `8719` CURINGA6–CURINGA9: não utilizados explicitamente no fluxo atual; reservar para futuras operações especiais.
- Permissões auxiliares (consultas de apoio, computadas via `Dt.CodigoPermissao * Configuracao.QtdPermissao + Configuracao.Localizar` no legado):
  - Localizar Serventia (para autocomplete/listagem de unidades)
  - Localizar ArquivoTipo (com filtro adicional por tipos de jurisprudência)
  - Localizar Usuário/Serventia/Grupo (magistrados por serventia)
  - Localizar ServentiaSubtipo (órgão/matéria)

### 2) Regras e validações
- Requisito mínimo de filtros: é obrigatório informar pelo menos um dos campos abaixo, caso contrário retornar 400 (mensagem: “É necessário informar algum filtro da pesquisa.”):
  - `texto`, `numeroProcesso`, `idServentia`, `idServentiaSubTipo`, `idArea`, `instancia`, `idRealizador`, `idTipoArquivo`, `dataPublicacaoIni`, `dataPublicacaoFim`.
- Normalização de filtros numéricos/códigos:
  - Valores "0" para `instancia`, `idArea` e `idServentiaSubTipo` devem ser tratados como nulos (não filtrar por esses campos).
- Datas de publicação:
  - Aceitar formato `dd/MM/yyyy` (conforme máscara da UI legada). Validar formato; rejeitar 400 se inválido.
  - Quando ambos informados, permitir intervalo aberto ou fechado; o legado não valida ordem, mas recomenda-se validar `ini <= fim` (400 se inconsistente).
- Número do processo:
  - Validar máscara CNJ `(NNNNNNN-NN.NNNN.N.NN.NNNN)`; rejeitar 400 se formato inválido.
- Paginação e limites:
  - Parâmetros: `page` (0-based) e `size` (itens por página). Valor padrão de `size`: 10. Permitir 10, 20 ou 50. Rejeitar `size > 50` com 400.
- Download do inteiro teor (PDF):
  - Exige validação de reCAPTCHA (token `g-recaptcha-response`). Sem token ou inválido: 403.
  - Se arquivo indisponível/bloqueado: 409 com mensagem adequada.
  - Se `idArquivo` inexistente: 404.
- Tipos de ato (ArquivoTipo):
  - A lista deve ser filtrada para permitir apenas: Acórdão, Decisão, Despacho(s), Ementa, Relatório e Voto, Sentença(s) e Voto (regra do método `filtrarPorTiposEspecificos`).

### 3) Modelos de dados (DTO)
- Pesquisa (baseada em `PesquisaAvancadaDt`):
  - `texto`: string (busca textual, suporta aspas para busca exata)
  - `numeroProcesso`: string (CNJ)
  - `idServentia`: string
  - `serventia`: string (descrição)
  - `idServentiaSubTipo`: string | null
  - `serventiaSubTipo`: string (descrição) | null
  - `idRealizador`: string (usuário/magistrado)
  - `realizador`: string (nome)
  - `dataPublicacaoIni`: string (dd/MM/yyyy)
  - `dataPublicacaoFim`: string (dd/MM/yyyy)
  - `idTipoArquivo`: string
  - `tipoArquivo`: string (descrição)
  - `instancia`: string | null ("16" 1º Grau; "151" Turmas Recursais; "15" Tribunal)
  - `idArea`: string | null (p.ex.: "1" Cível; "2" Criminal)
  - `qtdeItensUsuario`: number (10|20|50)

- Resultado (resumo do `Hit`):
  - `idArquivo`: string
  - `numero`: string (número do processo ou do documento)
  - `serventia`: string (campo `serv`)
  - `realizador`: string (magistrado)
  - `tipoAto`: string (campo `tipo_arq`)
  - `dataPublicacao`: string
  - `dataSessaoOuMovimentacao`: string
  - `textoDestacado`: string (trecho do documento com marcações; pode ser `extra` ou corpo)
  - `highlight`: array<string> (ocorrências no inteiro teor)
  - `tookMs`: number (tempo de resposta do provedor de busca)

### 4) Endpoints REST propostos

1) Consulta de jurisprudência
- Método: GET
- Caminho: `/consultar-jurisprudencia`
- Permissão: `8712` LOCALIZAR (pública)
- Query params:
  - `texto`, `numeroProcesso`, `idServentia`, `serventia`, `idServentiaSubTipo`, `serventiaSubTipo`, `idRealizador`, `realizador`, `dataPublicacaoIni`, `dataPublicacaoFim`, `idTipoArquivo`, `tipoArquivo`, `instancia`, `idArea`
  - `page` (default 0), `size` (default 10; máx. 50)
- Ação/regra: Delegar a mecanismo de busca (ex.: ElasticSearch) montando os parâmetros equivalentes a `getParamsQuery(...)` do legado.
- Retorno 200 (JSON):
  - `total`, `tookMs`, `page`, `size`, `paginas` (total de páginas), `itens`: array de Resultado.
- Erros: 400 (sem filtros ou parâmetros inválidos).

2) Preparar/limpar pesquisa
- Método: POST
- Caminho: `/consultar-jurisprudencia/preparar`
- Permissão: `8714` NOVO (pública)
- Corpo: vazio
- Ação/regra: Retornar contexto inicial com `qtdeItensUsuario = 10` e campos nulos/vazios.
- Retorno 200 (JSON): DTO de pesquisa inicial e metadados úteis (ex.: limites de `size`).

3) Download do inteiro teor (PDF)
- Método: GET
- Caminho: `/consultar-jurisprudencia/arquivos/{idArquivo}/download`
- Permissão: `8711` IMPRIMIR (pública, porém condicionada a reCAPTCHA)
- Query/Header: `g-recaptcha-response` obrigatório
- Ação/regra: Entregar PDF correspondente; se indisponível/bloqueado, retornar erro específico.
- Retorno 200: `application/pdf` (Content-Disposition: attachment)
- Erros: 403 (reCAPTCHA), 404 (arquivo inexistente), 409 (arquivo bloqueado/indisponível).

4) Consultas auxiliares (combos/autocomplete)
- Serventias
  - GET `/serventias?search={texto}&page={n}` — permissão auxiliar Localizar Serventia.
  - Retorna: `id`, `descricao`, `tipo`, `uf`.
- Tipos de Ato (ArquivoTipo)
  - GET `/arquivos-tipo?search={texto}&page={n}&apenasJurisprudencia=true` — aplica filtro de tipos específicos (Acórdão, Decisão, Despacho, Ementa, Relatório e Voto, Sentença(s) e Voto).
  - Retorna: `id`, `descricao`.
- Usuários (Magistrados por Serventia)
  - GET `/usuarios?search={texto}&serventiaId={id}&page={n}` — permissão auxiliar Localizar Usuário/Serventia/Grupo.
  - Retorna: `id`, `nome`, `grupo`.
- Serventia Subtipo (Órgão/Matéria)
  - GET `/serventias/subtipos?search={texto}&page={n}` — permissão auxiliar Localizar ServentiaSubtipo.
  - Retorna: `id`, `descricao`.
- Instâncias
  - GET `/combos/instancias` — lista padrão: 1º Grau (16), Turmas Recursais (151), Tribunal (15), além de opção "Todas" (0).
- Áreas
  - GET `/combos/areas?instancia={id}` — lista dependente da instância; incluir opção "Todas" (0). Valores usuais: Cível (1), Criminal (2).

Observação: o legado também oferece escolha de `qtdeItensPagina` (10/20/50) — manter esta convenção nos endpoints acima.

### 5) Respostas e códigos de status
- 200 OK — sucesso das consultas e preparação; entrega de PDF.
- 201 Created — não aplicável nesta migração (não cria recursos).
- 400 Bad Request — filtros ausentes ou inválidos; formato de datas/numeroProcesso; `size` inválido.
- 403 Forbidden — reCAPTCHA inválido/ausente para download.
- 404 Not Found — arquivo inexistente.
- 409 Conflict — arquivo bloqueado/indisponível.

### 6) Observações de segurança
- Acesso: público (`GrupoDt.ACESSO_PUBLICO`).
- Proteções recomendadas:
  - reCAPTCHA obrigatório para download de PDFs.
  - Rate limiting/anti-scraping em consultas e downloads.
  - Sanitização de entrada (ex.: escapando aspas em `texto`, como no legado) e logs/auditoria mínima (IP/UA) para troubleshooting.

### 7) Itens a decidir na migração
- Padronização de destaques (`highlight`) no retorno da busca (HTML vs. marcações simples).
- Representação de campos de data (string vs. ISO 8601); validação `ini <= fim` no backend.
- Tratamento de `LOCALIZARDWR` (8713): mapeamento definitivo para endpoints REST de autocomplete.
- Política de cache/bust para combos estáticos (instâncias/áreas) e parametrização futura.
- Estratégia de autenticação opcional (se futuramente houver dados restritos ou taxas mais altas de requisições).


### 8) Funcionalidades relacionadas no diretório (mapeadas para REST)

8.1) Consulta de Publicações (ConsultaPublicacaoCt)
- Objetivo: consulta pública de publicações por texto livre ou por campos específicos.
- Permissão: base `871`; localizar `8712`; imprimir `8711`; consultas auxiliares conforme seção 4.
- Regras:
  - Dois modos de busca: `tipoConsulta=texto` (texto livre obrigatório) ou `tipoConsulta=campo` (exige ao menos um filtro dentre texto/número/serventia/magistrado/tipo/data).
  - Padrão de paginação: 15 itens por página (modo texto); no legado, modo campo usa 1 item por página; na migração, unificar para `size` configurável (máx. 50).
  - Download de inteiro teor exige reCAPTCHA.
- Endpoints:
  - GET `/consultar-publicacao` — Query: `tipoConsulta=texto|campo`, `textoDigitado` (quando texto), `texto`, `numeroProcesso`, `idServentia`, `idRealizador`, `idTipoArquivo`, `dataPublicacaoIni`, `dataPublicacaoFim`, `page`, `size`.
    - Retorno 200: `total`, `tookMs`, `page`, `size`, `itens` (estrutura de Resultado similar à seção 3, sem campos de instância/área/subtipo).
    - Erros: 400 (sem texto no modo texto; sem filtros no modo campo).
  - POST `/consultar-publicacao/preparar` — limpa/retorna contexto de pesquisa (200).
  - GET `/consultar-publicacao/arquivos/{idArquivo}/download` — reCAPTCHA obrigatório; 200 PDF; 403/404/409 conforme regra.
  - Auxiliares: mesmas de serventia, arquivo-tipo (com filtro de tipos específicos), usuários por serventia.

8.2) Download de Arquivo Público por movimento (BuscaArquivoPublicoCt)
- Objetivo: entregar PDF público a partir de `Id_MovimentacaoArquivo` e `hash` de validação.
- Permissão: base `871` (público). Não exige reCAPTCHA; valida por hash MD5 de compatibilidade: `md5("null" + idMovimentacaoArquivo + idProcesso)`.
- Endpoints:
  - GET `/publicacoes/arquivo-publico` — Query: `Id_MovimentacaoArquivo`, `hash`.
    - Sucesso: 200 `application/pdf` (Content-Disposition: inline).
    - Erros: 400 (parâmetros ausentes, hash inválido, arquivo não público no momento); 409 (arquivo bloqueado/indisponível). Opcionalmente mapear 404 quando `Id_MovimentacaoArquivo` inexistente.

8.3) Validação de Documento Público e Intimações do Dia Anterior (PendenciaPublicaCt)
- Objetivo: validar/capturar PDF por código público e obter arquivo texto com intimações do dia anterior (com cache em contexto).
- Permissão: base `871` (público). Validação por reCAPTCHA no download por código.
- Endpoints:
  - GET `/pendencia-publica/validar` — Query: `c` (ou `codPublicacao`); Header/Query: `g-recaptcha-response` obrigatório.
    - Sucesso: 200 `application/pdf` (attachment) com nome prefixado `Doc_{codigo}`.
    - Erros: 400 (código inválido); 403 (reCAPTCHA); 409 (arquivo bloqueado/indisponível).
  - GET `/pendencia-publica/intimacoes-dia-anterior` — Query: opcional `opcaoPublicacao=1|2|3`.
    - Sucesso: 200 `text/plain` com lista (“ListaIntimacoes”).
    - Em processamento: 202/409 com mensagem “Relatório em processamento, tente novamente”.
    - Erro de geração: 500/409 com mensagem orientativa.
  - Observação: no legado há cache de arquivo e janela de validade no `ServletContext` — manter estratégia equivalente no backend REST.

8.4) Geração de PDF de Intimações por Data (PublicacaoIntimacaoPDFCt e PublicacaoIntimacaoPDFV2Ct)
- Objetivo: gerar e entregar PDF contendo intimações de uma data específica, por escopo (2º Grau, 1º Grau Capital, 1º Grau Interior). V2 usa consulta otimizada.
- Permissão: base `871` (público). Sem reCAPTCHA.
- Endpoints:
  - POST `/intimacoes/pdf/gerar` — Body/Query: `dataPublicacao` (dd/MM/yyyy; default dia anterior), `opcaoPublicacao=1|2|3`.
    - Sucesso: 200 `application/pdf` (attachment com nome: `Intimacao_{tipo}_{data}.pdf`).
    - Não encontrado: 404/409 com mensagem “Não foram encontradas intimações ... em {dataPublicacao}”.
  - GET `/intimacoes/pdf/status` — Query: token/sessão (se adotado); retorna JSON `{"flag":"0|1"}` indicando pronto.
    - Em processamento: 200 `{"flag":"0"}`; pronto: 200 `{"flag":"1"}`.

8.5) Geração de PDF de Publicações por Data (PublicacaoPDFCt)
- Objetivo: gerar PDF concatenado com publicações de uma data específica (2º Grau; 1º Grau Capital; 1º Grau Interior).
- Permissão: base `871` (público). Sem reCAPTCHA.
- Endpoints:
  - POST `/publicacoes/pdf/gerar` — Body/Query: `dataPublicacao` (dd/MM/yyyy; default dia anterior), `opcaoPublicacao=1|2|3`.
    - Sucesso: 200 `application/pdf` (attachment com nome: `Publicacao_{tipo}_{data}.pdf`).
    - Não encontrado: 404/409 com mensagem adequada.
  - GET `/publicacoes/pdf/status` — JSON `{"flag":"0|1"}` para polling de UI.

8.6) JSPs (referência — ignorar UI na migração)
- `ConsultaJurisprudencia.jsp`, `ConsultaPublicacaoElasticSearch.jsp` — telas de consulta/resultados.
- `Padroes/Localizar.jsp` — diálogo genérico de autocomplete/listas.
- `GerarIntimacaoPDF.jsp`, `GerarPublicacaoPDF.jsp` — telas acionadoras de geração de PDFs.
- `ValidarDocumentoPublico.jsp` — tela de validação/consulta por código.
- `Erro.jsp` — apresentação de mensagens de erro públicas.


