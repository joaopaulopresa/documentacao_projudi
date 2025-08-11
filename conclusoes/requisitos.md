## Requisitos – Conclusões (Troca de Responsável)

Breve: Migrar a funcionalidade de troca de responsável por conclusões (lote e por processo específico) para endpoints REST, mantendo validações de negócio, auditoria e restrição por serventia/permissão.

### 1) Permissões
- Código base: 805 – "Conclusões"
- Subpermissões:
  - 8054 – "Trocar Responsável" (acesso/preparação da troca)
  - 8055 – "Conclusões SALVAR" (efetivação/persistência da troca)

Fonte: `doc/conclusoes/conclusoes.json` e `Permissao()` em `TrocarConclusaoMagistradoCt.java`.

### 2) Regras e validações
- Troca em lote (serventia):
  - Transferir as N conclusões mais antigas do magistrado atual para o novo magistrado, ambos da mesma serventia do usuário solicitante.
  - Validações de negócio via `verificarTrocaResponsavelConclusaoLote(atualResponsavelId, novoResponsavelId, quantidade)`.
  - Regras mínimas: `quantidade > 0`; `atualResponsavelId != novoResponsavelId`; IDs válidos; perfis/lotação habilitados.
  - Apenas conclusões em aberto entram na contagem (implícito pela regra de negócio).
- Troca por processo específico:
  - Permitida somente se o processo possuir conclusão aberta (`consultarConclusaoAbertaProcesso(idProcesso, null)`).
  - Em caso de inexistência, retornar erro de negócio (conflito) informando que o processo não está concluso.
- Restrição por serventia:
  - Listas de responsáveis são limitadas à serventia do usuário ou do processo.
- Auditoria obrigatória:
  - Registrar `idUsuarioLog` e `ipComputadorLog` em operações de efetivação.
- Perfis/grupos:
  - Exigir permissão 805 (base) + 8054/8055 conforme etapa; normalmente perfis de magistrado/chefia de gabinete/serventia.

Referências: `TrocarConclusaoMagistradoCt.java` (switch `Configuracao.Novo`, `Salvar`, `SalvarResultado`, `Curinga6`).

### 3) Modelos de dados (DTO)
- ServentiaCargo (resumo para combos):
  - `id`: string
  - `serventiaCargo`: string
  - `serventia`: string
  - `cargoTipo`: string
  - `nomeUsuario`: string
- TrocaConclusaoLoteRequest (body):
  - `atualResponsavelId`: string (id de `ServentiaCargo`)
  - `novoResponsavelId`: string (id de `ServentiaCargo`)
  - `quantidade`: number (inteiro > 0)
- TrocaConclusaoProcessoRequest (body):
  - `processoId`: string
  - `conclusaoId`: string (quando disponível; pode ser omitido se o backend localizar a conclusão aberta)
  - `novoResponsavelId`: string (id de `ServentiaCargo`)
- Respostas comuns:
  - `mensagem`: string
  - `erros`: array de strings (opcional)
  - `responsaveis`: `ServentiaCargo[]` (para endpoints de consulta)

Observação: Os DTOs refletem o mínimo necessário observado no legado e podem ser enriquecidos conforme necessidade.

### 4) Endpoints REST propostos

1) Consultas auxiliares (combos/listas)
- GET `/conclusoes/responsaveis`
  - Permissão: 805
  - Query: `serventiaId` (opcional; padrão = serventia do usuário)
  - Ação: listar magistrados responsáveis aptos à troca dentro da serventia
  - Retorno: `200 { responsaveis: ServentiaCargo[] }`

- GET `/processos/{processoId}/conclusao/responsaveis`
  - Permissão: 805
  - Ação: validar se há conclusão aberta e listar responsáveis (mesma serventia do processo)
  - Retorno: `200 { responsaveis: ServentiaCargo[] }`; `409` se não houver conclusão aberta

2) Troca em lote (serventia)
- POST `/conclusoes/trocas/preparar`
  - Permissão: 8054
  - Body: `TrocaConclusaoLoteRequest`
  - Ação: preparação textual da operação (ex.: mensagem de confirmação: "Troca das N conclusões mais antigas...")
  - Retorno: `200 { mensagem }`

- POST `/conclusoes/trocas/confirmar`
  - Permissão: 8054
  - Body: `TrocaConclusaoLoteRequest`
  - Ação: validação de negócio, retornando mensagens de bloqueio quando houver (espelha `verificarTrocaResponsavelConclusaoLote`)
  - Retorno: `200 { mensagem }` ou `400 { erros }`

- POST `/conclusoes/trocas`
  - Permissão: 8055
  - Body: `TrocaConclusaoLoteRequest`
  - Ação: efetivar/persistir a troca em lote, com auditoria (`idUsuarioLog`, `ipComputadorLog`)
  - Retorno: `200 { mensagem }`

3) Troca por processo específico
- POST `/processos/{processoId}/conclusao/troca/preparar`
  - Permissão: 8054
  - Body: `{ conclusaoId?, novoResponsavelId }`
  - Ação: preparar a troca; quando `conclusaoId` não for informado, o backend tentará localizar a conclusão aberta
  - Retorno: `200 { mensagem }`; `409` se o processo não estiver concluso

- POST `/processos/{processoId}/conclusao/troca`
  - Permissão: 8055
  - Body: `TrocaConclusaoProcessoRequest`
  - Ação: efetivar a troca do responsável pela conclusão do processo (1º grau)
  - Retorno: `200 { mensagem }`

### 5) Respostas e códigos de status
- 200: sucesso (consulta/preparação/validação/efetivação)
- 400: falha de validação (parâmetros ou regras de negócio)
- 403: falta de permissão (805/8054/8055)
- 404: recurso não encontrado (processo/responsável inexistente)
- 409: conflito de negócio (ex.: processo não concluso)

### 6) Observações de segurança
- Autenticação e autorização obrigatórias; checar permissão base 805 e específicas por etapa (8054/8055).
- Restringir operações à serventia do usuário (e do processo, no fluxo por processo).
- Auditoria: registrar `idUsuarioLog` e `ipComputadorLog` em operações de efetivação.
- Dados pessoais de usuários (magistrados) devem observar LGPD.

### 7) Itens a decidir na migração
- Idempotência/controle de concorrência para operações em lote.
- Paginação/ordenação nas listas de responsáveis.
- Definição de limites seguros para `quantidade` em lote.
- Estratégia de resolução de `conclusaoId` quando omitido (por processo).
- Geração de eventos/logs operacionais para rastreabilidade.


