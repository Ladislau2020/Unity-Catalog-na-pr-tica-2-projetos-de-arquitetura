# Unity Catalog na prática — 2 projetos de arquitetura

Nov 3, 2026 · @André

## Visão geral

Dois projetos completos, cada um aplicando uma das estratégias de catálogo descritas no módulo [Apply naming conventions](https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/2-apply-naming-conventions?pivots=text) do Microsoft Learn, com fontes de dados gratuitas do Databricks Marketplace e compute clássico (Classic compute).

|  | Projeto A — Vertix Retail Group | Projeto B — DormeBem Online |
| --- | --- | --- |
| Estratégia do Learn | Separate catalogs for each environment (`{domain}_{environment}`) | Organize by layer within environment catalogs (`{domain}_{layer}`) |
| Catálogos | `sales_dev`, `sales_test`, `sales_prod` | `dev`, `prod` |
| Schemas | `raw`, `staging`, `curated` | `web_bronze`, `web_silver`, `web_gold`, `sales_*`, `customer_*`, `marketing_gold` |
| Fonte (Marketplace) | Simulated Retail Customer Data | Simulated E-Commerce Clickstream Data |
| Negócio | Grupo de retalho com 3 subsidiárias | E-commerce de colchões |
| Onde fica o isolamento | No catálogo (1 catálogo por ambiente) | No workspace (1 workspace por ambiente, catálogo ligado a ele) |

Legenda usada no documento:

- **\[Learn\]** — padrão escrito literalmente na página do Microsoft Learn.
- **\[Arquiteto\]** — extensão minha para cobrir objetos que a página não nomeia (volumes, warehouses, políticas, grupos, alertas, Genie). São propostas coerentes com o padrão, não regra oficial.

Pré-requisito: compute clássico exige um workspace Azure Databricks pago ou em trial com Unity Catalog. O Databricks Free Edition só tem compute serverless, por isso não serve para este exercício.

## Fontes de dados (Databricks Marketplace, gratuitas)

As duas listagens são da própria Databricks, gratuitas e de acesso imediato; ao instalar, o Marketplace cria um catálogo **só de leitura** (Delta Sharing) cujo nome você escolhe.

