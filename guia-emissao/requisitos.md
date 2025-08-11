## Requisitos – Guia Emissão

### Escopo
Migração da funcionalidade legada de emissão e gestão de guias (processuais e iniciais) para endpoints REST. Inclui: consulta/listagem por fluxos, impressão (PDF), desconto, parcelamento, reemissão de vencimento, cancelamento/desfazer, encaminhamento/desfazer à financeira, alteração de status (aguardando pagamento, pagamento deferido para final do processo, baixada com gratuidade), e intimação da parte responsável por guia final de débito. Ignorar aspectos de UI/JSP.

### Permissões
- Base: `CodigoPermissao = 646` (Guia Emissão)
- Subpermissões (do `guia-emissao.json`):
  - `IMPRIMIR (6461)`
  - `LOCALIZAR (6462)`
  - `ExcluirResultado (6460)`
  - `Novo (6464)`
  - `SalvarResultado (6465)`
  - `CURINGA6 (6466)`
  - `CURINGA7 (6467)`
  - `CURINGA9 (6469)`
- Auxiliares (buscas acopladas no módulo):
  - `ProcessoDt.CODIGO_PERMISSAO_CONSULTA_PROCESSO * QtdPermissao + Localizar` (consulta de processos)
  - `ServentiaDt.CodigoPermissao * QtdPermissao + Localizar` (consulta de serventias)
  - `ProcessoTipoDt.CodigoPermissao * QtdPermissao + Localizar` (consulta de tipos de processo)

### Regras e validações
- Impressão (PDF):
  - Requer captcha válido; caso inválido: 400 com "Código do Captcha não confere!".
  - Bloquear se `guiaPossuiRestricaoEmissaoComJuroMultaCorrecao = true` (mensagem `MENSAGEM_GUIA_ENVIADA_PROTESTO_FINANCEIRA`).
  - Para guias iniciais de 1º/2º grau, preencher apelante/apelado a partir de requerente/requerido; completar comarca/código; definir número do processo.
- Listagens/consultas (LOCALIZAR):
  - CURINGA6: listar guias iniciais emitidas pelo usuário logado.
  - CURINGA7: listar para Contadoria (paginado por serventia).
  - CURINGA8: listar para Financeira (paginado).
  - 10: listar com trânsito em julgado há 5 anos (Financeira).
  - CURINGA9: listar vencidas (Contadoria).
- Operações "Novo/Salvar/SalvarResultado" (preparar/confirmar):
  - Desconto: guia deve estar "Aguardando Pagamento" e não vencida (`MENSAGEM_GUIA_DEVE_ESTAR_AGUARDANDO_PAGAMENTO_E_NAO_VENCIDA`).
  - Parcelamento: validação via negócio; se `quantidadeParcelas > QUANTIDADE_MAXIMA_PARCELAS_SEM_DECISAO (5)`, exigir `motivoParcelamento` e registrar.
  - Reemissão: adicionar novo vencimento (+15 dias, conforme flag do negócio).
  - Alterar quantidade máxima de parcelas: registrar valor e motivo.
  - Acordo: percentual aplicado por faixas de parcelas (2–6: 80%; 7–12: 60%; 13–24: 40%; padrão 100%).
- Cancelamento e correlatas (Excluir/ExcluirResultado):
  - Cancelar guia emitida; se boleto já emitido, sinalizar conflito e exigir confirmação explícita.
  - Desfazer cancelamento.
  - Encaminhar para Financeira: permitido apenas para "Guia Final de Débito".
  - Desfazer encaminhamento: permitido apenas para guias com status "Enviada à Financeira".
  - Intimar parte responsável pela Guia Final de Débito (gera intimação e mensagem de sucesso com identificação da parte/guia).
- Alterações de status:
  - Aguardando Pagamento (a partir de "Aguardando deferimento").
  - Pagamento deferido para o final do processo.
  - Baixada com gratuidade.
- Auditoria obrigatória em todas as operações mutáveis: `idUsuarioLog` e `ipComputadorLog`.

