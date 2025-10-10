# 1) Deux modèles différents

## A. Relationnel (SQL)

* **Un modèle tabulaire** : **tables** (Articles, Auteurs, Catégories), **lignes**, **colonnes**.
* **Schéma strict** : types fixés (VARCHAR, DATE…), **contrôles d’intégrité** (PK, FK, UNIQUE).
* **Normalisation** : on **évite la duplication** (ex : une table `authors` + table de jointure `article_authors`).
* **Transactions ACID** : cohérence forte, multi-lignes, multi-tables.
* **Requêtes** en **SQL** (SELECT/INSERT/UPDATE/DELETE), **JOINS**, **GROUP BY**.
* **Cas d’usage** : systèmes métiers (OLTP), finance, commandes, CRM… où la **cohérence** et les **relations** priment.

## B. Documents (Elasticsearch)

* **Un modèle “document”** : chaque **document JSON** contient les champs nécessaires, souvent **dénormalisé** (on met les auteurs directement dans l’article).
* **Schéma souple** mais **mappé** (text/keyword/date…), pas de FK/PK au sens SQL.
* **Index inversé** pour **recherche plein texte** (analyzers, stemming, fuzzy, scoring BM25).
* **Near-Real-Time** : écritures visibles après un **refresh** (≈1s), pas de transactions multi-docs.
* **Requêtes** via **Query DSL** (JSON), **KQL** (Kibana), **ES|QL** (style SQL) + **aggrégations** distribuées.
* **Cas d’usage** : recherche textuelle (“Google-like”), logs/metrics, analytics temps réel, autosuggest, filtrage rapide avec facettes.

👉 **TL;DR**

* **SQL** = cohérence, relations, transactions, reporting structuré.
* **Elasticsearch** = **recherche** rapide et intelligente, **filtrage** + **agrégations** massives, **scalabilité horizontale**.



# 2) Ton dataset “news” : comment on le modélise ?

## A. En relationnel (ex : PostgreSQL)

### Schéma

```sql
CREATE TABLE categories (
  id SERIAL PRIMARY KEY,
  name TEXT UNIQUE NOT NULL
);

CREATE TABLE authors (
  id SERIAL PRIMARY KEY,
  name TEXT UNIQUE NOT NULL
);

CREATE TABLE articles (
  id SERIAL PRIMARY KEY,
  category_id INT REFERENCES categories(id),
  headline TEXT NOT NULL,
  short_description TEXT,
  link TEXT UNIQUE,
  date DATE NOT NULL
);

-- relation N-N : un article peut avoir plusieurs auteurs
CREATE TABLE article_authors (
  article_id INT REFERENCES articles(id),
  author_id INT REFERENCES authors(id),
  PRIMARY KEY (article_id, author_id)
);
```

### Requêtes typiques

* Derniers articles :

```sql
SELECT a.*
FROM articles a
ORDER BY a.date DESC
LIMIT 10;
```

* Articles “POLITICS” entre deux dates :

```sql
SELECT a.*
FROM articles a
JOIN categories c ON c.id = a.category_id
WHERE c.name = 'POLITICS'
  AND a.date >= '2018-05-24' AND a.date < '2018-05-27'
ORDER BY a.date DESC
LIMIT 20;
```

* Top catégories (group by) :

```sql
SELECT c.name, COUNT(*) AS n
FROM articles a
JOIN categories c ON c.id = a.category_id
GROUP BY c.name
ORDER BY n DESC
LIMIT 10;
```

* **Plein texte** (il faut configurer le FT de Postgres : `to_tsvector / to_tsquery`) :

```sql
SELECT a.*
FROM articles a
WHERE to_tsvector('english', a.headline || ' ' || a.short_description)
      @@ plainto_tsquery('english', 'Trump')
ORDER BY a.date DESC
LIMIT 20;
```

- **Forces** : cohérence, jointures propres, intégrité.
- **Limites** : la recherche FT est bonne mais **moins riche** que celle d’un moteur dédié (synonymes, fuzziness fine, ranker, highlight, scale-out…).



## B. En Elasticsearch (index “news”)

### Mapping minimal

```json
PUT _index_template/news-template
{
  "index_patterns": ["news*"],
  "template": {
    "mappings": {
      "properties": {
        "date":    { "type": "date", "format": "yyyy-MM-dd||strict_date_optional_time" },
        "category":{ "type": "keyword" },
        "authors": { "type": "keyword" },        // liste de noms exacts
        "headline":{ "type": "text" },           // plein texte
        "short_description": { "type": "text" }, // plein texte
        "link":   { "type": "keyword" }
      }
    }
  }
}
```

### Document type

```json
{
  "category": "POLITICS",
  "headline": "Trump Suggests North Korea Summit Could Still Happen",
  "authors": ["Roberta Rampton","Christine Kim"],
  "link": "https://www.huffingtonpost.com/entry/trump-north-korea-summit_us_5b0823c7e4b0568a880aee31",
  "short_description": "He had canceled the planned June 12 summit less than a day earlier.",
  "date": "2018-05-25"
}
```

