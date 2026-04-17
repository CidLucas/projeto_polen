# Pipeline de Preço Justo — Materiais Recicláveis

Pipeline automatizado para gerar a **tabela de preço justo (R$/kg)** por material × estado, a partir de dados de notas fiscais e catálogo de estoque.

---

## Estrutura de Arquivos

```
projetos/polen/
├── preco_justo_pipeline.ipynb   # Notebook principal do pipeline
├── products_invoices.csv        # Dados de notas fiscais (modo teste/local)
├── consulta_estoque.csv         # Catálogo de estoque (modo teste/local)
├── output/                      # Diretório de saída (criado automaticamente)
│   ├── preco_justo_YYYY-MM-DD.csv   # Resultado datado
│   └── preco_justo_latest.csv       # Resultado mais recente (sobrescrito)
└── README.md
```

### Onde colocar os arquivos

| Arquivo | Localização | Descrição |
|---|---|---|
| `preco_justo_pipeline.ipynb` | `projetos/polen/` | O notebook do pipeline. Deve estar neste diretório. |
| `service_account.json` | Raiz do repositório (`../../service_account.json` relativo ao notebook) | Credenciais do Google Cloud para acesso ao BigQuery. Necessário apenas no modo produção. |
| `products_invoices.csv` | `projetos/polen/` | Exportação da tabela `dataform.products_invoices` do BigQuery. Necessário apenas no modo teste. |
| `consulta_estoque.csv` | `projetos/polen/` | Exportação da tabela `omie_etl_hive.consulta_estoque` do BigQuery. Necessário apenas no modo teste. |

---

## Modos de Execução

### Modo Teste (CSV local)

O notebook vem configurado para ler dos CSVs locais (`products_invoices.csv` e `consulta_estoque.csv`). Basta executar célula por célula.

### Modo Produção (BigQuery)

Na célula **"2. Exportação do BigQuery"**, comente o bloco de CSV local e descomente o bloco de BigQuery:

```python
# Comentar estas linhas:
# df_invoices = pd.read_csv("products_invoices.csv", usecols=COLS_INVOICES)
# df_estoque = pd.read_csv("consulta_estoque.csv", usecols=["c_descricao"])

# Descomentar estas linhas:
creds = service_account.Credentials.from_service_account_file(SERVICE_ACCOUNT_FILE)
client = bigquery.Client(credentials=creds, project=PROJECT_ID)
df_invoices = client.query(QUERY_INVOICES).to_dataframe()
df_estoque = client.query(QUERY_ESTOQUE).to_dataframe()
```

**Projeto BigQuery:** `analytics-big-query-242119`
**Tabelas:**
- `dataform.products_invoices`
- `omie_etl_hive.consulta_estoque`

---

## Dependências

```bash
pip install google-cloud-bigquery db-dtypes pandas numpy scikit-learn scipy
```

Ou execute a primeira célula do notebook que roda `!pip install google-cloud-bigquery db-dtypes`.

---

## Configurando o Cron Job

O notebook é projetado para rodar automaticamente a cada 2 semanas via `cron`.

### 1. Converter o notebook para script Python (opcional)

Se preferir executar como `.py` ao invés de `.ipynb`:

```bash
jupyter nbconvert --to script preco_justo_pipeline.ipynb
```

Isso gera `preco_justo_pipeline.py`.

### 2. Executar o notebook diretamente via cron

```bash
# Rodar o notebook in-place (atualiza outputs dentro do .ipynb)
jupyter nbconvert --to notebook --execute preco_justo_pipeline.ipynb --output preco_justo_pipeline.ipynb
```

### 3. Configurar o crontab

```bash
crontab -e
```

Adicione uma das linhas abaixo:

**A cada 2 semanas (1º e 15 de cada mês, às 6h):**

```cron
0 6 1,15 * * cd /caminho/para/projetos/polen && /caminho/para/python -m jupyter nbconvert --to notebook --execute preco_justo_pipeline.ipynb --output preco_justo_pipeline.ipynb >> /var/log/preco_justo.log 2>&1
```

**Se converteu para `.py`:**

```cron
0 6 1,15 * * cd /caminho/para/projetos/polen && /caminho/para/python preco_justo_pipeline.py >> /var/log/preco_justo.log 2>&1
```

> **Importante:** Substitua `/caminho/para/` pelo caminho absoluto no servidor. Use `which python` e `which jupyter` para encontrar os executáveis corretos.

### 4. Verificar o cron

```bash
# Listar crons ativas
crontab -l

# Verificar logs
tail -f /var/log/preco_justo.log
```

---

## Saída: CSV de Preço Justo

A cada execução, o pipeline gera dois arquivos em `output/`:

| Arquivo | Descrição |
|---|---|
| `preco_justo_YYYY-MM-DD.csv` | Resultado datado (histórico) |
| `preco_justo_latest.csv` | Sempre o resultado mais recente (sobrescrito a cada execução) |

