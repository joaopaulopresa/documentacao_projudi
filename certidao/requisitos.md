## Requisitos – Certidão (Emissão/Consulta/Validação)

Escopo: migrar os fluxos legados (JSP/Servlet) do módulo de Certidões para endpoints REST. Cobre emissão por guia (narrativa, prática forense, negativa/positiva, licitação, interdição, autoria, execução), certidões específicas (execução CPC, circunstanciada), antecedentes/criminais, negativa/positiva (1º e 2º grau), variações públicas, validação/impressão de PDFs, consultas auxiliares (estado civil, profissão, cidade, usuários/serventia, modelos) e processamento assíncrono (batch).


### 1) Permissões

- Código base: `447` – Certidão

- Subpermissões relevantes (extraídas de certidao.json):
  - Gerar Certidão (`905`)
    - `9051` IMPRIMIR
    - `9052` LOCALIZAR
    - `9053` LOCALIZARDWR (Ajax)
    - `9054` NOVO (preparar)
    - `9055` SALVAR (emitir)
    - `9056` CURINGA6 (pré-visualizar)
    - `9057` CURINGA7
    - `9058` CURINGA8
    - `9059` CURINGA9
  - Guias de Certidão (`661`)
    - `6611` IMPRIMIR
    - `6612` Localizar
    - `6613` Localizar Ajax
    - `6614` NOVO
    - `6615` SALVAR
    - `6616` CURINGA6
    - `6617` CURINGA7
    - `6618` CURINGA8
    - `6619` CURINGA9
  - Itens globais baseados no módulo Certidão:
    - `4471` IMPRIMIR
    - `4472` LOCALIZAR
    - `4473` LOCALIZARDWR
    - `4475` SALVAR
    - `4470` ExcluirResultado
  - Acesso público: respeita `GrupoDt.ACESSO_PUBLICO` quando o endpoint for público (ex.: validação/emissão por guia pública, consultas públicas de negativa/positiva).

Observação: as operações Ajax de consulta (combos) usam o padrão `<PermissaoDoDTO>.CodigoPermissao * QtdPermissao + Configuracao.Localizar` no legado; na migração, padronizar como GETs abertos para perfis com a permissão base do DTO requerido.


### 2) Regras e validações de negócio

- Emissão por Guia (1º Grau) – `CertidaoGuiaCt`/`CertidaoGuiaPublicaCt`
  - Verificar se o processo é sigiloso/segredo de justiça quando acesso público; se for, bloquear com mensagem explícita.
  - Conferir pagamento da guia (Itau/SPG) e se já foi utilizada. Bloquear emissão quando não paga; quando já utilizada, disponibilizar o PDF emitido (download) sem gerar novo.
  - Conferir serventia da guia versus serventia do usuário quando não público; se divergir, bloquear.
  - Restrições por perfil/área: distribuidor Cível não utiliza guia Criminal e vice-versa.
  - Certidão Negativa/Positiva (online): se houver processos e o usuário for público, bloquear emissão positiva (orientar emissão no distribuidor).
  - Batch/relatório: quando tipo for relatório (não online), disparar geração assíncrona, com polling de status antes da entrega do arquivo.
  - Auditoria: em todas as gravações, preencher `idUsuarioLog` e `ipComputadorLog`.

- Certidão Narrativa (com ou sem custas)
  - Validar CPF/CNPJ quando informado.
  - Incluir movimentações do processo no corpo do texto antes de montar o modelo.

- Certidão Prática Forense
  - Modos: Quantitativa (retorna quantidade total de processos no período/âmbito) e Descritiva (lista processos). Pode somar resultados de SPG.
  - Exige OAB (número, complemento, UF) e período.

- Certidão Negativa/Positiva – 1º Grau (consulta pública)
  - Validar obrigatórios mínimos (nome/cpf/data conforme regra do negócio); executar verificação prévia `VerificarCertidaoNegativaPositivaPublica`.
  - Consultar Projudi + SPG; considerar flags de integração SEEU quando Criminal.
  - Calcular valores de custas e exibir formato monetário.

- Certidão Negativa/Positiva – 2º Grau
  - Seleção por área (Cível/Criminal), pessoa (Física/Jurídica) e período; montar texto a partir do modelo conforme há ou não processos.

- Certidão Execução CPC
  - Localizar por número do processo; montar texto e disponibilizar PDF.

