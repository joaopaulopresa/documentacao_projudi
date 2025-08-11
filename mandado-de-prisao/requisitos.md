## Requisitos – Mandado de Prisão

Escopo: migrar a funcionalidade de Mandado de Prisão do legado (JSP/Servlet) para endpoints REST. Cobre emissão, expedição (com assinatura), impressão/visualização, consultas e finalização (cumprimento, revogação, retirada de sigilo). Ignorar UI.

### 1) Permissões
- **Código base**: 740 (Mandado de Prisão)
- **Subpermissões** (do JSON):
  - 7400 – EXCLUIR
  - 7401 – IMPRIMIR
  - 7402 – LOCALIZAR
  - 7404 – NOVO
  - 7405 – SALVAR
  - 7407 – CURINGA7 (ações do juiz: listar/expedir/assinar)
  - 7408 – CURINGA8 (menu Cumprimento: consultar/visualizar)
  - 7409 – CURINGA9 (finalização: cumprir/revogar/retirar sigilo)
- **Permissões auxiliares para combos** (por entidade): `ArquivoTipo`, `Modelo`, `MandadoPrisaoStatus`, `PrisaoTipo`, `RegimeExecucao`, `Assunto`, `Processo`, `ProcessoParte` (consultas via LOCALIZAR de cada módulo)

### 2) Regras e validações
- **Perfis/escopos**:
  - Juiz: visualiza listas gerais (listarTodos=true) e expede mandado; consultas por `serventiaCargo` quando aplicável.
  - Serventia: emite mandado e lista por `id_Serventia`; no “Novo”, sigilo default para falso.
- **Verificações de negócio**:
  - Excluir: `verificarExcluir(mandado)`. Bloqueia exclusão conforme status.
  - Emitir: bloqueia se já possuir `mandadoPrisaoStatusCodigo`. Executa `Verificar(mandado, isJuiz)` antes de emitir.
  - Expedir: exige `Verificar(mandado, isJuiz)`; gera número se ausente; requer arquivo assinado presente (lista de arquivos da sessão) para efetivar expedição.
  - Finalização:
    - Cumprir: `verificarCumprimento(anexos, mandado)`; efetiva operação=1.
    - Revogar: exige ao menos 1 arquivo de decisão; operação=2.
    - Retirar sigilo: exige ao menos 1 arquivo de decisão; operação=3.
  - Cálculo de validade: requer `tempo(ano,mes,dia)` e data de nascimento do sentenciado (`ProcessoParte`). Erro se ausente.
- **Estados/Fluxos principais**:
  - Emitido → Expedido → Impresso → Cumprido/Revogado. “Sigiloso” pode ser retirado via fluxo de finalização.
- **Auditoria**: preencher sempre `idUsuarioLog` e `ipComputadorLog` (no `MandadoPrisaoDt` e em `processoPartePrisaoDt`).

### 3) Modelos de dados (DTO)
- **MandadoPrisao** (principais campos – strings salvo indicação):
  - Identificação/processo: `id`, `id_Processo`, `processoTipo`, `processoNumero`, `digitoVerificador`, `ano`, `processoNumeroCompleto`
  - Parte: `id_ProcessoParte`, `processoParte`, `dataNascimento`, `nomeMae`, `nomePai`, `sexo`, `cpf`, `naturalidade`, `ufNaturalidade`, `enderecoCompleto`, `idEndereco`
  - Dados do mandado: `mandadoPrisaoNumero`, `dataExpedicao`, `dataValidade`, `mandadoPrisaoOrigemCodigo`/`id_MandadoPrisaoOrigem`/`mandadoPrisaoOrigem`, `origem`, `numeroOrigem`, `localRecolhimento`, `valorFianca`, `sinteseDecisao`, `sigilo` (boolean lógico via string), `mandadoPrisaoStatusCodigo`/`id_MandadoPrisaoStatus`/`mandadoPrisaoStatus`, `forumCodigo`
  - Pena/prazo: `prazoPrisao`, `tempoPenaAno`, `tempoPenaMes`, `tempoPenaDia`, `tempoPenaTotalDias`, `tempoPenaTotalAnos`, `id_RegimeExecucao`/`regimeExecucao`
  - Tipo de prisão: `prisaoTipoCodigo`, `id_PrisaoTipo`/`prisaoTipo`
  - Datas/eventos: `dataEmissao`, `dataImpressao`, `dataPrisao`, `dataCumprimento`
  - Usuários ação: `idUsuarioServentiaEmissao`, `idUsuarioServentiaExpedicao`, `idUsuarioServentiaImpressao`, `idUsuarioServentiaCumprimento`
  - Arquivo/modelo: `idArquivoTipo`/`arquivoTipo`, `idModelo`/`modelo`, `textoEditor`
  - Auditoria: `idUsuarioLog`, `ipComputadorLog`, `dataAtualizacao`
  - Listas auxiliares: `listaMandadoPrisaoOrigem`, `listaMandadoPrisaoStatus`, `listaPrisaoTipo`, `listaRegime`, `listaArquivo`
