## Requisitos – Modelo (Cadastro e Gestão de Modelos)

Escopo: migrar a funcionalidade legada de cadastro, consulta, edição, exclusão, cópia e entrega do conteúdo de modelos (documentos/HTML) para endpoints REST. Incluir configuração de pré‑preenchimento vinculada ao modelo e consultas auxiliares (tipo de arquivo, serventia, usuário/serventia). Ignorar detalhes de tela/JSP.

### 1) Permissões
- Código base: 166 (Modelo)
- Subpermissões (conforme `doc/modelo/modelo.json`):
  - 1660 – Excluir Modelo
  - 1661 – Imprimir Modelo
  - 1662 – Localizar Modelo
  - 1663 – LocalizarDWR Modelo (retorno JSON do texto/HTML)
  - 1664 – Novo Modelo
  - 1665 – Salvar Modelo
  - 1666 – Copiar Modelo (CURINGA6)
  - 1668 – Salvar Configuração Movimentação (C8) – pré‑preenchimento

Observações de mapeamento (Servlet → REST):
- NOVO → POST `/modelos/preparar`
- SALVAR (pré‑validação) → POST `/modelos/confirmar`
- SALVARRESULTADO (persistência) → POST/PUT conforme criação/edição
- LOCALIZAR → GET `/modelos`
- EXCLUIR/EXCLUIRRESULTADO → DELETE `/modelos/{id}`
- CURINGA6 (copiar) → POST `/modelos/{id}/copiar`
- CURINGA7 (código de barras) → GET `/modelos/codigo-barra?numero=...` (retorna imagem)
- CURINGA8 (salvar pré‑preenchimento) → POST `/modelos/{id}/configuracoes/pre-preenchimento`
- LOCALIZARDWR (texto para e‑Carta) → GET `/modelos/{id}/texto`

### 2) Regras e validações
- Obrigatórios ao salvar: `modelo` (descrição) e `idArquivoTipo`. Rejeitar se ausentes.
- Validação de edição/salvamento (negócio):
  - Apenas o proprietário (mesma `idUsuarioServentia`), o chefe do proprietário ou administrador podem alterar/excluir.
  - Não permitir excluir modelos genéricos (sem `idServentia` e sem `idUsuarioServentia`), exceto se administrador.
  - Não permitir excluir/alterar modelos de outra serventia (comparar com `idServentia` do usuário), exceto se administrador.
- Regra específica de locomoção (mandado):
  - Campo opcional `qtdLocomocao` só é aplicável quando o `idArquivoTipo` corresponde a Mandado (`ArquivoTipoDt.MANDADO`).
  - E somente visível/permitido para usuários do grupo “Gerenciamento de Tabelas” na serventia `ServentiaDt.GERENCIAMENTO_SISTEMA_PRODUDI`.
- Cópia de modelo (copiar): ao copiar, zerar campos de identificação e de autoria (`id`, `modeloCodigo`, `idServentia`, `idUsuarioServentia`, `qtdLocomocao`), mantendo `texto` e `idArquivoTipo` como base do novo rascunho.
- Pré‑preenchimento (C8): persistir configuração associada ao modelo; retornar 400 com mensagem de erro quando validações falharem.
- Consulta/paginação: suportar filtros por `modelo` (descrição), `idArquivoTipo` e `arquivoTipo` (texto), com paginação (`pagina`, `tamanho`).
- Auditoria: preencher a partir do contexto autenticado (não do payload) `idUsuarioLog` e `ipComputadorLog` em todas as operações de escrita.

### 3) Modelos de dados (DTOs)
- Modelo
  - id: string
  - modelo: string (descrição) [obrigatório]
  - modeloCodigo: string
  - texto: string (HTML)
  - idArquivoTipo: string [obrigatório]
  - arquivoTipo: string (rótulo)
  - idServentia: string
  - serventia: string (rótulo)
  - idUsuarioServentia: string
  - usuarioServentia: string (rótulo)
  - codigoTemp: string
  - serventiaCodigo: string
  - qtdLocomocao: string|null (apenas para mandado e perfil específico)
  - Computados no backend (somente leitura):
    - criador: string ("Serventia", "Usuário Serventia" ou "Genérico")

- Configuração de Pré‑Preenchimento (C8)
  - nomeArquivo: string
  - movimentacaoTipo/idMovimentacaoTipo: string
  - movimentacaoComplemento: string
  - classificadorPendencia/idClassificadorPendencia: string
  - classificadorProcesso1/idClassificadorProcesso1: string
  - classificadorProcesso2/idClassificadorProcesso2: string
  - naoGerarVerificarProcesso: "S"|"N"
  - listaPendencias: array de ids (opcional)

Observações:
- O legado expõe um grande conjunto de variáveis de substituição (placeholders) para mail‑merge em `ModeloDt` (constantes). Elas não são editadas via endpoints, mas são usadas pelo mecanismo de renderização de texto/HTML.

### 4) Endpoints REST propostos

