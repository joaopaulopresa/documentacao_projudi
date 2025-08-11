## Requisitos – Autos Conclusos (Análise, Assinatura e Sessões 2º Grau)

### Escopo e objetivo
Documentar a funcionalidade de “Autos Conclusos” do legado JSP/Servlet para migração a endpoints REST. Foco em operações, parâmetros, validações e permissões. Ignorar navegação/telas.

Fontes: `AnalisarConclusaoCt.java`, `AssinarConclusaoCt.java`, `AssinarSessaoSegundoGrauCt.java`, DTO `AnaliseConclusaoDt.java` e permissões em `analisar-autos-conclusos.json`.

### Permissões relevantes
- Código base da funcionalidade: **271 – Analisar Autos Conclusos**
- Subpermissões (conforme `analisar-autos-conclusos.json`):
  - **2712 – LOCALIZAR**: Consulta de conclusões não analisadas
  - **2714 – NOVO**: Preparar/abrir análise (simples/múltipla)
  - **2715 – SALVAR**: Salvar pré-análise/analysis (inclui confirmar/desfazer)
  - **2716 – CURINGA6**: Fluxos específicos (ex.: pré-análise múltipla)
  - **2717 – CURINGA7**: Consulta de conclusões finalizadas
  - **2718 – CURINGA8**: Retirar prioridade de conclusão
  - **2710 – EXCLUIR**: Descartar conclusão

Observação: validações adicionais por perfil/grupo (ex.: assistente/assessor/desembargador, fluxo UPJ, segundo grau) são respeitadas conforme regras dos servlets.

### Regras e validações principais
- Número de processo: deve conter dígito verificador (formato com “.”). Caso contrário, retornar erro “Número do Processo no formato incorreto.”
- Pré-análise múltipla bloqueada para Presidência/Vice (legado lança erro).
- Verificações de negócio ao salvar:
  - Pré-análise: `verificarPreAnaliseConclusao`, depois `salvarPreAnaliseConclusao`.
  - Análise: `verificarAnaliseConclusao`, `podeAnalisarConclusao`, depois `salvarAnaliseConclusao`.
- “Guardar para assinar” marca a pré-análise como pendente de assinatura.
- Descarte: remove status “aguardando assinatura” ou descarta análise conforme operação.
- Filtros de lista podem excluir votos/ementas dependendo do tipo (voto/ementa) e perfil.

### Modelos de dados (principais campos)
Baseado em `AnaliseConclusaoDt` e parâmetros capturados:
- Identificação e classificação
  - `idMovimentacaoTipo`, `movimentacaoTipo`, `movimentacaoComplemento`
  - `idClassificadorProcesso1`, `classificadorProcesso1`
  - `idClassificadorProcesso2`, `classificadorProcesso2`
  - `idClassificadorPendencia`, `classificadorPendencia`
  - `idTipoPendencia` (alias `tipo`)
- Arquivo/modelo/texto
  - `idArquivoTipo`, `arquivoTipo`, `nomeArquivo`
  - `idModelo`, `modelo`, `textoEditor` (HTML do despacho/voto/ementa)
  - Listas: `listaArquivos`, `listaPendenciasGerar`
- Consulta e filtros
  - `numeroProcesso`, `dataInicial`, `dataFinal`
  - `preAnalise` (boolean), `pendenteAssinatura` (boolean)
  - `pedidoAssistenciaIsencao` ("0"|"1"|"2")
- Sessões 2º grau (quando aplicável)
  - `processoTipoSessao`, `idProcessoTipoSessao`
  - `audienciaStatus`, `audienciaStatusCodigo`

Notas:
- Para assinaturas, o legado prepara conteúdo HTML dos arquivos para assinatura digital (cliente ou servidor). No REST, tratar como conteúdo a assinar + metadados (nome, tipo, mime).

---

## Endpoints REST propostos

