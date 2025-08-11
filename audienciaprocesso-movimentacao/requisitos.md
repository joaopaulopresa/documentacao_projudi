## Requisitos – Movimentação de Audiência/Processo

### Escopo e objetivo
Documentar a funcionalidade de movimentação de processos vinculados a audiências/sessões (inclui pré-análise, inserção/alteração de extrato da ata, encaminhamentos, votar por maioria, pendências geradas, movimentação em lote), visando migração a endpoints REST. Ignorar navegação/telas.

Fontes: `AudienciaProcessoMovimentacaoCt.java`, DTO `AudienciaMovimentacaoDt.java` e permissões em `audienciaprocesso-movimentacao.json`.

### Permissões relevantes
- Código base: **395 – AudienciaProcesso Movimentação** (classe usa `AudienciaMovimentacaoDt.CodigoPermissaoAudienciaProcesso`)
- Subpermissões:
  - **3954 – NOVO**: Iniciar movimentação (carregar dados da audiência/processo; preparar lote)
  - **3955 – SALVAR**: Confirmar movimentação (persistir alterações e pendências)
- Consultas auxiliares dependem de permissões de cada entidade relacionada: `Classificador`, `ArquivoTipo`, `Modelo`, `Serventia`, `ServentiaCargo`, `ProcessoParteAdvogado`, `MovimentacaoTipo` etc.

### Regras e validações principais
- Tipos de movimentação (parâmetro `TipoAudienciaProcessoMovimentacao`):
  - “1” pré-análise; “3” início de sessão; “4” sessão adiada; “5” retirada de pauta; “6” desmarcada. Ajusta flags no DTO.
- Cenários suportados:
  - Inserir/alterar extrato da ata de julgamento (validações específicas).
  - Pré-análise de sessão (guardar para assinar, redistribuição opcional de voto/ementa).
  - Movimentação individual ou em lote (conjunto de `audicenciasProcesso`).
  - Marcar julgado mérito, acordo/valor, classificador, modelo/arquivo, ementa (quando aplicável).
  - Encaminhamentos e verificação de processo (flags).
  - Sustentação oral (incluir/remover advogados).
- Segundo grau (sessões):
  - Regras e perfis específicos (magistrado 2º grau, gabinete, UPJ). Chama métodos `salvarMovimentacaoAudienciaProcessoSessaoSegundoGrau` ou `salvarMovimentacaoAudienciaProcesso` conforme tipo.
  - Criação/alteração de “Elaboração de Voto” (pendência) quando indicado.
- Validações antes de persistir:
  - `verificarMovimentacaoAudienciaProcesso` e `podeMovimentarAudiencia` (ou `verificarMovimentacaoAudiencia` nos casos de ata).
- Auditoria obrigatória: `idUsuarioLog`, `ipComputadorLog`.

### Modelo de dados (principais campos)
Do `AudienciaMovimentacaoDt`:
- Audiência/processo: `audienciaDt` (inclui `audienciaProcessoDt` e `processoDt`).
- Status e listas: `audienciaStatusCodigo`, `audienciaStatus`, `listaAudienciaProcessoStatus`.
- Arquivo/Texto: `idArquivoTipo`, `arquivoTipo`, `idModelo`, `modelo`, `textoEditor`, `listaArquivos`.
- Classificador/Relatório: `idClassificador`, `classificador`, `idRelatorio`, `hashRelatorio`.
- Ementa: `idArquivoTipoEmenta`, `arquivoTipoEmenta`, `idModeloEmenta`, `modeloEmenta`, `textoEditorEmenta`, `nomeArquivo`, `nomeArquivoEmenta`.
- Sessão (remarcação): `idNovaSessao`, `dataNovaSessao`, `listaSessoesAbertas`.
- Passos: `passo1`, `passo2`, `passo3`.
- Flags/controle: `ehPreAnalise`, `alteracaoExtratoAta`, `inserirExtratoAtaJulgamento`, `votoPorMaioria`, `pendenteAssinatura`, `isMovimentacaoSessaoDesmarcada`, `isMovimentacaoSessaoRetiradaDePauta`, `encaminhamento`, `verificarProcesso`.
- Composição de turma: presidente, relator, votantes (1º–4º), MP, redator.
- Sustentação oral: `sustentacaoOral`, `listaAdvogadosSustentacaoOral`, inclusão/remoção.
- Pendências geradas: `listaPendenciasGerar`, `idPendenciaVotoGerada`, `idPendenciaEmentaGerada`, `idServentiaCargoVotoEmentaGerada`.
- Movimentação tipo: `idMovimentacaoTipo`, `movimentacaoTipo`.

