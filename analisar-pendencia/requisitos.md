## Requisitos – Analisar Pendência (Pré-análise, Análise, Assinatura e Impressão)

### Escopo e objetivo
Documentar a funcionalidade de “Analisar Pendência” do legado JSP/Servlet para migração a endpoints REST. Foco em operações, parâmetros, validações e permissões. Ignorar navegação/telas.

Fontes: `AnalisarPendenciaCt.java`, `AssinarPendenciaCt.java`, DTO `AnalisePendenciaDt.java` e permissões em `analisar-pendencia.json`.

### Permissões relevantes
- Código base da funcionalidade: **561 – Analisar Pendência**
- Subpermissões (conforme `analisar-pendencia.json`):
  - **5612 – LOCALIZAR**: Consulta Pendências Não Analisadas
  - **5613 – LOCALIZAR DWR**: Suporte a listagens auxiliares (no REST consolidar em endpoints de consulta)
  - **5614 – NOVO**: Preparar/abrir análise (simples/múltipla)
  - **5615 – SALVAR**: Salvar pré-análise/análise; confirmação/descarte
  - **5616 – CURINGA6**: Preparar análise de uma pré-análise múltipla; fluxo especial de assinatura em lote
  - **5617 – CURINGA7**: Consulta Pendências Analisadas
  - **5618 – CURINGA8**: Gerar Voto (cria pendência de voto)
  - **5610 – EXCLUIR**: Descartar análise
  - **5611 – IMPRIMIR**: Emissão/entrega de PDF após assinatura (autoridades)

### Regras e validações principais
- Número de processo: deve conter dígito verificador (formato com “.”). Caso contrário, retornar erro “Número do Processo no formato incorreto.”
- Pré-análise vs. análise:
  - Pré-análise: `verificarPreAnalisePendencia` e depois `salvarPreAnalisePendencia`.
  - Análise: `verificarAnalisePendencia` e depois `salvarAnalisePendencia`.
- “Guardar para assinar”: marca a pré-análise como pendente de assinatura.
- Descarte: remove status “aguardando assinatura” (retorna a “pré-analisadas”) ou descarta análise em andamento.
- Assinatura: restrições de perfil; geração opcional de PDF para autoridades.
- Geração de voto/ementa: cria pendências específicas (voto/ementa) a partir de uma pendência base.

### Modelos de dados (principais campos)
Baseado em `AnalisePendenciaDt` e parâmetros capturados:
- Identificação e classificação
  - `idMovimentacaoTipo`, `movimentacaoTipo`, `movimentacaoComplemento`
  - `idPendenciaTipo` (alias `tipo`), `unidadeTrabalho`
- Arquivo/modelo/texto
  - `idArquivoTipo`, `arquivoTipo`, `nomeArquivo`
  - `idModelo`, `modelo`, `textoEditor` (HTML), listas: `listaArquivos`, `listaPendenciasFechar`
  - Campos de ementa: `idArquivoTipoEmenta`, `arquivoTipoEmenta`, `nomeArquivoEmenta`, `idModeloEmenta`, `modeloEmenta`, `textoEditorEmenta`
- Consulta e filtros
  - `numeroProcesso`, `dataInicial`, `dataFinal`, `preAnalise` (boolean)
  - `pendenteAssinatura` (boolean)
- Vínculos e histórico
  - `arquivoPreAnalise`, `arquivoPreAnaliseEmenta`, `usuarioPreAnalise`, `dataPreAnalise`, `historicoPendencia`
- Metadados e logs
  - `idUsuarioLog`, `ipComputadorLog`
- Itens específicos (quando aplicável)
  - Voto/Ementa: `idPendenciaVotoGerada`, `idPendenciaEmentaGerada`, `idServentiaCargoVotoEmentaGerada`
  - Mandados/Alvará: campos de prazo, locomocoes, contas, etc. (expostos apenas quando necessários nos fluxos correspondentes)

---

## Endpoints REST propostos

### 1) Pendências – Consulta e resultados
- GET `/pendencias`
  - Permissão: 5612 (LOCALIZAR)
  - Query: `numeroProcesso?`, `idPendenciaTipo?`
  - Retorna: lista de pendências não analisadas do usuário.
  - Validações: formato do número do processo.

- GET `/pendencias/analisadas`
  - Permissão: 5617 (CURINGA7)
  - Query: `numeroProcesso?` ou (`dataInicial` e `dataFinal`) – obrigatório ao menos um critério válido
  - Retorna: lista de pendências analisadas.
  - Validações: formato do número do processo; parâmetros obrigatórios.

