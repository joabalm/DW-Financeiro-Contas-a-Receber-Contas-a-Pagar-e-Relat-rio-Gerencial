# DW Financeiro: Contas a Receber, Contas a Pagar e Relatório Gerencial

> ETL em camadas (Bronze–Silver–Gold) com PostgreSQL na nuvem e dashboards em Power BI para análise de resultado, inadimplência e despesas por centro de custo.

Este projeto implementa um **Data Warehouse Financeiro** completo, desde a geração de dados sintéticos até a camada de visualização em **Power BI**, passando por um processo de **ETL em camadas (Bronze → Silver → Gold)** e carga em um **banco PostgreSQL na nuvem (Railway)**.

O foco é responder perguntas típicas de gestão financeira:

- Qual é o **resultado previsto** e o **resultado realizado** ao longo do tempo?
- Como está o **saldo em aberto** da empresa?
- Qual o nível de **inadimplência da carteira de clientes**?
- Como se comportam as **despesas por centro de custo** e por tipo de fornecedor?

---

## 1. Arquitetura do Projeto

Pipeline em camadas:

1. **Bronze**  
   - CSVs sintéticos de:
     - `contas_a_receber`
     - `contas_a_pagar`
     - `clientes`
     - `fornecedores`
     - `centros_custo`
     - `plano_contas`
     - `calendario`  
   - Dados crus, sem tratamento.

2. **Silver (notebook `bronze_to_silver_financeiro.ipynb`)**  
   - Limpeza e padronização:
     - Conversão de datas.
     - Normalização de status (ABERTA, ATRASADA, PAGA, CANCELADA).
     - Cálculo de `valor_em_aberto`, `dias_atraso`, `flag_atrasada`, `flag_em_aberto`.
   - Salvamento em `parquet` na pasta `SILVER`.

3. **Gold (notebook `silver_to_gold_financeiro.ipynb`)**  
   - Modelagem dimensional em **esquema estrela**:
     - Dimensões:
       - `dim_tempo`
       - `dim_cliente`
       - `dim_fornecedor`
       - `dim_centro_custo`
       - `dim_plano_contas`
     - Fatos:
       - `fato_contas_receber`
       - `fato_contas_pagar`
   - Criação de chaves de tempo (`id_tempo_emissao`, `id_tempo_vencimento`, `id_tempo_pagamento`) a partir da `dim_tempo`.
   - Persistência em `parquet` na pasta `GOLD`.

4. **DW em PostgreSQL (notebook `load_gold_to_postgres.ipynb` + script SQL)**  
   - Criação do schema `dw_financeiro` e tabelas:
     - `dim_tempo`, `dim_cliente`, `dim_fornecedor`,
       `dim_centro_custo`, `dim_plano_contas`,
       `fato_contas_receber`, `fato_contas_pagar`.
   - Carga dos arquivos Gold para o banco PostgreSQL hospedado na Railway.

5. **Camada de Visualização (Power BI)**  
   - Conexão direta ao PostgreSQL (`dw_financeiro`).
   - Criação do modelo tabular com relacionamentos 1:* entre dimensões e fatos.
   - Construção de painéis gerenciais.

---

## 2. Modelo Dimensional

### Dimensões

- **dim_tempo**
  - `id_tempo` (YYYYMMDD)
  - `data`, `ano`, `mes`, `nome_mes`, `dia`, `nome_dia_semana`
  - `ano_mes` (ex.: 2025-01), `trimestre`

- **dim_cliente**
  - `id_cliente`, `nome_cliente`, `segmento`, `cidade`, `estado`, `regiao`

- **dim_fornecedor**
  - `id_fornecedor`, `nome_fornecedor`, `tipo_fornecedor`, `cidade`, `estado`

- **dim_centro_custo**
  - `id_centro_custo`, `nome_centro_custo`, `tipo` (Operacional, Administrativo, etc.)

