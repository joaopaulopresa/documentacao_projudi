## Requisitos – Funcionalidades mapeadas (Juiz de 1º Grau)

- ### Analisar Autos Conclusos
  - Breve: Consulta, pré-análise, análise e assinatura de autos conclusos, incluindo descarte e consulta de concluídos. Permissões, validações e endpoints REST propostos.
  - Documento: [requisitos.md](./analisar-autos-conclusos/requisitos.md)

- ### Analisar Pendência
  - Breve: Consulta, pré-análise, análise e assinatura de pendências (inclui voto/ementa quando aplicável), descarte, impressão para autoridades e consultas auxiliares.
  - Documento: [requisitos.md](./analisar-pendencia/requisitos.md)

- ### Assessor (Gestão de Assessores/Assistentes)
  - Breve: Cadastro/edição de assessor, vínculo com serventia do chefe, ativação/desativação de vínculo e controle de permissão “guardar para assinar”, com referências (cidade, RG, bairro).
  - Documento: [requisitos.md](./assessor/requisitos.md)

- ### Audiência x Processo (Troca de Responsável)
  - Breve: Preparação e confirmação da troca de responsável de audiências/processos (individual ou em lote), com consultas auxiliares (serventia/cargo, audiência, processo, status).
  - Documento: [requisitos.md](./audienciaprocesso/requisitos.md)

- ### Movimentação de Audiência/Processo
  - Breve: Movimentar processos em audiências/sessões (pré-análise, inserção/alteração de extrato da ata, votação por maioria, sustentação oral, pendências geradas, lote e regras de 2º grau quando aplicável).
  - Documento: [requisitos.md](./audienciaprocesso-movimentacao/requisitos.md)


- ### Agenda de Audiências (Geração e Gestão de Agendas Livres)
  - Breve: Preparar, validar e gerar agendas de audiências; consultar agendas livres; reservar, liberar e excluir agendas em lote, com auditoria e regras por perfil/cargo.
  - Documento: [requisitos.md](./agenda/requisitos.md)


 - ### Audiências (Consulta, Agendamento e Público)
   - Breve: Consultas por fluxo e público; preparação/validação; agendamento manual e automático (inclui carta precatória/CEJUSC); impressão e consultas auxiliares.
   - Documento: [requisitos.md](./audiencias/requisitos.md)

- ### Busca de Processo
  - Breve: Consulta, visualização e operações auxiliares sobre processos judiciais (pública, por advogado, por código de acesso, sigilosos, situação do processo, responsáveis e download de arquivos).
  - Documento: [requisitos.md](./busca-processo/requisitos.md)

 - ### Celulares do Usuário (Liberação/Bloqueio)
   - Breve: Consulta, cadastro/edição, liberação, bloqueio e exclusão de celulares vinculados ao usuário, com validações, auditoria e permissões específicas.
   - Documento: [requisitos.md](./celular/requisitos.md)

- ### Certidão (Emissão/Consulta/Validação)
  - Breve: Emissão por guia (narrativa, prática forense, negativa/positiva e variações), execuções (CPC/circunstanciada), antecedentes, negativa/positiva (1º e 2º grau), fluxo público, validação/impressão de PDFs, consultas auxiliares e batch.
  - Documento: [requisitos.md](./certidao/requisitos.md)

 - ### Classificador
   - Breve: Cadastro/edição/exclusão de classificadores e classificação/desclassificação em lote de processos, com validações por perfil e consultas auxiliares.
   - Documento: [requisitos.md](./classificador/requisitos.md)

 - ### Conclusões (Troca de Responsável)
   - Breve: Preparação, validação e efetivação da troca de responsável por conclusões em lote e por processo específico, com consultas de responsáveis por serventia e regras de negócio/auditoria.
   - Documento: [requisitos.md](./conclusoes/requisitos.md)

- ### Configuração Pré-Análise Automática
  - Breve: Parametrização por classificador para geração automática de pré‑análises, incluindo preparação, validação, gravação, exclusão, operação especial de geração e consultas auxiliares.
  - Documento: [requisitos.md](./configuracao-pre-analise-automatica/requisitos.md)

- ### Consultar Audiência DRS (por Processo)
  - Breve: Consulta de audiências do DRS a partir do número do processo, com bloqueio de download em casos de segredo de justiça e suporte a SPG/SSG/Projudi.
  - Documento: [requisitos.md](./consultar-audiencia-drs/requisitos.md)

- ### Consultar Jurisprudência
  - Breve: Consulta pública de jurisprudência com filtros (texto, processo, instância, área, órgão/matéria, serventia, magistrado, tipo de ato, datas), paginação, destaques e download do inteiro teor (PDF), incluindo consultas auxiliares de apoio.
  - Documento: [requisitos.md](./consultar-jurisprudencia/requisitos.md)