### 2) Pré-análise e Análise (criação/atualização)
- POST `/pendencias/pre-analises`
  - Permissão: 5615 (SALVAR)
  - Body: {
    `pendencias`: [ids],
    `idArquivoTipo`, `nomeArquivo`, `arquivoTipo`,
    `textoEditor` (HTML), `idModelo?`,
    `guardarParaAssinar`: boolean
  }
  - Ação: cria/atualiza pré-análise; se `guardarParaAssinar=true`, marca como pendente de assinatura.
  - Regras: `verificarPreAnalisePendencia` antes de persistir.

- PATCH `/pendencias/pre-analises/{id}/texto-parcial`
  - Permissão: 5615 (SALVAR)
  - Body: `{ textoEditor }`
  - Ação: salva texto parcial da pré-análise (quando fluxo exigir).

- POST `/pendencias/analises`
  - Permissão: 5615 (SALVAR)
  - Body: {
    `pendencias`: [ids],
    `idArquivoTipo`, `nomeArquivo`, `arquivoTipo`,
    `textoEditor` (HTML)
  }
  - Ação: efetiva análise e fecha pendências selecionadas. Regras: `verificarAnalisePendencia`.

- DELETE `/pendencias/{id}/analise`
  - Permissão: 5610 (EXCLUIR)
  - Ação: descarta a análise da pendência.

### 3) Geração de Voto/Ementa
- POST `/pendencias/{id}/gerar-voto`
  - Permissão: 5618 (CURINGA8) e perfis elegíveis
  - Ação: cria pendência de voto vinculada.

- POST `/pendencias/{id}/gerar-ementa`
  - Permissão: 561 (base) e perfis elegíveis
  - Ação: cria pendência de ementa vinculada.

### 4) Assinatura de Pré-análises
- GET `/pendencias/pre-analises/pendentes-assinatura`
  - Permissão: 561 (base), condicionado a perfil
  - Query: `numeroProcesso?`, `idPendenciaTipo?`
  - Retorna: lista de pré-análises pendentes de assinatura (com hash/token quando necessário).

- POST `/pendencias/pre-analises/assinatura/preparar`
  - Permissão: 561 (base)
  - Body: `{ pendencias: [ids] }`
  - Retorna: conteúdos a assinar [{ `idPendencia`, `nomeArquivo`, `idArquivoTipo`, `arquivoTipo`, `conteudoHtml` }].

- POST `/pendencias/pre-analises/assinatura/confirmar`
  - Permissão: 5615 (SALVAR)
  - Body: lista de itens assinados (map pendência → arquivo assinado + metadados de assinatura)
  - Ação: valida e salva como análise concluída (aplica `verificarAnalisePendencia` quando aplicável).

- POST `/pendencias/pre-analises/assinatura/descarte`
  - Permissão: 5615 (SALVAR)
  - Body: `{ pendencias: [ids] }`
  - Ação: remove status “aguardando assinatura”, retornando para “pré-analisadas”.

- POST `/pendencias/pre-analises/{id}/assinar`
  - Permissão: 5616 (CURINGA6) + 561 (base)
  - Body: `{ senhaCertificado, salvarSenha? }`
  - Ação: assinatura no servidor do conteúdo e posterior salvamento da análise.

### 5) Impressão (autoridades)
- POST `/pendencias/assinatura/imprimir`
  - Permissão: 5611 (IMPRIMIR)
  - Body: identificação do lote/processos assinados
  - Ação: emite PDF consolidado das pendências assinadas (quando perfil de autoridade).

### 6) Endpoints auxiliares de consulta (combos/listagens)
Usados para preencher seletores e validações nas operações acima.

- GET `/movimentacoes-tipo?query=`
  - Retorna tipos de movimentação habilitados para usuário/grupo.

- GET `/arquivos-tipo?query=`
  - Retorna tipos de arquivo disponíveis ao usuário/grupo.

- GET `/modelos?query=&idArquivoTipo=`
  - Retorna modelos do usuário (pode preencher texto ao selecionar).

---

## Respostas e códigos de status (resumo)
- 200: sucesso em consultas e operações de confirmação/salvamento.
- 201: criação de pré-análise/análise quando aplicável.
- 400: validação de parâmetros (ex.: número de processo inválido, campos obrigatórios).
- 403: usuário sem permissão para a ação.
- 404: entidade não localizada (pendência/modelo/etc.).
- 409: conflito de negócio (ex.: não pode analisar/assinar no estado atual).

## Observações de segurança
- Todas as rotas exigem autenticação e avaliação de permissão (código 561 e subcódigos quando aplicável), além de checagens por perfil/grupo do usuário.
- Para assinatura no servidor, o certificado do usuário deve estar previamente carregado/validado; a senha não deve ser persistida quando `salvarSenha=false`.

## Itens a decidir na migração
- Formato e armazenamento de arquivos assinados (.p7s, metadados de assinatura) e PDFs emitidos.
- Tokens/hashes para acesso seguro a conteúdos pré-assinados.
- Paginação/ordenação nas consultas de listas.


