
# 1) Deux familles de bases de données

## 1.1 Modèle relationnel (SQL) vs Elasticsearch (documents)

| Aspect               | Base relationnelle (SQL)                                    | Elasticsearch (documents)                                                                                          |                                          |
| -------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | ---------------------------------------- |
| Modèle de données    | Tables, lignes, colonnes. Schéma strict, normalisation.     | Documents JSON dénormalisés. Schéma souple mais mappé (mapping).                                                   |                                          |
| Identité & relations | Clés primaires/étrangères, JOINS, contraintes d’intégrité.  | Pas de FK/JOINS classiques. Relations rarement utilisées (join parent/child coûteux). Dénormalisation recommandée. |                                          |
| Transactions         | ACID multi-lignes/multi-tables.                             | ACID par document uniquement. Pas de transactions multi-documents.                                                 |                                          |
| Requêtage            | SQL (SELECT/INSERT/UPDATE/DELETE), JOINS, GROUP BY.         | Query DSL (JSON), KQL (barre de recherche Kibana), ES                                                              | QL (style SQL), agrégations distribuées. |
| Texte intégral       | Possible (FTS Postgres/MySQL) mais secondaire.              | Fonction première : index inversé, analyzers, fuzzy, scoring, highlight.                                           |                                          |
| Cohérence temporelle | Lectures immédiatement cohérentes après commit.             | Near-real-time (visible après refresh ~1s).                                                                        |                                          |
| Scalabilité          | Verticale par défaut, sharding possible mais plus complexe. | Horizontale par construction (shards primaires + réplicas répartis sur plusieurs nœuds).                           |                                          |
| Cas d’usage forts    | Systèmes métiers, finance, ERP/CRM, forte intégrité.        | Recherche type « Google », logs/metrics, filtres à facettes, analytics quasi temps réel.                           |                                          |
| Coût des écritures   | Optimisé pour transactions fréquentes et cohérentes.        | Écritures rapides mais conçues pour lecture/cherche massive; reindex nécessaire si structure évolue.               |                                          |

Conclusion pratique : SQL pour la cohérence relationnelle et les transactions. Elasticsearch pour la recherche textuelle, le filtrage ultra-rapide, les agrégations et la scalabilité.

---

# 2) Ton dataset « news » : modélisation

## 2.1 En SQL (ex. PostgreSQL)

Tables recommandées (normalisation) :

* `categories(id, name UNIQUE)`
* `authors(id, name UNIQUE)`
* `articles(id, category_id FK, headline, short_description, link UNIQUE, date DATE)`
* `article_authors(article_id FK, author_id FK)` pour N-N

Avantages : intégrité, JOINS propres. Inconvénient : recherche texte intégral moins riche, scaling recherche/facettes moins fluide.

## 2.2 En Elasticsearch (index `news`)

Un document = un article :

```json
{
  "category": "POLITICS",              // keyword
  "authors": ["Ron Dicker"],           // keyword (liste)
  "headline": "Titre ...",             // text
  "short_description": "Résumé ...",   // text
  "link": "https://...",               // keyword
  "date": "2018-05-25"                 // date
}
```

Mapping minimal conseillé :

```json
PUT _index_template/news-template
{
  "index_patterns": ["news*"],
  "template": {
    "mappings": {
      "properties": {
        "date":    { "type": "date", "format": "yyyy-MM-dd||strict_date_optional_time" },
        "category":{ "type": "keyword" },
        "authors": { "type": "keyword" },
        "headline":{ "type": "text" },
        "short_description": { "type": "text" },
        "link":   { "type": "keyword" }
}}}}
```

Avantages : recherche/filtrage/aggrégations rapides et scalables. Inconvénient : pas de JOINS, dénormalisation, near-real-time.

---

# 3) Les langages et « DSL/DQL/KQL/ES|QL »

