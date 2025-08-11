## Requisitos – Gestão de Assessores/Assistentes

### Escopo e objetivo
Documentar a funcionalidade “Assessor” do legado JSP/Servlet para migração a endpoints REST. Foco em operações, parâmetros, validações e permissões. Ignorar navegação/telas.

Fontes: `AssessorCt.java`, DTO `AssistenteDt.java` e permissões em `assessor.json`.

### Permissões relevantes
- Código base da funcionalidade: **257 – Assessor** (classe `AssistenteDt.CodigoPermissao`)
- Subpermissões (conforme `assessor.json`):
  - **2572 – LOCALIZAR**: Consultas (lista/seleção)
  - **2573 – LOCALIZAR DWR**: Consultas auxiliares (referências)
  - **2574 – NOVO**: Preparar/limpar dados para cadastro
  - **2575 – SALVAR**: Salvar dados (criar/atualizar, vincular à serventia, bloquear/liberar “guardar para assinar”)
  - **2570 – EXCLUIR**: Ativar/Desativar vínculo do assessor na serventia

### Regras e validações principais
- Criação/edição de assessor:
  - Validações de negócio em `UsuarioNe.VerificarAssessor` (ex.: integridade de CPF, obrigatórios, unicidade, formato de e-mail, etc.).
  - Definição da situação funcional de acordo com o grupo do usuário chefe: advogado, MP, magistrado ou padrão.
  - Senha inicial padrão para novos usuários (legado: “12345”), exigir troca posterior.
- Vínculo assessor ↔ serventia/grupo do chefe:
  - Validações em `UsuarioNe.VerificarAssistenteServentia` antes de `salvarAssistenteServentia`.
  - Ativar/Desativar vínculo por `Id_UsuarioServentiaGrupo`.
- Controle “guardar para assinar” por assessor:
  - `libereAssistenteGuardarParaAssinar` e `bloqueieAssistenteGuardarParaAssinar`.
- Consultas auxiliares de referência (cidade, órgão expedidor do RG, bairro) para preencher endereço do assessor.

### Modelo de dados (principais campos do assessor)
Derivados de parâmetros aceitos em `AssessorCt`:
- Identificação: `id`, `usuario` (login/CPF), `cpf`, `rg`, `rgDataExpedicao`, `idRgOrgaoExpedidor`/`rgOrgaoExpedidor`.
- Dados pessoais: `nome`, `sexo`, `dataNascimento`, `email`, `telefone`, `celular`.
- Acesso: `senha` (criação), `grupo`, `grupoCodigo`, `grupoTipoCodigo`.
- Vínculos: `idUsuarioServentia` (chefe), `idUsuarioServentiaGrupo` (vínculo), `idServentia`, `serventia`.
- Naturalidade e endereço: `idCidade`/`cidade`, `idEndereco`/`endereco`, `logradouro`, `numero`, `complemento`, `idBairro`/`bairro`, `bairroIdCidade`/`bairroCidade`, `bairroUf`, `cep`.
- Auditoria: `idUsuarioLog`, `ipComputadorLog`.

---

## Endpoints REST propostos

### 1) Consultas e seleção
- GET `/assessores`
  - Permissão: 2572 (LOCALIZAR)
  - Query: `usuario?`, `nome?`, `cpf?`
  - Retorna lista de assessores vinculados à serventia do chefe atual (dados essenciais + status do vínculo/guardar-para-assinar).

- GET `/usuarios`
  - Permissão: 2572 (LOCALIZAR)
  - Query: `query` (busca por nome/usuário/RG/CPF)
  - Uso: selecionar usuário existente para vincular como assessor.

- GET `/assessores/{id}`
  - Permissão: 2572 (LOCALIZAR)
  - Retorna dados completos do assessor e seus vínculos com serventias do chefe.

