## Requisitos – Movimentação de Processo (Genérica)

Escopo: migrar a funcionalidade legado de movimentação genérica de processos para endpoints REST. Cobre preparação, validação e efetivação de movimentações (individual e em lote), geração de pendências e anexos, operações especiais (validar/invalidar) e consultas auxiliares. Ignora detalhes de UI/JSP.

### 1) Permissões
- Código base: `Movimentacao` (`permissao_codigo` 269)
- Subpermissões principais (do legado):
  - `Localizar` (2692) – Listar movimentações/consulta; inclui variação DWR (2693)
  - `Salvar` (2695) – Preparação/confirmar dados antes da efetivação
  - `Curinga6` (2696) – Validar movimentação
  - `Curinga7` (2697) – Invalidar/alterar visibilidade, requer `tipoBloqueio`
  - `Curinga8` (2698) – Tratamentos Ajax (substituído por endpoints REST de upload/lista)
- Permissões auxiliares usadas nas consultas:
  - `MovimentacaoTipo`, `Classificador`, `ArquivoTipo`, `Modelo`, `AreaDistribuicao`, `Serventia`, `ServentiaCargo`, `ProcessoTipo`, `Processo` (consulta). Cada uma segue seu módulo específico.

### 2) Regras e validações
- Preparação/checagem de elegibilidade:
  - Verificar se o(s) processo(s) pode(m) ser movimentado(s) pelo usuário atual: regras de grupo/perfil, situação do processo, restrições de segundo grau e gabinete quando aplicável.
  - Em lote: aceitar vetor de `processos[]` ou seleção por classificador/serventia/cargo.
  - Quando contexto de devolução de precatória/ofício estiver ativo, predefinir tipo de movimentação “OFÍCIO(S) EXPEDIDO(S)” (código 113) e complemento oriundo do contexto.
  - Se houver acesso liberado a outra serventia para o processo, marcar `acessoOutraServentia` para permitir a movimentação cruzada.
- Validações de negócio no confirmar/efetivar:
  - `verificarMovimentacaoGenerica` + `isPodeMovimentarProcesso` obrigatórias antes de salvar.
  - Execução penal: processos de execução penal (exceto ANPP) em serventias subtipo execução penal só podem ser movimentados para arquivamento; demais movimentações devem ser bloqueadas no final com mensagem.
  - Arquivamento: exibir/retornar `mensagemAdvertencia` quando a validação de pedido de arquivamento apontar advertências.
  - Atualização criminal: para tipos “Juntada -> Petição -> Denúncia” e “Decisão -> Recebimento -> Denúncia”, atualizar datas no processo criminal.
- Operações especiais:
  - Validar movimentação: checa `podeMudarStatusMovimentacao` e marca como válida.
  - Invalidar movimentação: idem, exigindo `tipoBloqueio` para restrição de visibilidade/acesso aos arquivos.
- Auditoria obrigatória em todos os endpoints de escrita: `idUsuarioLog`, `ipComputadorLog`.

### 3) Modelos de dados (DTOs)
- Movimentacao (base para request/response):
  - Identificação: `id`, `idMovimentacaoTipo`, `movimentacaoTipo`, `movimentacaoTipoCodigo`
  - Processo: `idProcesso` (único) ou `processos[]` (lote), `processoNumero`, `digitoVerificador`
  - Usuário: `idUsuarioRealizador`, `usuarioRealizador`
  - Conteúdo: `textoEditor` (HTML), `complemento`, `idArquivoTipo`, `arquivoTipo`, `idModelo`, `modelo`
  - Classificador: `idClassificador`, `classificador`, `idClassificadorPendencia`, `classificadorPendencia`
  - Prioridade: `idProcessoPrioridade`, `processoPrioridade`, `processoPrioridadeCodigo`
  - Pendências a gerar: `pendencias[]` (tipos e dados mínimos)
  - Arquivos: `arquivos[]` (nome, tipo, tamanho, hash/assinatura quando aplicável)
  - Fluxo/flags: `multiplo`, `passo1/2/3`, `acessoOutraServentia`, `redirecionaOutraServentia`, `idRedirecionaOutraServentia`
  - Auditoria: `idUsuarioLog`, `ipComputadorLog`, `dataRealizacao`, `palavraChave`
- Encaminhamento (quando aplicável): `idServentia`, `serventia`, `idServentiaCargo`, `serventiaCargo`, `idAreaDistribuicao`, `areaDistribuicao`.

Observações:
- Suportar operação em lote: `processos[]` deve ser permitido em preparar/confirmar/efetivar.
- Campos de apoio (combos) obtidos via endpoints auxiliares.

### 4) Endpoints REST propostos

