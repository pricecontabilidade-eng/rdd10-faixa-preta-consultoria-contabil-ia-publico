# Documentação técnica — Consultoria Contábil com IA

**Faixa Preta RDD10+ 2026/2 | Versão pública V3 | Price Contabilidade**

> **Nota de escopo:** documentação de arquitetura e configuração, não comprovação de execução. Blueprint de referência com oito módulos. Importação visual relatada no Make, porém salvamento bloqueado pela falta de Data Store válido. Não foram realizados testes de execução desta cópia. O TCC original da Faixa Marrom não foi modificado.

## 1. Contexto e problema resolvido

A proposta integra dados fiscais de NF-e já processados na camada Prata com uma camada Ouro para apoiar diagnósticos gerenciais. Evita interpretar XML bruto diretamente no agente e centraliza dados estruturados antes da análise. A saída pretendida é um relatório no Google Docs.

## 2. Arquitetura e fluxo de dados

`Data Store Prata → Iterator → Set Variable → POST Supabase/Ouro → Array Aggregator → GET Supabase/Ouro → Agente de IA → Google Docs`

Os IDs e a ordem de exportação são: **1, 2, 6, 7, 13, 9, 8, 14**. Não há alteração dos mapeamentos, prompts, configurações ou ordem em relação ao arquivo V2_SEGURO recebido.

## 3. Módulos, entradas, saídas e configurações

| ID | Módulo Make (versão do export) | Entrada / função | Saída / cuidado |
|---|---|---|---|
| 1 | Data Store — Search records (`datastore:SearchRecord` v1) | Consulta registros da Prata; Data Store não vinculado na cópia | Dados da NF-e, inclusive `itens`; associar Data Store compatível |
| 2 | Iterator (`builtin:BasicFeeder` v1) | Array `{{1.data.itens}}` | Um bundle por item |
| 6 | Set variable (`util:SetVariable2` v1) | Variável `item_ouro`, valor `{{`2`}}` | Item preparado no ciclo de execução |
| 7 | HTTP Make a request (`http:MakeRequest` v4) | POST JSON para `/rest/v1/ouro_nfe_consultoria`; `on_conflict=chave_nfe,codigo_produto` | Gravação/upsert pretendida; requer URL, credencial, tabela, índice único e validação do corpo |
| 13 | Array aggregator (`builtin:BasicAggregator` v1) | Agrega `{{7.statusCode}}`, origem módulo 1 | Reúne respostas antes da consulta |
| 9 | HTTP Make a request (`http:MakeRequest` v4) | GET `/rest/v1/ouro_nfe_consultoria` com `select=*` | Dados Ouro; sem paginação explícita, risco de volume/contexto |
| 8 | Make AI Agent (`ai-local-agent:RunLocalAIAgent` v0) | Instruções de análise gerencial + `{{9.data}}` | Resposta textual, sem invenção de dados |
| 14 | Google Docs — Create a document (`google-docs:createADocument` v1) | Conteúdo `{{8.response}}` | Documento `DIAGNÓSTICO GERENCIAL - CAMADA OURO - AGENTE IA` no My Drive |

As versões indicam o **export analisado**, não garantia de compatibilidade futura.

## 4. Dados e regras de negócio

A camada Prata expõe identificadores de NF-e, datas, emitente/destinatário, UF, valores tributários e array de itens. Cada item inclui código, descrição, NCM, CFOP, quantidade, valor unitário e total. O POST mapeia chave, data, ano, mês, período, produto, quantidade, valores e participantes. A chave de conflito declarada é `chave_nfe,codigo_produto`: confirmar unicidade real dos itens; produtos repetidos na mesma NF-e podem exigir identificador de linha.

O agente foi instruído a consolidar faturamento, quantidade, faturamento por produto, média por unidade, produtos em destaque, distribuição por destinatário e limitações. **Cuidado:** `faturamento_nfe` é repetido por item no POST; somá-lo sem deduplicar por chave de NF-e pode superestimar receita. A documentação não demonstra que a análise evite essa duplicação.

## 5. Pré-requisitos, permissões e conexões

- **Make:** acesso a cenários, Data Stores e conexões; importar em cenário isolado.
- **Data Store Prata:** selecionar estrutura compatível com os campos exportados; a cópia não contém Data Store funcional.
- **Supabase:** tabela `ouro_nfe_consultoria` com colunas e tipos compatíveis, restrição única adequada ao `on_conflict`, permissões de INSERT/SELECT e políticas RLS revisadas. Configurar credenciais no cofre do Make, nunca no JSON.
- **Agente de IA:** conexão de provedor válida e limites de custo/contexto.
- **Google Docs:** conexão com permissão de criar documentos na pasta pretendida.

