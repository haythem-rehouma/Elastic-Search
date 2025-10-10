
# 1) Les briques de base

* **Cluster** : l’ensemble Elasticsearch.
* **Nœud (node)** : une machine (ou conteneur) qui fait tourner ES. Tu en as 1.
* **Index** : une “table” logique (ex. `news`).
* **Document** : un objet JSON (une “ligne”) stocké dans un index.
* **Mapping** : le “schéma” des champs (type `text`, `keyword`, `date`, etc.).

## Shard, c’est quoi ?

* Un **shard** = un **morceau physique** d’un index (un segment de données avec son index inversé).
* 2 types :

  * **Primary shard** : la copie “originale”.
  * **Replica shard** : une **copie** du primary (pour haute dispo + plus de lectures en parallèle).
* Pourquoi ?

  * **Scalabilité** : un index trop gros → on le découpe en plusieurs shards, répartis sur plusieurs nœuds.
  * **Perf** : plus de parallélisme en lecture.
  * **Tolérance aux pannes** : si un nœud tombe, un replica prend le relais.

> En single-node dev: 1 primary, 0 replica suffit.

# 2) Les types de champs (crucial)

* **`text`** : pour la **recherche plein texte**. ES **analyse** (tokenise, lowercase…) → `match`, `multi_match`.
* **`keyword`** : **valeur exacte**, non analysée → **filtres exacts**, **aggrégations**, **tri**.
* **`date`** : comparaisons et histogrammes.
* Souvent un champ texte a **`.keyword`** en sous-champ pour l’exact.

# 3) “Query DSL” / “KQL” / “ES|QL” — c’est quoi ?

* **Query DSL** (ce que tu envoies en JSON dans Dev Tools / via `curl`)
  → **langage officiel d’Elasticsearch** (le plus complet).
  Ex.:

  ```http
  GET news/_search
  {
    "query": { "term": { "category": "POLITICS" } },
    "sort": [{ "date": "desc" }]
  }
  ```
* **KQL** (Kibana Query Language) — barre de recherche dans **Discover**
  → simple pour filtrer/chercher:
  `category : "POLITICS" and date >= "2018-05-24"`
* **ES|QL** — style SQL lisible (onglet ES|QL)
  → super pour stats/analyses:

  ```sql
  FROM news
  | WHERE date >= "2018-05-24"
  | STATS count() BY category
  | ORDER BY count DESC
  ```

**Quand utiliser quoi ?**

* **Découverte rapide** → **KQL**
* **Tout faire / options avancées** → **Query DSL**
* **Analytique lisible / “group by”** → **ES|QL**

# 4) Les requêtes de base (concepts)

* **Plein texte** : `match`, `multi_match` (sur `text`)
* **Filtre exact** : `term`, `terms` (sur `keyword` ou `field.keyword`)
* **Plage** : `range` (dates/nombres)
* **Combinaisons** : `bool` → `must` (ET), `should` (OU), `must_not` (NON), `filter` (filtres non scorés)
* **Tri** : `sort` (souvent par `date`)
* **Pagination** :

  * simple: `from`/`size`
  * **grosses volumétries**: `search_after` (recommandé)
* **Aggregations** : “group by” côté ES (`terms`, `date_histogram`, `top_hits`, etc.)
* **Highlight** : surlignage des mots trouvés en plein texte

# 5) Ingestion (comment ES lit tes données)

* **Bulk API** en **NDJSON** (1 action + 1 doc par ligne). Rapide/robuste.
* **Index template** : pour **typer correctement** dès l’entrée (ex. `date` en `date`, `authors` en `keyword`).
* **Ingest pipeline** : transformations à l’entrée (parser dates, champs dérivés…).

# 6) Pourquoi cette stack (ELK/Elastic Stack) ?

* **Elasticsearch** : moteur de recherche/analytiques temps réel, **scalable** (shards/replicas), **API JSON**.
* **Kibana** : UI pour explorer (Discover), requêter (Dev Tools), visualiser (Lens), faire des dashboards.
* (**Beats/Agent/Logstash** optionnels) : ingestion depuis logs, systèmes, files, etc.
* Tu obtiens : **ingest → indexation → recherche/analyse → visualisation** au même endroit, très vite, et ça **scale**.

# 7) Petits pièges & réflexes

* “`field of type text is not aggregatable`” → utilise **`field.keyword`** ou mappe en `keyword`.
* Dates : bien définir le **format** (`"yyyy-MM-dd"` chez toi).
* Disque presque plein → indices **read-only** (désactive les thresholds en dev ou libère de l’espace).
* Dev en single-node → **1 primary / 0 replica** OK.
* **Ne mélange pas** plein texte (`text`) et égalité stricte (`keyword`).

# 8) Mini “carte mentale”

* **Je veux chercher des mots** → champ `text` → `match/multi_match`
* **Je veux filtrer exactement / grouper / trier** → `keyword`
* **Je veux du SQL-like lisible** → **ES|QL**
* **Je veux 100% de contrôle** → **Query DSL**
* **Je veux voir vite** → **KQL** dans Discover

