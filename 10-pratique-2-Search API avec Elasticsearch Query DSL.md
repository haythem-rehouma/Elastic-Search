- Allez à **Kibana → Dev Tools → Console** pour notre  jeu de données (index : `news`).

```http
// Vérifs rapides
GET /
GET _cat/indices?v
GET news/_count
GET news/_mapping
```

```http
// Derniers articles par date décroissante
GET news/_search
{
  "size": 10,
  "sort": [{ "date": "desc" }]
}
```

```http
// Filtre exact sur une catégorie
GET news/_search
{
  "size": 10,
  "query": { "term": { "category": "POLITICS" } },
  "sort": [{ "date": "desc" }]
}
```

```http
// Plusieurs catégories + plage de dates
GET news/_search
{
  "size": 20,
  "query": {
    "bool": {
      "filter": [
        { "terms": { "category": ["POLITICS","WORLD NEWS"] } },
        { "range": { "date": { "gte": "2018-05-20", "lt": "2018-05-27" } } }
      ]
    }
  },
  "sort": [{ "date": "desc" }]
}
```

```http
// Plein texte (boost du titre) + surlignage
GET news/_search
{
  "size": 20,
  "query": {
    "multi_match": {
      "query": "Trump",
      "fields": ["headline^2","short_description"]
    }
  },
  "highlight": {
    "fields": { "headline": {}, "short_description": {} }
  }
}
```

```http
// Plein texte + filtre exact + dates
GET news/_search
{
  "size": 20,
  "query": {
    "bool": {
      "must": {
        "multi_match": {
          "query": "North Korea",
          "fields": ["headline","short_description"]
        }
      },
      "filter": [
        { "term": { "category": "WORLD NEWS" } },
        { "range": { "date": { "gte": "2018-05-24", "lte": "2018-05-26" } } }
      ]
    }
  },
  "sort": [{ "date": "desc" }]
}
```

```http
// Pagination simple
GET news/_search
{
  "from": 0,
  "size": 10,
  "sort": [{ "date": "desc" }]
}
```

```http
// Pagination robuste avec search_after (1re page)
GET news/_search
{
  "size": 10,
  "sort": [{ "date": "desc" }, { "_id": "asc" }]
}
// Copie les 2 valeurs "sort" du dernier hit puis :
```

```http
// Pagination robuste avec search_after (page suivante)
GET news/_search
{
  "size": 10,
  "sort": [{ "date": "desc" }, { "_id": "asc" }],
  "search_after": ["2018-05-26T00:00:00Z", "ID_DERNIER_HIT"]
}
```

```http
// TOP catégories (group by)
GET news/_search
{
  "size": 0,
  "aggs": {
    "by_category": { "terms": { "field": "category", "size": 10 } }
  }
}
```

```http
// Histogramme par jour pour POLITICS
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

```http
// Top auteurs sur une période
// Si aggrégations refusées sur "authors", essaie "authors.keyword"
GET news/_search
{
  "size": 0,
  "query": { "range": { "date": { "gte": "2018-05-24", "lte": "2018-05-26" } } },
  "aggs": {
    "by_author": { "terms": { "field": "authors", "size": 10 } }
  }
}
```

```http
// 2 derniers titres par catégorie (aperçu)
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

```http
// Rechercher par auteur exact
GET news/_search
{
  "size": 10,
  "query": { "term": { "authors": "Ron Dicker" } },
  "sort": [{ "date": "desc" }]
}
// Variante si besoin : { "term": { "authors.keyword": "Ron Dicker" } }
```

```http
// Articles qui contiennent 'Weinstein' dans le titre OU la description
GET news/_search
{
  "size": 15,
  "query": {
    "multi_match": {
      "query": "Weinstein",
      "fields": ["headline","short_description"],
      "operator": "or"
    }
  },
  "sort": [{ "date": "desc" }]
}
```

```http
// Filtrer par domaine du lien (HuffPost) via préfixe
GET news/_search
{
  "size": 10,
  "query": {
    "prefix": { "link": "https://www.huffingtonpost.com/entry/" }
  },
  "sort": [{ "date": "desc" }]
}
```

```http
// Requête booléenne plus stricte (ET/OU/NOT)
GET news/_search
{
  "size": 20,
  "query": {
    "bool": {
      "must": [
        { "match_phrase": { "headline": "North Korea" } }
      ],
      "should": [
        { "match": { "short_description": "summit" } }
      ],
      "must_not": [
        { "term": { "category": "ENTERTAINMENT" } }
      ],
      "filter": [
        { "range": { "date": { "gte": "2018-05-24", "lte": "2018-05-26" } } }
      ]
    }
  },
  "sort": [{ "date": "desc" }]
}
```

---

## (Optionnel) ES|QL (onglet **ES|QL**)

```sql
-- Compter les articles par catégorie
FROM news
| STATS count() BY category
| ORDER BY count DESC
| LIMIT 10
```

```sql
-- Filtre par date + top auteurs
FROM news
| WHERE date BETWEEN "2018-05-24" AND "2018-05-26"
| STATS count() BY authors
| ORDER BY count DESC
| LIMIT 10
```

```sql
-- Plein texte sur Trump (titre ou description)
FROM news
| WHERE MATCH(headline, "Trump") OR MATCH(short_description, "Trump")
| KEEP headline, category, authors, date, link
| ORDER BY date DESC
| LIMIT 20
```

# Exercice :

- Donnez les 15 titres les plus récents de WORLD NEWS contenant North Korea
