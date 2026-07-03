# Rapport de verification Budget Rill / APIs

Date: 2026-07-03

## Objectif

Verifier la coherence entre:

- le snapshot de donnees `data_snapshot/data.csv`;
- la metrics view `meterics/bigquery_budget_metrics_2025.yaml`;
- les APIs Rill dans `rill_apis/`;
- les chemins Rill live attendus `apis/` et `metrics/`;
- le catalogue de skills dans `skills/`.

Le focus etait volontairement mis sur la qualite analytique de la couche metrics/API, pas sur la finalisation de `budget_reports_agent.json`.

## Donnees inspectees

Le fichier principal `data_snapshot/data.csv` est un TSV, pas un CSV virgule.

Echantillon inspecte:

- 2 000 lignes;
- 43 colonnes;
- periode: annee 2025 uniquement;
- dimensions fortes: `country`, `region`, `brand`, `subbrand`, `product`, `media`, `submedia`, `digital_lever`, `status`, `workflow_process`, `flight_status`, `flight_progress`;
- dimensions tres incompletes: `retail_media` vide a 98.4%, `on_site_off_site` vide a 82.0%.

Constats metriques sur l'echantillon:

| Mesure | Total | Lignes non nulles/non zero |
| --- | ---: | ---: |
| `media_net_eur` | 47 068 790.49 | 951 |
| `committed_total_eur` | 93 842 012.16 | 1 265 |
| `gross_investment_eur` | 47 386 628.59 | 995 |
| `technical_costs_eur` | 77 032.66 | 62 |
| `non_media_costs_eur` | 317 838.10 | 185 |
| `media_agency_fees_eur` | 81 327.11 | 47 |
| `media_taxes_eur` | 38 919.48 | 61 |
| `proposal_amount` | 0.00 | 0 |
| `purchase_order_amount` | 0.00 | 0 |
| `rate_card_eur` | 0.00 | 0 |

Conclusion: `proposal_amount`, `purchase_order_amount` et `rate_card_eur` existent dans le modele, mais ne portent pas d'information dans ce snapshot. Les garder est coherent pour compatibilite, mais les analyses par defaut ne doivent pas s'appuyer dessus.

## Problemes trouves

1. Les APIs `filter_*` utilisaient `committed_eur`, colonne absente du dataset. La colonne correcte est `committed_total_eur`.

2. `rill_apis/budget_top_countries.yaml` contenait un ancien bloc SQL residuel apres le `LIMIT`, avec un `FROM ranked` inexistant dans la requete finale. L'API etait donc invalide.

3. Les APIs analytiques initiales ne couvraient pas les dimensions pourtant presentes et utiles dans les donnees: `subbrand`, `product`, `submedia`.

4. Il manquait une API generique pour:
   - comparer deux metriques ou deux valeurs de dimension;
   - construire un tableau croise dynamique en format exploitable.

5. Le catalogue `skills/skill_index.yaml` restait centre sur `brand`, `country`, `media`, `status`, `workflow_process`, alors que la metrics view expose plus de dimensions utiles.

## Corrections realisees

### Metrics view

Fichier modifie:

- `meterics/bigquery_budget_metrics_2025.yaml`
- miroir ajoute pour Rill live: `metrics/bigquery_budget_metrics_2025.yaml`

Mesures ajoutees:

- `budget_line_count`;
- `active_media_net_line_count`;
- `active_committed_line_count`;
- `committed_vs_media_net_ratio`;
- `gross_investment_vs_media_net_ratio`;
- `non_media_cost_rate_eur`;
- `avg_media_net_per_source_row_eur`.

But: donner a l'agent des indicateurs plus robustes que les champs PO/proposal/rate card, qui sont tous a zero dans l'echantillon.

### APIs corrigees

Fichiers corriges:

- `rill_apis/filter_brands.yaml`;
- `rill_apis/filter_countries.yaml`;
- `rill_apis/filter_media.yaml`;
- `rill_apis/filter_statuses.yaml`;
- `rill_apis/filter_workflows.yaml`;
- `rill_apis/budget_top_countries.yaml`.

Les memes fichiers sont aussi presents dans `apis/`, car l'instance Rill port-forwardee annonce ses ressources sous `/apis/*.yaml`.

Changements:

- remplacement de `committed_eur` par `committed_total_eur`;
- suppression du bloc SQL obsolete dans `budget_top_countries`.

### APIs enrichies

Les filtres `subbrand`, `product`, `submedia` ont ete ajoutes aux APIs existantes:

- `budget_dashboard_data_filtered`;
- `budget_monthly_trend`;
- `budget_top_brands`;
- `budget_top_countries`;
- `budget_top_media`;
- `budget_brand_financial_detail`.

### Nouvelles APIs de listes

Ajoute:

- `rill_apis/filter_subbrands.yaml`;
- `rill_apis/filter_products.yaml`;
- `rill_apis/filter_submedia.yaml`.

Miroirs Rill:

- `apis/filter_subbrands.yaml`;
- `apis/filter_products.yaml`;
- `apis/filter_submedia.yaml`.

Ces APIs renvoient les valeurs canoniques, les variantes normalisees, le volume de lignes et les agregats budget principaux.

