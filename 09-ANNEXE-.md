# Partie 1 — Import

* **Kibana UI** : *ML → Data Visualizer → File* → index `news`, time field `date`, `category/authors` en keyword → Import.
* **Terminal** : template `news*` (types), `_bulk` NDJSON vers `news`, `/_refresh`, vérifier avec `news/_count`.

# Sanity checks (Dev Tools)

* `GET /` — cluster OK
* `GET _cat/indices?v` — index `news` présent
* `GET news/_count` — volume
* `GET news/_mapping` — types (`text/keyword/date`)

# KQL (Discover)

* Catégorie : `category : "POLITICS"`
* Période : `date >= "2018-05-20" and date < "2018-06-01"`
* Plein texte : `headline : "Trump" or short_description : "Trump"`
* Combinée : `category : ("WORLD NEWS" or "POLITICS") and authors : "Ron Dicker" and date >= "2018-05-25"`

# ES|QL (Discover/Dev Tools)

* Comptage par catégorie :

  ```sql
  FROM news | STATS count() BY category | ORDER BY count DESC | LIMIT 10
  ```
* Top auteurs (période) :

  ```sql
  FROM news | WHERE date BETWEEN "2018-05-20" AND "2018-05-31"
  | STATS count() BY authors | ORDER BY count DESC | LIMIT 10
  ```
* Plein texte :

  ```sql
  FROM news | WHERE MATCH(headline,"Trump") OR MATCH(short_description,"Trump") | LIMIT 20
  ```
* Histogramme/jour (POLITICS) :

  ```sql
  FROM news | WHERE category == "POLITICS"
  | EVAL day = DATE_TRUNC(1 days, date) | STATS count() BY day | ORDER BY day
  ```

# DSL JSON (Dev Tools)

* Plein texte multi-champs (boost titre) :

  ```http
  GET news/_search
  { "query": { "multi_match": { "query": "Trump", "fields": ["headline^2","short_description"] } }, "size": 20 }
  ```
* Filtre exact + dates + tri :

  ```http
  GET news/_search
  { "query": { "bool": { "must":[{"term":{"category":"POLITICS"}}],
               "filter":[{"range":{"date":{"gte":"2018-05-20","lt":"2018-06-01"}}}] } },
    "sort":[{"date":"desc"}] }
  ```
* Aggregation Top catégories :

  ```http
  GET news/_search
  { "size":0, "aggs":{ "by_category":{ "terms":{"field":"category","size":10} } } }
  ```
* Histogramme POLITICS par jour :

  ```http
  GET news/_search
  { "size":0, "query":{"term":{"category":"POLITICS"}},
    "aggs":{"per_day":{"date_histogram":{"field":"date","calendar_interval":"day"}}} }
  ```
* Highlight :

  ```http
  GET news/_search
  { "query":{"multi_match":{"query":"North Korea","fields":["headline","short_description"]}},
    "highlight":{"fields":{"headline":{},"short_description":{}}} }
  ```

# Tri & pagination

* Pagination simple :

  ```http
  GET news/_search { "from":0, "size":10, "sort":[{"date":"desc"}] }
  ```
* `search_after` (grands volumes) :

  ```http
  GET news/_search
  { "size":10, "sort":[{"date":"desc"},{"_id":"asc"}], "search_after":["<last_date>","<last_id>"] }
  ```

# cURL (exemple)

```bash
curl -u elastic:$ELASTIC_PASSWORD 'http://localhost:9200/news/_search?pretty' \
 -H 'Content-Type: application/json' -d '{
  "query": { "term": { "category": "POLITICS" } },
  "size": 5, "sort": [{ "date": "desc" }]
}'
```

# Rappels mapping

* **text** : `headline`, `short_description` → plein texte
* **keyword** : `category`, `authors`, `link` → filtres/aggs
* **date** : `date` → range, histogrammes