### Modelos de dados (DTO)
- `GuiaEmissao` (principal – base para payloads):
  - Identificação e vínculo: `id`, `idProcesso`, `numeroProcesso`, `idGuiaTipo`, `idGuiaStatus`, `numeroGuiaCompleto`, `idServentia`, `serventia`, `idComarca`, `comarca`, `comarcaCodigo`.
  - Partes/representantes: `requerente`, `requerido`, `listaRequerentes`, `listaRequeridos`, `listaOutrasPartes`, `listaAdvogados`, `listaAssuntos`, `idProcessoParteResponsavelGuia`, `nomeProcessoParteResponsavelGuia`.
  - Itens/valores: `listaGuiaItemDt`, campos de custas/atos/honorários e derivados (conforme legado).
  - Parcelamento/desconto: `quantidadeParcelas`, `porcentagemDesconto`, `motivoParcelamento`, `quantidadeMaximaParcelas` (alteração), constantes: `QUANTIDADE_MAXIMA_PARCELAS_SEM_DECISAO = 5`.
  - Flags de fluxo/integrações: `guiaFinalDebito` (derivado), `guiaEnviadaFinanceira`, `guiaPossuiRestricaoEmissaoComJuroMultaCorrecao`, `guiaEmitidaSPG/SSG`.
  - Auditoria: `idUsuario`, `idUsuarioLog`, `ipComputadorLog`, datas relevantes (emissão, vencimento, recebimento, cancelamento, base de atualização).

### Endpoints REST propostos

1) Consultas/listagens
- GET `/guias-emissao`
  - Permissão: `LOCALIZAR (6462)`
  - Query: `modo` ∈ {`usuario-iniciais`, `contadoria`, `contadoria-vencidas`, `financeira`, `financeira-5-anos`}, `serventiaId?`, `pagina?`
  - Ação: retorna lista por fluxo (equivalente a CURINGA6/7/8/9/10)
  - Retorno: 200 + lista paginada

2) Impressão (PDF)
- GET `/guias-emissao/{id}/pdf?captcha=...`
  - Permissão: `IMPRIMIR (6461)`
  - Ação: valida captcha e restrição de emissão; gera PDF
  - Retorno: 200 `application/pdf`; 400 captcha; 409 restrição de emissão

3) Desconto/Parcelamento/Reemissão/Acordo/Alterar máx. parcelas – preparar e confirmar
- POST `/guias-emissao/{id}/desconto/preparar`
  - Permissão: `Novo (6464)`
  - Body: `{ porcentagemDesconto }`
  - Regras: guia aguardando pagamento e não vencida
  - Retorno: 200 com dados de confirmação
- POST `/guias-emissao/{id}/desconto/confirmar`
  - Permissão: `SalvarResultado (6465)`
  - Body: `{ porcentagemDesconto }`
  - Ação: aplicar desconto; auditoria
  - Retorno: 200 com mensagem de sucesso; 409 conflito

- POST `/guias-emissao/{id}/parcelamento/preparar`
  - Permissão: `Novo (6464)`
  - Body: `{ quantidadeParcelas, motivoParcelamento? }`
  - Regras: validação negócio; motivo obrigatório se `quantidadeParcelas > 5`
  - Retorno: 200 confirmação
- POST `/guias-emissao/{id}/parcelamento/confirmar`
  - Permissão: `SalvarResultado (6465)`
  - Body: `{ quantidadeParcelas, motivoParcelamento }`
  - Ação: gerar guia parcelada; auditoria
  - Retorno: 200/409

- POST `/guias-emissao/{id}/reemitir/preparar`
  - Permissão: `Novo (6464)`
  - Body: `{ adicionar15Dias: true }`
  - Retorno: 200 confirmação
- POST `/guias-emissao/{id}/reemitir/confirmar`
  - Permissão: `SalvarResultado (6465)`
  - Body: `{ adicionar15Dias: true }`
  - Ação: atualizar data de vencimento (+15 dias)
  - Retorno: 200/409