- Certidão Circunstanciada (Execução)
  - Pesquisa de sentenciado por parâmetros combinados; lista processos; seleção carrega dados e monta texto de modelo (editor). Consultas de modelos por tipo de arquivo.

- Combos/Consultas auxiliares
  - Estado Civil, Profissão, Cidade, Usuário/Serventia, Modelos: GET com filtros, paginação simples (posições) e retorno JSON.

- Validações gerais
  - Formatos: CPF/CNPJ (com dígitos), datas válidas, número de processo quando aplicável.
  - Perfis/grupos: regras especiais para distribuidor, juventude infracional (exibir/check “menor infrator”), acesso público.
  - LGPD: dados pessoais sensíveis (nome, CPF, documentos, textos) devem ser protegidos; logs e arquivos acessíveis apenas a perfis autorizados.
  - Auditoria obrigatória em toda emissão/salvamento de certidão: `idUsuarioLog`, `ipComputadorLog`.


### 3) Modelos de dados (DTOs principais)

- CertidaoGuia
  - Identificação do requerente: `nome`, `cpf`, `cnpj`, `tipoPessoa`, `naturalidade(id/descricao)`, `sexo`, `dataNascimento`, `rg`, `rgOrgaoExpedidor(id/sigla)`, `rgDataExpedicao`, `estadoCivil(id/descricao)`, `profissao(id/descricao)`, `domicilio`, `comarca`.
  - Guia/modelo/contexto: `numeroGuia`, `id_Comarca`, `id_Modelo`, `id_Guia`, `id_GuiaTipo`, `tipoCertidao` (ONLINE/RELATORIO), `pessoaJuridica`, `fimJudicial`, `averbacaoCusta`.
  - Processo: `idProcesso`, `protocolo`, `juizo`, `natureza`, `valorAcao`, `requerenteProcesso`, `advogadoRequerente`, `requerido`, `advogadoRequerido`, `fase`, `dataSentenca`, `dataBaixa`, `dataDistribuicao`, `movimentacoesProcesso`, `listaProcessos`.
  - Prática Forense: `oab`, `oabComplemento`, `oabUf`, `oabUfCodigo`, `textoForense`, `anoInicial/final`, `mesInicial/final`, `tipo` (Quantitativa/Descritiva), `quantidade`, `modeloCodigo`.
  - Narrativa: `textoCertidao`, `texto`.
  - Custas: `custaCertidao`, `custaTaxaJudiciaria`, `custaTotal`, `guiaEmissaoDt`, `dadosCertidaoGravadosSPG`, `quantidadeSPG`.
  - Integrações/flags: `integracaoSEEUHabilitada`, `sucessoIntegracaoSEEU`.
  - Responsáveis: `id_UsuarioEscrivaoResponsavel`, `nomeUsuarioEscrivaoResponsavel`.

- CertidaoNegativaPositiva
  - Dados pessoais/processuais: `nome`, `cpfCnpj`, `tipoPessoa`, `dataNascimento`, `nomeMae`, `nomePai`, `sexo`, `identidade`, `estadoCivil`, `profissao`, `domicilio`, `nacionalidade`.
  - Contexto: `serventia`, `comarca`, `id_comarca`, `comarcaCodigo`, `areaCodigo` (Cível/Criminal), `Territorio` (E=Estadual, C=Comarca), `Finalidade`, `NumeroGuiaCertidao`, `InteressePessoal`.
  - Listas/documentos: `listaProcessos`, `listaProcessosRequerente`, `listaNomesComarcasComProcessos`, `documento` (bytes), `nomeDocumento`.
  - Integrações/flags: `integracaoSEEUHabilitada`, `sucessoIntegracaoSEEU`, `pessoaJuridica`, `certidaoEmitida`, `tipoGuia`, `id_Guia`, `id_GuiaStatus`.
  - Texto/valores: `texto`, `valorTaxa`, `valortotal`, `valorCertidao`, `dataApresentacao`, `dataPagamento`.

- CertidaoAntecedenteCriminal
  - Pessoais: `nome`, `nomePai`, `nomeMae`, `naturalidade(id/descricao)`, `profissao(id/descricao)`, `estadoCivil`, `dataNascimento`, `sexo`, `identidade`, `cpfCnpj`, `domicilio`, `nacionalidade`, `rg`.
  - Flags/contexto: `comarcaCodigo`, `comarca`, `chkMenorInfrator`, `listaProcessosImprimirCertidao`, `retornoProcessoAntecedenteCriminalDt`.
  - Texto: `texto`.

