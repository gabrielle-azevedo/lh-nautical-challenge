# LH Nautical — Desafio de Dados

> **Transformando dados desorganizados em inteligência estratégica para o varejo náutico.**

---

##  Contexto

A **LH Nautical** é uma empresa líder no varejo de peças e acessórios para embarcações, operando em modelo híbrido com loja física em Florianópolis e e-commerce de alcance nacional.
**Missão:** atuar como profissional de dados e transformar esse cenário — desde a limpeza das bases até a geração de insights preditivos e sistemas de recomendação.

---

##  Estrutura do Repositório

```
lh-nautical-data-challenge/
│
├──  data/
│   ├── vendas_limpo.csv
│   ├── produtos_limpo.csv
│   ├── clientes_limpo.csv
│   ├── custos_resumido.csv
│   ├── custos_expandido.csv
│   ├── base_completa.csv
│   └── recomendacoes.csv
│
├──  notebooks/
│   ├── 01_EDA.ipynb
│   ├── 02_Tratamento.ipynb
│   ├── 03_Analise_Vendas.ipynb
│   ├── 04_Analise_Clientes.ipynb
│   ├── 05_Previsao_Demanda.ipynb
│   └── 06_Recomendacao.ipynb
│
├──  apresentacao/
│   └── LH_Nautical_Apresentacao.pptx
│
├──  banco/
│   └── lh_nautical.db
│
└── README.md
```

---

##  Notebooks — Descrição

### `01_EDA.ipynb` — Exploração Inicial
Análise exploratória das 4 bases brutas recebidas.

**O que foi feito:**
- Verificação de shape, dtypes e primeiras linhas de cada base
- Contagem de valores nulos e duplicatas por coluna
- Estatísticas descritivas (min, max, média, desvio padrão)
- Análise do período coberto pelos dados de vendas
- Identificação dos problemas de qualidade em cada base

**Bases analisadas:**
| Base | Linhas | Colunas | Principais problemas |
|---|---|---|---|
| vendas_2023_2024 | — | id, id_client, id_product, qtd, total, sale_date | Verificar duplicatas e valores negativos |
| produtos_raw | — | code, name, price, actual_category | Duplicatas e textos despadronizados |
| custos_importacoes | — | product_id, product_name, category, historic_data | JSON aninhado na coluna historic_data |
| clientes_crm | — | full_name, email, code, location | E-mails com `#` e location despadronizado |

---

### `02_Tratamento.ipynb` — Limpeza e Padronização
Tratamento de todos os problemas identificados na EDA.

**O que foi feito:**
- Remoção de duplicatas (linhas inteiras iguais)
- Remoção de registros com nulos em colunas críticas
- Correção de e-mails com `#` → `@`
- Separação da coluna `location` em `cidade` e `estado` (regex + mapeamento de UFs)
- Padronização de texto com `.str.strip().str.title()`
- Conversão de tipos: datas com `pd.to_datetime()`, valores com `pd.to_numeric()`
- Remoção de vendas com `qtd <= 0` e `total <= 0`
- Expansão do JSON `historic_data` em linhas separadas (custos)
- Extração do custo mais recente por produto

**Bases geradas:**
- `vendas_limpo.csv`
- `produtos_limpo.csv`
- `clientes_limpo.csv`
- `custos_resumido.csv`
- `custos_expandido.csv`

---

### `03_Analise_Vendas.ipynb` — Performance Comercial
Análise de faturamento, produtos e sazonalidade via SQL.

**O que foi feito:**
- Criação do banco SQLite `lh_nautical.db` com as 4 tabelas limpas
- Join das 4 tabelas em uma `base_completa`
- Faturamento total por ano (2023 vs 2024)
- Faturamento mensal com variação percentual
- Top 10 produtos por faturamento e por quantidade
- Faturamento por categoria de produto
- Faturamento por estado (ranking nacional)
- Ticket médio por período

**Principais queries:**
```sql
-- Faturamento por ano
SELECT strftime('%Y', sale_date) AS ano,
       ROUND(SUM(total), 2) AS faturamento_total
FROM base_completa
GROUP BY ano ORDER BY ano;

-- Top produtos
SELECT name, SUM(qtd) AS total_qtd, ROUND(SUM(total), 2) AS total_faturado
FROM base_completa
GROUP BY name ORDER BY total_faturado DESC LIMIT 10;
```

---

### `04_Analise_Clientes.ipynb` — Segmentação e Valor
Identificação dos clientes mais valiosos e análise de comportamento.

**O que foi feito:**
- Top 10 clientes por faturamento acumulado
- Frequência de compra por cliente
- Clientes recorrentes (compraram em 2023 e 2024)
- Distribuição de clientes por estado
- Segmentação RFM simplificada:
  -  Alto Valor: acima de R$ 10.000
  -  Médio Valor: entre R$ 5.000 e R$ 10.000
  -  Baixo Valor: abaixo de R$ 5.000