| Listagem | Projeto | O que traz | Nome do catálogo ao instalar \[Arquiteto\] |
| --- | --- | --- | --- |
| [Simulated Retail Customer Data](https://marketplace.databricks.com/details/a82597f6-5ada-49d5-b934-d6c9dece16a1/Databricks_Simulated-Retail-Customer-Data) | A | Schema `v01`: tabelas `customers`, `sales`, `sales_orders`. Schema `v02`: volume `retail-pipeline` com 7 dias de encomendas de 3 subsidiárias fictícias (2025-11-01 a 2025-11-07) e `customer_changes_daily` em JSON (novos clientes, alterações, eliminações) | `src_mkt_retail_customers` |
| [Simulated E-Commerce Clickstream Data](https://marketplace.databricks.com/details/ca87a8be-9a17-4469-acee-d992d6cd16ca/Databricks_Simulated-ECommerce-Clickstream-Data) | B | Ficheiros com eventos, transações, utilizadores e produtos de um retalhista de colchões, de 2020-06-19 a 2020-07-04 | `src_mkt_ecommerce_clickstream` |

Regras para os catálogos de origem \[Arquiteto\]:

- Prefixo `src_mkt_` deixa claro que é dado de terceiros, só leitura, fora do medalhão. Segue a lógica do Learn para cenários de partilha: o nome indica propósito e origem.
- Um único catálogo de origem serve dev, test e prod. Nenhum ambiente escreve nele.
- Os ficheiros do volume partilhado são **copiados dia a dia** para um volume `landing` próprio. Isso simula a chegada incremental de dados e permite usar Auto Loader com checkpoint no seu storage.
- Antes de construir, abra cada tabela/volume e confirme os nomes reais das colunas e pastas. Os nomes de colunas citados mais abaixo são os esperados e devem ser validados.

## Convenções de nomenclatura comuns aos dois projetos

Todo o nome segue `minusculas_com_underscore`, sem versão no nome (`_v1`, `_final`, `_updated` vão para comentários ou table properties) e sem pontos, espaços ou barras.

| Objeto | Padrão | Exemplo | Origem |
| --- | --- | --- | --- |
| Catálogo por domínio + ambiente | `{domain}_{environment}` | `sales_prod` | Learn |
| Catálogo por ambiente | `{environment}` | `prod` | Learn |
| Schema por camada | `bronze` / `silver` / `gold` ou `raw` / `staging` / `curated` | `raw` | Learn |
| Schema domínio + camada | `{domain}_{layer}` | `web_silver` | Learn |
| Tabela | substantivo descritivo e curto | `cleaned_orders` | Learn |
| View / materialized view | `vw_{nome}` | `vw_customer_stats` | Learn |
| Tabela temporária de staging | `tmp_{nome}` | `tmp_import_staging` | Learn |
| Catálogo partilhado externamente | `{propósito}_shared` | `customer_analytics_shared` | Learn |
| Caminho de storage | `/{env}/{layer}/{domain}/{table}/` | `/prod/gold/sales/monthly_revenue/` | Learn |
| Cluster | `{environment}_{domain}_cluster` | `dev_sales_cluster` | Learn |
| Job | `job_{layer}_{purpose}` | `job_bronze_orders_ingestion` | Learn |
| Lakeflow pipeline | `pipe_{domain_ou_propósito}` | `pipe_orders_processing` | Learn |
| Pipeline de streaming | `stream_{source}_to_{target}` | `stream_landing_to_bronze` | Learn |
| Notebook | `notebook_{layer}_{purpose}` | `notebook_silver_orders_transformation` | Learn |
| Repositório Git | `repo_{project_name}` | `repo_sales_project` | Learn |
| SQL query | `query_{purpose}` | `query_monthly_sales_report` | Learn |
| Dashboard | `dashboard_{domain}_{purpose}` | `dashboard_sales_performance` | Learn |
| Volume | substantivo da função | `landing`, `checkpoints` | Arquiteto |
| SQL warehouse | `wh_{environment}_{domain}_{uso}` | `wh_prod_sales_bi` | Arquiteto |
| Cluster policy | `policy_{environment}_{uso}` | `policy_prod_jobs` | Arquiteto |
| Instance pool | `pool_{environment}_{node}` | `pool_prod_d4ds` | Arquiteto |
| Job cluster (chave no job) | `{environment}_etl_cluster` | `prod_etl_cluster` | Arquiteto (nome do Learn) |
| Job orquestrador | `job_e2e_{domain}_{frequência}` | `job_e2e_sales_daily` | Arquiteto |
| Função (UDF, máscara) | `fn_{purpose}` | `fn_mask_email` | Arquiteto |
| Alerta SQL | `alert_{purpose}` | `alert_orders_rejected_rate` | Arquiteto |
| Genie space | `genie_{domain}_{purpose}` | `genie_sales_questions` | Arquiteto |
| Grupo de contas | `grp_{domain}_{papel}` | `grp_sales_engineers` | Arquiteto |
| Service principal | `sp_{domain}_{environment}_{uso}` | `sp_sales_prod_deploy` | Arquiteto |
| Tags de compute | `env`, `domain`, `project`, `owner`, `cost_center` | `env=prod` | Arquiteto |

Decisão de arquiteto que difere do diagrama do Learn: o diagrama mostra tabelas como `orders_dev` e `orders_test`. Aqui **as tabelas têm o mesmo nome em todos os ambientes** (`orders`). Só o catálogo muda, passado como parâmetro. Assim o mesmo código corre em dev, test e prod sem alterações, que é o objetivo do CI/CD.

Todo notebook, job e pipeline recebe o parâmetro `catalog` (Projeto A) ou `env` (Projeto B) e nunca tem o nome do ambiente escrito no código.

## Projeto A — Environment isolation (Vertix Retail Group)

Um catálogo por ambiente (`sales_dev`, `sales_test`, `sales_prod`) dá o isolamento mais forte: permissões concedidas ao nível do catálogo e zero risco de um job de dev escrever em produção.

### Cenário de negócio

O Vertix Retail Group é um grupo de retalho com três subsidiárias, cada uma com o seu sistema de vendas. Hoje cada subsidiária envia um ficheiro diário de encomendas e a direção financeira consolida tudo em Excel, com três dias de atraso.

Além disso, o CRM central exporta diariamente as alterações de clientes (registos, atualizações de perfil e pedidos de eliminação). O departamento jurídico exige que um cliente eliminado desapareça das camadas de consumo no ciclo seguinte (direito ao esquecimento, RGPD).

O CDO pediu uma plataforma de vendas consolidada com três ambientes, porque uma alteração mal testada no ano passado corrompeu o relatório de fecho mensal. Requisito número 1: **nenhum programador pode escrever em produção**; só o pipeline de CI/CD.

### Perguntas de negócio a responder

1. Qual é a receita diária consolidada e por subsidiária?
2. Qual é o ticket médio e o número de encomendas por cliente, por subsidiária?
3. Quem são os 20 clientes de maior valor (customer lifetime value)?
4. Quantos clientes novos, atualizados e eliminados entram por dia?
5. Que produtos geram mais receita em cada subsidiária?
6. Qual é a percentagem de encomendas rejeitadas por qualidade (sem cliente válido, valor negativo, data inválida)?
7. Os clientes eliminados já não aparecem em nenhuma tabela `curated`? (controlo RGPD)

### Estrutura Unity Catalog

Camadas: `raw` = bronze, `staging` = silver, `curated` = gold (nomes do diagrama de environment isolation do Learn). A estrutura é idêntica nos três catálogos.

```txt
src_mkt_retail_customers          (catálogo de origem, só leitura, partilhado)
├─ v01  → customers, sales, sales_orders
└─ v02  → volume retail-pipeline

sales_dev | sales_test | sales_prod
├─ raw                              (schema — dado como chegou)
│  ├─ landing                       (volume: /orders/, /customer_changes/)
│  ├─ checkpoints                   (volume: checkpoints do Auto Loader)
│  ├─ raw_orders                    (Auto Loader, encomendas das 3 subsidiárias)
│  ├─ raw_customer_changes          (Auto Loader, JSON de alterações)
│  ├─ raw_customers                 (snapshot inicial de v01.customers)
│  ├─ raw_sales                     (snapshot de v01.sales)
│  └─ raw_sales_orders              (snapshot de v01.sales_orders)
├─ staging                          (schema — limpo e validado)
│  ├─ cleaned_orders                (tipos corrigidos, deduplicado, subsidiária normalizada)
│  ├─ rejected_orders               (linhas que falharam expectativas + motivo)
│  ├─ cleaned_sales                 (linhas de venda por produto)
│  ├─ validated_customers           (SCD tipo 2 com histórico)
│  └─ tmp_orders_dedup              (staging temporário do MERGE)
└─ curated                          (schema — pronto para negócio)
   ├─ orders                        (encomendas válidas, só clientes ativos)
   ├─ customers                     (versão atual, sem eliminados)
   ├─ daily_revenue_by_subsidiary
   ├─ product_revenue_by_subsidiary
   ├─ customer_lifetime_value
   ├─ customer_changes_daily_summary
   ├─ vw_customer_stats             (view)
   └─ vw_subsidiary_kpis            (view)
```

Função de governação: `sales_prod.curated.fn_mask_email` aplicada como column mask à coluna de e-mail de `customers`, visível em claro só para `grp_sales_admins`.

Storage (managed location por catálogo), no padrão `/{env}/{layer}/{domain}/`:

```txt
abfss://datalake@<storage>.dfs.core.windows.net/dev/raw/sales/
abfss://datalake@<storage>.dfs.core.windows.net/test/staging/sales/
abfss://datalake@<storage>.dfs.core.windows.net/prod/curated/sales/
```

Com 3 catálogos, a managed location fica ao nível do schema: `sales_prod.raw` → `/prod/raw/sales/`, `sales_prod.curated` → `/prod/curated/sales/`. Nomes de recursos: storage credential `cred_datalake_sales`, external locations `extloc_dev_sales`, `extloc_test_sales`, `extloc_prod_sales`.

### Compute (Classic)

| Nome | Tipo | Ambiente | Quem usa | Configuração sugerida |
| --- | --- | --- | --- | --- |
| `dev_sales_cluster` | All-purpose | dev | Engenheiros | Standard access mode, 1–2 workers `Standard_D4ds_v5`, auto-termination 30 min, `policy_dev_interactive` |
| `test_sales_cluster` | All-purpose | test | Engenheiro de release, só para depuração | Dedicated, 1 worker, auto-termination 15 min, normalmente desligado |
| `ad_hoc_analysis_cluster` | All-purpose | dev | Analistas (Python/exploração) | Standard, 1 worker, auto-termination 20 min, só lê `sales_prod.curated` |
| `dev_etl_cluster` | Job cluster (chave no job) | dev | Jobs de dev | 1 worker, `policy_dev_jobs` |
| `test_etl_cluster` | Job cluster | test | Jobs de test | 2 workers, `policy_test_jobs` |
| `prod_etl_cluster` | Job cluster | prod | Jobs de prod | Autoscale 2–4, `pool_prod_d4ds`, `policy_prod_jobs` |
| Compute do pipeline | Lakeflow classic | por ambiente | `pipe_customer_changes_cdc` | Modo triggered, Core/Pro, `policy_prod_pipelines` em prod |
| `wh_dev_sales_adhoc` | SQL warehouse Pro | dev | Engenheiros e queries de teste | 2X-Small, auto-stop 10 min |
| `wh_prod_sales_bi` | SQL warehouse Pro | prod | Dashboards, alertas, Genie | Small, 1–2 clusters, auto-stop 15 min |

Runtime: a versão LTS mais recente igual nos três ambientes. Todas as policies impõem as tags `env`, `domain=sales`, `project=vertix_sales`, `owner`. Com elas o custo por ambiente sai de `system.billing.usage`.

### Jobs e pipelines

| Nome | Tipo | O que faz | Agenda (prod) | Depende de |
| --- | --- | --- | --- | --- |
| `job_raw_landing_simulation` | Job (notebook) | Copia 1 dia de ficheiros do volume partilhado para `raw.landing` (parâmetro `load_date`) | Diário 05:00 | — |
| `job_raw_snapshot_ingestion` | Job (notebook) | Carga inicial de `v01` para `raw_customers`, `raw_sales`, `raw_sales_orders` | Manual, uma vez | — |
| `job_raw_orders_ingestion` | Job (notebook, Auto Loader `availableNow`) | `landing/orders/` → `raw_orders` | Pelo orquestrador | landing |
| `pipe_customer_changes_cdc` | Lakeflow Declarative Pipeline | `landing/customer_changes/` → `raw_customer_changes` → `validated_customers` (AUTO CDC, SCD tipo 2, eliminações aplicadas) | Pelo orquestrador | landing |
| `job_staging_orders_transformation` | Job (notebook) | `raw_orders` → `cleaned_orders` + `rejected_orders` (MERGE via `tmp_orders_dedup`) | Pelo orquestrador | raw\_orders |
| `job_curated_sales_aggregation` | Job (notebook) | Constrói `orders`, `customers`, receitas e CLV | Pelo orquestrador | staging |
| `job_curated_gdpr_check` | Job (SQL task) | Falha se existir em `curated` algum cliente eliminado | Pelo orquestrador | curated |
| `job_e2e_sales_daily` | Job orquestrador | Encadeia todos os anteriores e atualiza o dashboard | Diário 05:00 | — |

Com Databricks Asset Bundles, o `mode: development` acrescenta automaticamente o prefixo `[dev <utilizador>]` aos jobs em dev. Assim os nomes de jobs ficam iguais em todos os ambientes sem colidir.

### Repositório e notebooks

```txt
repo_sales_project/
├─ databricks.yml                     (bundle: targets dev, test, prod → variável catalog)
├─ resources/
│  ├─ job_e2e_sales_daily.yml
│  ├─ pipe_customer_changes_cdc.yml
│  └─ dashboard_sales_performance.yml
├─ src/
│  ├─ setup/
│  │  ├─ notebook_setup_catalog_objects
│  │  └─ notebook_setup_grants
│  ├─ raw/
│  │  ├─ notebook_raw_landing_simulation
│  │  ├─ notebook_raw_snapshot_ingestion
│  │  └─ notebook_raw_orders_ingestion
│  ├─ staging/
│  │  ├─ notebook_staging_orders_transformation
│  │  └─ pipe_customer_changes_cdc.sql
│  ├─ curated/
│  │  ├─ notebook_curated_sales_aggregation
│  │  ├─ notebook_curated_customer_lifetime_value
│  │  └─ notebook_curated_gdpr_check
│  └─ quality/
│     └─ notebook_quality_naming_validation
├─ sql/
│  ├─ query_daily_revenue_by_subsidiary.sql
│  ├─ query_average_ticket_by_subsidiary.sql
│  ├─ query_top_customers_lifetime_value.sql
│  ├─ query_customer_changes_daily.sql
│  ├─ query_product_revenue_ranking.sql
│  └─ query_orders_rejected_rate.sql
└─ tests/
   └─ test_staging_orders_transformation.py
```

Primeira célula de todos os notebooks:

```python
dbutils.widgets.text("catalog", "sales_dev")
catalog = dbutils.widgets.get("catalog")
spark.sql(f"USE CATALOG {catalog}")
```

### Queries, dashboards e alertas

| Objeto | Responde à pergunta |
| --- | --- |
| `query_daily_revenue_by_subsidiary` | 1 |
| `query_average_ticket_by_subsidiary` | 2 |
| `query_top_customers_lifetime_value` | 3 |
| `query_customer_changes_daily` | 4 |
| `query_product_revenue_ranking` | 5 |
| `query_orders_rejected_rate` | 6 |
| `dashboard_sales_performance` (AI/BI, em `wh_prod_sales_bi`) | 1, 2, 3, 5 |
| `dashboard_sales_data_quality` | 4, 6, 7 |
| `alert_orders_rejected_rate` | Dispara se rejeições > 5% no dia |
| `genie_sales_questions` | Perguntas livres sobre `sales_prod.curated` |

Pasta de workspace para estes objetos: `/Workspace/Shared/vertix_sales/{queries,dashboards}`.

### Permissões

| Grupo / principal | `sales_dev` | `sales_test` | `sales_prod` |
| --- | --- | --- | --- |
| `grp_sales_engineers` | `ALL PRIVILEGES` | `USE CATALOG`, `SELECT` | `USE CATALOG`, `SELECT` (só leitura para depuração) |
| `grp_sales_analysts` | — | — | `USE CATALOG`, `USE SCHEMA` + `SELECT` só em `curated` |
| `grp_sales_admins` | `MANAGE` | `MANAGE` | `MANAGE`, owner do catálogo |
| `sp_sales_test_deploy` | — | escrita total | — |
| `sp_sales_prod_deploy` | — | — | escrita total (único que escreve) |

Se tiver workspaces separados, use workspace-catalog binding: `sales_prod` fica ligado só ao workspace de produção.

### Script de criação (excerto)

```sql
CREATE CATALOG IF NOT EXISTS sales_prod
  COMMENT 'Vertix Retail Group — vendas consolidadas, produção';

CREATE SCHEMA IF NOT EXISTS sales_prod.raw
  MANAGED LOCATION 'abfss://datalake@<storage>.dfs.core.windows.net/prod/raw/sales/';
CREATE SCHEMA IF NOT EXISTS sales_prod.staging
  MANAGED LOCATION 'abfss://datalake@<storage>.dfs.core.windows.net/prod/staging/sales/';
CREATE SCHEMA IF NOT EXISTS sales_prod.curated
  MANAGED LOCATION 'abfss://datalake@<storage>.dfs.core.windows.net/prod/curated/sales/';

CREATE VOLUME IF NOT EXISTS sales_prod.raw.landing;
CREATE VOLUME IF NOT EXISTS sales_prod.raw.checkpoints;

GRANT USE CATALOG ON CATALOG sales_prod TO `grp_sales_analysts`;
GRANT USE SCHEMA, SELECT ON SCHEMA sales_prod.curated TO `grp_sales_analysts`;
GRANT USE CATALOG, SELECT ON CATALOG sales_prod TO `grp_sales_engineers`;
```

## Projeto B — Organize by layer within environment catalogs (DormeBem Online)

O ambiente é separado pelo workspace e cada workspace tem um catálogo (`dev`, `prod`); dentro dele, os schemas combinam domínio e camada (`web_bronze`, `sales_gold`). Toda a produção fica num único catálogo, o que simplifica a governação.

### Cenário de negócio

A DormeBem Online vende colchões só pela internet. O site regista cada evento de navegação (página vista, produto visto, adicionar ao carrinho, checkout, compra), a origem do tráfego e o dispositivo.

A equipa de marketing gasta a maior parte do orçamento em campanhas pagas e não sabe que canal traz compradores, nem onde os visitantes desistem. A equipa comercial quer saber que colchões são muito vistos mas pouco comprados. A direção quer medir o impacto da campanha do feriado de 4 de julho, que coincide com o fim do período dos dados.

Três equipas de domínio partilham a plataforma: **web** (clickstream), **sales** (transações e produtos) e **customer** (utilizadores). O domínio **marketing** não tem dados próprios: só consome e cruza os outros na camada gold.

### Perguntas de negócio a responder

1. Qual é o funil de conversão (visita → produto visto → carrinho → checkout → compra) por origem de tráfego e dispositivo?
2. Que canal (first touch) gera mais receita e mais compradores?
3. Qual é a taxa de abandono de carrinho por dia?
4. Que produtos são muito vistos e pouco comprados (taxa view-to-buy)?
5. Quanto tempo passa entre o primeiro contacto e a primeira compra?
6. Como evoluem receita diária e ticket médio, e houve efeito na semana do 4 de julho?
7. Em que estados/regiões (campo de geografia dos eventos) se concentram as compras?

Colunas esperadas no dataset, a confirmar ao instalar: eventos com `user_id`, `event_name`, `event_timestamp`, `device`, `geo`, `traffic_source`, `items`; vendas com `order_id`, `email`, `transaction_timestamp`, `purchase_revenue_in_usd`, `items`; utilizadores com `user_id`, `email`, `user_first_touch_timestamp`; produtos com `item_id`, `name`, `price`.

### Estrutura Unity Catalog

```txt
src_mkt_ecommerce_clickstream      (catálogo de origem, só leitura)

dev  (ligado ao workspace ws_dormebem_dev)
prod (ligado ao workspace ws_dormebem_prod)
├─ web_bronze
│  ├─ landing                       (volume: ficheiros de eventos por dia)
│  ├─ checkpoints                   (volume)
│  └─ raw_events
├─ web_silver
│  ├─ cleaned_events                (timestamps convertidos, deduplicado, items explodidos)
│  └─ sessions                      (sessão = 30 min de inatividade por user_id)
├─ web_gold
│  ├─ funnel_daily
│  ├─ traffic_source_performance
│  └─ vw_funnel_by_device
├─ sales_bronze
│  ├─ landing                       (volume)
│  ├─ raw_transactions
│  └─ raw_products
├─ sales_silver
│  ├─ cleaned_transactions
│  ├─ transaction_items             (uma linha por item vendido)
│  └─ products
├─ sales_gold
│  ├─ daily_revenue
│  ├─ product_performance           (vistas, carrinhos, compras, view-to-buy)
│  └─ vw_sales_summary
├─ customer_bronze
│  └─ raw_users
├─ customer_silver
│  └─ validated_users
├─ customer_gold
│  ├─ customer_lifetime_value
│  └─ vw_customer_stats
└─ marketing_gold                   (domínio consumidor: só lê silver/gold dos outros)
   ├─ attribution_first_touch
   ├─ cart_abandonment_daily
   ├─ time_to_first_purchase
   └─ vw_campaign_july4_impact
```

Por que não há `marketing_bronze` nem `marketing_silver`: um schema só existe quando tem dados. Criar schemas vazios só para “completar o padrão” é ruído.

Governação: `prod.customer_silver.fn_mask_email` como column mask em `validated_users.email` e `cleaned_transactions.email`.

Storage no padrão exato do Learn `/{env}/{layer}/{domain}/{table}/`, com managed location por schema:

```txt
prod.web_bronze      → abfss://datalake@<storage>.dfs.core.windows.net/prod/bronze/web/
prod.sales_gold      → abfss://datalake@<storage>.dfs.core.windows.net/prod/gold/sales/
prod.marketing_gold  → abfss://datalake@<storage>.dfs.core.windows.net/prod/gold/marketing/
dev.web_bronze       → abfss://datalake@<storage>.dfs.core.windows.net/dev/bronze/web/
```

Nomes de recursos: `cred_datalake_dormebem`, `extloc_dev`, `extloc_prod`.

### Compute (Classic)

| Nome | Tipo | Workspace | Quem usa | Configuração sugerida |
| --- | --- | --- | --- | --- |
| `dev_web_cluster` | All-purpose | dev | Equipa web | Standard, 1–2 workers, auto-termination 30 min, `policy_dev_interactive` |
| `dev_sales_cluster` | All-purpose | dev | Equipas sales e customer | Standard, 1–2 workers, auto-termination 30 min |
| `ad_hoc_analysis_cluster` | All-purpose | prod | Analistas de marketing | Standard, 1 worker, auto-termination 20 min, só lê schemas `*_gold` |
| `dev_etl_cluster` | Job cluster | dev | Jobs de dev | 1 worker, `policy_dev_jobs` |
| `prod_etl_cluster` | Job cluster | prod | Jobs batch de prod | Autoscale 2–4, `pool_prod_d4ds`, `policy_prod_jobs` |
| `prod_streaming_cluster` | Job cluster | prod | `stream_landing_to_bronze` | 2 workers fixos (streaming não beneficia de autoscale clássico), `policy_prod_streaming` |
| Compute do pipeline | Lakeflow classic | por workspace | `pipe_sales_processing` | Modo triggered, `policy_prod_pipelines` em prod |
| `wh_dev_analytics` | SQL warehouse Pro | dev | Desenvolvimento de queries | 2X-Small, auto-stop 10 min |
| `wh_prod_bi` | SQL warehouse Pro | prod | Dashboards, alertas, Genie | Small, 1–2 clusters, auto-stop 15 min |

Aqui o cluster é por domínio (`dev_web_cluster`, `dev_sales_cluster`) porque várias equipas partilham o mesmo catálogo; no Projeto A havia um só domínio. Tags: `env`, `domain`, `project=dormebem`, `owner`.

### Jobs e pipelines

| Nome | Tipo | O que faz | Agenda (prod) | Depende de |
| --- | --- | --- | --- | --- |
| `job_bronze_landing_simulation` | Job (notebook) | Copia 1 dia de ficheiros da origem para os volumes `landing` (parâmetro `load_date`) | Diário 04:30 | — |
| `stream_landing_to_bronze` | Job streaming (Auto Loader) | `web_bronze.landing` → `raw_events` | Contínuo ou `availableNow` de hora a hora | landing |
| `job_bronze_users_ingestion` | Job (notebook) | Utilizadores → `customer_bronze.raw_users` | Pelo orquestrador | landing |
| `pipe_sales_processing` | Lakeflow Declarative Pipeline | `sales_bronze` → `sales_silver` com expectativas de qualidade | Pelo orquestrador | landing |
| `job_silver_events_transformation` | Job (notebook) | `raw_events` → `cleaned_events` | Pelo orquestrador | raw\_events |
| `job_silver_sessions_build` | Job (notebook) | `cleaned_events` → `sessions` | Pelo orquestrador | cleaned\_events |
| `job_silver_users_validation` | Job (notebook) | `raw_users` → `validated_users` | Pelo orquestrador | raw\_users |
| `job_gold_funnel_aggregation` | Job (notebook) | `funnel_daily`, `traffic_source_performance` | Pelo orquestrador | silver web |
| `job_gold_sales_aggregation` | Job (notebook) | `daily_revenue`, `product_performance`, `customer_lifetime_value` | Pelo orquestrador | silver sales/customer |
| `job_gold_marketing_attribution` | Job (notebook) | Todas as tabelas de `marketing_gold` | Pelo orquestrador | gold web + sales |
| `job_e2e_ecommerce_daily` | Job orquestrador | Encadeia bronze → silver → gold e atualiza dashboards | Diário 05:00 | — |

O parâmetro `env` (`dev` ou `prod`) é o nome do catálogo e vem do target do bundle.

### Repositório e notebooks

Estrutura por camada, exatamente como no exemplo do Learn, com subpastas por domínio:

```txt
repo_ecommerce_project/
├─ databricks.yml                     (targets dev e prod → variável env, workspace host)
├─ resources/
│  ├─ job_e2e_ecommerce_daily.yml
│  ├─ stream_landing_to_bronze.yml
│  └─ pipe_sales_processing.yml
├─ setup/
│  ├─ notebook_setup_catalog_objects
│  └─ notebook_setup_grants
├─ bronze/
│  ├─ notebook_bronze_landing_simulation
│  ├─ notebook_bronze_events_ingestion
│  └─ notebook_bronze_users_ingestion
├─ silver/
│  ├─ notebook_silver_events_transformation
│  ├─ notebook_silver_sessions_build
│  ├─ notebook_silver_users_validation
│  └─ pipe_sales_processing.sql
├─ gold/
│  ├─ notebook_gold_funnel_aggregation
│  ├─ notebook_gold_sales_aggregation
│  └─ notebook_gold_marketing_attribution
├─ sql/
│  ├─ query_funnel_conversion_by_source.sql
│  ├─ query_revenue_by_first_touch.sql
│  ├─ query_cart_abandonment_rate.sql
│  ├─ query_product_view_to_buy.sql
│  ├─ query_time_to_first_purchase.sql
│  ├─ query_daily_revenue_trend.sql
│  └─ query_purchases_by_region.sql
└─ tests/
```

Primeira célula de todos os notebooks:

```python
dbutils.widgets.text("env", "dev")
env = dbutils.widgets.get("env")
spark.sql(f"USE CATALOG {env}")
# depois: spark.table("web_silver.cleaned_events")
```

### Queries, dashboards e alertas

| Objeto | Responde à pergunta |
| --- | --- |
| `query_funnel_conversion_by_source` | 1 |
| `query_revenue_by_first_touch` | 2 |
| `query_cart_abandonment_rate` | 3 |
| `query_product_view_to_buy` | 4 |
| `query_time_to_first_purchase` | 5 |
| `query_daily_revenue_trend` | 6 |
| `query_purchases_by_region` | 7 |
| `dashboard_web_funnel` | 1, 3 |
| `dashboard_marketing_attribution` | 2, 5, 6 |
| `dashboard_sales_performance` | 4, 6, 7 |
| `alert_cart_abandonment_spike` | Dispara se o abandono subir mais de 10 pontos face à média de 7 dias |
| `genie_marketing_questions` | Perguntas livres sobre `prod.marketing_gold` e `prod.sales_gold` |

### Permissões

Como vários domínios partilham o catálogo, as permissões descem ao nível do **schema**. É a principal diferença operacional face ao Projeto A.

| Grupo / principal | Catálogo `dev` | Catálogo `prod` |
| --- | --- | --- |
| `grp_web_engineers` | Escrita em `web_*` | `SELECT` em `web_*` |
| `grp_sales_engineers` | Escrita em `sales_*` e `customer_*` | `SELECT` em `sales_*` e `customer_*` |
| `grp_marketing_analysts` | — | `SELECT` em `*_gold` e `marketing_gold` |
| `grp_platform_admins` | `MANAGE` | `MANAGE`, owner do catálogo |
| `sp_ecommerce_prod_deploy` | — | Escrita em todos os schemas (único que escreve) |

### Script de criação (excerto)

```sql
CREATE CATALOG IF NOT EXISTS prod
  COMMENT 'DormeBem Online — produção, todos os domínios';

CREATE SCHEMA IF NOT EXISTS prod.web_bronze
  MANAGED LOCATION 'abfss://datalake@<storage>.dfs.core.windows.net/prod/bronze/web/';
CREATE SCHEMA IF NOT EXISTS prod.web_silver
  MANAGED LOCATION 'abfss://datalake@<storage>.dfs.core.windows.net/prod/silver/web/';
CREATE SCHEMA IF NOT EXISTS prod.web_gold
  MANAGED LOCATION 'abfss://datalake@<storage>.dfs.core.windows.net/prod/gold/web/';
-- repetir para sales_*, customer_* e marketing_gold

CREATE VOLUME IF NOT EXISTS prod.web_bronze.landing;
CREATE VOLUME IF NOT EXISTS prod.web_bronze.checkpoints;

GRANT USE CATALOG ON CATALOG prod TO `grp_marketing_analysts`;
GRANT USE SCHEMA, SELECT ON SCHEMA prod.marketing_gold TO `grp_marketing_analysts`;
GRANT USE SCHEMA, SELECT ON SCHEMA prod.web_silver TO `grp_web_engineers`;
```

No workspace de produção, o catálogo `prod` fica com binding só para `ws_dormebem_prod`, e `dev` só para `ws_dormebem_dev`. Se só tiver um workspace para o exercício, crie os dois catálogos nele e explique no vídeo que, num cenário real, o binding faria a separação.

## Comparação A vs B

Escolha A quando o risco de um erro chegar à produção é a maior preocupação; escolha B quando há vários domínios e a separação de ambientes já é feita por workspace.

| Critério | A — `{domain}_{environment}` | B — `{environment}` + `{domain}_{layer}` |
| --- | --- | --- |
| Onde fica a fronteira de ambiente | No catálogo | No workspace (binding do catálogo) |
| Número de catálogos | Domínios × ambientes (3 aqui; 15 com 5 domínios) | Um por ambiente (2 aqui, sempre 2–3) |
| Granularidade das permissões | Catálogo (simples) | Schema (mais grants, mais fino) |
| Nome completo de uma tabela | `sales_prod.curated.orders` | `prod.sales_gold.daily_revenue` |
| Parâmetro no código | `catalog` | `env` |
| Adicionar um domínio novo | Criar 3 catálogos | Criar 3 schemas em cada catálogo |
| Visão única da produção | Espalhada por vários `*_prod` | Tudo em `prod` |
| Risco típico | Proliferar catálogos sem gestão | Grant errado num schema expõe outro domínio |
| Bom para | Um domínio crítico, regulação forte, poucas equipas | Plataforma com vários domínios, equipa de plataforma central |

Nos dois casos o Learn recomenda as mesmas práticas de implementação: documentar as convenções num sítio central, restringir `CREATE SCHEMA` e `CREATE TABLE` a quem deve criar, validar nomes no CI/CD e rever periodicamente os objetos existentes pelo `information_schema`.

## Roteiro de implementação

A mesma sequência serve para os dois projetos; só mudam os nomes. Cada passo pode ser um episódio de vídeo.

1. **Documento de convenções**: publique a tabela de nomenclatura deste documento no README do repositório.
2. **Grupos e service principals**: crie `grp_*` e `sp_*` na conta e adicione-os ao workspace.
3. **Storage**: storage credential, external locations e pastas `/{env}/{layer}/{domain}/` no ADLS Gen2.
4. **Marketplace**: instale as listagens com os nomes `src_mkt_retail_customers` e `src_mkt_ecommerce_clickstream`; dê `USE CATALOG` + `SELECT` só aos service principals e engenheiros.
5. **Compute**: crie as cluster policies, o instance pool, os clusters all-purpose e os SQL warehouses Pro, com tags.
6. **Catálogos, schemas e volumes**: corra `notebook_setup_catalog_objects` com o parâmetro de ambiente.
7. **Permissões**: corra `notebook_setup_grants`; teste com um utilizador analista que não consegue escrever.
8. **Repositório e bundle**: crie `repo_sales_project` / `repo_ecommerce_project` no Git, ligue-o como Git folder e configure os targets do `databricks.yml`.
9. **Bronze / raw**: simulação de landing + Auto Loader. Corra 1 dia, confirme, depois os restantes 6 (A) ou os 16 dias (B).
10. **Silver / staging**: transformações, expectativas de qualidade, SCD tipo 2 (A) ou sessões (B).
11. **Gold / curated**: agregações que respondem às perguntas de negócio.
12. **Orquestração**: `job_e2e_*` com dependências, job clusters e notificações de falha.
13. **Consumo**: queries, dashboards, alertas e Genie space no warehouse de produção.
14. **Promoção**: `databricks bundle deploy -t test` e depois `-t prod`, sempre pelo service principal.
15. **Validação de nomes**: `notebook_quality_naming_validation` como passo do CI/CD (query abaixo).

### Validação automática de nomenclatura

Query para o passo 15. Devolve objetos que violam as regras; o CI/CD falha se devolver linhas.

```sql
SELECT * FROM (
SELECT table_catalog, table_schema, table_name, table_type,
  CASE
    WHEN table_type = 'VIEW' AND table_name NOT LIKE 'vw\\_%'  THEN 'view sem prefixo vw_'
    WHEN table_type <> 'VIEW' AND table_name LIKE 'vw\\_%'     THEN 'tabela com prefixo de view'
    WHEN table_name RLIKE '(_v[0-9]+|_final|_updated|_new|_old)$' THEN 'versão no nome'
    WHEN table_name RLIKE '_(dev|test|prod)$'                 THEN 'ambiente no nome da tabela'
    WHEN table_name LIKE 'tmp\\_%'                            THEN 'tabela tmp_ esquecida'
  END AS violation
FROM system.information_schema.tables
WHERE table_catalog IN ('sales_dev','sales_test','sales_prod')   -- Projeto B: ('dev','prod')
  AND table_schema <> 'information_schema'
) WHERE violation IS NOT NULL;
```

Para schemas no Projeto B, verifique também que todos seguem `{domain}_{layer}`:

```sql
SELECT catalog_name, schema_name
FROM system.information_schema.schemata
WHERE catalog_name IN ('dev','prod')
  AND schema_name <> 'information_schema'
  AND NOT schema_name RLIKE '^(web|sales|customer|marketing)_(bronze|silver|gold)$';
```

### Sources

- [Apply naming conventions — Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/2-apply-naming-conventions?pivots=text)
- [Simulated Retail Customer Data — Databricks Marketplace](https://marketplace.databricks.com/details/a82597f6-5ada-49d5-b934-d6c9dece16a1/Databricks_Simulated-Retail-Customer-Data)
- [Simulated E-Commerce Clickstream Data — Databricks Marketplace](https://marketplace.databricks.com/details/ca87a8be-9a17-4469-acee-d992d6cd16ca/Databricks_Simulated-ECommerce-Clickstream-Data)