### Colunas do CSV

| Coluna | Tipo | Descrição |
|---|---|---|
| `cluster` | string | Nome do cluster de material (ex: `PEAD::SUCATA DE PEAD CRISTAL`) |
| `material_base` | string | Grupo base do material (ex: `PEAD`, `PET`, `PP`) |
| `state` | string | UF do emitente (ex: `SP`, `PR`, `SC`) |
| `dominant_sector` | string | Setor CNAE mais frequente no par cluster×estado |
| `n_obs` | int | Número de transações usadas no cálculo |
| `n_months_data` | int | Quantidade de meses com dados |
| `n_products` | int | Produtos distintos no cluster |
| `median_price` | float | Preço mediano (R$/kg) |
| `Q1` | float | 1º quartil do preço (R$/kg) |
| `Q3` | float | 3º quartil do preço (R$/kg) |
| `fair_buy` | float | Preço justo de compra = Q1 (R$/kg) |
| `fair_sell` | float | Preço justo de venda = Q3 (R$/kg) |
| `fair_target` | float | Preço justo alvo = mediana (R$/kg) |
| `predicted_next` | float | Preço mediano do último mês disponível (proxy para próximo mês) |
| `heterogeneity` | float | Coeficiente de variação — quanto menor, mais estável o preço |
| `top_products` | string | Top 3 produtos por volume no cluster |
| `quality` | string | Tier de qualidade: A (excellent), B (good), C (fair) |
| `run_date` | string | Data de execução do pipeline (YYYY-MM-DD) |

### Como usar o CSV

```python
import pandas as pd

df = pd.read_csv("output/preco_justo_latest.csv")

# Filtrar por material e estado
pet_sp = df[(df["material_base"] == "PET") & (df["state"] == "SP")]
print(pet_sp[["cluster", "fair_buy", "fair_target", "fair_sell", "quality"]])

# Apenas pares de alta confiança
high_quality = df[df["quality"].isin(["A (excellent)", "B (good)"])]
```

### Interpretação dos preços

- **`fair_buy` (Q1):** Preço abaixo do qual 25% das transações ocorrem. Referência para quem **compra** material.
- **`fair_target` (mediana):** Preço central do mercado. Referência para negociação.
- **`fair_sell` (Q3):** Preço acima do qual 25% das transações ocorrem. Referência para quem **vende** material.
- **`heterogeneity`:** CV < 0.3 = preço muito estável, CV > 0.5 = alta variação.
- **`quality`:**
  - **A** — ≥100 transações e CV < 0.3 (alta confiança)
  - **B** — ≥50 transações e CV < 0.5 (boa confiança)
  - **C** — ≥20 transações (confiança moderada)

---

## Etapas do Pipeline

1. **Exportação de dados** — Lê notas fiscais e catálogo de estoque (BigQuery ou CSV)
2. **Limpeza** — Remove nulos, preços zerados, materiais desconhecidos, filtra anos 2023-2025
3. **Classificação** — Classifica natureza da operação (venda/compra) e grupo de material (PEAD, PET, PP, etc.)
4. **TF-IDF Matching** — Cruza descrições das notas com o catálogo de estoque usando similaridade textual (char n-grams + word n-grams, 50/50)
5. **Clusterização** — Agrupa produtos similares via linkage hierárquico (distância cosseno, corte 0.50). Desambigua por preço com KMeans quando a razão max/min > 2.0
6. **Remoção de outliers** — IQR (k=1.5) por cluster × estado. Remove materiais com < 10 obs e clusters com < 50 obs
7. **Agregação mensal** — Calcula mediana, média, Q25, Q75 por mês × cluster × estado. Gera lag-1m (preço do mês anterior)
8. **Tabela de preço justo** — Gera Q1, mediana, Q3, CV, setor dominante, tier de qualidade por cluster × estado
9. **Exportação** — Salva CSVs datado e `latest`

---

## Parâmetros Ajustáveis

Definidos na célula de configuração:

| Parâmetro | Valor padrão | Descrição |
|---|---|---|
| `TFIDF_THRESHOLD` | 0.25 | Score mínimo para considerar um match TF-IDF |
| `MATCH_SCORE_MIN` | 0.6 | Score mínimo para manter o registro após matching |
| `MIN_OBS_FAIR_PRICE` | 5 | Obs mínimas para incluir na tabela de preço justo |
| `PRICE_RATIO_THRESHOLD` | 2.0 | Razão max/min de preço para split do cluster |
| `NAME_DIST_THRESHOLD` | 0.50 | Distância de corte para linkage hierárquico |
| `MIN_MAT_OBS` | 10 | Obs mínimas por material |
| `MIN_CLUSTER_OBS` | 50 | Obs mínimas por cluster |