### Requêtes typiques (Query DSL)

* Derniers articles :

```http
GET news/_search
{
  "size": 10,
  "sort": [{ "date": "desc" }]
}
```

* Filtre exact + dates :

```http
GET news/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "category": "POLITICS" } },
        { "range": { "date": { "gte": "2018-05-24", "lt": "2018-05-27" } } }
      ]
    }
  },
  "sort": [{ "date": "desc" }]
}
```

* **Plein texte + highlight** :

```http
GET news/_search
{
  "query": {
    "multi_match": {
      "query": "Trump",
      "fields": ["headline^2","short_description"]
    }
  },
  "highlight": { "fields": { "headline": {}, "short_description": {} } }
}
```

* **Group by** (aggrégations distribuées) :

```http
GET news/_search
{
  "size": 0,
  "aggs": { "by_category": { "terms": { "field": "category", "size": 10 } } }
}
```

- **Forces** : recherche textuelle au top (fuzzy, score, highlight), agrégations rapides à grande échelle, **UX Kibana** (facettes, Discover, dashboards).
- **Limites** : pas de FK/JOINS classiques, **dénormalisation** fréquente, **pas de transactions multi-docs**, cohérence **near-real-time**.



# 3) “Document” vs “Table” : que devient une table ?

* **Table `articles`** → **Index `news`** (1 **document** par article).
* **Table `categories`** → soit **champ `category`** (keyword) dans le doc, soit un index à part si tu dois gérer un référentiel riche (rare en search).
* **Table `authors` + `article_authors` (N-N)** → en ES, on met **`authors` comme tableau** de `keyword`.
  Si tu veux garder une vraie relation, ES propose **`join` parent/child**… mais c’est **coûteux**. En général en ES on **dénormalise**.

**Idée clé** :

* SQL = **normalisé** (éviter duplication) → **JOINS**.
* ES = **dénormalisé** (rapide à lire/chercher) → **pas de JOINS**, on **réplique** un peu d’info.



# 4) “DSL”, “DQL”, “KQL”, “ES|QL”… c’est quoi ?

* **DSL (Elasticsearch Query DSL)** : le **langage JSON** officiel des requêtes ES (`_search`, `bool`, `match`, `aggs`, etc.).
* **KQL (Kibana Query Language)** : mini-langage dans **Discover** pour filtrer vite (`category:"POLITICS" AND date >= "2018-05-24"`).
* **ES|QL** : langage **style SQL** d’Elastic pour requêtes lisibles (filter, STATS/GROUP BY, ORDER BY…).
* **DQL** : en SQL “classique”, ça veut dire **Data Query Language** (SELECT). **Ce n’est pas un langage Elastic.**
  (Dans Elastic, on parle plutôt de **DSL** / **KQL** / **ES|QL**.)



# 5) Shards, replicas, scaling (simple)

* Un **index** ES est découpé en **shards** (morceaux Lucene).

  * **Primary shard** = copie “maître”
  * **Replica shard** = copie de secours / lecture en parallèle
* **Pourquoi ?**

  * **Scalabilité** horizontale (répartir les shards sur plusieurs nœuds)
  * **Haute dispo** (si un nœud tombe, un replica sert les requêtes)
* En dev **single-node** : **1 primary, 0 replica** suffit.



# 6) Choisir : relationnel, Elasticsearch… ou **les deux** ?

### Prends **SQL** si tu veux surtout…

* **Transactions** (multi-lignes, multi-tables), contraintes d’intégrité.
* Beaucoup de **mises à jour** cohérentes, reporting métier précis.
* Des **JOINS** complexes, référentiels rigides.

### Prends **Elasticsearch** si tu veux surtout…

* **Recherche plein texte** (classement, fuzzy, highlight, “like Google”).
* **Filtrage** ultra rapide + **tableaux de bord** (Kibana).
* **Agrégations** massives quasi temps réel (logs, analytics, observability).
* **Scalabilité horizontale** simple.

### Le **pattern courant** (et gagnant)

* **SQL = source canonique** (vérité, transactions).
* **ES = couche de recherche/analytics** (indexation asynchrone depuis SQL).
* Tu profites **des deux** sans renoncer à la cohérence.



# 7) Les pièges classiques

* Utiliser `text` dans des **aggrégations** → « not aggregatable » → prendre la variante **`field.keyword`** ou mapper en `keyword`.
* **Dates** au mauvais format → mapper correctement (`yyyy-MM-dd` chez toi).
* Penser qu’ES remplace un SGBD **transactionnel** → non (ES a ACID **par document**, pas multi-docs).
* Oublier la **dénormalisation** : mettre les auteurs en **array** dans
