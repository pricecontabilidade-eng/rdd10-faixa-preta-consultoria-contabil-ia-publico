# Consultoria Contábil com IA — Faixa Preta RDD10+

**Price Contabilidade | RDD10+ — Faixa Preta 2026/2**

## Objetivo

Documentar uma arquitetura de consultoria contábil gerencial que transforma registros de NF-e processados na camada Prata em dados da camada Ouro, consulta os resultados no Supabase, solicita análise a um agente de IA e gera um documento no Google Docs.

**Escopo e evidência:** este repositório é uma **referência técnica pública anonimizada**. O cenário original desenvolvido no TCC da Faixa Marrom permanece separado. O Blueprint foi verificado quanto à estrutura JSON e à presença de oito módulos. A importação visual foi relatada como realizada em ambiente Make, mas o salvamento ficou bloqueado pela ausência de Data Store válido. **Não há evidência de execução bem-sucedida da cópia pública nem de resultados reais.**

## Arquitetura

`Data Store Prata (Make) → Iterator de itens → Variável de item → HTTP POST Supabase/Ouro → Agregador → HTTP GET Supabase/Ouro → Agente de IA → Google Docs`

## Arquivos

- [`RDD10_FAIXA_PRETA_CONSULTORIA_CONTABIL_IA_V3_PUBLICO.blueprint.json`](RDD10_FAIXA_PRETA_CONSULTORIA_CONTABIL_IA_V3_PUBLICO.blueprint.json) — Blueprint de referência com oito módulos, mapeamentos e prompts preservados; exige reconfiguração de conexões.
- [`FAIXA_PRETA_DOCUMENTACAO_TECNICA_V3_PUBLICO.md`](FAIXA_PRETA_DOCUMENTACAO_TECNICA_V3_PUBLICO.md) — descrição módulo a módulo, requisitos, versões, custos, limitações, segurança e checklist.

## Pré-requisitos

Conta Make com permissões para importar cenários e criar/selecionar Data Stores; Data Store Prata com estrutura compatível; projeto Supabase com tabela Ouro e chave única compatível; conexão HTTP autenticada com privilégios mínimos; conexão de provedor de IA; conta Google com acesso ao Google Docs. Credenciais devem ser cadastradas **somente no gerenciador seguro de conexões do Make**.

## Como importar

1. No Make, crie um cenário de testes isolado e importe o arquivo `.blueprint.json`.
2. Associe o Data Store de entrada e confira os campos e o mapeamento dos itens.
3. Configure as conexões HTTP para seu projeto Supabase e confirme tabela, colunas, políticas de acesso e `on_conflict`.
4. Configure a conexão do agente de IA e a conexão Google Docs.
5. Valide os mapeamentos, o JSON dinâmico e o tamanho dos lotes **antes de executar**.
6. Execute exclusivamente com dados sintéticos e registre resultados, duração, operações e custos. Não execute na base original.

## Segurança e limites

O arquivo contém placeholders como `SEU-PROJETO` e `CONFIGURAR_*`; não inclui credenciais funcionais conhecidas. A revisão automatizada não substitui inspeção humana. O JSON mantém o esquema dos dados e os prompts, mas não inclui NF-e reais. A consulta `select=*` não possui paginação explícita e o agente recebe `9.data` integralmente: controle volume e custo antes de usar.

## Evidência de demonstração

**Link público do cenário Make / vídeo:** não disponibilizado. Não é apresentada execução como concluída.

## Autoria

Price Contabilidade — projeto educacional RDD10+ | Faixa Preta 2026/2.