### 1) Conclusões – Consulta e resultados
- GET `/conclusoes`
  - Permissão: 2712 (LOCALIZAR)
  - Query: `numeroProcesso?`, `idClassificadorPendencia?`, `idTipoPendencia?`, `incluirVotoEmenta?` (default excluir quando “todas” no legado)
  - Retorna: lista de conclusões pendentes do usuário, com dados mínimos do processo, pendência, prioridade e tipo.
  - Validações: formato do número do processo.

- GET `/conclusoes/finalizadas`
  - Permissão: 2717 (CURINGA7)
  - Query: `numeroProcesso?` ou (`dataInicial` e `dataFinal`) – obrigatório ao menos um critério válido
  - Retorna: lista de conclusões finalizadas no período/processo.
  - Validações: formato do número do processo; parâmetros obrigatórios.

### 2) Análise e Pré-análise (criação/atualização)
- POST `/conclusoes/pre-analises`
  - Permissão: 2715 (SALVAR)
  - Body: {
    `pendencias`: [ids],
    `idArquivoTipo`, `nomeArquivo`, `arquivoTipo`,
    `textoEditor` (HTML), `idModelo?`,
    `listaPendenciasGerar` (lista de tipos/itens),
    `guardarParaAssinar`: boolean
  }
  - Ação: cria/atualiza pré-análise. Se `guardarParaAssinar=true`, marca como pendente de assinatura.
  - Regras: `verificarPreAnaliseConclusao` antes de persistir.

- PATCH `/conclusoes/pre-analises/{id}/texto-parcial`
  - Permissão: 2715 (SALVAR)
  - Body: `{ textoEditor }`
  - Ação: salva texto parcial da pré-análise.

- POST `/conclusoes/analises`
  - Permissão: 2715 (SALVAR)
  - Body: {
    `pendencias`: [ids],
    `idArquivoTipo`, `nomeArquivo`, `arquivoTipo`,
    `textoEditor` (HTML), `listaPendenciasGerar`
  }
  - Ação: efetiva análise (fecha pendências selecionadas). Regras: `verificarAnaliseConclusao` e `podeAnalisarConclusao`.

- DELETE `/conclusoes/{id}/analise`
  - Permissão: 2710 (EXCLUIR)
  - Ação: descarta uma análise pendente (retorna pendência ao estado anterior apropriado).

- POST `/conclusoes/retirar-prioridade`
  - Permissão: 2718 (CURINGA8)
  - Body: `{ idPendencia }`
  - Ação: remove prioridade da pendência.

- GET `/conclusoes/analises/{id}`
  - Permissão: 271 (base) ou 2717 quando aplicável
  - Ação: retorna detalhes de uma análise finalizada (dados do texto/arquivo, histórico de pré-análises, metadados para download de arquivos).

### 3) Assinatura de Pré-análises (primeiro grau)
- GET `/conclusoes/pre-analises/pendentes-assinatura`
  - Permissão: 271 (base), condicionado a perfil
  - Query: `numeroProcesso?`, `idClassificador?`, `idTipoPendencia?`
  - Retorna: lista de pré-análises pendentes de assinatura do usuário, com token/hash de validação quando necessário.

- POST `/conclusoes/pre-analises/assinatura/preparar`
  - Permissão: 271 (base)
  - Body: `{ pendencias: [ids] }`
  - Retorna: array com conteúdos a assinar [{ `idPendencia`, `nomeArquivo`, `idArquivoTipo`, `arquivoTipo`, `conteudoHtml` }].

- POST `/conclusoes/pre-analises/assinatura/confirmar`
  - Permissão: 2715 (SALVAR)
  - Body: lista de itens assinados (map pendência → arquivo assinado + metadados assinatura)
  - Ação: valida e salva como análise concluída (aplica `verificarAnaliseConclusao` e `podeAnalisarConclusao`).

- POST `/conclusoes/pre-analises/assinatura/descarte`
  - Permissão: 2715 (SALVAR)
  - Body: `{ pendencias: [ids] }`
  - Ação: remove status “aguardando assinatura”, retornando para “pré-analisadas”.

