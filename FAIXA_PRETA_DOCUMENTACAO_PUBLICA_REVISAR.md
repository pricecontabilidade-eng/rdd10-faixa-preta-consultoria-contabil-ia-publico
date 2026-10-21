> **Nota de escopo:** Esta página documenta a arquitetura e o procedimento de configuração. Não constitui prova de execução da cópia anonimizada, nem concede acesso ao cenário, ao banco de dados ou às credenciais. Os parâmetros e mapeamentos descritos são referências técnicas, não valores de produção.

# Consultoria Contábil com IA — Make + Supabase + Camada Ouro

**RDD10+ Faixa Preta 2026/2 | Versão pública de referência — documentação técnica**  
**Tipo:** Documentação de arquitetura Make.com | **Blueprint público V3:** disponível no repositório GitHub público, em versão anonimizada, separada do projeto original privado.  
**Estado:** cenário desenvolvido para o TCC da Faixa Marrom; cópia de referência anonimizada. **Importação visual dos 8 módulos realizada no Make; salvamento bloqueado por ausência de Data Store válido. Não executado nesta cópia.**

## 1. Contexto e problema resolvido
O projeto demonstra uma aplicação de consultoria contábil gerencial com dados estruturados de NF-e. Em vez de produzir relatórios manuais a partir de registros fiscais, o cenário lê itens, organiza-os em uma tabela analítica (Camada Ouro), consulta o conjunto e pede a um agente de IA um diagnóstico gerencial baseado somente nos dados recebidos. O resultado é criado em Google Docs. A saída é apoio à análise humana, não substitui validação contábil/fiscal nem representa parecer tributário automático.

## 2. Arquitetura e fluxo
`Make Data Store (registros NF-e) → separação de itens → variável → HTTP POST/upsert Supabase Ouro → agregação de status → HTTP GET Supabase Ouro → agente IA → Google Docs`.

**Atenção à nomenclatura:** o módulo 1 está nomeado “BUSCA DADOS PROCESSADOS DA PRATA”, mas o rótulo de Data Store visível no export original era `BRONZE_NFE`. Isso **não comprova** que os dados tenham passado pela Prata. Confirme origem, transformação e qualidade do conjunto antes de utilizá-lo em produção.

## 3. Oito módulos — função, entradas e configuração

| ID | Módulo Make | Função | Entrada / configuração indispensável | Saída |
|---|---|---|---|---|
| 1 | `datastore:SearchRecord` v1 | Busca registros fiscais no Data Store | Vincular Data Store próprio com `chave_nfe`, `data_emissao`, `nome_emitente`, `nome_destinatario`, `valor_total_nfe` e `itens[]` | Registro `1.data` |
| 2 | `builtin:BasicFeeder` v1 | Percorre o array de itens | Array mapeado `{{1.data.itens}}` | Item individual `2.value[]` |
| 6 | `util:SetVariable2` v1 | Disponibiliza item para o ciclo | Variável `item_ouro`, escopo `roundtrip`, valor mapeado do módulo 2 | Variável de contexto |
| 7 | `http:MakeRequest` v4 | POST com resolução de duplicidade no Supabase | URL `/rest/v1/ouro_nfe_consultoria`; credencial Supabase em cofre Make; `Prefer: resolution=merge-duplicates,return=representation`; `on_conflict=chave_nfe,codigo_produto` | Resposta HTTP e status |
| 13 | `builtin:BasicAggregator` v1 | Agrega resultados de gravação | Fonte/feeder módulo 1; campo `statusCode` do módulo 7 | Agregado de status |
| 9 | `http:MakeRequest` v4 | Consulta a tabela Ouro | GET da mesma URL; parâmetro `select=*`; credencial de leitura autorizada | `9.data` |
| 8 | `ai-local-agent:RunLocalAIAgent` v0 | Produz diagnóstico com base nos dados consultados | Conectar provedor IA; entrada contém `{{9.data}}`; manter prompts originais | `8.response` |
| 14 | `google-docs:createADocument` v1 | Cria documento com análise | Conectar Google; título do diagnóstico; conteúdo `{{8.response}}`; selecionar Drive/pasta autorizados | Documento Google Docs |

