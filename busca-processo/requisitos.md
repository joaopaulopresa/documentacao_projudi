## Requisitos – Busca de Processo

Escopo: migrar a funcionalidade de busca e visualização de processos do legado (JSP/Servlet) para endpoints REST. Ignorar fluxos de tela; focar em permissões, regras/validações, modelos e endpoints.

### 1) Permissões
- Código base: `152` – "Busca Processo" (ProcessoDt.CODIGO_PERMISSAO_CONSULTA_PROCESSO)
- Subpermissões (do `busca-processo.json`):
  - `1521` – "IMPRIMIR": Ver Situação do Processo
  - `1522` – "Localizar": Consulta Geral de Processos
  - `1523` – "Localizar Ajax": Consultas auxiliares JSON (DWR no legado)
  - `1526` – "CURINGA6": Download de arquivos do processo
  - `1529` – "Segredo Justiça / Navegação de Arquivos"
- Relacionadas no módulo Processo (quando aplicável):
  - `AudienciaMovimentacaoDt.CodigoPermissaoAudienciaProcesso` (ver/movimentar audiências em situação do processo)
  - `ProcessoParteAdvogadoDt.CodigoPermissao` (habilitar advogados em responsáveis)

### 2) Regras e validações
- ReCAPTCHA obrigatório em consultas públicas:
  - Consulta pública geral e por código de acesso exigem verificação anti-robô.
- Validações de entrada por caso de uso:
  - Código de acesso: `numeroProcesso` obrigatório no formato com `.` ou `-`; `codigoAcesso` obrigatório.
  - Advogado: `oabNumero`, `oabComplemento`, `oabUf` obrigatórios.
  - Inquérito: `inquerito` obrigatório.
  - Precatórios (público): `cnpj` (Fonte Pagadora) obrigatório.
  - Validação de existência de número de processo: aceita flag opcional `desconsideraExecPen`.
- Sigilo e perfis:
  - Consulta a processos sigilosos por número permitida apenas a: Magistrado, Assessor de Magistrado, MP, Delegado. Caso contrário, erro de permissão.
  - Exibição de dados sensíveis condicionada à serventia do usuário ou perfil (ex.: situação do processo e voto/ementa em 2º grau).
- Hash anti-robô para acesso a "e outros" (partes completas):
  - Necessário enviar `codigoHash` válido gerado por sessão/usuário para listar todas as partes do processo.
- Download de arquivos de movimentação:
  - Validação de hash composta por `idMovimentacaoArquivo + idProcesso` e parâmetro `hash`.
  - Parâmetros opcionais: `recibo`, `codigoVerificacao`, `acessibilidade`, `CodigoIA` (URL temporária), `eCarta`, `cnj`/`Comunicacao`.
- Paginação:
  - Todas as listagens devem aceitar paginação (`page`, `pageSize`); respostas devem retornar `quantidadePaginas` quando aplicável.
- Auditoria:
  - Registrar `idUsuarioLog` e `ipComputadorLog` em operações sensíveis (downloads/relatórios/PDFs).

### 3) Modelos de dados (DTO)
- Processo (baseado em `ProcessoDt`/`ProcessoDtGen`):
  - Identificação: `id`, `processoNumero`, `digitoVerificador`, `processoNumeroCompleto`.
  - Classificadores e situação: `id_ProcessoTipo`, `processoTipo`, `id_ProcessoFase`, `processoFase`, `id_ProcessoStatus`, `processoStatus`, `segredoJustica`, `sigiloso`, `prioridade` (código/descrição), `is100Digital`, `processoFisicoTipo`.
  - Localização: `id_Serventia`, `serventia`, `comarca`, `serventiaTipoCodigo`, `serventiaSubTipoCodigo`.
  - Assuntos/objeto: `id_Assunto`, `assunto`, `id_ObjetoPedido`, `objetoPedido`.
  - Valores/datas: `valor`, `dataRecebimento`, `dataArquivamento`, `dataPrescricao`, `dataTransitoJulgado`.
  - Relações: `id_Recurso` + `recursoDt`, listas de partes: `listaPolosAtivos`, `listaPolosPassivos`, `listaOutrasPartes`, `listaAdvogados`.
  - Movimentações: `listaMovimentacoes` (e `listaMovimentacoesFisico`).
  - Indicadores: `existePeticaoPendente`, `existeGuiasPendentes`, `efeitoSuspensivo`, `penhora`, `localizador`.
