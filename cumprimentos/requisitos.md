## Requisitos – Cumprimentos (Pendências)

Breve: Migração da funcionalidade de Cumprimentos/Pendências do legado (JSP/Servlet) para REST. Cobre consultas por fluxos, criação e reabertura, expedição/distribuição/encaminhamento, finalizações (inclui variações como marcar audiência, gerar ofício, por concluso), pré‑análises, intimações/citações e operações em lote, além de consultas auxiliares (tipos, status, serventias, cargos, modelos, etc.). Ignora telas/JSP.

### 1) Permissões
- **Código base**: 112 (Cumprimentos)
- **Subpermissões relevantes (extraídas de `cumprimentos.json`)**:
  - **263 – Abertos**: consultar pendências abertas da serventia
  - **374 – Acompanhamento**: lista pendências aguardando visto da serventia
  - **805 – Conclusões**
    - **8054 – Trocar Responsável** (TrocarConclusaoMagistrado)
    - **8055 – Conclusões SALVAR**
  - **108 – Configuração Pré‑Análise Automática** (PendenciaConfigConclusaoAuto)
    - 1086 CURINGA6; 1080 EXCLUIR; 1084 NOVO; 1085 SALVAR
  - **373 – Criar Pendência**
  - **377 – Finalizados** (serventia)
  - **741 – Mandado de Prisão** (consulta)
  - **289 – Meus**
    - 283 Acompanhamento; 324 Com prazo decorrido; 284 Finalizados; 265 Reservados/Pré‑Analisados; 500 Respondidos
  - **547 – Pendentes** (não analisadas)
  - **549 – Pré‑Análises**
    - 552 Finalizadas; 551 Múltiplas; 550 Simples
  - **399 – Publicações**
    - 402 Consulta Textual; 280 Criar Publicação; 397 Listar Publicações (3972 LOCALIZAR; 3976/3977/3978 CURINGA6/7/8)
  - **507 – Respondidos** (serventia)
  - **1126/1127/1128/1129 – Pendência CURINGA 6/7/8/9**: operações especiais (acompanhamentos diversos; criação; distribuição)
  - **1121 – IMPRIMIR**
  - **1122 – LOCALIZAR**
  - **1125 – SALVAR**
  - **1123 – LOCALIZARDWR** (acesso externo)
  - **114 – Pendência Arquivo**
    - 1142 Localizar; 1143 LocalizarDWR; 1146 Baixar Arquivo

Observação: além do menu “Cumprimentos”, o módulo utiliza permissões auxiliares de entidades relacionadas em buscas/combos (ex.: tipos/subtipos/status de pendência, serventias, comarcas, prioridades, arquivos/modelos).

### 2) Regras e validações
- **Autorização por perfis/grupos**: uso de perfis como assessor/assessorMP/assessorAdvogado afeta distribuição automática ao abrir pendência; checagens como `isPodeMovimentar`, `isPodeRedistribuir`, `isPodeTrocarResponsavel`, `isPodeEncaminhar`, `isPodeReservarNumeroMandadoCentral`.
- **Hash de acesso**: operações por link exigem `id + hash` válidos (verificação do código hash antes de abrir/operar pendência).
- **Pré‑checagens na abertura**: `verificarPreviaResolverPendencia(usuario, pendencia, serventia)` para alertas (ex.: requisitos de mandado).
- **Intimação via telefone**: ao preparar/finalizar, força status “Aguardando Retorno”. Validação obrigatória de parte: erro quando `id_ProcessoParte` ausente.
- **Central de Mandados**:
  - Reservar previamente número do mandado quando a comarca possui central implantada e o usuário pode reservar.
  - Expedição pode ser para central vinculada ou comarcas contíguas; validação de limite por oficial pode exigir confirmação explícita (diálogo/confirmar).
- **Validações de negócio**: `verificarExpedir`, `validarArquivosPendencia`, `verificarEncaminhar`, `verificarAlteracaoPendenciaTipo`, `verificarAlvaraSoltura`.
- **Domicílio Eletrônico**: fluxo específico de expedição eletrônica.
- **Alteração de Tipo**: “Efetuar Troca” condicionado a verificação; suportado para intimação/citação/mandado/carta precatória e cálculos.
- **Finalizações com efeitos colaterais**:
  - Movimentar processo; marcar audiência; gerar ofício comunicatório; por concluso; elaboração de voto (com/sem movimentação).