---

## Endpoints REST propostos

### 1) Preparar movimentação
- POST `/audiencias-processos/movimentacao/preparar`
  - Permissão: 3954 (NOVO)
  - Body: `{ idAudienciaProcesso }` ou `{ audienciasProcesso: [ids] }`, `tipoAudienciaProcessoMovimentacao?`, `fluxo?`, flags opcionais
  - Ação: carrega dados completos para movimentar (individual ou lote), popula listas auxiliares (status, tipos de pendência), define flags.

### 2) Confirmar movimentação
- POST `/audiencias-processos/movimentacao/confirmar`
  - Permissão: 3955 (SALVAR)
  - Body: campos relevantes do `AudienciaMovimentacaoDt` (status selecionado, arquivos/listas, texto/ementa, composição, flags, pendências a gerar)
  - Regras: valida com `verificarMovimentacaoAudienciaProcesso` e `podeMovimentarAudiencia`; salva conforme tipo (sessão 2º grau x demais; individual x lote).
  - Retornos distintos:
    - Sessão 2º grau: mensagens específicas (início/adiamento/realizada), handling de pré-análise com redistribuição opcional.
    - Lote: retorna lista atualizada de sessões abertas quando aplicável.

### 3) Operações específicas
- POST `/audiencias-processos/movimentacao/extrato-ata/validar`
  - Valida inclusão/alteração do extrato da ata para o contexto informado.

- POST `/audiencias-processos/movimentacao/elaboracao-voto`
  - Cria/atualiza pendência de elaboração de voto para o processo da sessão, quando aplicável.

- POST `/audiencias-processos/movimentacao/descartar-pre-analise`
  - Descarta pré-análise pendente da sessão.

- POST `/audiencias-processos/movimentacao/sustentacao-oral/adicionar`
  - Body: `{ idProcessoParteAdvogado, presente, dispensou }`
  - Ação: adiciona advogado à sustentação oral da sessão.

- POST `/audiencias-processos/movimentacao/sustentacao-oral/remover`
  - Body: `{ index }` ou `{ idProcessoParteAdvogado }`
  - Ação: remove advogado da lista de sustentação oral.

### 4) Consultas auxiliares
- GET `/classificadores?query=&serventiaId=&fluxoUPJ=`
- GET `/arquivos-tipo?query=`
- GET `/modelos?query=&idArquivoTipo=&ementa=S|N`
- GET `/serventias?tipo=promotoria&query=`
- GET `/serventia-cargos?contexto=presidente|relator|votante1|votante2|votante3|votante4|mp|redator&serventiaId=&serventiaTipo=&filtroEncaminhamentoGabinete=`
- GET `/processos/{id}/advogados?query=`
- GET `/movimentacoes-tipo?query=`
  - Permissões: conforme entidades referenciadas; filtros conforme sessão/usuario (serventia/subtipo/grupo).

---

## Respostas e códigos de status (resumo)
- 200: sucesso em preparar/confirmar, validações e operações específicas.
- 201: não aplicável (movimentação altera estado existente; se gerar pendência nova, 201 opcional com payload).
- 400: parâmetros inválidos ou inconsistentes (flags, ids, combinações).
- 403: usuário sem permissão/perfil para a ação.
- 404: entidade não localizada (audiência/processo, modelo, arquivo-tipo, cargo, advogado).
- 409: conflito de negócio (ex.: não pode movimentar no estado atual; validações da ata; regras de 2º grau).

## Observações de segurança
- Autenticação e checagem de permissão (395/3954/3955) e perfil (2º grau, gabinete, UPJ, analista, magistrado).
- Auditoria com `idUsuarioLog` e `ipComputadorLog`.
- Dados textuais de extrato/voto/ementa possivelmente sensíveis: proteger em trânsito/armazenamento; versionar e registrar autor/tempo.

## Itens a decidir na migração
- Estratégia transacional para operações em lote (consistência x desempenho).
- Formato de anexos/arquivos e política de assinatura, quando aplicável.
- Paginação em consultas auxiliares.