## 6. Importação, configuração e teste

1. Preserve o original e importe o Blueprint V3 em **novo cenário isolado** no Make.
2. Reassocie o Data Store e confira estrutura, disponibilidade de `itens` e permissões.
3. No HTTP POST (7), substitua somente o placeholder da URL pelo endpoint do seu projeto; configure autenticação no Make, sem colar segredo em documentação; confira colunas e `on_conflict`.
4. No HTTP GET (9), configure o mesmo projeto/tabela e limite/pagine a consulta conforme volume esperado. Qualquer alteração funcional deve ser documentada em uma nova revisão do Blueprint.
5. Reassocie agente e Google Docs; confirme os tokens de mapeamento no editor visual.
6. Teste com 1 NF-e sintética de 1 item; confira retorno HTTP, registro Ouro, saída do agente e documento.
7. Teste duplicidade, NF-e com itens repetidos, caracteres especiais em descrições, ausência de dados, falha de autenticação e volume maior. Registre evidências e custos.

**Estado dos testes nesta entrega:** inspeção e validação estrutural do JSON concluídas; importação/salvamento e execução em ambiente configurado **não comprovados**.

## 7. Versões dos conectores identificadas no export

Data Store SearchRecord v1; BasicFeeder v1; SetVariable2 v1; HTTP MakeRequest v4 (módulos 7 e 9); BasicAggregator v1; Make AI Agent RunLocalAIAgent v0; Google Docs createADocument v1. Estas são versões observadas, não garantia de disponibilidade futura.

## 8. Segurança, custos e desempenho

**Segurança:** placeholders `SEU-PROJETO` e `CONFIGURAR_*`; conexões com IDs nulos (`0`) exigem configuração. Nenhuma chave de acesso funcional conhecida foi identificada por inspeção automatizada, mas a revisão humana do conteúdo completo é obrigatória antes da publicação. Não exponha NF-e reais, CNPJ de clientes, XML bruto, chaves, tokens, IDs privados ou links com acesso privilegiado. O Data Store de origem inclui campo `xml_raw` no esquema exportado, mesmo sem registros reais.

**Custos:** não há medição financeira ou de duração comprovada. Estime por execução: operações Make + chamadas/consumo de IA + infraestrutura Supabase + armazenamento Google. Compare lotes sintéticos de 10, 100 e 1.000 itens apenas após configurar limites e permissões. `select=*` e envio integral de `9.data` ao agente podem exceder contexto e elevar custo. O parâmetro de histórico do agente deve ser revisado para evitar gasto desnecessário.

**Riscos funcionais:** corpo JSON do módulo 7 interpola textos sem escape explícito; caracteres como aspas podem quebrar JSON. Há possível supercontagem de faturamento por NF-e e conflito entre linhas com mesmo produto. Verifique paginação e dados retornados antes de usar como evidência de resultado.

## 9. Demonstração e rastreabilidade

- **Origem:** TCC Faixa Marrom RDD10+ — Camada Ouro + Agente IA.
- **Blueprint público:** `RDD10_FAIXA_PRETA_CONSULTORIA_CONTABIL_IA_V3_PUBLICO.blueprint.json`.
- **README:** `README_FAIXA_PRETA_V3_PUBLICO.md` (renomear para `README.md` no repositório, substituindo o README anterior somente após revisão).
- **Link externo do cenário Make / vídeo:** não disponibilizado; não inventar evidências.
- **Prova de execução:** não disponível nesta cópia; configuração não equivale a execução.

## 10. Checklist para publicação

- [x] JSON sintaticamente válido e oito módulos preservados.
- [x] Placeholders de projeto e conexões presentes.
- [x] README e documentação detalhada preparados.
- [x] Requisitos, módulos, versões, riscos e estimativa de custos documentados.
- [ ] Revisão humana de segurança do JSON, README e documentação.
- [ ] Conferir e publicar os três arquivos no repositório público.
- [ ] Conferir link público em janela anônima.
- [ ] Preencher `contexto_projeto` e tags **contabilidade**, **make**, **nfe**, **supabase** na plataforma de submissão, caso os campos existam.
- [ ] Adicionar link externo de cenário ou demonstração **somente se houver link real e seguro**.
- [ ] Reconfigurar Data Store, conexões e executar testes sintéticos em cenário isolado, se exigido pela avaliação.

**Autoria:** Price Contabilidade — RDD10+ Faixa Preta 2026/2.