- **Marcação de leitura**: “Marcar Lido” e “Marcar Lido Aguardando Parecer” para intimações/citações; lote para “Marcar Como Lidas/Marcar Visto”.
- **Operações em lote**: liberar pendências reservadas; marcar lidas/visto; distribuir selecionadas.
- **Prazos**: listas de prazo decorrido, a decorrer e devolução de autos têm ações de “Vistar”/movimentar em massa; contagem/paginação controladas.
- **Auditoria**: sempre preencher `idUsuarioLog` e `ipComputadorLog` nas operações.

### 3) Modelos de dados (DTO base)
Baseado em `PendenciaDtGen` e `PendenciaDt`:
- **Identificação e vínculo**: `id`, `id_PendenciaPai`, `hash`, `id_Processo`, `processoNumero`, `processoNumeroCompleto`, `id_ProcessoParte`, `nomeParte`, `id_Movimentacao`, `id_Classificador`.
- **Tipificação e status**: `id_PendenciaTipo`, `pendenciaTipo`, `pendenciaTipoCodigo`, `id_PendenciaStatus`, `pendenciaStatusCodigo`, `statusPendenciaRetorno`, `listaStatusAguardandoRetorno`.
- **Responsáveis e perfis**: `responsaveis` (lista), `magistradoResponsaveisAtuais` (lista), `id_UsuarioCadastrador/Finalizador`, `usuarioCadastrador/Finalizador`, `serventiaUsuarioCadastrador/Finalizador`, `id_ServentiaCargo`, `serventiaResponsavel`, `podeLiberar`.
- **Datas e prazos**: `dataInicio`, `dataFim`, `dataLimite`, `dataDistribuicao`, `prazo`, `dataVisto`, `dataTemp`; auxiliares de audiência: `dataAudiencia`, `horaAudiencia`, `telefoneAudiencia`, `enderecoAudiencia`, `linkAudiencia`.
- **Prioridade**: `id_ProcessoPrioridade`, `processoPrioridade`, `processoPrioridadeCodigo`, `pendenciaPrioridadeCodigoTexto`, `pendenciaPrioridadeOrdem`.
- **Arquivos**: `ListaArquivos` (vinculados), utilitários `getArquivosResposta`, downloads e assinados; suporte a `idModelo`, `codModeloCorreio`, `maoPropriaCorreio`.
- **Mandados/Correios**: `id_MandadoJudicial`, `id_MandadoTipo`, `listaTiposMandados`, `prazoMandado`, `codigoPrazoMandado`, `idPendenciaCorreios`, flags ECarta via status, número reservado (`numeroReservadoMandadoExpedir`).
- **Outros campos**: `cpfCnpjParte`, `id_ProcessoCustaTipo`, `processoTipo/Sessao`, `precatorioDt`, `alvaraNumero`, `listaContas` (alvará eletrônico), `eventoTipo/id_EventoTipo/dataEvento`, `pedidoSegredoJustica`, `isAnalisePendenciaAssincrona`, `Id_PendenciaSubTipo`.

Observações: métodos auxiliares determinam tipo (mandado, carta precatória, conclusão, voto/ementa, verificação), estado (em andamento, aguardando retorno/cumprimento), prioridade e seleção em lote.

### 4) Endpoints REST propostos

4.1 Consultas (GET)
- **/pendencias/abertas-serventia**: filtros opcionais `tipo`, `subtipo`, `status`, `numeroProcesso`, `prioridade`, `serventia`, `comarca`, `dataIni`, `dataFim`, `pagina`, `tamanho`.
- **/pendencias/serventia/finalizadas**; **/pendencias/serventia/respondidas**.
- **/pendencias/minhas/reservadas | pre-analisadas | acompanhamento | prazo-decorrido | finalizadas | respondidas**.
- **/pendencias/prazos/decorridos**: filtros `tipo`, `numeroProcesso`, `pagina`; ações em lote por endpoint separado.
- **/pendencias/prazos/a-decorrer**; **/pendencias/prazos/devolucao-autos**.
- **/pendencias/intimacoes/lidas**; **/pendencias/intimacoes/distribuidas** (promotoria/procuradoria por filtro `orgao`).
- **/pendencias/expedidas-serventia**.
- **/publicacoes** (consulta textual por `q`, datas, tipo); **/publicacoes/listar**.