### 3.1. Dados esperados no Data Store
A estrutura declarada no export inclui: `cnpj_emitente`, `xml_raw`, `origem`, `received_at`, `processed`, `processed_at`, `chave_nfe`, `numero_nfe`, `serie_nfe`, `data_emissao`, `cnpj_destinatario`, `nome_emitente`, `nome_destinatario`, `uf_emitente`, `uf_destinatario`, `valor_total_nfe`, `valor_produtos`, `valor_icms`, `valor_pis`, `valor_cofins`, `valor_ipi`, `valor_cbs`, `valor_ibs`, `itens[]`. Cada item contém `numero_item`, `codigo_produto`, `descricao`, `ncm`, `cfop`, `quantidade`, `valor_unitario` e `valor_total_item`. A presença de campos no esquema não significa que todos os XMLs contenham os respectivos tributos.

### 3.2. Mapeamento principal da camada Ouro
O POST grava `chave_nfe`, `data_emissao`, `ano`, `mes`, `periodo`, `codigo_produto`, `descricao_produto`, `ncm`, `cfop`, `quantidade_vendida`, `valor_unitario`, `faturamento_produto`, `faturamento_nfe`, `nome_emitente`, `nome_destinatario`. `ano`, `mes` e `periodo` são derivados de `data_emissao`. O upsert pressupõe restrição UNIQUE compatível com `(chave_nfe,codigo_produto)`; se uma mesma NF-e repetir o código em itens diferentes, esta chave pode sobrescrever registros — avalie antes de produção.

### 3.3. Agente IA
O **System Prompt** limita o agente aos dados da Camada Ouro e proíbe inventar fatos. O **Input** solicita faturamento total, quantidade, faturamento por produto, média por unidade, destaques por produto e destinatário, inconsistências e limitações. O arquivo JSON preserva integralmente os prompts. O export registra configuração `defaultModel=medium` e metadado de modelo `gpt-5-nano`; confirme disponibilidade e preço na conta de destino. Não use resultados sem conferência: valores da NF-e podem se repetir por item, de modo que somar `faturamento_nfe` em nível de item pode duplicar totais.

## 4. Pré-requisitos e permissões
1. **Make.com:** conta com acesso a Data Stores, HTTP, agente IA e Google Docs; permissão para importar e editar cenários; Data Store e estrutura de dados criados previamente.
2. **Supabase:** projeto ativo; tabela `public.ouro_nfe_consultoria` com colunas e tipos compatíveis; índice/constraint para upsert; acesso REST API autorizado. Configure permissões mínimas para leitura e escrita conforme RLS/políticas de sua instalação; nunca desabilite RLS apenas para facilitar testes.
3. **Credencial Supabase:** criar no gerenciador de credenciais do Make (não no JSON público). Preferir credencial de escopo mínimo; **não publicar** `service_role`, `secret`, `anon` ou tokens em README, prints ou URL.
4. **Agente de IA:** conexão válida no Make, limites de uso e política de tratamento de dados compatíveis com LGPD. Use dados sintéticos ou anonimizados em demonstrações públicas.
5. **Google Docs/Drive:** autenticação OAuth autorizada, permissão de criar documento na pasta escolhida e política de compartilhamento restrita.

## 5. Como importar e configurar — passo a passo
1. Faça cópia do JSON e abra Make → Scenarios → criar cenário → importar Blueprint. **A importação pode exigir ajustes porque o export foi anonimizado.**
2. No módulo **1**, selecione um Data Store **da sua conta**, com campos compatíveis; não utilize o identificador de origem.
3. Confira no módulo **2** o array `{{1.data.itens}}`; confira a variável no módulo **6**.
4. No módulo **7**, troque `https://SEU-PROJETO.supabase.co` pela URL de **seu** projeto e vincule credencial de API **no cofre do Make**; mantenha método POST, headers, parâmetro `on_conflict` e JSON do body. Não insira chave em texto puro no arquivo.
5. No módulo **9**, aplique a mesma URL e credencial apropriada para GET; revise o risco de `select=*` retornar mais dados do que o necessário.
6. No módulo **8**, selecione sua conexão de IA; confira os prompts sem alterá-los e valide o volume de dados de entrada antes de executar.
7. No módulo **14**, conecte sua conta Google, escolha a pasta permitida e confira o mapeamento `{{8.response}}`.
8. Execute com **uma NF-e sintética**, contendo 1–2 itens e sem CPF/CNPJ reais. Confira status do POST, quantidade de linhas no Ouro, conteúdo do GET, consistência dos totais no diagnóstico e criação do documento.
9. Teste novamente o mesmo documento para validar idempotência, verificando a chave de conflito. Ative agendamento somente após revisão de segurança, custos, duplicidades e tratamento de erros.