Consultas
- GET `/modelos`
  - Permissão: 1662 (Localizar)
  - Query: `modelo`, `idArquivoTipo`, `arquivoTipo`, `pagina` (default 0), `tamanho` (default 20)
  - Ação: lista modelos conforme filtros e escopo do usuário (serventia/proprietário)
  - Retorno: 200 com `{ itens: [ { id, modelo, criador, idArquivoTipo, arquivoTipo } ], pagina, tamanho, total }`

- GET `/modelos/{id}`
  - Permissão: 1662 (Localizar)
  - Ação: consulta detalhada do modelo
  - Retorno: 200 com DTO Modelo; 404 se não encontrado ou fora do escopo

- GET `/modelos/{id}/texto`
  - Permissão: 1663 (LocalizarDWR)
  - Ação: retorna JSON com `id`, `modelo`, `texto` (HTML) para consumo direto por clientes (ex.: e‑Carta)
  - Retorno: 200; 404 se não encontrado

Criação/Edição
- POST `/modelos/preparar`
  - Permissão: 1664 (Novo)
  - Body: opcional (pode aceitar `idArquivoTipo` para pré‑popular)
  - Ação: retorna DTO inicial com defaults e metadados auxiliares (ex.: combos permitidos)
  - Retorno: 200 com DTO base

- POST `/modelos/confirmar`
  - Permissão: 1665 (Salvar)
  - Body: DTO Modelo (sem campos de auditoria)
  - Ação: executa validações de negócio de edição/criação; não persiste
  - Retorno: 200 se válido; 400 com detalhes de validação

- POST `/modelos`
  - Permissão: 1665 (Salvar)
  - Body: DTO Modelo (sem `id`)
  - Ação: cria um novo modelo após validar regras; auditoria do usuário atual
  - Retorno: 201 com DTO criado; 400 validação; 409 conflito de negócio

- PUT `/modelos/{id}`
  - Permissão: 1665 (Salvar)
  - Body: DTO Modelo (campos editáveis)
  - Ação: atualiza um modelo existente após validação (proprietário/chefe/admin e serventia)
  - Retorno: 200 com DTO atualizado; 400 validação; 403 permissão; 404 não encontrado; 409 conflito

Exclusão
- DELETE `/modelos/{id}`
  - Permissão: 1660 (Excluir)
  - Ação: excluir modelo respeitando regras (não excluir genéricos, nem de outra serventia; admin pode)
  - Retorno: 204; 403/404 conforme regra

Operações especiais
- POST `/modelos/{id}/copiar`
  - Permissão: 1666 (Copiar)
  - Ação: clona o modelo, zerando identificadores/campos de autoria; retorna rascunho criado
  - Retorno: 201 com DTO do novo modelo

- POST `/modelos/{id}/configuracoes/pre-preenchimento`
  - Permissão: 1668 (Salvar Configuração Movimentação – C8)
  - Body: DTO de configuração de pré‑preenchimento (ver seção 3)
  - Ação: persiste a configuração associada ao modelo
  - Retorno: 200; 400 com mensagem quando inválido

- GET `/modelos/codigo-barra`
  - Permissão: 1662 (Localizar) – uso utilitário associado ao módulo
  - Query: `numero` (string)
  - Ação: gera imagem de código de barras do número informado
  - Retorno: 200 image/png; 400 se parâmetro inválido

Auxiliares (combos)
- GET `/arquivo-tipos`
  - Permissão: conforme módulo ArquivoTipo (consulta)
  - Query: `descricao`, `pagina`, `tamanho`
  - Ação: lista tipos de arquivo disponíveis ao usuário

- GET `/serventias` e GET `/usuarios/serventias`
  - Permissão: conforme módulos respectivos
  - Ação: consultas auxiliares para preenchimento de autoria/escopo

### 5) Respostas e códigos de status
- 200: sucesso em consultas/validações/atualizações
- 201: criação de recurso (novo modelo; cópia)
- 204: exclusão sem corpo
- 400: erro de validação (campos obrigatórios, regras de negócio como pré‑preenchimento)
- 403: usuário sem permissão ou fora do escopo (serventia/propriedade)
- 404: não encontrado
- 409: conflito de negócio (ex.: tentativa de excluir genérico fora das regras)

### 6) Observações de segurança
- Autenticação/Autorização obrigatórias; aplicar verificação de grupo, serventia e propriedade do recurso.
- Conteúdo `texto` é HTML: aplicar sanitização/whitelisting no backend antes de persistir/entregar.
- LGPD: variáveis de substituição podem conter dados pessoais; proteger acesso por permissão/escopo e registrar auditoria.

### 7) Itens a decidir na migração
- Tamanho máximo de `texto` (HTML) e política de sanitização.
- Paginação padrão (`tamanho` e limites máximos) nas consultas.
- Se o endpoint de código de barras deve permanecer neste módulo ou migrar para um utilitário geral.
- Exposição opcional de uma lista de placeholders suportados para ajudar o editor de modelos (pode ser derivado das constantes em `ModeloDt`).