- Filtros de busca (baseados em `BuscaProcessoDt` usado no controlador):
  - Gerais: `processoNumero`, `nomeParte`, `cpfCnpjParte`, `id_Serventia`, `id_ProcessoTipo`, `id_Assunto`, `id_Classificador`, `isProprio`, `sigiloso`.
  - Advogado: `oabNumero`, `oabComplemento`, `oabUf`.
  - Público/precatórios: `cnpj`, `fontePagadora`.
  - Inquérito/execução penal: `inquerito`, `idProcessoExecucao`.
  - Responsáveis/relator: `id_Relator`, `id_UsuarioServentia`.

### 4) Endpoints REST propostos

Preparação
- POST `/busca-processos/preparar`
  - Permissão: 152
  - Body: `{ tipoConsultaProcesso, idServentia? }`
  - Ação: inicializa contexto padrão da busca (define serventia atual quando aplicável). Retorna dados mínimos para UI de filtros.

Consultas/listagens
- GET `/processos`
  - Permissão: 1522
  - Query: filtros gerais (vide DTO), `page`, `pageSize`.
  - Retorno: lista paginada + `quantidadePaginas`.
- GET `/processos/publico`
  - Permissão: pública controlada; exige ReCAPTCHA válido.
  - Query: filtros análogos aos da pública; `page`, `pageSize`.
- GET `/processos/precatorios-publico`
  - Permissão: pública; ReCAPTCHA recomendado.
  - Query: `cnpj` (obrigatório), `fontePagadora?`, `page`, `pageSize`.
- GET `/processos/codigo-acesso`
  - Permissão: pública; ReCAPTCHA obrigatório.
  - Query: `numeroProcesso` (formato com `.` ou `-`), `codigoAcesso`.
  - Ação: valida e retorna o processo único; restringe campos conforme LGPD/sigilo.
- GET `/processos/juiz`
  - Permissão: 1522
  - Query: filtros por responsável/juiz; `sigiloso=true` permite busca por número sigiloso se perfil autorizado.
- GET `/processos/advogado`
  - Permissão: 1522
  - Query: `oabNumero`, `oabComplemento`, `oabUf` (obrigatórios); `page`, `pageSize`.
- GET `/processos/defensor-escritorio`
  - Permissão: 1522
  - Query: `idServentia`, `idUsuarioServentia` (obrigatórios); `page`, `pageSize`.
- GET `/processos/inquerito`
  - Permissão: 1522
  - Query: `inquerito` (obrigatório); `page`, `pageSize`.
- GET `/processos/relator`
  - Permissão: 1522
  - Query: `idRelator`, `idProcessoTipo?`; `page`, `pageSize`.
- GET `/processos/arquivados/sem-motivo`
  - Permissão: 1522
  - Query: `idServentia`, `page`, `pageSize`.
- GET `/processos/inconsistencias/polo-passivo`
  - Permissão: 1522
  - Query: `idServentia`, `page`, `pageSize`.
- GET `/processos/guias/atraso`
  - Permissão: 1522; `idServentia`, `page`, `pageSize`.
- GET `/processos/contas/saldo`
  - Permissão: 1522; `idServentia`, `page`, `pageSize`.
- GET `/processos/prisao/fora-prazo`
  - Permissão: 1522; dados do usuário (juiz) considerados; `page`, `pageSize`.
- GET `/processos/prescritos`
  - Permissão: 1522; `idServentia`, `idComarca`, `page`, `pageSize`.
- GET `/processos/encaminhados`
  - Permissão: 1522; `idServentia`, `page`, `pageSize`.
- GET `/processos/recursos`
  - Permissão: 1522; `idServentia`, `page`, `pageSize`.
- GET `/processos/sem-assunto`
  - Permissão: 1522; `idServentia`, `page`, `pageSize`.
- GET `/processos/com-assunto-pai`
  - Permissão: 1522; `idServentia`, `page`, `pageSize`.
- GET `/processos/com-classe-pai`
  - Permissão: 1522; `idServentia`, `page`, `pageSize`.