- ### Cumprimentos (Pendências)
  - Breve: Consultas por fluxos (abertas, minhas, prazo decorrido, finalizadas/respondidas, expedidas, intimações/citações), criação e reabertura, expedição/distribuição/encaminhamento, finalizações e operações em lote, com consultas auxiliares (tipos, status, serventias/cargos, modelos) e auditoria.
  - Documento: [requisitos.md](./cumprimentos/requisitos.md)

- ### Dados do Usuário
  - Breve: Consulta de usuários com filtros (grupo, serventia, nome), exibição de detalhes, ativação/desativação, reset de senha, alteração de senha do logado (CURINGA6), consultas auxiliares (grupos/serventias) e relatório PDF.
  - Documento: [requisitos.md](./dados-usuario/requisitos.md)
 
 - ### Descartar Pendência
   - Breve: Descartar pendências de processo (inclusive em lote), operações especiais de intimação (aguardando parecer/finalização), pré‑análise de precatória, troca de classificador, consultas auxiliares e geração de código de acesso/PDF.
   - Documento: [requisitos.md](./descartar-pendencia/requisitos.md)

 - ### Distribuição por Serventia (Relatório)
   - Breve: Consulta auxiliar (área, serventia, usuário) e geração de relatório PDF sintético/analítico de processos distribuídos por serventia, com validações de escopo por perfil.
   - Documento: [requisitos.md](./distribuicao/requisitos.md)


- ### Enviar Mídias – Upload
  - Breve: Preparação, validação, upload (URL assinada ou via servidor com chunks) e efetivação de movimentação “Mídia Publicada” em processo, com limites por usuário, sanitização e auditoria.
  - Documento: [requisitos.md](./enviar-midias-upload/requisitos.md)

  - ### Itens do Relatório de Estatística de Produtividade
    - Breve: Consulta, cadastro/edição e exclusão de itens de estatística de produtividade, incluindo geração de relatório (PDF/TXT) e modos de localização leve (auto‑complete), com permissões e auditoria.
    - Documento: [requisitos.md](./estatisticaprodutividadeitem/requisitos.md)

 - ### Gráficos – Processos por Comarca/Serventia/Item de Produtividade
   - Breve: Preparação do período, consultas auxiliares (comarca/serventia/itens) e geração de PDF de gráficos por comarca, por serventia e por itens de produtividade, com restrições de escopo por grupo (Estatística vs. demais usuários).
   - Documento: [requisitos.md](./graficos-funcoes/requisitos.md)

- ### Guia Emissão
  - Breve: Consulta/listagem por fluxos, impressão de guia (PDF) com validações, desconto, parcelamento, reemissão, cancelamento/desfazer, encaminhamento/desfazer à financeira, alterações de status e intimação do responsável por guia final de débito.
  - Documento: [requisitos.md](./guia-emissao/requisitos.md)

- ### Liminar Deferida (Processos não julgados)
  - Breve: Consulta paginada de processos com liminar deferida ainda não julgados por limiar de dias, com escopo por perfil (Estatística, Desembargador, demais) e geração de relatório PDF.
  - Documento: [requisitos.md](./liminar-deferida/requisitos.md)

- ### Mandado de Prisão
  - Breve: Emissão (serventia), expedição/assinatura (juiz), impressão/visualização, consultas por fluxo e finalização (cumprir, revogar, retirar sigilo), com endpoints auxiliares de combos e validações de negócio.
  - Documento: [requisitos.md](./mandado-de-prisao/requisitos.md)

 - ### Modelo (Cadastro e Gestão de Modelos)
   - Breve: Consulta, criação/edição, exclusão, cópia e entrega do texto (HTML) de modelos, com configuração de pré‑preenchimento e consultas auxiliares (tipo de arquivo/serventia), validações por escopo/propriedade e auditoria.
   - Documento: [requisitos.md](./modelo/requisitos.md)

- ### Movimentação de Processo (Genérica)
  - Breve: Preparar, validar e efetivar movimentações processuais individuais e em lote, gerar pendências e anexos, validar/invalidar movimentações e consultar tipos/apoios.
  - Documento: [requisitos.md](./movimentacao/requisitos.md)

- ### Movimentação de Arquivo
  - Breve: Consulta e operações sobre arquivos de movimentações (listar, validar, alterar visibilidade, publicar/despublicar) e fluxos híbrido/digital (converter, adicionar peças, retornar), com auditoria e regras por perfil.
  - Documento: [requisitos.md](./movimentacao-arquivo/requisitos.md)

- ### Movimentação Tipo (Parametrização por Usuário e Grupo)
  - Breve: Consulta de opções e gestão dos vínculos usuário×tipo de movimentação (incluir/excluir individual e em lote), com permissões 684/6846, validações e auditoria.
  - Documento: [requisitos.md](./movimentacao-tipo/requisitos.md)

- ### Pendência Arquivo
  - Breve: Consulta, criação/edição e exclusão do vínculo arquivo×pendência, download protegido/público e alteração de status, com validação de negócio e auditoria.
  - Documento: [requisitos.md](./pendencia-arquivo/requisitos.md)