Preparação/fluxo principal
- POST `/movimentacoes/preparar`
  - Permissão: 269 (Movimentacao)
  - Body: { processos[], idMovimentacaoTipo?, classificador?, idClassificadorPendencia?, encaminhamento?, acessoOutraServentia? }
  - Ação: verifica elegibilidade dos processos, monta contexto inicial, retorna `listaPendenciaTipos` permitidos e quaisquer mensagens de atenção (execução penal, devolução, UPJ/2º grau).
  - Retorno: 200 OK, { contextoMovimentacao, listaPendenciaTipos, mensagens[] }

- POST `/movimentacoes/confirmar`
  - Permissão: 2695 (Salvar)
  - Body: { dados da movimentação + pendencias[], arquivos[] }
  - Ação: `verificarMovimentacaoGenerica`, validações de arquivamento, compila resumo para confirmação (inclui advertências).
  - Retorno: 200 OK, { resumo, mensagensAdvertencia[] }; 400 validação

- POST `/movimentacoes`
  - Permissão: 269 (Movimentacao)
  - Body: { dados finais confirmados da movimentação }
  - Ação: `isPodeMovimentarProcesso` e `salvarMovimentacaoGenerica`. Atualiza datas criminais quando aplicável. Aceita lote.
  - Retorno: 201 Created, { idsMovimentacoes[], processosAfetados[] }; 409 conflito de negócio; 400 validação

- DELETE `/movimentacoes/{id}`
  - Permissão: 269 (Movimentacao) ou específica de exclusão no módulo (se aplicável)
  - Ação: excluir movimentação existente (fluxo genérico legado prevê exclusão).
  - Retorno: 200/204; 404 se inexistente; 403 se sem permissão

Operações especiais
- POST `/movimentacoes/{id}/validar`
  - Permissão: 2696 (Curinga6)
  - Ação: valida movimentação após checagem de permissão.
  - Retorno: 200 OK; 403/404

- POST `/movimentacoes/{id}/invalidar`
  - Permissão: 2697 (Curinga7)
  - Body: { tipoBloqueio }
  - Ação: invalida/ajusta visibilidade, aplicando restrição conforme `tipoBloqueio`.
  - Retorno: 200 OK; 400/403/404

Consultas/listagens
- GET `/movimentacoes`
  - Permissão: 2692 (Localizar)
  - Query: `idProcesso`|`numeroProcesso`, `palavraChave`, `dataDe/Até`, paginação (`pos`, `limit`)
  - Ação: listar movimentações do processo/consulta textual, conforme regras de visibilidade.
  - Retorno: 200 OK, paginação

Auxiliares (combos/apoio)
- GET `/movimentacoes/tipos` – lista tipos por grupo/usuário (permissão do módulo MovimentacaoTipo)
- GET `/classificadores` e `/classificadores/pendencias` – por serventia/UPJ quando aplicável
- GET `/tipos-arquivo` – tipos de arquivo permitidos por grupo
- GET `/modelos` – modelos do usuário/serventia para o `idArquivoTipo`
- GET `/areas-distribuicao`
- GET `/serventias`
- GET `/serventias-cargos` – considerar filtro especial para gabinetes de 2º grau
- GET `/processos-tipo` – classes de processo por serventia
- GET `/processos` – consulta leve para seleção

Observações de lote
- Além de `processos[]`, suportar seleção por classificador/serventia/cargo: endpoint auxiliar
  - GET `/processos/por-classificador` – `idClassificador`, `idServentia`, `idServentiaCargo?`

### 5) Respostas e códigos de status
- 200 sucesso; 201 criação; 204 sem conteúdo
- 400 erro de validação (ex.: campos obrigatórios, formato)
- 403 sem permissão
- 404 não encontrado (movimentação/processo)
- 409 conflito de negócio (bloqueios de estado, regras de execução penal, impedimentos de sessão/2º grau)

### 6) Observações de segurança
- Autenticação e autorização obrigatórias em todos os endpoints.
- Dados sensíveis: arquivos e `textoEditor` devem ser sanitizados; manter trilha de auditoria (`idUsuarioLog`, `ipComputadorLog`).
- Respeitar regras de visibilidade/sigilo ao listar/entregar dados de movimentações e anexos.

### 7) Itens a decidir na migração
- Upload/assinatura de arquivos: fluxo (URL assinada vs. upload direto) e limites (tamanho, tipos permitidos).
- Tratamento de operações “redirecionaOutraServentia”: algumas hoje redirecionam para outros módulos; decidir se ficam como endpoints dedicados neste domínio ou mantidos nos módulos específicos.
- Paginação padrão e ordenação para listagens por processo.
- Escopo de exclusão: permitir ou não apagar movimentações já efetivadas (política e trilha de auditoria).