- **ProcessoPartePrisaoDt** (aninhado em `MandadoPrisao`): `id_ProcessoParte`, `nome`, `dataPrisao`, `id_PrisaoTipo`/`prisaoTipo`, `id_LocalCumpPena`/`localCumpPena`, `prazoPrisao`, `id_UsuarioLog`, `ipComputadorLog`

Observações: `sigilo` tratado como string no legado; padronizar como boolean no REST, mantendo compatibilidade de entrada se necessário.

### 4) Endpoints REST propostos

#### 4.1 Consultas
- GET `/mandados-prisao`
  - Permissão: 7402 (LOCALIZAR)
  - Query: `processoId` (obrig.), `escopo=juiz|serventia` (define listarTodos), `status[]`, `tipoPrisao`, `pagina`
  - Ação: lista mandados do processo; para juiz, retorna todos; para serventia, filtra por `id_Serventia`.
  - Retorno: lista de `MandadoPrisao` (resumo) + paginação

- GET `/mandados-prisao/cumprimento`
  - Permissão: 7408 (CURINGA8)
  - Query: `numeroProcesso`, `dataInicial`, `dataFinal`, `statusCodigo`, `tipoCodigo`, `escopo`, `pagina`
  - Ação: consulta JSON do legado `listarMandadoPrisaoServentiaJSON` (equivalente REST), respeitando escopo juiz/serventia.
  - Retorno: lista resumida paginada

- GET `/mandados-prisao/{id}`
  - Permissão: 7408 (visualização via Cumprimento) ou 7402
  - Ação: carrega o mandado e listas auxiliares para edição/visualização.
  - Retorno: `MandadoPrisao` completo

#### 4.2 Preparação/Emissão (Serventia)
- POST `/mandados-prisao/preparar`
  - Permissão: 7404 (NOVO)
  - Body: `{ processoId }`
  - Ação: inicializa DTO limpo com dados do processo e listas auxiliares; para serventia, define `sigilo=false` por padrão.
  - Retorno: `MandadoPrisao` (draft)

- POST `/mandados-prisao/{id}/emitir`
  - Permissão: 7406/7407 analógico ao fluxo da serventia; considerar 740 (base) + regra de perfil
  - Body: `MandadoPrisao` (dados necessários)
  - Regras: bloqueia se já possuir `statusCodigo`; executa `Verificar(mandado, isJuiz)`; registra emissão pela serventia.
  - Retorno: 200 com `MandadoPrisao` atualizado (status=EMITIDO)

#### 4.3 Expedição e Assinatura (Juiz)
- POST `/mandados-prisao/{id}/preparar-expedicao`
  - Permissão: 7407 (CURINGA7)
  - Ação: valida; gera número se ausente; monta modelo do texto do mandado; determina `idArquivoTipo` “Mandado de Prisão”.
  - Retorno: `{ texto, idArquivoTipo, arquivoTipo }`

- POST `/mandados-prisao/{id}/assinar`
  - Permissão: 7407
  - Body: `{ arquivoAssinadoId | arquivoAssinadoConteudo, hashAssinatura, metadados }`
  - Ação: armazena arquivo assinado vinculado ao mandado (substitui a “lista de arquivos” da sessão do legado).
  - Retorno: 201 (referência do arquivo assinado)

- POST `/mandados-prisao/{id}/expedir`
  - Permissão: 7407
  - Body: `{ arquivoAssinadoId }`
  - Regras: exige arquivo assinado; efetiva expedição e limpa pendências temporárias.
  - Retorno: 200; status=EXPEDIDO

