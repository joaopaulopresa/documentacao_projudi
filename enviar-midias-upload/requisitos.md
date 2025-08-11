## Requisitos – Enviar Mídias – Upload

Escopo: migrar a funcionalidade legada de upload de mídias (JSP/Servlet `MidiaPublicada`) para endpoints REST. Foca em permissões, regras/validações, modelo de dados e endpoints. Ignorar UI.

### 1) Permissões
- Código base: `347` – Enviar Mídias – Upload
- Subpermissões (legado JSON):
  - `3474` – NOVO (preparar)
  - `3475` – SALVAR (validar dados – data/hora/complemento)
  - `3477` – CURINGA7 (gerar URL assinada para upload direto no storage)
  - `3479` – CURINGA9 (upload via servidor com chunks)
- Observação: existe fluxo `SALVARRESULTADO` no servlet (efetivar movimentação). Não consta no JSON anexo; considerar como ação efetivadora sob o código base 347, ou mapear permissão específica na migração.

### 2) Regras e validações
- Contexto exigido: operação vinculada a um processo existente.
- Auditoria: registrar `idUsuarioLog` e `ipComputadorLog` em todas as operações que alteram estado.
- Dados obrigatórios para efetivar movimentação:
  - `dataRealizacao` (dd/MM/yyyy) – validar formato.
  - `horaRealizacao` (HH:mm) – validar formato.
  - `complemento` (opcional; até 80 caracteres no legado).
- Upload de arquivos:
  - Nome do arquivo sanitizado (remoção de caracteres inválidos) antes de persistir/enviar.
  - Suporte a upload em partes (chunks) com cabeçalho `Content-Range`.
  - Limites por usuário (obter de configuração): quantidade máxima de arquivos, tamanho máximo por arquivo (MB), tipos MIME/extensões permitidas.
  - Dois modos suportados:
    - URL assinada (direto no storage, ex.: S3/Ceph) – geração prévia de URL com tempo de expiração e método definido (PUT/POST), possivelmente com upload sequencial.
    - Upload via servidor – recepção multipart/chunk, composição em diretório temporário e, ao final, envio ao storage e retorno do URL de download.
- Efetivação: após upload completo dos arquivos e validação de data/hora, gerar movimentação do processo "Mídia Publicada", usando complemento: "<Data dd/MM/yyyy HH:mm> - <texto opcional>".
- Erros de negócio/validação devem retornar mensagem clara (ex.: data/hora inválidas; tipo/tamanho não permitido; upload incompleto).

### 3) Modelos de dados (DTO)
- MidiaPublicada
  - `processoId`: string | number
  - `dataHora`: string ISO (armazenado a partir de `dataRealizacao` + `horaRealizacao`)
  - `complemento`: string
  - `idUsuarioLog`: string
  - `ipComputadorLog`: string
  - `arquivos`: MidiaPublicadaArquivo[]
- MidiaPublicadaArquivo
  - `id` (gerado na preparação ou pelo storage)
  - `nomeOriginal`
  - `nomeSanitizado`
  - `contentType`
  - `tamanhoBytes`
  - `uploadCompleto`: boolean
  - `urlDownload` (preenchido após envio ao storage)
  - `mensagemErro` (quando aplicável)
- ConfigUpload (retorno de preparação)
  - `quantidadeMaxima`
  - `tamanhoMaximoMB`
  - `tiposPermitidos`

Observações:
- No legado, campos de auditoria e contexto de processo são populados no servidor; na migração, trafegar explicitamente (ou recuperar do token/contexto). 

### 4) Endpoints REST propostos

Preparação e validação
- POST `/midias-publicadas/preparar`
  - Permissão: 3474 (NOVO)
  - Body: `{ processoId }`
  - Ação: inicia contexto de upload, limpa arquivos pendentes, retorna parâmetros de upload (limites, tipos) e um `uploadSessionId` (sugestão) para correlacionar etapas.
  - Retorno 200: `{ uploadSessionId, config: ConfigUpload }`