**Principais queries:**
```sql
-- Segmentação por valor
SELECT full_name,
       ROUND(SUM(total), 2) AS valor_total,
       CASE
           WHEN SUM(total) >= 10000 THEN 'Alto Valor'
           WHEN SUM(total) >= 5000  THEN 'Médio Valor'
           ELSE 'Baixo Valor'
       END AS segmento
FROM base_completa
GROUP BY id_client, full_name
ORDER BY valor_total DESC;
```

---

### `05_Previsao_Demanda.ipynb` — Machine Learning com Prophet
Previsão de faturamento diário para os próximos 90 dias.

**O que foi feito:**
- Preparação da série temporal com agregação diária
- Preenchimento de dias sem venda com valor 0
- Treinamento do modelo Prophet com sazonalidades anual e semanal
- Previsão de 90 dias à frente com intervalo de confiança
- Visualização da tendência, sazonalidade semanal e anual

**Principais insights:**
- Tendência de crescimento estável ao longo do período
- Novembro é o pico anual (Black Friday)
- Abril/Maio representam o vale do ano
- Sexta-feira e Sábado são os melhores dias da semana

**Configuração do modelo:**
```python
modelo = Prophet(
    yearly_seasonality=True,
    weekly_seasonality=True,
    daily_seasonality=False,
    seasonality_mode="multiplicative"
)
```

---

### `06_Recomendacao.ipynb` — Sistema de Recomendação
Recomendação de produtos por similaridade entre clientes.

**O que foi feito:**
- Criação da matriz cliente × produto com quantidade comprada
- Cálculo de similaridade de cosseno entre clientes
- Função de recomendação: identifica clientes similares e sugere produtos ainda não comprados
- Geração de top 3 recomendações para cada cliente
- Enriquecimento com nome, preço e categoria do produto

**Lógica:**
```
Cliente A comprou Produto 1 e Produto 2
Cliente B comprou Produto 1 e Produto 3
→ Recomendação para A: Produto 3
→ Recomendação para B: Produto 2
```

**Output:** `recomendacoes.csv` com id_client, produto recomendado, nome, preço e categoria.

---

## Tecnologias Utilizadas

| Ferramenta | Uso |
|---|---|
| **Python 3.12** | Linguagem principal |
| **Pandas** | Manipulação e análise de dados |
| **SQLite3** | Banco de dados relacional |
| **Prophet** | Previsão de séries temporais |
| **Scikit-learn** | Similaridade de cosseno |
| **Matplotlib** | Visualizações |
| **Google Colab** | Ambiente de desenvolvimento |

---

## Como Reproduzir

### 1. Clone o repositório
```bash
git clone https://github.com/seu-usuario/lh-nautical-data-challenge.git
cd lh-nautical-data-challenge
```

### 2. Instale as dependências
```bash
pip install pandas prophet scikit-learn matplotlib pandasql
```

### 3. Execute os notebooks em ordem
```
01_EDA.ipynb          → exploração inicial
02_Tratamento.ipynb   → limpeza e padronização
03_Analise_Vendas.ipynb → análise comercial
04_Analise_Clientes.ipynb → segmentação
05_Previsao_Demanda.ipynb → modelo Prophet
06_Recomendacao.ipynb → sistema de recomendação
```

> Execute sempre na ordem numérica — cada notebook depende das saídas do anterior.

---

## Principais Insights

### Vendas
- Faturamento em crescimento estável de 2023 para 2024
- Novembro é o mês de maior pico (Black Friday)
- Sexta-feira e Sábado concentram os maiores volumes

### Clientes
- 18% dos clientes são de Alto Valor (acima de R$ 10.000)
- SC lidera o faturamento por estado — sede da loja física
- SP e RJ representam ~29% do faturamento do e-commerce

### Produtos
- Motores e Embarcações dominam o ranking de faturamento
- Top 3 produtos concentram grande parte da receita

---

## 📁 Arquivos de Saída

| Arquivo | Descrição |
|---|---|
| `vendas_limpo.csv` | Base de vendas tratada |
| `produtos_limpo.csv` | Catálogo de produtos limpo |
| `clientes_limpo.csv` | CRM de clientes padronizado |
| `custos_resumido.csv` | Custo atual por produto (USD) |
| `custos_expandido.csv` | Histórico completo de custos |
| `base_completa.csv` | Join das 4 bases |
| `recomendacoes.csv` | Top 3 recomendações por cliente |
| `lh_nautical.db` | Banco SQLite com todas as tabelas |

---

*Desafio de Dados — Incidium Academy*
