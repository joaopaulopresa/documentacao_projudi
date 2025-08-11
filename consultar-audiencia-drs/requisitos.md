## Requisitos – Consultar Audiência DRS (por Processo)

Escopo: migrar a funcionalidade legada de consulta de audiências DRS por número de processo (JSP/Servlet) para endpoints REST. Foco em regras, permissões, modelos de dados e endpoints; sem descrição de telas.

### 1) Permissões
- **Código base**: `251` – Consultar Audiencia DRS
- **Subpermissões** (do legado):
  - `2512` – LOCALIZAR (consulta)
  - `2514` – NOVO (preparar/limpar contexto)

### 2) Regras e validações
- **Consulta por número do processo**:
  - Chamada de negócio: `AudienciaDRSNe.consulteProcessoComAudiencias(numeroProcesso)`
  - Se não localizar em SPG, SSG ou Projudi: retornar 404.
  - Se localizar e não houver audiências: retornar 200 com lista vazia e mensagem informativa.
- **Segredo de justiça**:
  - Projudi (eletrônico) com segredo: permitir visualizar metadados, mas bloquear download via endpoint; mensagem orientando a acessar a movimentação no Projudi.
  - SPG (físico) com segredo: permitir visualizar metadados, mas bloquear download via endpoint; mensagem orientando a solicitar o DVD na serventia.
  - Quando bloqueado, retornar 403 em endpoints de download e/ou sinalizar `canDownload=false` na listagem.
- **Validação de formato** do número do processo: CNJ padrão `NNNNNNN-DD.AAAA.J.TR.OOOO` (ex.: `5000280-28.2010.8.09.0059`), aceitar variações sem pontuação; normalizar.
- **Auditoria**: sempre registrar `idUsuarioLog` e `ipComputadorLog`.

### 3) Modelos de dados (DTO)
- `RetornoDRSProcesso` (resumo):
  - `processoNumeroCompleto: string`
  - `sistema: "PROJUDI" | "SPG" | "SSG"` (ou flags de presença conforme legado)
  - `possuiProcesso: boolean`
  - `possuiAudiencias: boolean`
  - `segredoJustica: boolean`
  - `mensagemBloqueioDownload?: string` (preenchida quando `segredoJustica=true`)
  - `processoDetalhes?: { ... }` (campos variam por sistema)
  - `polosAtivos: Parte[]`
  - `polosPassivos: Parte[]`
  - `audiencias: Audiencia[]`
- `Parte` (Projudi): `nome, nomeMae, cpfCnpj, dataNascimento`
- `Parte` (SPG/SSG): `nome, filiacao, cpf, dataNascimento`
- `ProcessoDetalhes` (Projudi): `serventia, processoTipo, processoFase, classificador, dataRecebimento`
- `ProcessoDetalhes` (SPG): `serventia, classe, fase, classificador, dataDistribuicao`
- `ProcessoDetalhes` (SSG): `codComarca, nomeComarca`
- `Audiencia`:
  - `dataHora: string` (ISO 8601)
  - `urlDownloadCriptografado?: string` (quando permitido)
  - `canDownload: boolean`
  - `anexos: Anexo[]`
- `Anexo`:
  - `urlDownloadCriptografado?: string` (quando permitido)
  - `canDownload: boolean`

Observações:
- O legado não expõe identificadores estáveis para audiências/anexos; considerar índice estável, hash de data/URL ou ID retornado pelo DRS caso disponível na integração.
- O campo legado `browserEhIE` é desnecessário em REST.

### 4) Endpoints REST propostos

- Consulta/preparação
  - POST `/audiencias-drs-processos/preparar`
    - Permissão: `2514` (NOVO)
    - Corpo: vazio
    - Ação: inicializa/retorna payload base (pode retornar apenas `{ ok: true }`); sem efeitos colaterais.
    - Resposta: 200

- Consulta por processo
  - GET `/audiencias-drs-processos`
    - Permissão: `2512` (LOCALIZAR)
    - Query: `numeroProcesso` (obrigatório)
    - Ação: invoca a consulta, normaliza número CNJ, determina sistema (Projudi/SPG/SSG), popula partes, detalhes e audiências.
    - Regras:
      - Não encontrado: 404
      - Sem audiências: 200 com `audiencias: []` e `possuiAudiencias=false`
      - Segredo: incluir `segredoJustica=true`, `canDownload=false` em cada item e `mensagemBloqueioDownload`
    - Resposta: 200 (objeto `RetornoDRSProcesso`)

- Download de vídeo da audiência
  - GET `/audiencias-drs-processos/{numeroProcesso}/audiencias/{indice}/download`
    - Permissão: `2512` (LOCALIZAR)
    - Ação: quando permitido, redireciona (302) para `urlDownloadCriptografado` retornada pelo DRS; quando segredo, 403.
    - Respostas: 302 (Location), 403, 404 (índice inválido)

- Download de anexo de audiência
  - GET `/audiencias-drs-processos/{numeroProcesso}/audiencias/{indice}/anexos/{indiceAnexo}/download`
    - Permissão: `2512` (LOCALIZAR)
    - Regras/Respostas: iguais ao download de vídeo.

### 5) Respostas e códigos de status
- 200: sucesso; também para caso sem audiências (lista vazia)
- 302: redirecionamento para URL de download quando permitido
- 400: validação (ex.: `numeroProcesso` inválido)
- 403: segredo de justiça (download bloqueado)
- 404: processo não localizado (SPG/SSG/Projudi) ou item inexistente

### 6) Observações de segurança
- Autenticação/autorização obrigatória pelas permissões informadas.
- Dados pessoais (partes, filiação, CPF, datas) sujeitos à LGPD; assegurar minimização e finalidade explícita.
- URLs de download devem ser temporárias/assinadas; não persistir nem expor além do necessário.
- Auditoria de todas as chamadas: `idUsuarioLog`, `ipComputadorLog`, data/hora e parâmetros de consulta.

### 7) Itens a decidir na migração
- Definição de identificadores estáveis para audiências/anexos quando a origem não fornecer IDs.
- Tempo de expiração e política de geração de URLs de download (presign/redirect vs. proxy).
- Padrão de normalização/validação do número CNJ (tolerância a máscaras/variações).
- Padrão de mensagens ao usuário para casos de segredo e de ausência de audiências (campo dedicado vs. i18n).