- POST `/guias-emissao/{id}/alterar-quantidade-maxima-parcelas/preparar`
  - Permissão: `Novo (6464)`
  - Body: `{ quantidadeMaximaParcelas }`
  - Retorno: 200 confirmação
- POST `/guias-emissao/{id}/alterar-quantidade-maxima-parcelas/confirmar`
  - Permissão: `SalvarResultado (6465)`
  - Body: `{ quantidadeMaximaParcelas, motivoParcelamento }`
  - Ação: atualizar configuração
  - Retorno: 200/409

- POST `/guias-emissao/{id}/acordo/preparar`
  - Permissão: `Novo (6464)`
  - Body: `{ quantidadeParcelas, idAcordo }`
  - Retorno: 200 confirmação (percentual calculado por faixas)
- POST `/guias-emissao/{id}/acordo/confirmar`
  - Permissão: `SalvarResultado (6465)`
  - Body: `{ quantidadeParcelas, idAcordo, motivoParcelamento? }`
  - Ação: gerar guia de acordo (percentuais: 2–6:80%, 7–12:60%, 13–24:40%, padrão 100%)
  - Retorno: 200/409

4) Cancelamento e correlatas
- POST `/guias-emissao/{id}/cancelar`
  - Permissão: `CURINGA9 (6469)` ou `ExcluirResultado (6460)`
  - Regras: se boleto emitido, retornar 409 a menos que `forcar=true`
  - Retorno: 200/409
- POST `/guias-emissao/{id}/desfazer-cancelamento`
  - Permissão: `CURINGA8 (6468 – não mapeado no JSON, usar fluxo ExcluirResultado)`
  - Retorno: 200/409
- POST `/guias-emissao/{id}/encaminhar-financeiro`
  - Permissão: `CURINGA7 (6467)`
  - Regra: apenas Guia Final de Débito
  - Retorno: 200/409
- POST `/guias-emissao/{id}/desfazer-encaminhar-financeiro`
  - Permissão: `Editar/ExcluirResultado` (fluxo legado)
  - Regra: apenas se status "Enviada à Financeira"
  - Retorno: 200/409
- POST `/guias-emissao/{id}/intimar-responsavel-final-debito`
  - Permissão: `CURINGA6 (6466)`
  - Ação: gerar intimação da parte responsável
  - Retorno: 201/409

5) Alterações de status
- POST `/guias-emissao/{id}/status/aguardando-pagamento`
  - Permissão: `SalvarResultado (6465)`
  - Origem: alteração a partir de "Aguardando deferimento"
  - Retorno: 200/404
- POST `/guias-emissao/{id}/status/pagamento-deferido-final-processo`
  - Permissão: `SalvarResultado (6465)`
  - Retorno: 200/404
- POST `/guias-emissao/{id}/status/baixada-com-gratuidade`
  - Permissão: `SalvarResultado (6465)`
  - Retorno: 200/404

6) Endpoints auxiliares (combos/listagens)
- GET `/processos` (consulta leve por descrição/numero)
- GET `/serventias`
- GET `/processo-tipos`

### Respostas e códigos de status
- 200 sucesso; 201 criação (intimação); 400 validação (captcha, parâmetros ausentes); 403 permissão; 404 não encontrado; 409 conflito de negócio (restrição de impressão, boleto já emitido, estado incompatível).

### Observações de segurança
- Autenticação/autorização obrigatória; validar perfil/grupo e escopo de serventia.
- Registrar `idUsuarioLog` e `ipComputadorLog` em todas as operações mutáveis.
- Dados sensíveis: PDFs e metadados de partes/advogados – aplicar regras de sigilo/LGPD.

### Itens a decidir na migração
- Captcha na rota de impressão (mecanismo REST adequado: token de desafio vs. captcha visual).
- Estratégia de streaming/armazenamento de PDF.
- Padrão para confirmação de operações potencialmente destrutivas (ex.: cancelar guia com boleto emitido – flag `forcar`).
- Padronização de mensagens para cliente vs. logs internos.


