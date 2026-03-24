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
│   ├── vendas_2023-2024.ipynb
|   ├── custos_importacao.ipynb
|   ├── vendas_2023-2024.ipynb
│   ├── clientes_crm.ipynb
│   ├── Analise_de_Vendas.ipynb
│   ├── Analise_clientes.ipynb
│   ├── Prev_demand.ipynb
│   └── sist_recomen.ipynb
│
├──  apresentacao/
│   └── LH_Nautical_Apresentacao.ppsx
│
├──  banco/
│   └── lh_nautical.db
│
└── README.md
```

---

##  Notebooks — Descrição

## Notebooks

---

### `vendas_2023-2024.ipynb`
**EDA e tratamento da base de vendas**

Carrega o arquivo `vendas_2023_2024.csv` e faz a exploração inicial: shape, tipos de coluna, nulos e duplicatas. Gera tabelas de resumo separadas para cada coluna — quantidade vendida (`qtd`), valor financeiro (`total`), clientes únicos, produtos únicos e período coberto pelas datas.

Nenhuma limpeza pesada foi necessária nessa base. A saída é o dataframe tratado, pronto para ser carregado no banco SQLite na etapa seguinte.

**Colunas trabalhadas:** `id`, `id_client`, `id_product`, `qtd`, `total`, `sale_date`

---

### `produtos.ipynb`
**EDA e tratamento do catálogo de produtos**

Carrega `produtos_raw.csv` e inspeciona as quatro colunas disponíveis. O principal trabalho aqui foi na coluna `price`, que chegou com o prefixo `R$` em formato texto — foi convertida para numérico após limpeza da string. A coluna `actual_category` passou por padronização de texto para remover acentos e inconsistências de capitalização.

Duplicatas foram identificadas e removidas. O notebook salva a base limpa como `produtos_limpo.csv`.

**Colunas trabalhadas:** `code`, `name`, `price`, `actual_category`

---

### `clientes_crm.ipynb`
**EDA e tratamento da base de clientes**

O arquivo `clientes_crm.json` tinha dois problemas principais: e-mails com `#` no lugar de `@`, e a coluna `location` com formatos completamente inconsistentes — a mesma informação aparecia como `"PE , Recife"`, `"AC , Rio Branco"`, `"PB/Cabedelo"` e várias outras variações.

A solução para o `location` foi uma função de regex que separa cidade e estado independente do separador usado (vírgula, hífen ou barra), com um dicionário de correções manuais para os casos que não foram resolvidos automaticamente. A coluna original foi descartada após a separação.

A base limpa foi salva como `clientes_limpo.csv`.

**Colunas trabalhadas:** `full_name`, `email`, `code`, `location` → `cidade`, `estado`

---

### `custos_importacao.ipynb`
**EDA e tratamento dos custos de importação**

Carrega `custos_importacao.json`. A complexidade desse notebook está na coluna `historic_data`, que armazena uma lista de objetos JSON com datas e preços em dólar — um histórico de variações de custo por produto.

O tratamento teve duas etapas: primeiro extraiu o custo mais recente de cada produto para criar uma coluna `custo_atual_usd` (base resumida), depois expandiu o histórico completo em linhas separadas para análises temporais (base expandida).

Resultado: dois arquivos — `custos_resumido.csv` e `custos_expandido.csv`.

**Colunas trabalhadas:** `product_id`, `product_name`, `category`, `historic_data`

---

### `Analise_de_Vendas.ipynb`
**Modelagem SQL e análise de faturamento**

Cria o banco SQLite `lh_nautical.db`, carrega as quatro bases limpas como tabelas e constrói a `base_completa` via JOIN entre vendas, produtos, clientes e custos. Essa tabela é a base de todas as análises subsequentes.

As análises de vendas cobrem faturamento por ano, faturamento mensal, top 10 produtos por receita, faturamento por categoria e faturamento por estado. Os resultados são exportados também como `base_completa.csv`.

**Dependências:** `vendas_limpo.csv`, `produtos_limpo.csv`, `clientes_limpo.csv`, `custos_resumido.csv`

---

### `Analise_clientes.ipynb`
**Segmentação e análise de comportamento de clientes**

Parte da `base_completa` no banco para responder perguntas sobre o perfil dos clientes. Cobre quatro análises: ranking dos top 10 clientes por faturamento, segmentação RFM simplificada por faixas de valor (Alto, Médio e Baixo), frequência de compra por cliente e distribuição geográfica por estado.

A segmentação usa `SUM(total) >= 55.000.000` como corte para Alto Valor — threshold definido a partir da distribuição real dos dados.

**Dependências:** `lh_nautical.db` com a tabela `base_completa`

---

### `Prev_demand.ipynb`
**Previsão de demanda com Prophet**

Instala e usa a biblioteca Prophet (Meta) para prever o faturamento diário dos próximos 90 dias. A série temporal é preparada com agregação diária e preenchimento de dias sem venda com zero — etapa importante para não inflar a média do modelo.

O modelo é configurado com sazonalidade anual e semanal no modo multiplicativo, que performa melhor em dados de varejo com variação proporcional. Os gráficos de saída mostram a previsão com intervalo de confiança e os componentes separados de tendência, sazonalidade semanal e anual.

**Dependências:** `lh_nautical.db`, biblioteca `prophet`

---

### `sist_recomen.ipynb`
**Sistema de recomendação por similaridade de cosseno**

Monta uma matriz cliente × produto com as quantidades compradas, calcula a similaridade de cosseno entre todos os pares de clientes e usa os clientes mais similares para sugerir produtos que o cliente alvo ainda não comprou.

Para cada cliente são geradas 3 recomendações. O resultado final é enriquecido com nome, preço e categoria do produto via JOIN no banco e exportado como `recomendacoes.csv`.

**Dependências:** `lh_nautical.db`, biblioteca `scikit-learn`

---

## Arquivos gerados

| Arquivo | Gerado em |
|---|---|
| `vendas_limpo.csv` | `vendas_2023-2024.ipynb` |
| `produtos_limpo.csv` | `produtos.ipynb` |
| `clientes_limpo.csv` | `clientes_crm.ipynb` |
| `custos_resumido.csv` | `custos_importacao.ipynb` |
| `custos_expandido.csv` | `custos_importacao.ipynb` |
| `lh_nautical.db` | `Analise_de_Vendas.ipynb` |
| `base_completa.csv` | `Analise_de_Vendas.ipynb` |
| `recomendacoes.csv` | `sist_recomen.ipynb` |

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
```
vendas_2023-2024.ipynb
produtos.ipynb
clientes_crm.ipynb
custos_importacao.ipynb
        ↓
Analise_de_Vendas.ipynb
Analise_clientes.ipynb
        ↓
Prev_demand.ipynb
sist_recomen.ipynb
```

> Execute sempre nessa sequência. Os notebooks de análise dependem das bases limpas geradas nos primeiros quatro.

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