## 6. Segurança e anonimização — medidas documentadas para a cópia de referência
- URL real do projeto Supabase foi substituída por `https://SEU-PROJETO.supabase.co` nos módulos HTTP 7 e 9.
- Identificadores de credenciais nos parâmetros `apiKeyKeychain`, `makeConnectionId` e `__IMTCONN__` foram substituídos por `0` (marcador técnico de conexão ausente, **não é uma credencial funcional**).
- Identificador do Data Store de origem foi substituído; é obrigatório selecionar o Data Store correto na conta de destino.
- Rótulos pessoais de conexões foram substituídos; não há chave de API em texto claro nos campos de autenticação examinados.
- **Limite da verificação:** inspeção estática do arquivo não substitui teste de importação, inspeção manual de todos os campos nem auditoria da plataforma de destino. Não publicar XML real, dados de clientes, capturas de tela com segredos ou links com tokens.

## 7. Versões dos conectores identificadas no export
Data Store SearchRecord **v1**; BasicFeeder **v1**; SetVariable2 **v1**; HTTP MakeRequest **v4** (módulos 7 e 9); BasicAggregator **v1**; Make AI Agent `RunLocalAIAgent` **v0**; Google Docs createADocument **v1**. São versões **observadas no arquivo**, não garantia de disponibilidade futura.

## 8. Custos e desempenho — como estimar
**Não há medição de custo financeiro ou tempo médio de execução anexada a este artefato.** Para estimar, registre em um teste: quantidade de registros NF-e, itens por documento, operações/créditos Make consumidos, tokens de entrada e saída do modelo, custo do plano Supabase e armazenamento Google. Calcule `custo por execução = operações Make + custo IA + parcela de infraestrutura`; compare cenários de 10, 100 e 1.000 itens. A consulta `select=*` e a inclusão integral de `9.data` no prompt podem elevar custos e exceder contexto; imponha limites/paginação conforme o caso.

## 9. Demonstração e rastreabilidade
**Documentação pública:** esta página descreve a arquitetura e os pré-requisitos da cópia anonimizada. O Blueprint V3 está disponível no repositório público [RDD10 — Faixa Preta — Consultoria Contábil com IA](https://github.com/pricecontabilidade-eng/rdd10-faixa-preta-consultoria-contabil-ia-publico), junto do README e da documentação técnica V3. O projeto original permanece separado em repositório privado. O link do GitHub não substitui a seleção do artefato na Biblioteca RDD10+. **Link público de demonstração de execução do cenário Make:** não disponibilizado.  
**Referência de origem:** TCC Faixa Marrom RDD10+ — Camada Ouro + Agente IA.  
**Prova de execução:** o JSON demonstra configuração, **não comprova sozinho** que o cenário executou corretamente ou que os resultados são reais. Caso disponível, publique demonstração com dados sintéticos e sem credenciais.

## 10. Checklist de publicação
- [x] JSON do Make presente e sintaticamente válido; 8 módulos preservados.
- [x] A documentação registra a substituição de endereço de projeto e IDs de conexões na cópia de referência; o JSON requer revisão independente antes de qualquer divulgação.
- [x] README com módulos, pré-requisitos, versões, permissões, custos e passos de importação.
- [x] Repositório GitHub público com Blueprint V3, README e documentação técnica V3 disponibilizado; acesso público confirmado em janela anônima.
- [ ] Atualizar esta documentação legada no GitHub e confirmar o conteúdo atualizado em janela anônima.
- [ ] Confirmar tags **contabilidade, make, nfe, supabase** na Biblioteca.
- [ ] Preencher **contexto_projeto** se a plataforma disponibilizar esse campo.
- [x] Conferir importação visual dos 8 módulos no Make (realizada).
- [ ] Resolver Data Store válido e salvar cenário de demonstração (não realizado; conta sem espaço disponível para criar outro Data Store).
- [ ] Testar execução em ambiente isolado com dados sintéticos (não realizado; não executar no projeto original).
- [ ] Revisar manualmente o conteúdo público e eventuais anexos para ausência de dados reais e segredos.