- POST `/conclusoes/pre-analises/{id}/assinar`
  - Permissão: 2716 (CURINGA6) + 271 (base)
  - Body: `{ senhaCertificado, salvarSenha? }`
  - Ação: fluxo de assinatura no servidor (quando aplicável). Após assinar, salva análise.

### 4) Sessões de Segundo Grau – Assinatura (relatórios, votos e ementas)
- GET `/sessoes-2g/pendentes-assinatura`
  - Permissão: 271 (base) + perfil de 2º grau
  - Retorna: lista de processos em sessões de 2º grau com voto/ementa pendentes de assinatura para o usuário (inclui verificações de relatoria/redatoria).

- POST `/sessoes-2g/assinatura/preparar`
  - Permissão: 271 (base)
  - Body: `{ sessoes: [idsAudienciaProcesso] }`
  - Retorna: conteúdos a assinar por sessão: para cada sessão, arquivos de `RELATORIO_VOTO` e `EMENTA` com `{ nomeArquivo, idArquivoTipo, arquivoTipo, conteudoHtml }`.

- POST `/sessoes-2g/assinatura/confirmar`
  - Permissão: 2715 (SALVAR)
  - Body: lista de arquivos assinados por sessão
  - Ação: valida e salva movimentação da audiência (sessão 2º grau), gerando/acoplando os arquivos assinados.

- POST `/sessoes-2g/assinatura/descarte`
  - Permissão: 2715 (SALVAR)
  - Body: `{ sessoes: [idsAudienciaProcesso] }`
  - Ação: descarta “aguardando assinatura” quando não há texto de voto/ementa válido; retorna para estado de pré-analisadas.

- POST `/sessoes-2g/assinar`
  - Permissão: 2716 (CURINGA6) + 271 (base) + perfil 2º grau
  - Body: `{ sessoes: [idsAudienciaProcesso], senhaCertificado, salvarSenha? }`
  - Ação: assinatura no servidor dos votos/ementas e persistência da movimentação.

### 5) Endpoints auxiliares de consulta (combos/listagens)
Usados para preencher seletores e validações nas operações acima.

- GET `/movimentacoes-tipo?query=`
  - Retorna tipos de movimentação habilitados para usuário/grupo.

- GET `/classificadores?query=&serventiaId=&fluxoUPJ=`
  - Retorna classificadores disponíveis (ajusta por serventia/UPJ). Pode aceitar `consultaClassificador=2` para contexto de localizar.

- GET `/arquivos-tipo?query=`
  - Retorna tipos de arquivo.

- GET `/modelos?query=&idArquivoTipo=`
  - Retorna modelos do usuário (pode preencher texto e pendências sugeridas ao selecionar).

- GET `/areas-distribuicao?query=`

- GET `/serventias?query=&idAreaDistribuicao=`
  - Requer `idAreaDistribuicao` válido.

- GET `/serventia-cargos?query=&idServentiaProcesso=`
  - Retorna responsáveis/cargos para encaminhamento (gabinete).

---

## Respostas e códigos de status (resumo)
- 200: sucesso em consultas e operações síncronas de confirmação/salvamento.
- 201: criação de pré-análise/análise quando aplicável.
- 400: validação de parâmetros (ex.: número de processo inválido, campos obrigatórios).
- 403: usuário sem permissão para a ação.
- 404: entidade não localizada (pendência/sessão/modelo/etc.).
- 409: conflito de negócio (ex.: não pode analisar/assinar no estado atual).

## Observações de segurança
- Todas as rotas exigem autenticação e avaliação de permissão (código 271 e subcódigos quando aplicável).
- Para assinatura no servidor, o certificado do usuário deve estar previamente carregado/validado na sessão segura; a senha não deve ser persistida quando `salvarSenha=false`.

## Itens a decidir na migração
- Formato dos “arquivos assinados” (ex.: anexar .p7s, metadados de assinatura) e política de armazenamento.
- Tokens/hashes para download/acesso seguro a conteúdos pré-assinados.
- Paginação e ordenação nas consultas de listas.


