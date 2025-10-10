- Allez à **“Dev Tools → Console”**. :'objectif est d'écrire des requêtes prêtes pour notre index `news`.
(collez chaque bloc tel quel dans **Kibana → Dev Tools → Console**, et exécute avec ▶)



# 0) Sanity checks

```http
GET /
GET _cat/indices?v
GET news/_count
GET news/_mapping
```

* `/_mapping` te montre les types de champs (ex: `text`, `keyword`, `date`).
  Pour des **filtres exacts** et des **aggrégations**, privilégie les champs `keyword` (ex: `category`, `authors`).
  Pour le **plein texte**, utilise les champs `text` (ex: `headline`, `short_description`).

---

# 1) Lire des documents

## Derniers documents (par date décroissante)

```http
GET news/_search
{
  "size": 10,
  "sort": [{ "date": "desc" }]
}
```

## Filtre exact (catégorie)

```http
GET news/_search
{
  "size": 10,
  "query": { "term": { "category": "POLITICS" } },
  "sort": [{ "date": "desc" }]
}
```

## Plusieurs catégories + plage de dates

```http
GET news/_search
{
  "size": 20,
  "query": {
    "bool": {
      "filter": [
        { "terms": { "category": ["POLITICS", "WORLD NEWS"] } },
        { "range": { "date": { "gte": "2018-05-20", "lt": "2018-06-01" } } }
      ]
    }
  },
  "sort": [{ "date": "desc" }]
}
```

## Plein texte dans titre + description (avec boost du titre)

```http
GET news/_search
{
  "size": 20,
  "query": {
    "multi_match": {
      "query": "Trump",
      "fields": ["headline^2", "short_description"]
    }
  },
  "highlight": {
    "fields": { "headline": {}, "short_description": {} }
  }
}
```

## Combiner plein texte + filtres exacts

```http
GET news/_search
{
  "size": 20,
  "query": {
    "bool": {
      "must": {
        "multi_match": {
          "query": "North Korea",
          "fields": ["headline", "short_description"]
        }
      },
      "filter": [
        { "term": { "category": "WORLD NEWS" } },
        { "range": { "date": { "gte": "2018-05-20", "lt": "2018-06-01" } } }
      ]
    }
  },
  "sort": [{ "date": "desc" }]
}
```

---

# 2) Tri & pagination

## Pagination simple (jusqu’à ~10k docs)

```http
GET news/_search
{
  "from": 0,
  "size": 10,
  "sort": [{ "date": "desc" }]
}
```

## Pagination “grande échelle” (recommandée) avec `search_after`

1ʳᵉ page :

```http
GET news/_search
{
  "size": 10,
  "sort": [{ "date": "desc" }, { "_id": "asc" }]
}
```

Prends les 2 dernières valeurs de `sort` du dernier hit, puis :

```http
GET news/_search
{
  "size": 10,
  "sort": [{ "date": "desc" }, { "_id": "asc" }],
  "search_after": ["2018-05-26T00:00:00Z", "SOME_ID_HERE"]
}
```

---

# 3) Aggrégations (stats/group by)

## Compter par catégorie (Top 10)

```http
GET news/_search
{
  "size": 0,
  "aggs": {
    "by_category": { "terms": { "field": "category", "size": 10 } }
  }
}
```

## Histogramme par jour (POLITICS)

```http
GET news/_search
{
  "size": 0,
  "query": { "term": { "category": "POLITICS" } },
  "aggs": {
    "per_day": {
      "date_histogram": { "field": "date", "calendar_interval": "day" }
    }
  }
}
```

## Top auteurs sur une période

```http
GET news/_search
{
  "size": 0,
  "query": { "range": { "date": { "gte": "2018-05-20", "lt": "2018-05-31" } } },
  "aggs": {
    "by_author": {
      "terms": { "field": "authors", "size": 10 }
    }
  }
}
```

## Top N titres par catégorie (ex: 2 titres)

```http
GET news/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category", "size": 5 },
      "aggs": {
        "latest_titles": {
          "top_hits": {
            "size": 2,
            "_source": ["headline","date","link"],
            "sort": [{ "date": "desc" }]
          }
        }
      }
    }
  }
}
```

---

# 4) ES|QL (option lisible “style SQL”)

Tu peux aussi exécuter ces requêtes dans **Dev Tools** (onglet **ES|QL**) :

## Compter par catégorie

```sql
FROM news
| STATS count() BY category
| ORDER BY count DESC
| LIMIT 10
```

## Filtrer par date + top auteurs

```sql
FROM news
| WHERE date BETWEEN "2018-05-20" AND "2018-05-31"
| STATS count() BY authors
| ORDER BY count DESC
| LIMIT 10
```

## Plein texte avec `MATCH`

```sql
FROM news
| WHERE MATCH(headline, "Trump") OR MATCH(short_description, "Trump")
| KEEP headline, category, authors, date, link
| ORDER BY date DESC
| LIMIT 20
```

## Histogramme (agrégé par jour)

```sql
FROM news
| WHERE category == "POLITICS"
| EVAL day = DATE_TRUNC(1 days, date)
| STATS count() BY day
| ORDER BY day
```

---

# 5) Petits tips

* Erreur `field [X] of type [text] ... not aggregatable` → utilise la variante **`keyword`** du champ si elle existe (ex: `authors.keyword`) OU veille à ingérer `authors` en `keyword` (ce que tu as probablement déjà).
* Dates : assure-toi que `date` est bien mappé en `date`. Si tu as des formats non ISO, ajoute un pipeline d’ingest ou remappe.
* Pour **sauver** une requête Discover, clique **Save** → tu pourras l’utiliser dans un **Dashboard**.



# Exercice :

- Trouvez le *Top 5 catégories entre 2018-05-20 et 2018-05-31 puis listez des 10 titres les plus récents de POLITICS*