- **dim_plano_contas**
  - `id_plano_contas`, `codigo_contabil`, `descricao`, `grupo` (Receita/Despesa), `tipo`

### Fatos

- **fato_contas_receber**
  - Chaves:
    - `id_conta_receber`
    - `id_cliente`, `id_centro_custo`, `id_plano_contas`
    - `id_tempo_emissao`, `id_tempo_vencimento`, `id_tempo_pagamento`
  - Medidas:
    - `valor_original`, `valor_pago`, `valor_em_aberto`
    - `juros_multa`, `desconto`, `dias_atraso`
    - `flag_atrasada`, `flag_em_aberto`
  - Datas brutas: `data_emissao`, `data_vencimento`, `data_pagamento`

- **fato_contas_pagar**
  - Chaves:
    - `id_conta_pagar`
    - `id_fornecedor`, `id_centro_custo`, `id_plano_contas`
    - `id_tempo_emissao`, `id_tempo_vencimento`, `id_tempo_pagamento`
  - Medidas:
    - `valor_original`, `valor_pago`, `valor_em_aberto`
    - `juros_multa`, `desconto`, `dias_atraso`
    - `flag_atrasada`, `flag_em_aberto`
  - Datas brutas: `data_emissao`, `data_vencimento`, `data_pagamento`

---

## 3. Principais Medidas (DAX)

### Contas a Receber

- `CR Valor Original` = SUM( valor_original )  
- `CR Valor Pago` = SUM( valor_pago )  
- `CR Valor em Aberto` = SUM( valor_em_aberto )  
- `CR Valor em Aberto Atrasado` = SUM( valor_em_aberto ) apenas para títulos com `flag_atrasada = 1`  
- `CR % Carteira em Atraso` = `CR Valor em Aberto Atrasado / CR Valor em Aberto`  
- `CR Titulos em Atraso (Qtde)` = quantidade de títulos com `flag_atrasada = 1`  
- `CR Atraso Médio (dias)` = média de `dias_atraso` para títulos atrasados.

### Contas a Pagar

- `CP Valor Original`, `CP Valor Pago`, `CP Valor em Aberto`  
- `CP Valor em Aberto Atrasado`  
- `CP % Obrigações em Atraso`  
- `CP Titulos em Atraso (Qtde)`  
- `CP Atraso Médio (dias)`

### Visão consolidada

- `Resultado Previsto` = CR Valor Original − CP Valor Original  
- `Resultado Realizado` = CR Valor Pago − CP Valor Pago  
- `Saldo em Aberto` = CR Valor em Aberto − CP Valor em Aberto  

---

## 4. Dashboards construídos (Power BI)

O arquivo `powerbi/relatorio_financeiro.pbix` possui três páginas principais:

1. **Visão Geral Financeira**
   - Cards: Resultado Realizado, Resultado Previsto, Saldo em Aberto, títulos em atraso (CR/CP).
   - Gráfico de linhas: Resultado ao longo do tempo.
   - Gráfico de colunas: Receitas vs Despesas por mês.
   - Gráfico de barras: Pagamento por tipo de fornecedor.

2. **Inadimplência & Risco**
   - Cards: % da carteira em atraso, Receitas em Aberto, Receitas em Atraso, Atraso Médio (dias).
   - Gráfico de colunas: Atraso por mês.
   - Tabela detalhada de títulos em atraso (cliente, vencimento, dias em atraso, valor em aberto).
   - Ranking “Top clientes inadimplentes”.

3. **Despesas por Centro de Custo**
   - Cards: Despesas Previstas, Despesas Pagas, Despesas em Aberto, CP Títulos em Atraso, CP Atraso Médio (dias).
   - Gráfico de colunas empilhadas: Despesa por mês (Previsto x Pago).
   - Gráfico de barras horizontais: Despesa por Centro de Custo.
   - Tabela detalhada de títulos a pagar (fornecedor, centro de custo, vencimento, dias em atraso, valor em aberto).

---