### 2) Criação/atualização de assessor
- POST `/assessores`
  - Permissão: 2575 (SALVAR)
  - Body (exemplo): {
    `cpf`, `nome`, `sexo?`, `dataNascimento?`,
    `rg?`, `rgDataExpedicao?`, `idRgOrgaoExpedidor?`,
    `email?`, `telefone?`, `celular?`,
    `endereco`: { `logradouro?`, `numero?`, `complemento?`, `idBairro?`, `bairro?`, `bairroIdCidade?`, `bairroCidade?`, `bairroUf?`, `cep?` },
    `idCidade?`, `cidade?`
  }
  - Ação: cria assessor (aplica `VerificarAssessor` e `salvarAssistente`). Para novos usuários, senha inicial padrão; exigir troca posterior.

- PATCH `/assessores/{id}`
  - Permissão: 2575 (SALVAR)
  - Body: campos editáveis do assessor e do endereço
  - Ação: atualiza dados (aplica `VerificarAssessor`).

### 3) Vínculo assessor ↔ serventia do chefe
- POST `/assessores/{id}/vinculos`
  - Permissão: 2575 (SALVAR)
  - Body: `{ idServentia, idUsuarioServentiaChefe, grupoUsuarioChefe }`
  - Ação: valida (`VerificarAssistenteServentia`) e cria vínculo (`salvarAssistenteServentia`).

- POST `/assessores/vinculos/{idVinculo}/ativar`
  - Permissão: 2570 (EXCLUIR)
  - Ação: ativa vínculo (`ativarUsuarioServentiaGrupo`).

- POST `/assessores/vinculos/{idVinculo}/desativar`
  - Permissão: 2570 (EXCLUIR)
  - Ação: desativa vínculo (`desativarUsuarioServentiaGrupo`).

### 4) Controle “guardar para assinar”
- POST `/assessores/{id}/guardar-para-assinar/liberar`
  - Permissão: 2575 (SALVAR)
  - Ação: liberação (`libereAssistenteGuardarParaAssinar`).

- POST `/assessores/{id}/guardar-para-assinar/bloquear`
  - Permissão: 2575 (SALVAR)
  - Ação: bloqueio (`bloqueieAssistenteGuardarParaAssinar`).

### 5) Endpoints auxiliares (referências)
- GET `/referencias/cidades?cidade=&uf=&page=`
  - Permissão: 2573 (LOCALIZAR DWR)
  - Retorna: cidades (`consultarDescricaoCidadeJSON`).

- GET `/referencias/rg-orgaos?sigla=&nome=&page=`
  - Permissão: 2573 (LOCALIZAR DWR)
  - Retorna: órgãos expedidores de RG (`consultarDescricaoRgOrgaoExpedidorJSON`).

- GET `/referencias/bairros?bairro=&cidade=&uf=&page=`
  - Permissão: 2573 (LOCALIZAR DWR)
  - Retorna: bairros com cidade/UF (`consultarDescricaoBairroJSON`).

---

## Respostas e códigos de status (resumo)
- 200: sucesso em consultas e operações.
- 201: criação de assessor/vínculo.
- 400: validação de parâmetros (ex.: campos obrigatórios, CPF inválido/duplicado).
- 403: usuário sem permissão para a ação.
- 404: entidade não localizada (assessor, vínculo, referências).
- 409: conflito de negócio (ex.: vínculo já existente, bloqueio/liberação incoerente com estado atual).

## Observações de segurança
- Todas as rotas exigem autenticação e avaliação de permissão (código 257 e subcódigos quando aplicável), além de checagens por perfil/grupo do usuário chefe.
- Dados sensíveis (CPF, endereço, contatos) devem ser trafegados e armazenados conforme LGPD.
- Senhas nunca devem retornar em respostas. Para novos usuários, exigir troca de senha no primeiro acesso.

## Itens a decidir na migração
- Política de redefinição/troca de senha no onboarding dos assessores.
- Paginação/ordenação em consultas (lista de usuários/assessores/referências).
- Normalização de referências (cidade/bairro/órgão RG) e cache local no cliente.