4.2 Preparação/Confirmação (POST)
- **/pendencias/preparar**: corpo com dados mínimos (processo, tipo, parte, prioridade); retorna contexto, listas auxiliares e validações pendentes.
- **/pendencias/confirmar**: cria de fato a pendência (201); validações de domínio e auditoria.
- **/pendencias/{id}/reabrir/preparar** → **/pendencias/{id}/reabrir/confirmar**: reabre mantendo histórico e regras específicas (inclui caminho de impressão pós‑reabertura quando aplicável).
- **/pendencias/{id}/alterar-tipo/preparar** → **/pendencias/{id}/alterar-tipo/confirmar**: valida `novoTipo` e dependências.

4.3 Operações de resolução
- **/pendencias/{id}/encaminhar**: body com `serventiaDestino`, `serventiaCargoDestino`; aplica `verificarEncaminhar`.
- Expedição/Distribuição (aceitam upload/lista de `arquivos` e parâmetros de expedição):
  - **/pendencias/{id}/expedir** | **/pendencias/{id}/expedir-imprimir**
  - **/pendencias/{id}/expedir-domicilio-eletronico**
  - **/pendencias/{id}/distribuir** | **/pendencias/{id}/distribuir-imprimir**
- **/pendencias/{id}/finalizar** e variantes especializadas (todas exigem auditoria e validação de arquivos quando necessário):
  - **/finalizar/movimentar-processo**
  - **/finalizar/marcando-audiencia**
  - **/finalizar/gerar-oficio-comunicatorio**
  - **/finalizar/averbacao-custas**
  - **/finalizar/por-concluso**
  - **/finalizar/elaboracao-voto** | **/finalizar/elaboracao-voto-movimentando**
- **/pendencias/{id}/concluir-aguardando-retorno**: valida arquivos, aplica regra de alvará quando cabível.
- **/pendencias/{id}/marcar-lido** | **/pendencias/{id}/marcar-lido-aguardando-parecer** (intimações/citações).
- **/pendencias/{id}/guardar-para-assinar**.

4.4 Operações em lote (POST)
- **/pendencias/lote/liberar**: body `pendencias: ["{id}@#!@{hash}", ...]`.
- **/pendencias/lote/distribuir**: body `pendencias: [id...]`, parâmetros de expedição.
- **/pendencias/lote/marcar-visto** | **/pendencias/lote/marcar-lidas**: body `pendencias: [id...]`.

4.5 Arquivos e impressão
- **/pendencias/{id}/arquivos** (POST upload, GET listar); **/pendencias/{id}/arquivos-assinados** (GET).
- **/pendencias/{id}/imprimir**: geração/entrega de PDFs (retornar URL/ID de job; quando “imprimir” for combinado com expedição/distribuição, retornar também os documentos gerados).

4.6 Endpoints auxiliares (combos)
- **/pendencias/tipos**; **/pendencias/subtipos?tipoId=**; **/pendencias/status**.
- **/serventias**; **/serventias/expedir** (filtradas por tipo e central de mandados); **/serventias/{id}/cargos** (`onlyMagistrados`, `encaminhamento`).
- **/comarcas**; **/prioridades-processo**.
- **/processos**; **/processos/partes**.
- **/arquivos/tipos?grupo=**; **/modelos?arquivoTipoId=**; **/serventias/tipos**.

Para todos os endpoints, incluir `idUsuarioLog` e `ipComputadorLog` no header/corpo conforme padrão do backend.

### 5) Respostas e códigos de status
- 200 sucesso; 201 criação; 400 validação (ex.: parte ausente, arquivos inválidos);
- 403 permissão (perfil/serventia/tipo de operação não autorizado);
- 404 não encontrado (pendência/processo/modelo/serventia);
- 409 conflito de negócio (ex.: limite por oficial; estado incompatível para finalizar; hash inválido).

### 6) Observações de segurança
- **Autenticação/Autorização**: respeitar perfis e subpermissões listadas; mover a verificação de hash do legado para controle de autorização/escopos no REST.
- **Dados sensíveis**: atenção a dados pessoais (LGPD), documentos anexos, números de mandado, segredo de justiça e Domicílio Eletrônico.
- **Assinatura/arquivos**: uploads, assinatura e hash de arquivos devem manter integridade e trilha de auditoria.

### 7) Itens a decidir na migração
- Paginação e ordenação padrão em cada listagem; limites e tokens de continuação.
- Atomicidade em operações de lote (tudo ou nada x parcial com relatório de erros).
- Modelo de geração de PDFs (sincrono x job assíncrono) e URLs temporárias.
- Estratégia para “reserva de número de mandado” (transação e expiração/cancelamento).
- Padronização dos “fluxos” do legado em filtros ou recursos distintos no REST.
- Endpoints de configuração da Pré‑Análise Automática (criação/edição/exclusão e execução) e sua orquestração.