### Nouveaux resolvers

Ajoute:

- `rill_apis/resolve_subbrand.yaml`;
- `rill_apis/resolve_product.yaml`;
- `rill_apis/resolve_submedia.yaml`.

Miroirs Rill:

- `apis/resolve_subbrand.yaml`;
- `apis/resolve_product.yaml`;
- `apis/resolve_submedia.yaml`.

Ils reprennent la logique existante exact/prefix/contains/fuzzy pour eviter que l'agent utilise directement des valeurs utilisateur approximatives.

### Nouvelles APIs analytiques

Ajoute:

- `rill_apis/budget_compare_metrics.yaml`;
- `rill_apis/budget_pivot_table.yaml`.

Miroirs Rill:

- `apis/budget_compare_metrics.yaml`;
- `apis/budget_pivot_table.yaml`.

`budget_compare_metrics` permet:

- comparaison de deux metriques sur un meme scope filtre;
- comparaison par dimension (`country`, `brand`, `media`, `subbrand`, etc.);
- sortie long-form: `segment`, `metric`, `value_eur`, `row_count`.

`budget_pivot_table` permet:

- lignes dynamiques via `row_dimension`;
- colonnes dynamiques via `column_dimension`;
- metrique dynamique via `metric`;
- sortie long-form pivotable: `row_value`, `column_value`, `value_eur`, `row_count`.

### Skills / catalogue

Ajoute:

- `skills/filter_subbrands.yaml`;
- `skills/filter_products.yaml`;
- `skills/filter_submedia.yaml`;
- `skills/resolve_subbrand.yaml`;
- `skills/resolve_product.yaml`;
- `skills/resolve_submedia.yaml`;
- `skills/budget_compare_metrics.yaml`;
- `skills/budget_pivot_table.yaml`.

Mis a jour:

- `skills/skill_index.yaml`.

Le catalogue connait maintenant les dimensions:

- `brand`;
- `country`;
- `region`;
- `media`;
- `subbrand`;
- `product`;
- `submedia`;
- `digital_lever`;
- `status`;
- `workflow_process`;
- `flight_status`;
- `flight_progress`.

## Validation effectuee

1. Parsing YAML:

- 50 fichiers YAML valides dans `rill_apis/`, `skills/`, `meterics/`.

2. Rill MCP:

- metrics view exposee: `bigquery_budget_metrics_2025`;
- statut projet: `parse_errors: []`.

3. Validation directe sur le port-forward Rill:

- base API testee: `http://localhost:9009/v1/instances/default/api`;
- endpoint MCP teste: `http://localhost:9009/mcp`;
- MCP initialize OK, serveur Rill: `v0.87.7`;
- `filter_brands?limit=1` repond en HTTP 200 avec `total_committed_eur`;
- `filter_subbrands?limit=1` repond en HTTP 200;
- `budget_top_countries?limit=2` repond en HTTP 200;
- `budget_compare_metrics?brand=No%20Brand&metric_a=media_net&metric_b=committed` repond en HTTP 200;
- `budget_pivot_table?row_dimension=brand&column_dimension=country&metric=media_net&limit=2` repond en HTTP 200.

Conclusion live: le port-forward actif sur `9009` sert bien l'instance Rill Kubernetes mise a jour. La configuration MCP suivante est coherente avec l'etat teste:

```json
{
  "mcpServers": {
    "rill": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "http://localhost:9009/mcp"
      ]
    }
  }
}
```

4. Validation donnees:

- lecture du snapshot en TSV;
- verification des totaux des mesures principales;
- verification que les dimensions ajoutees existent bien dans la metrics view et dans le snapshot;
- verification d'un exemple pivot brand x country sur `media_net_eur`.

Exemple top pivot `brand x country` par `media_net_eur`:

| Brand | Country | Media Net EUR |
| --- | --- | ---: |
| No Brand | China | 8 234 913.44 |
| No Brand | United States | 4 969 451.88 |
| L'Oreal Skincare | China | 4 114 593.75 |
| No Brand | United Kingdom | 1 699 231.04 |
| Elseve Haircare | Brazil | 877 294.40 |

## Recommandations

1. Pour les reponses par defaut, privilegier:
   - `media_net`;
   - `committed`;
   - `gross_investment`;
   - `non_media_costs`;
   - `technical_costs`;
   - `media_agency_fees`;
   - `media_taxes`.

2. Eviter de mettre en avant `proposal_amount`, `purchase_order_amount`, `proposal_to_po_gap` et `rate_card` tant que la source reste a zero sur ces colonnes.

3. Ne pas utiliser `retail_media` et `on_site_off_site` comme dimensions principales pour l'instant: elles sont trop vides dans l'echantillon.

4. Pour les tableaux croises dynamiques, garder la sortie long-form cote API et laisser le frontend/agent pivoter l'affichage. C'est plus stable que de generer dynamiquement des colonnes SQL.

5. Prochaine etape cote agent: mettre a jour les composants Langflow pour ajouter les nouvelles dimensions/endpoints dans les mappings internes. Je ne l'ai pas fait ici volontairement, car la demande etait de ne pas se concentrer sur l'agent incomplet.