- POST `/midias-publicadas/confirmar`
  - Permissão: 3475 (SALVAR)
  - Body: `{ uploadSessionId, processoId, dataRealizacao, horaRealizacao, complemento }`
  - Ação: valida data/hora; compõe `dataHora` e mantém em sessão/servidor para etapa final.
  - 200: `{ sucesso: true, dataHoraISO }`
  - 400: `{ sucesso: false, mensagem }` (erros de formato/obrigatórios)

Upload – modo URL assinada (direto no storage)
- POST `/midias-publicadas/gerar-url-upload`
  - Permissão: 3477 (CURINGA7)
  - Body: `{ uploadSessionId, processoId, nomeArquivo, contentType, tamanhoBytes }`
  - Ação: gera URL assinada do storage e metadados (método, headers) para o cliente enviar o arquivo diretamente.
  - 200: `{ urlAssinada, metodo, headers, expiresAt }`

Upload – modo servidor (multipart/chunk)
- POST `/midias-publicadas/upload-arquivo`
  - Permissão: 3479 (CURINGA9)
  - Headers: `Content-Range` (quando chunked)
  - Query/Body: `{ uploadSessionId, processoId, contentType? }` + multipart `files[]`
  - Ação: recebe parte/arquivo, grava em pasta temporária; ao concluir, envia ao storage e retorna metadados (nome, tamanho, url).
  - 200: `{ files: [{ name, size, url? , error? }] }` ou `{ sucesso: true, info }` para progresso/estado intermediário
  - 400/409: conflitos de chunks, tipo/tamanho inválidos, etc.

Efetivação
- POST `/midias-publicadas/efetivar`
  - Permissão: 347 (base) – ou permissão específica a definir para SALVARRESULTADO
  - Body: `{ uploadSessionId, processoId }` (dataHora/complemento já confirmados)
  - Ação: gera movimentação "Mídia Publicada" no processo, vinculando os arquivos enviados; encerra a sessão de upload.
  - 200: `{ sucesso: true, idMovimentacao }`
  - 409: conflito de negócio (ex.: upload incompleto/sem arquivos)

Consultas auxiliares
- GET `/midias-publicadas/uploads`
  - Permissão: 347
  - Query: `{ uploadSessionId }`
  - Ação: lista o estado atual dos arquivos no upload em andamento.
  - 200: `{ arquivos: MidiaPublicadaArquivo[] }`

### 5) Respostas e códigos de status
- 200 OK: operações de preparação/validação/consulta e upload concluído.
- 201 Created: opcional para criação de sessão de upload.
- 400 Bad Request: validações de formato/obrigatórios (data/hora, tipos/tamanhos, range inválido).
- 403 Forbidden: falta de permissão (código 347 ou subpermissões).
- 404 Not Found: sessão ou processo inexistente.
- 409 Conflict: estado de upload inconsistente (ex.: arquivo não finalizado) ou regra de negócio na efetivação.

### 6) Observações de segurança
- Autenticação/Autorização obrigatória; checar permissão base 347 e subpermissões por operação.
- Processos sigilosos: validar acesso do usuário ao processo antes de gerar URL/upload/efetivar.
- URLs assinadas: prazo curto, escopo estrito (objeto único), método controlado; nunca expor credenciais do bucket.
- Sanitização de nomes de arquivo; rejeitar extensões não permitidas.
- Limitar tamanho/quantidade por configuração do usuário/perfil; considerar varredura antivírus antes de disponibilizar `urlDownload`.
- Auditar `idUsuarioLog`, `ipComputadorLog`, carimbo de data/hora, lista final de arquivos e resultado da efetivação.

### 7) Itens a decidir na migração
- Unificação do modo de upload (direto no storage vs. via servidor) ou manter ambos conforme ambiente/rede.
- Identificador de sessão (`uploadSessionId`) e política de expiração/limpeza de temporários.
- Estratégia de chunking (tamanho, paralelismo) e idempotência de partes.
- Política de erros e reprocessamento (retentativas) no envio ao storage.
- Permissão específica para efetivar (equivalente ao `SALVARRESULTADO`) ou reutilizar permissão base 347.
- Padrão do `urlDownload` (presign vs. público) e tempo de expiração.