- CertidaoSegundoGrauNegativaPositiva
  - `nome`, `nomeMae`, `dataNascimento`, `area`, `dataInicial`, `dataFinal`, `feito`, `cpfCnpj`, `pessoaTipo` (Física/Jurídica), `listaProcesso`, `texto`.

- CertidaoExecucaoCPC
  - `numero`, `serventia`, `natureza`, `promovente`, `advogadoPromovente`, `promovido`, `cpfCnpj`, `dataDistribuicao`, `valor`, `texto`.

- Certidao (Circunstanciada)
  - `requerente`, `cpfCnpj`, `domicilio(id/descricao)`, `rg`, `naturalidade(id/descricao)`, `estadoCivil(id/descricao)`, `sexo`, `profissao(id/descricao)`, `nomeMae`, `nomePai`, `dataNascimento`.
  - Contexto/modelo: `comarcaCodigo`, `certidaoTipo`, `processoNumeroCompleto`, `numeroGuia`, `id_Modelo`, `modelo`, `listaProcesso`.

Observações: todos os DTOs suportam preenchimento incremental (limpar/parcial), e convertem formatos monetários e datas para exibição. Os modelos são identificados por códigos inteiros associados a tipos de certidão e ao resultado (negativa/positiva; física/jurídica; 1º/2º grau; integração SEEU).


### 4) Endpoints REST propostos

Agrupados por caso de uso. Todos retornos em JSON, exceto quando indicado (PDF/bytes). Onde aplicável, suportar operação em lote (vetores) e paginação simples.

- Emissão por Guia (1º Grau)
  - POST `/certidoes/guias/preparar`
    - Permissão: 447/9054 (NOVO)
    - Ação: iniciar contexto limpo; preencher `id_UsuarioEscrivaoResponsavel` pelo usuário logado; retornar DTO base.
  - GET `/certidoes/guias`
    - Permissão: 447/9052 (LOCALIZAR)
    - Query: `numeroGuia`, `tipoCertidao` (O/R)
    - Ações/regra: validações (paga, já utilizada, serventia compatível, sigilo quando público); quando N/P online, consulta processos; para relatórios, disparar tarefa assíncrona.
    - Retorno: DTO completo (com `texto` se aplicável) e flags de status; em caso de positiva pública, retornar erro de negócio 409 com mensagem específica.
  - POST `/certidoes/guias/previa`
    - Permissão: 447/9056 (CURINGA6)
    - Body: DTO com campos necessários
    - Ação: gerar `texto` (Narrativa inclui movimentações; Prática Forense invoca cálculo/lista); retorno DTO com `texto`.
  - POST `/certidoes/guias/emitir`
    - Permissão: 447/9055 (SALVAR)
    - Body: DTO com `texto` montado
    - Ações: persistir `CertidaoValidacao` (30 dias validade), preencher auditoria e retornar PDF (bytes) e metadados (código de verificação).
    - Respostas: 201 (criado + arquivo); 400 validação; 409 conflito (guia já utilizada).
  - DELETE `/certidoes/guias/{numeroGuia}`
    - Permissão: 4470 (ExcluirResultado)
    - Ação: excluir certidão emitida por guia; validações de área/serventia; retorno 200 com mensagem.
  - PATCH `/certidoes/guias/processos/remove`
    - Permissão: 447/9052 (LOCALIZAR_AUTO_PAI)
    - Query: `index`
    - Ação: remover processo da lista da certidão em preparo; retornar DTO atualizado.
  - GET `/usuarios-serventia`
    - Permissão: `UsuarioServentiaDt` localizar
    - Query: `nome`, `posicao`
    - Ação: consulta de usuários da serventia do logado para preencher escrivão(ã) responsável.
  - Fluxo assíncrono (relatórios):
    - POST `/certidoes/guias/relatorios`
      - Inicia processamento em background; retorna `jobId`.
    - GET `/certidoes/guias/relatorios/{jobId}/status`
      - Retorna `ok`|`ainda-nao-terminou` e tempo decorrido.
    - GET `/certidoes/guias/relatorios/{jobId}/arquivo`
      - Entrega PDF quando pronto.

- Validação/Emissão Pública por Guia
  - GET `/publico/certidoes/validar`
    - Query: `numeroGuia`
    - Ação: verificar existência, pagamento e emissão; retornar status: `inexistente` | `nao_paga` | `certidao_nao_emitida` | `ok`.
  - GET `/publico/certidoes/arquivo`
    - Query: `numeroGuia`
    - Ação: quando emitida, retornar PDF publicado.