#### 4.4 Impressão/Arquivos
- GET `/mandados-prisao/{id}/arquivo-assinado`
  - Permissão: 7401 (IMPRIMIR) ou conforme regra de escopo
  - Ação: entrega o HTML assinado; retorna 403 caso não permitido (segredo/escopo).

- GET `/mandados-prisao/{id}/pdf`
  - Permissão: 7401
  - Ação: gera/entrega PDF do mandado ativo. Content-Type: `application/pdf`.

#### 4.5 Finalização (Cumprimento/Revogação/Retirada de Sigilo)
- POST `/mandados-prisao/{id}/cumprir`
  - Permissão: 7409 (CURINGA9)
  - Body: `{ anexos:[...], dataCumprimento, dataPrisao, localRecolhimento, observacoes }`
  - Regras: `verificarCumprimento`; operação=1.
  - Retorno: 200 e mensagem de sucesso

- POST `/mandados-prisao/{id}/revogar`
  - Permissão: 7409
  - Body: `{ anexos:[...], decisao }` (obrigatório ≥1 anexo)
  - Regras: operação=2.
  - Retorno: 200

- POST `/mandados-prisao/{id}/retirar-sigilo`
  - Permissão: 7409
  - Body: `{ anexos:[...], decisao }` (obrigatório ≥1 anexo)
  - Regras: operação=3.
  - Retorno: 200

#### 4.6 Exclusão
- DELETE `/mandados-prisao/{id}`
  - Permissão: 7400 (EXCLUIR)
  - Regras: `verificarExcluir(mandado)`; retorna 409 se houver impedimento de negócio.
  - Retorno: 204

#### 4.7 Auxiliares (combos/listagens)
- GET `/mandados-prisao/origens` – lista `MandadoPrisaoOrigem`
- GET `/mandados-prisao/tipos-prisao` – lista `PrisaoTipo`
- GET `/mandados-prisao/regimes` – lista `RegimeExecucao`
- GET `/mandados-prisao/status` – lista `MandadoPrisaoStatus`
- GET `/modelos` – filtra por `arquivoTipoId` e termo; escopo do usuário
- GET `/arquivos-tipo` – lista por grupo do usuário (para “Mandado de Prisão”)

#### 4.8 Operação especial
- POST `/mandados-prisao/calcular-validade`
  - Permissão: 740 (base)
  - Body: `{ tempoAno, tempoMes, tempoDia, idProcessoParte }`
  - Ação: converte para dias e calcula validade a partir da data de nascimento do sentenciado e data atual.
  - Retorno: JSON com resultado do cálculo; 400 se faltar data de nascimento

### 5) Respostas e códigos de status
- 200 OK: operações de consulta/efeito sem criação
- 201 Created: criação de arquivo assinado ou rascunho quando aplicável
- 204 No Content: exclusão
- 400 Bad Request: validações de formato/obrigatórios (ex.: falta de `dataNascimento` para cálculo; anexos obrigatórios)
- 403 Forbidden: falta de permissão/escopo (acesso a arquivo assinado, PDF)
- 404 Not Found: mandado/processo inexistente
- 409 Conflict: impedimentos de negócio (`verificarExcluir`, estados inválidos)

### 6) Observações de segurança
- Autenticação/Autorização por permissão base 740 e subpermissões específicas; restringir escopo por perfil (juiz/serventia).
- Conteúdo sensível: dados pessoais no texto/arquivos do mandado (LGPD). Minimizar exposição; mascarar quando for público.
- Assinatura: substituir “lista de arquivos da sessão” por fluxo de upload seguro (hash, carimbo do tempo, verificação de integridade) e política de retenção.

### 7) Itens a decidir na migração
- Modelo e assinatura: formato do arquivo assinado (HTML/PDF/A1/A3) e integração com provedor de assinatura.
- Geração de número do mandado: política/serviço central ou lógica local do módulo.
- Paginador e filtros padronizados nas consultas (incluindo ordenação).
- Padronização de `sigilo` como boolean no contrato REST.
- Estratégia para anexos/decisões nas finalizações (storage, metadados, categorias de arquivo).