| Terme     | Signification                                                           | Où l’utiliser                               | Commentaires                                                |     |                |       |       |          |         |
| --------- | ----------------------------------------------------------------------- | ------------------------------------------- | ----------------------------------------------------------- | --- | -------------- | ----- | ----- | -------- | ------- |
| Query DSL | Langage JSON natif d’Elasticsearch (`_search`, `bool`, `match`, `aggs`) | Kibana → Dev Tools → Console, ou via API    | Couvre 100% des fonctionnalités de requête.                 |     |                |       |       |          |         |
| KQL       | Kibana Query Language (syntaxe simple de filtres)                       | Kibana → Discover (barre de recherche)      | Ex. `category:"POLITICS" AND date >= "2018-05-24"`.         |     |                |       |       |          |         |
| ES        | QL                                                                      | Langage de requêtes « style SQL » d’Elastic | Kibana (Discover/Dev Tools, onglet ES                       | QL) | Lisible, `FROM | WHERE | STATS | ORDER BY | LIMIT`. |
| DQL       | En SQL traditionnel : Data Query Language (SELECT)                      | —                                           | Ce n’est pas un langage Elastic. Ne pas confondre avec DSL. |     |                |       |       |          |         |

---

# 4) Text vs Keyword vs Date : choisir le bon type

| Type      | Usage typique                        | Opérateurs recommandés                                     |                 |
| --------- | ------------------------------------ | ---------------------------------------------------------- | --------------- |
| `text`    | Plein texte, analyse linguistique    | `match`, `multi_match`, `match_phrase`, highlight          |                 |
| `keyword` | Filtre exact, agrégations, tri exact | `term`, `terms`, `prefix`, `wildcard`, agrégations `terms` |                 |
| `date`    | Filtres temporels, histogrammes      | `range`, `date_histogram`, ES                              | QL `DATE_TRUNC` |

Symptôme courant : « field of type [text] is not aggregatable » → utiliser la variante `.keyword` ou mapper directement en `keyword`.

---

# 5) Shards, réplicas et haute disponibilité (synthèse)

* Un index est découpé en **shards primaires**. Chaque primaire peut avoir **n réplicas**.
* Répartition des shards sur plusieurs nœuds = parallélisme des lectures/agrégations et tolérance aux pannes.
* Dev en single-node : 1 primaire, 0 replica suffit. Prod : réplicas ≥ 1 pour haute dispo.

---

# 6) Ingestion des données (rappel opérationnel)

* Fichier NDJSON (1 JSON/ligne) → endpoint `_bulk` avec ligne d’action + ligne document.
* UI Kibana « Upload a file » accepte NDJSON et propose un mapping initial.
* Pour gros volumes : `_bulk` par paquets, `refresh=false` pendant l’ingestion, `/_refresh` à la fin.

Exemple `_bulk` préparé depuis `news.ndjson` :

```bash
awk '{print "{\"index\":{\"_index\":\"news\"}}"; print}' data/news.ndjson > data/bulk.ndjson
```

Envoi (paquets, puis refresh) : voir ton guide précédent.

---

# 7) Exemples comparés : mêmes besoins, requêtes différentes

## 7.1 Derniers articles par date

* SQL :

```sql
SELECT * FROM articles ORDER BY date DESC LIMIT 10;
```

* Elasticsearch (DSL) :

```http
GET news/_search
{
  "size": 10,
  "sort": [{ "date": "desc" }]
}
```

* KQL (Discover) :

```
*  (et tri par date desc dans l’UI)
```

* ES|QL :

```sql
FROM news
| ORDER BY date DESC
| LIMIT 10
```

## 7.2 Filtre catégorie + plage de dates

* SQL :

```sql
SELECT a.*
FROM articles a
JOIN categories c ON c.id = a.category_id
WHERE c.name = 'POLITICS'
  AND a.date >= '2018-05-24' AND a.date < '2018-05-27'
ORDER BY a.date DESC
LIMIT 20;
```

* Elasticsearch (DSL) :

```http
GET news/_search
{
  "size": 20,
  "query": {
    "bool": {
      "filter": [
        { "term":  { "category": "POLITICS" } },
        { "range": { "date": { "gte": "2018-05-24", "lt": "2018-05-27" } } }
      ]
    }
  },
  "sort": [{ "date": "desc" }]
}
```