- Certidão Narrativa Sem Custas
  - POST `/certidoes/narrativa-sem-custas/preparar`
    - Ação: iniciar contexto com `idProcesso` (se fornecido); carregar movimentações.
  - GET `/certidoes/narrativa-sem-custas`
    - Query: `idProcesso`
    - Ação: preencher dados do processo e movimentações; custas zeradas.
  - POST `/certidoes/narrativa-sem-custas/previa`
    - Body: `Nome`, `Cpf`, `textoCertidao`
    - Ação: validar CPF; montar `texto` por modelo.
  - POST `/certidoes/narrativa-sem-custas/emitir`
    - Ação: persistir validação e retornar PDF.

- Certidão de Antecedentes/Informação Criminal (Criança/Adolescente quando aplicável)
  - POST `/certidoes/antecedentes/preparar`
    - Permissão: 447/9054 (NOVO)
    - Ação: iniciar contexto limpo; definir `chkMenorInfrator=false` por padrão; retornar DTO base.
  - GET `/certidoes/antecedentes`
    - Permissão: 447/9052 (LOCALIZAR)
    - Query: `Nome` (obrigatório), `Cpf`, `DataNascimento`, `NomeMae`, `NomePai`, `Id_Naturalidade`, `Rg`, `Sexo`, `chkMenorInfrator`
    - Ação: validações (CPF/Data); recuperar lista de processos (considerando Juventude Infracional quando aplicável); retornar DTO com `retornoProcessoAntecedenteCriminalDt` e `bloquearCampos=true`.
  - POST `/certidoes/antecedentes/emitir`
    - Body: DTO incluindo `listaProcessosCertidao` (seleção) e flags `impressaoTelaConsultaCertidao`, `chkMenorInfrator`
    - Ações: montar texto conforme perfil/SEEU e flags; quando não menor infrator e emissão pela tela de consulta, persistir validação (30 dias) e retornar PDF publicado; demais casos, retornar PDF direto.
  - PATCH `/certidoes/antecedentes/processos/remove`
    - Query: `index`
    - Ação: remover item da lista de processos do DTO em preparo e re-montar `texto`/modelo.

- Certidão Prática Forense
  - POST `/certidoes/pratica-forense/preparar`
  - GET `/certidoes/pratica-forense`
    - Query: `tipo` (Quantitativa|Descritiva), `oabNumero`, `oabComplemento`, `oabUf`, `periodo`
    - Ação: calcular quantidade (quantitativa) ou listar processos (descrita, Projudi+SPG), montar `texto`.
  - POST `/certidoes/pratica-forense/emitir`
    - Ação: gerar PDF (sem persistência de validação quando não aplicável ao fluxo vigente).

- Certidão Negativa/Positiva – 1º Grau (pública)
  - POST `/certidoes/negativa-positiva/preparar`
  - GET `/certidoes/negativa-positiva`
    - Query: `id_Comarca`, `Comarca`, `Nome`, `Cpf`, `NomeMae`, `DataNascimento`, `Area` (default Cível)
    - Ação: validações (pré-verificação), consulta Projudi + SPG, definir integrações (SEEU), calcular valores; retorna listas e indicadores.

- Certidão Negativa/Positiva – 2º Grau
  - POST `/certidoes/segundo-grau/np/preparar`
  - GET `/certidoes/segundo-grau/np`
    - Query: `Nome`, `NomeMae`, `DataNascimento`, `Area`, `DataInicial`, `DataFinal`, `Feito`, `PessoaTipo`, `Cpf`
    - Ação: consultar processos (SSG + repositório), montar `texto` pelo modelo adequado.
  - PATCH `/certidoes/segundo-grau/np/processos/remove`
    - Query: `index`
    - Ação: remover processo da lista e re-montar `texto`.
  - POST `/certidoes/segundo-grau/np/emitir`
    - Ação: persistir validação e entregar PDF `Certidao{cpfCnpj}`.

- Certidão Execução CPC
  - POST `/certidoes/execucao-cpc/preparar`
  - GET `/certidoes/execucao-cpc`
    - Query: `Numero`
    - Ação: recuperar processo; montar `texto`.
  - POST `/certidoes/execucao-cpc/emitir`
    - Ação: gerar PDF.