- GET `/processos/com-reus-presos`
  - Permissão: 1522; contexto do usuário; `page`, `pageSize`.
- GET `/processos/outras-informacoes`
  - Permissão: 1523
  - Query: filtros da listagem + indicadores adicionais (mesmo do legado DWR); retorno JSON.

Dados do processo e agregados
- GET `/processos/{id}`
  - Permissão: 152
  - Retorna dados completos do processo (ajustados por sigilo/perfil/nivel de acesso).
- GET `/processos/{id}/situacao`
  - Permissão: 1521
  - Retorna pendências abertas, conclusões, audiências (e voto/ementa quando aplicável ao perfil/serventia).
- GET `/processos/{id}/responsaveis`
  - Permissão: 152
  - Retorna responsáveis (juiz/relator/advogados) e desabilitados quando perfil permitir.
- GET `/processos/{id}/partes`
  - Permissão: 152; requer `codigoHash` válido na query.
  - Retorna todas as partes (inclusive "e outros").
- GET `/processos/{id}/execucao-penal/eventos`
  - Permissão: 152
  - Retorna lista de eventos da execução penal e metadados (atestado disponível, históricos PSC/LFS).
- GET `/processos/{id}/execucao-penal/atestado.pdf`
  - Permissão: 152
  - Retorna PDF do último atestado de pena a cumprir (gera se necessário).

Downloads e relatórios
- GET `/processos/{id}/movimentacoes/{idMov}/arquivo`
  - Permissão: 1526; requer `hash` válido.
  - Query: `recibo?`, `codigoVerificacao?`, `acessibilidade?`, `codigoIA?`.
  - Variações:
    - `eCarta=true` → PDF envelope e-Carta.
    - `cnj=true&comunicacao=...` → PDF CNJ (domicílio eletrônico).
- GET `/processos/relatorios/reus-presos.pdf`
  - Permissão: 1522 (ou específica); gera relatório de réus presos por serventia/responsável.

Verificações auxiliares e combos
- GET `/processos/validar-numero`
  - Permissão: 1522; Query: `numProcesso`, `desconsideraExecPen?`; 200 quando existe, 404 quando não.
- GET `/tipos-processo`, `/assuntos`, `/areas`, `/serventias`, `/classificadores`, `/serventia-cargos`, `/usuarios-serventia`
  - Permissão: 1523/1522
  - Query: `search` e parâmetros específicos (ex.: `outrasServentias=true`, `primeiroGrau=1`).

### 5) Respostas e códigos de status
- 200 sucesso; 206 conteúdo parcial (downloads grandes, se aplicável); 201 não se aplica (sem criação aqui).
- 400 validação (campos obrigatórios, formato do número do processo, OAB incompleta, etc.).
- 403 permissão (sigiloso sem perfil, hash inválido, sem permissão de download/situação).
- 404 recurso não encontrado (processo inexistente, número inválido, combos vazios).
- 409 conflito de negócio (quando regras do domínio impedirem execução).

### 6) Observações de segurança
- Autenticação e autorização por permissão e perfil/grupo (1º/2º grau, gabinete, MP, delegado, estagiário).
- Dados sensíveis (sigilo, vítimas/menores) devem ser mascarados quando aplicável; em acesso público limitar campos expostos.
- ReCAPTCHA obrigatório em consultas públicas para mitigação de scraping.
- Hash anti-robô obrigatório para endpoints de partes completas e alguns downloads.
- Auditoria completa: `idUsuarioLog` e `ipComputadorLog` nos downloads/relatórios e operações críticas.

### 7) Itens a decidir na migração
- Substituição do mecanismo de `codigoHash` por token assinado por servidor (curto prazo de expiração) e escopo por processo.
- Especificação do payload mínimo do endpoint de "preparar" (listas auxiliares iniciais?).
- Política de paginação padrão (tamanho de página e limites máximos) e forma de retorno de `quantidadePaginas`.
- Campos expostos nos modos público/código de acesso (LGPD) e granularidade por perfil.
- Unificação de variações de download em um endpoint com parâmetros vs. múltiplos endpoints específicos.
- Rate limiting adicional além de ReCAPTCHA para endpoints públicos.