* KQL :

```
category : "POLITICS" and date >= "2018-05-24" and date < "2018-05-27"
```

* ES|QL :

```sql
FROM news
| WHERE category == "POLITICS" AND date BETWEEN "2018-05-24" AND "2018-05-27"
| ORDER BY date DESC
| LIMIT 20
```

## 7.3 Plein texte (titre + description) avec mise en évidence

* SQL (Postgres FTS, simplifié) :

```sql
SELECT a.*
FROM articles a
WHERE to_tsvector('english', a.headline || ' ' || a.short_description)
      @@ plainto_tsquery('english', 'Trump')
ORDER BY a.date DESC
LIMIT 20;
```

* Elasticsearch (DSL) :

```http
GET news/_search
{
  "size": 20,
  "query": {
    "multi_match": {
      "query": "Trump",
      "fields": ["headline^2","short_description"]
    }
  },
  "highlight": { "fields": { "headline": {}, "short_description": {} } }
}
```

* KQL :

```
headline : "Trump" or short_description : "Trump"
```

* ES|QL :

```sql
FROM news
| WHERE MATCH(headline, "Trump") OR MATCH(short_description, "Trump")
| KEEP headline, category, authors, date, link
| ORDER BY date DESC
| LIMIT 20
```

## 7.4 Group by / agrégations

* SQL :

```sql
SELECT c.name, COUNT(*) AS n
FROM articles a
JOIN categories c ON c.id = a.category_id
GROUP BY c.name
ORDER BY n DESC
LIMIT 10;
```

* Elasticsearch (DSL) :

```http
GET news/_search
{
  "size": 0,
  "aggs": {
    "by_category": { "terms": { "field": "category", "size": 10 } }
  }
}
```

* ES|QL :

```sql
FROM news
| STATS count() BY category
| ORDER BY count DESC
| LIMIT 10
```

---

# 8) Quand choisir quoi ? (guideline simple)

| Besoin dominant                                         | Choix naturel                                |
| ------------------------------------------------------- | -------------------------------------------- |
| Intégrité forte, contraintes, transactions multi-tables | Base relationnelle (SQL)                     |
| Recherche texte riche, filtres à facettes, autosuggest  | Elasticsearch                                |
| Dashboards quasi temps réel sur gros volumes            | Elasticsearch + Kibana                       |
| Reporting comptable/juridique précis                    | SQL (ou data-warehouse/BI)                   |
| Système métier avec workflows                           | SQL au cœur ; ES en miroir pour la recherche |

Patron gagnant fréquent : SQL comme « source de vérité », Elasticsearch comme « moteur de recherche/analytics » alimenté par ETL/ingest.

---

# 9) Pièges fréquents et bonnes pratiques

* Agrégations sur `text` → erreur « not aggregatable » : utiliser `keyword` ou `field.keyword`.
* Dates : définir un `format` compatible et envoyer des dates cohérentes.
* Trop de petites mises à jour document par document : regrouper par `_bulk` quand possible.
* Changer la structure des champs après ingestion : nécessite souvent un reindex (nouvel index + re-ingestion).
* Indexation de champs inutiles en `text` : mapper en `keyword` si on fait surtout des filtres/aggs.
* Production : prévoir réplicas, monitoring (Kibana Stack Monitoring), snapshots (sauvegardes).

---

# 10) Ce qu’il faut retenir en une page

* SQL et Elasticsearch ne se remplacent pas : ils se complètent.
* Elasticsearch excelle pour la recherche texte, le filtrage, les agrégations et la scalabilité horizontale.
* Ton dataset « news » s’exprime naturellement en documents : `category`/`authors` en `keyword`, `headline`/`short_description` en `text`, `date` en `date`.
* Trois façons de requêter ES : Query DSL (complet), KQL (simple), ES|QL (lisible).
* La performance et l’UX de Kibana viennent du sharding, de l’index inversé et des agrégations distribuées.
* En dev : 1 shard primaire, 0 replica. En prod : au moins 1 replica, gestion de la capacité et du mapping en amont.

