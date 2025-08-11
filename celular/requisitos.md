## Requisitos – Celulares do Usuário (Liberação/Bloqueio)

Escopo: migrar a funcionalidade legada (JSP/Servlet `UsuarioFone`) para endpoints REST. Ignorar UI. Cobre permissões, regras/validações, modelo de dados e endpoints necessários.

### 1) Permissões
- Código base: `291` (Celular)
- Subpermissões mapeadas do legado (`celular.json`):
  - `2910` – EXCLUIR
  - `2911` – IMPRIMIR
  - `2912` – LOCALIZAR
  - `2913` – LOCALIZARDWR
  - `2914` – NOVO
  - `2915` – SALVAR
  - `2916` – CURINGA6 (bloquear/liberar aparelho)
  - `2917` – CURINGA7 (reservado – não usado no código atual)
  - `2918` – CURINGA8 (reservado – não usado no código atual)

Observação: o legado também consulta usuários via permissão de `UsuarioDt` para localizar o usuário vinculado (case especial no controlador).

### 2) Regras e validações
- Negócio
  - Listar celulares vinculados ao usuário logado.
  - Cadastrar/editar celular (telefone e IMEI) para um usuário.
  - Liberar e bloquear um celular:
    - Liberado quando há `dataLiberacao` preenchida (regra do método `isLiberado`).
    - Bloquear: revogar liberação (definição de persistência a decidir; ver Itens a decidir).
  - Excluir cadastro de celular.
  - Busca de cadastros (listagem) com paginação.
- Validações (executadas por `UsuarioFoneNe.Verificar` no legado; explicitar no REST):
  - `fone`: obrigatório; formato de telefone nacional (DDD + número). Ex.: `^\(?\d{2}\)?\s?\d{4,5}-?\d{4}$`.
  - `imei`: obrigatório; 15 dígitos numéricos (Luhn para IMEI opcional).
  - Unicidade recomendada: evitar duplicidade de `imei` ativo por usuário.
  - `codigo`/`codigoValidade`: se usados para confirmação, exigir quando operação assim demandar.
  - Consistência de estado: não liberar aparelho já liberado; não bloquear aparelho já bloqueado.
- Perfis/Autorização
  - Todas as operações exigem autenticação.
  - Verificação de permissão por subação:
    - Consultar: `2912`.
    - Novo/Salvar: `2914`/`2915`.
    - Bloquear/Liberação: `2916`.
    - Excluir: `2910`.
    - Imprimir (se implementado): `2911`.
- Auditoria (obrigatório em todas as operações de escrita)
  - `idUsuarioLog`: ID do usuário logado.
  - `ipComputadorLog`: IP de origem da requisição.

### 3) Modelos de dados (DTO)
Baseado em `UsuarioFoneDtGen`/`UsuarioFoneDt`:
- `id` (`Id_UsuFone`): string/UUID
- `idUsuario` (`Id_Usuario`): string
- `usuario` (`Usuario`): string (identificador legível do usuário)
- `imei` (`Imei`): string (15 dígitos)
- `fone` (`Fone`): string (telefone)
- `codigo` (`Codigo`): string (código de verificação)
- `codigoValidade` (`CodigoValidade`): string/data (validade do código)
- `dataPedido` (`DataPedido`): string/data
- `dataLiberacao` (`DataLiberacao`): string/data (preenchida quando liberado)
- `codigoTemp` (`CodigoTemp`): string
- Flags/derivados:
  - `liberado`: boolean (derivado: `dataLiberacao` não vazia)
- Auditoria (no payload de escrita): `idUsuarioLog`, `ipComputadorLog`

### 4) Endpoints REST propostos

Consulta/Listagem
- GET `/celulares`
  - Permissão: `2912`
  - Query: `termo` (busca por fone/IMEI/usuario), `page`, `pageSize`, `idUsuario` (opcional)
  - Ação: retorna lista paginada de celulares
  - Retorno: `{ itens: UsuarioFone[], total: number }`

- GET `/celulares/me`
  - Permissão: `2912`
  - Ação: lista celulares vinculados ao usuário autenticado
  - Retorno: `UsuarioFone[]`

- GET `/celulares/{id}`
  - Permissão: `2912`
  - Ação: consulta detalhada por ID
  - Retorno: `UsuarioFone`

Cadastro/Edição
- POST `/celulares`
  - Permissão: `2914`/`2915`
  - Body: `UsuarioFone` (sem `id`)
  - Ação: valida e cria novo registro
  - Retornos: `201` com recurso criado; `400` se validação falhar

- PUT `/celulares/{id}`
  - Permissão: `2915`
  - Body: `UsuarioFone`
  - Ação: valida e atualiza registro existente
  - Retornos: `200`; `404` se não encontrado; `400` validação

Operações especiais
- POST `/celulares/{id}/liberar`
  - Permissão: `2916`
  - Body: opcional `{ codigo, codigoValidade }` se política exigir
  - Ação: libera o aparelho (preenche `dataLiberacao`)
  - Retornos: `200`; `409` se já liberado; `404` se não encontrado

- POST `/celulares/{id}/bloquear`
  - Permissão: `2916`
  - Ação: bloqueia o aparelho (revoga liberação)
  - Retornos: `200`; `409` se já bloqueado; `404` se não encontrado

Exclusão
- DELETE `/celulares/{id}`
  - Permissão: `2910`
  - Ação: exclui o registro
  - Retornos: `204`; `404` se não encontrado; `409` se conflito de negócio

Auxiliares
- GET `/usuarios` (apoio à seleção de usuário)
  - Permissão: conforme módulo de usuários
  - Query: `termo`, `page`, `pageSize`
  - Retorno: `{ itens: { id, nome, ... }[], total }`

- GET/POST `/celulares/impressao` (opcional)
  - Permissão: `2911`
  - Ação: gerar/baixar relatório de celulares do usuário ou da serventia

### 5) Respostas e códigos de status
- 200 OK – sucesso em consultas e operações
- 201 Created – criação concluída
- 204 No Content – exclusão concluída
- 400 Bad Request – validação de dados
- 403 Forbidden – falta de permissão
- 404 Not Found – recurso não encontrado
- 409 Conflict – conflito de negócio (estado inconsistente)

### 6) Observações de segurança
- Autenticação obrigatória; autorização por subpermissão.
- LGPD: dados pessoais sensíveis (telefone/IMEI). Não expor além da necessidade.
- Auditoria: registrar `idUsuarioLog` e `ipComputadorLog` em POST/PUT/DELETE e operações especiais.
- Proteções contra enumeração: paginar resultados e aplicar filtros por escopo do usuário quando aplicável.

### 7) Itens a decidir na migração
- Semântica de bloqueio/liberação no armazenamento: limpar `dataLiberacao` ou manter histórico/flag dedicado (ex.: `bloqueado = true`).
- Política de códigos (`codigo`, `codigoValidade`) e uso do `hash` temporário: obrigar para liberação? fluxo de geração/validação.
- Limites: permitir múltiplos aparelhos por usuário? regras de unicidade de `imei`/`fone`.
- Implementar ou não endpoint de impressão (`2911`).
- Padrão de IDs (UUID vs. numérico) e formatação de datas no payload (`ISO 8601`).