- Certidão Circunstanciada (Execução)
  - POST `/certidoes/circunstanciada/preparar`
  - GET `/certidoes/circunstanciada`
    - Query: parâmetros de filtro (CPF/CNPJ, Nome, NomeMae, DataNascimento, NumeroProcesso)
    - Ação: validar filtros; listar sentenciados/processos.
  - GET `/certidoes/circunstanciada/modelos`
    - Query: `arquivoTipo=certidao-circunstanciada`, `nome`, `posicao`
    - Ação: listar modelos (paginado) para seleção.
  - POST `/certidoes/circunstanciada/editor`
    - Body: seleção de processo/modelo
    - Ação: recuperar processo e montar `texto` completo para edição (retornar `TextoEditor`).

- Consultas auxiliares (combos)
  - GET `/combos/estado-civil?nome=<..>&posicao=<..>`
  - GET `/combos/profissao?nome=<..>&posicao=<..>`
  - GET `/combos/cidade?cidade=<..>&uf=<..>&posicao=<..>`
  - GET `/usuarios-serventia?nome=<..>&posicao=<..>`
  - GET `/modelos?arquivoTipo=<..>&nome=<..>&posicao=<..>`


### 5) Respostas e códigos de status

- 200: sucesso (consultas; remoções de item da lista; prévias geradas).
- 201: criação/ emissão de certidão com persistência (`/emitir`) retornando metadados (código verificação) e/ou PDF.
- 400: validação de entrada (CPF/Data inválidos; campos obrigatórios não informados; OAB incompleta; guia não informada).
- 403: sem permissão (área/serventia incompatíveis; perfis inadequados; tentativa de emissão positiva no público).
- 404: não encontrado (guia inexistente; modelo inexistente; processo não localizado).
- 409: conflito de negócio (guia já utilizada; certidão positiva no fluxo público; guia de área divergente do perfil).


### 6) Observações de segurança

- Autenticação obrigatória para todos os endpoints, exceto os explicitamente públicos (`/publico/*` e consultas públicas da negativa/positiva) – ainda assim com limitação de dados sensíveis.
- Autorização baseada nas permissões mapeadas (base 447 e subpermissões); checagens adicionais por perfil/grupo (distribuidor cível/criminal; juventude infracional) e por serventia.
- Dados sensíveis: nomes, CPF/CNPJ, documentos e PDFs devem seguir LGPD; logs com IP e usuário; evitar exposição de dados de processos sob sigilo.
- PDFs gerados devem ter controle de validade (30 dias) e número/código de verificação quando aplicável.


### 7) Itens a decidir na migração

- Estrutura de job para relatórios (batch): adotar filas/worker e padronizar o polling (status/arquivo) ou usar Webhook/WebSocket.
- Persistência dos modelos e seleção em editor: padronizar contrato do endpoint de modelos e do payload de montagem.
- Padronizar nomenclatura dos tipos de guia (Narrativa, Prática Forense, Negativa/Positiva, Licitação, Interdição, Autoria, Execução) e seus IDs provenientes do SPG/Projudi.
- Unificar formatação monetária e de datas no backend (evitar regras no cliente).
- Estratégia de versionamento dos modelos (códigos variam por tipo/resultado e por integração SEEU).


### 8) Notas de mapeamento legado → REST

- `NOVO` → POST `/preparar`
- `LOCALIZAR` → GET listagens/consultas com filtros via query string
- `CURINGA6` → POST `/previa` (pré-visualização/geração de texto)
- `SALVAR` → POST `/emitir` (persistir/entregar PDF)
- `EXCLUIR`/`EXCLUIRRESULTADO` → DELETE ou POST dedicado de exclusão/confirmar-exclusão
- `IMPRIMIR` → emissão/entrega de PDF. Nos casos batch, dividir em status + arquivo.


Referências no legado
- Controladores: `CertidaoGuiaCt`, `CertidaoGuiaPublicaCt`, `CertidaoNarrativaSemCustasCt`, `CertidaoPraticaForenseCt`, `ConsultaCertidaoNegativaPositivaCt`, `CertidaoSegundoGrauNegativaPositivaCt`, `CertidaoExecucaoCPCCt`, `CertidaoExecucaoCt`.
- DTOs: `CertidaoGuiaDt`, `CertidaoNegativaPositivaDt`, `CertidaoAntecedenteCriminalDt`, `CertidaoSegundoGrauNegativaPositivaDt`, `CertidaoExecucaoCPCDt`, `CertidaoDt`/`CertidaoDtGen`.


