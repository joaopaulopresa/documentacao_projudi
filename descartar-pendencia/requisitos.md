## Requisitos – Descartar Pendência

Escopo: migrar a funcionalidade do legado (controlador `DescartarPendenciaProcessoCt`) para endpoints REST. O foco é o descarte de pendências (inclusive em lote) no contexto do processo, operações especiais associadas (intimação aguardando parecer, pré‑análise precatória), alteração de classificador e geração de código de acesso. Não descreve telas.

### 1) Permissões
- Código base: `482` – “Descartar Pendência” (Classe: `DescartarPendenciaProcessoCt.CodigoPermissao`).
- Subpermissões (conforme `descartar-pendencia.json`):
  - `4825` – SALVAR: Descartar pendências.
  - `4827` – CURINGA7: Marcar pendência de intimação como “Aguardando Parecer” (e possível finalização).
  - `4829` – CURINGA9: Geração de pré‑análise de precatória.
  - `4823` – Localizar Ajax: Consultas assíncronas auxiliares.
- Permissões auxiliares utilizadas pelo fluxo (de outros módulos):
  - Classificador – Localizar (para consulta de classificadores): `ClassificadorDt.CodigoPermissao * QtdPermissao + Localizar`.
  - Modelo/Impressão de “Código de Acesso” (consulta de modelos por tipo de arquivo e impressão PDF).

Observação: o controlador também manipula operações de liberação de acesso/peticionamento e elaboração de voto, que podem depender de permissões específicas desses módulos.

### 2) Regras e validações
- Descarta pendências em lote por processo: é necessário identificar o processo corrente (`processoDt` na sessão do legado) e a lista de pendências selecionadas.
- Hash de segurança para operações sensíveis em pendências individuais (ex.: intimação aguardando parecer): validar `UsuarioSessao.VerificarCodigoHash(idPendencia, hash)`.
- Pré‑condições por tipo de processo:
  - Bloqueio de impressão/código de acesso e algumas ações quando `processoDt.getProcessoTipoCodigo() == CALCULO_DE_LIQUIDACAO_DE_PENA` (processo físico).
  - Fluxos de precatória/precatorio apenas quando o processo é compatível (precatória/precatorio expedida online), senão erro.
  - Em processos apensos, peticionamento sujeito a `podePeticionarProcessoApenso`.
  - Em cartas precatórias arquivadas: peticionamento bloqueado.
- Anti‑abuso: “Solicitar Acesso” respeita janela de tempo mínima entre solicitações (valor parametrizado), com auditoria de IP e usuário.
- Alteração de classificador exige validação de hash da pendência e envio de `idNovoClassificador`; sem troca, retorna erro.
- Perfis/grupos com regras específicas (ex.: grupos de 2º grau habilitam visualizações/opções como voto/ementa; redirecionamentos por perfil). Para o REST, validar autorização e retornar 403 quando aplicável.
- Auditoria: enviar/registrar sempre `idUsuarioLog` e `ipComputadorLog` (ex.: `new LogDt(idUsuario, ip)` em alterações).

### 3) Modelos de dados (DTO)
- Pendência (`Pendencia`): `id`, `hash`, `id_PendenciaTipo`, `classificador`, `id_Classificador`, `codigoTemp` (flags de fluxo, ex.: pré‑análise), `minutaRelatorioVoto` (boolean), demais metadados.
- Processo (`Processo`): `id`, `id_Serventia`, `processoTipoCodigo`, `processoNumeroDigito`, `id_ProcessoPrincipal`, `dataArquivamento`, `segredoJustica`, flags de “precatoria/precatorio expedida online”.
- Usuário (`Usuario`): `idUsuario`, `usuario`, `grupoCodigo`, `grupoTipoCodigo`, perfil (ex.: assessor, magistrado, MP, advogado), `id_Serventia`, `ipComputadorLog`.
- Arquivo (`Arquivo`): `id`, `nome`, `tipo`, metadados e vínculo a pendência/processo.
- Modelo (`Modelo` para “Código de Acesso”): `id`, `modelo`, `texto` (HTML), `tipoArquivo`.

### 4) Endpoints REST propostos

1) Descartar pendências (lote por processo)
- POST `/descartar-pendencias/preparar`
  - Permissão: 4825 (SALVAR).
  - Body: `{ idProcesso: string, idsPendencias: string[] }`
  - Ação: valida existência/posse e prepara confirmação; retorna mensagem e ecoa itens.
  - Retorno: `200 OK` com `{ mensagem, idsPendencias }`.
- POST `/descartar-pendencias/confirmar`
  - Permissão: 4825 (SALVAR).
  - Body: `{ idProcesso: string, idsPendencias: string[] }`
  - Ação: efetiva `descartarPendencias` no processo.
  - Retorno: `200 OK` com `{ quantidadeDescartada }`.

2) Operações especiais – Intimação aguardando parecer
- POST `/pendencias/{id}/intimacao/finalizar`
  - Permissão: 482 (base) ou subpermissão específica conforme mapeamento.
  - Body: `{ hash: string }`
  - Ação: finaliza pendência de intimação aguardando parecer.
  - Retorno: `200 OK`.
- POST `/pendencias/{id}/intimacao/aguardando-parecer`
  - Permissão: 4827 (CURINGA7).
  - Body: `{ hash: string, finalizada: boolean }`
  - Ação: marca “aguardando parecer”; se `!finalizada`, também finaliza.
  - Retorno: `200 OK`.

3) Operação especial – Pré‑análise precatória
- POST `/pendencias/{id}/pre-analise-precatoria`
  - Permissão: 4829 (CURINGA9).
  - Body: `{ hash: string }`
  - Ação: gera entidade de pré‑análise para a pendência ligada à precatória.
  - Retorno: `201 Created` com `{ idPreAnalise }`.

4) Classificador da pendência
- GET `/pendencias/classificadores`
  - Permissão: do módulo Classificador (Localizar).
  - Query: `q` (texto), `idServentia`, `fluxoUPJ` (quando aplicável), `paginacao`.
  - Ação: consulta classificadores para seleção.
  - Retorno: `200 OK` com lista paginada.
- PUT `/pendencias/{id}/classificador`
  - Permissão: 482 (base) e hash válido.
  - Body: `{ hash: string, idNovoClassificador: string }`
  - Ação: troca o classificador; registra auditoria (`LogDt`).
  - Retorno: `200 OK`.

5) Arquivos – Precatória/Precatório (quando o processo for compatível)
- GET `/processos/{id}/precatorias/arquivos`
  - Permissão: 482 (base).
  - Query: `lista=expedidas|juntadas`.
  - Ação: lista arquivos vinculados ao fluxo de precatória/precatorio.
  - Retorno: `200 OK` com `{ arquivos: Arquivo[] }`.
- POST `/processos/{id}/precatorias/devolver`
  - Permissão: 482 (base).
  - Body: `{ arquivoIds: string[], statusDevolucao: string }`
  - Ação: prepara devolução (monta lista e contexto de movimentação); integração com módulo de movimentação.
  - Retorno: `200 OK`.
- POST `/processos/{id}/precatorias/encaminhar-oficio`
  - Permissão: 482 (base).
  - Body: `{ arquivoIds: string[] }`
  - Ação: prepara encaminhamento de ofício; integração com módulo de movimentação.
  - Retorno: `200 OK`.

6) Processo dependente – Ofício comunicatório
- GET `/processos/{id}/dependente/arquivos`
  - Permissão: 482 (base).
  - Query: `lista=juntadas`.
  - Ação: lista arquivos no dependente (requer `id_ProcessoPrincipal`).
  - Retorno: `200 OK`.
- POST `/processos/{id}/dependente/oficio/encaminhar`
  - Permissão: 482 (base).
  - Body: `{ arquivoIds: string[] }`
  - Ação: prepara encaminhamento de ofício comunicatório; integração com movimentação.
  - Retorno: `200 OK`.

7) Código de Acesso (modelos e PDF)
- GET `/processos/{id}/codigo-acesso/modelos`
  - Permissão: modelo/arquivo do tipo “Código de Acesso”.
  - Query: `q`, `paginacao`.
  - Ação: consultar modelos disponíveis.
  - Retorno: `200 OK` com lista paginada.
- GET `/processos/{id}/codigo-acesso/preview`
  - Permissão: idem acima.
  - Query: `modeloId`, `idParte` (opcional).
  - Ação: monta e retorna HTML do modelo preenchido.
  - Retorno: `200 OK` com `{ html }`.
- POST `/processos/{id}/codigo-acesso/pdf`
  - Permissão: idem acima.
  - Body: `{ modeloId: string, idParte?: string }`
  - Ação: converte HTML em PDF e retorna binário.
  - Retorno: `200 OK` (content-type `application/pdf`).

8) Situação do Processo (auxiliar)
- GET `/processos/{id}/situacao`
  - Permissão: 482 (base).
  - Ação: consultar pendências, conclusões e audiências em aberto para o processo (equivalente a `consultarSituacaoProcesso`).
  - Retorno: `200 OK` com `{ pendencias, conclusoes, audiencias }`.

### 5) Respostas e códigos de status
- 200 OK: operações de consulta e ações sem criação de novo recurso.
- 201 Created: geração de entidades como pré‑análise (quando aplicável).
- 400 Bad Request: parâmetros ausentes/invalidos (ex.: sem seleção de pendências; sem `idNovoClassificador`; processo incompatível com fluxo selecionado).
- 403 Forbidden: sem permissão ou hash inválido para a pendência.
- 404 Not Found: processo/pendência/arquivo inexistente ou não pertencente ao contexto do usuário.
- 409 Conflict: violações de negócio (ex.: peticionamento bloqueado por arquivamento; janela anti‑abuso ainda vigente).

### 6) Observações de segurança
- Autenticação obrigatória e autorização por permissão/subpermissão.
- Validação de hash para operações sensíveis em pendência.
- Auditoria: registrar `idUsuarioLog`, `ipComputadorLog`, timestamp e parâmetros essenciais em todas as mutações.
- Dados sensíveis: PDFs/HTMLs gerados podem conter dados pessoais; aplicar LGPD (controle de acesso, logs, retenção e mascaramento quando aplicável).

### 7) Itens a decidir na migração
- De/para dos fluxos que hoje “preparam contexto” para o módulo de movimentação: definir contratos claros entre serviços (ex.: payload com `arquivoIds`, `status`, `complemento`).
- Escopo de permissões auxiliares (Classificador, Modelos/Impressão) dentro desta API ou via serviços dedicados.
- Idempotência para descarte em lote e para marcações de intimação (repetições com o mesmo hash/ID).
- Paginação/ordenacão nas listagens de classificadores, arquivos e situação do processo.
- Janelas de anti‑abuso: parametrização e política de retorno (código/erro padronizado).


