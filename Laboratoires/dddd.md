génial — on y va tout de suite, **ultra-guidé de A à Z**, avec ton dataset JSON (catégorie, titre, auteur, lien, description, date). Tu vas :

1. lancer un mini-ELK (Elasticsearch + Kibana) en **Docker** (sécurité désactivée pour apprendre tranquille)
2. créer un **index `news`** avec un **mapping propre** (date, text/keyword, analyzers)
3. **ingérer** tes lignes JSON (au format **NDJSON / _bulk**)
4. exécuter **plein de requêtes** (match, phrase, multi_match, bool, range, minimum_should_match, aggregations, significant_text, date_histogram, top_hits, …)
5. construire **quelques visualisations** rapides dans **Kibana**
6. bonus: autocomplétion (suggester), surlignage (highlight), filtres analyzers FR/EN, export/backup.



# 0) Pré-requis rapides (déjà faits pour toi plus haut, mais je résume)

* Docker & Docker Compose installés ✅
* Port 9200 (ES) et 5601 (Kibana) libres ✅
* Un terminal sur ta VM ✅

---

# 1) Lancer Elasticsearch + Kibana (compose prêt à copier/coller)

Dans ton **HOME** ou le Desktop :

```bash
mkdir -p ~/elk-news && cd ~/elk-news
cat > docker-compose.yml <<'YAML'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.14.3
    container_name: es-news
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200"
    volumes:
      - esdata:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD-SHELL","curl -fsS http://localhost:9200 >/dev/null || exit 1"]
      interval: 5s
      timeout: 3s
      retries: 60

  kibana:
    image: docker.elastic.co/kibana/kibana:8.14.3
    container_name: kb-news
    depends_on:
      elasticsearch:
        condition: service_healthy
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - XPACK_SECURITY_ENABLED=false
    ports:
      - "5601:5601"

volumes:
  esdata:
YAML

docker compose up -d
docker compose ps
```

Test rapide :

```bash
curl -s http://localhost:9200 | jq '.'
# => "tagline": "You Know, for Search"
```

Ouvre **Kibana** : [http://localhost:5601](http://localhost:5601)
(aucun login demandé, sécurité désactivée pour le LAB)

---

# 2) Créer ton fichier **NDJSON** à partir de tes lignes

Tu m’as donné des **lignes JSON** (une par article). Pour `_bulk`, il faut **NDJSON** avec **l’action** avant chaque doc.

### 2.1. Colle ta liste telle quelle dans `raw.jsonl`

```bash
cd ~/elk-news
cat > raw.jsonl <<'JSON'
{"category": "POLITICS", "headline": "Ryan Zinke Looks To Reel Back Some Critics With 'Grand Pivot' To Conservation", "authors": "Chris D'Angelo", "link": "https://www.huffingtonpost.com/entry/ryan-zinke-reel-back-critics-grand-pivot-conservation_us_5b086c78e4b0fdb2aa538b3f", "short_description": "The interior secretary attempts damage control with hunting and fishing groups that didn\u2019t like his fossil fuel focus.", "date": "2018-05-26"}
{"category": "POLITICS", "headline": "Trump's Scottish Golf Resort Pays Women Significantly Less Than Men: Report", "authors": "Mary Papenfuss", "link": "https://www.huffingtonpost.com/entry/trump-scottish-golf-resort-pays-women-less-than-men_us_5b08ca29e4b0802d69cb4d37", "short_description": "And there are four times as many male as female executives.", "date": "2018-05-26"}
{"category": "WEIRD NEWS", "headline": "Weird Father's Day Gifts Your Dad Doesn't Know He Wants (But He Does)", "authors": "David Moye", "link": "https://www.huffingtonpost.com/entry/weird-fathers-day-gifts-2018_us_5b05bf18e4b05f0fc84438f6", "short_description": "Why buy a boring tie when you can give him testicle plush toys?", "date": "2018-05-26"}
{"category": "ENTERTAINMENT", "headline": "Twitter #PutStarWarsInOtherFilms And It Was Universally Entertaining", "authors": "Andy McDonald", "link": "https://www.huffingtonpost.com/entry/twitter-put-star-wars-in-other-films_us_5b098875e4b0568a880be92c", "short_description": "There's no such thing as too much \"Star Wars.\"", "date": "2018-05-26"}
{"category": "WEIRD NEWS", "headline": "Mystery 'Wolf-Like' Animal Reportedly Shot In Montana, Baffles Wildlife Officials", "authors": "Hilary Hanson", "link": "https://www.huffingtonpost.com/entry/montana-wolf-like-animal-mystery_us_5b04533de4b003dc7e4708af", "short_description": "\u201cWe have no idea what this was until we get a DNA report back.\"", "date": "2018-05-26"}
{"category": "WORLD NEWS", "headline": "North Korea Still Open To Talks After Trump Cancels Summit", "authors": "Josh Smith and Christine Kim, Reuters", "link": "https://www.huffingtonpost.com/entry/north-korea-reax-canceled-summit_us_5b07a93de4b0568a8809ccf2", "short_description": "Trump\u2019s announcement came after repeated threats by North Korea to pull out of the summit over what it saw as confrontational remarks by U.S. officials.", "date": "2018-05-25"}
# ... (colle toutes les lignes que tu m'as envoyées, exactement au même format, une par ligne)
JSON
```

### 2.2. Convertir en **NDJSON pour _bulk**

Chaque doc doit être précédé d’une ligne `{"index":{"_index":"news"}}` :

```bash
awk 'BEGIN{print ""} {print "{\"index\":{\"_index\":\"news\"}}"; print}' raw.jsonl > news.bulk.ndjson
wc -l raw.jsonl news.bulk.ndjson
# news.bulk.ndjson doit avoir 2x plus de lignes (action + source)
```

---

# 3) Créer l’index `news` avec un **mapping propre**

On définit types + analyzers. On garde simple et robuste :

* `date`: type `date` (format `yyyy-MM-dd`)
* `category`: `keyword` (exact) + `text` (recherche full-text) via sous-champ
* `authors`: `keyword` + `text`
* `headline`, `short_description`: `text` (analyzer standard) + sous-champ `keyword`
* `link`: `keyword`
* **Option**: un `normalizer` pour `category.keyword_lower`

```bash
curl -s -X PUT 'http://localhost:9200/news' -H 'Content-Type: application/json' -d '{
  "settings": {
    "number_of_shards": 1,
    "analysis": {
      "normalizer": {
        "lowercase_normalizer": { "type": "custom", "filter": ["lowercase"] }
      }
    }
  },
  "mappings": {
    "properties": {
      "date":    { "type": "date", "format": "yyyy-MM-dd" },
      "category": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword" },
          "keyword_lower": { "type": "keyword", "normalizer": "lowercase_normalizer" }
        }
      },
      "headline": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 } }
      },
      "authors": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword" } }
      },
      "short_description": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 } }
      },
      "link": { "type": "keyword" }
    }
  }
}'
```

Vérifie :

```bash
curl -s http://localhost:9200/news | jq '.'
```

---

# 4) **Ingestion** en masse (Bulk)

```bash
curl -s -H 'Content-Type: application/x-ndjson' -X POST 'http://localhost:9200/_bulk?pretty' --data-binary @news.bulk.ndjson
# => "errors" doit être false
curl -s 'http://localhost:9200/news/_count?pretty'
```

Si `errors: true`, c’est souvent un caractère UTF-8 mal échappé → recolle la ligne fautive.
(ton JSON est déjà propre; l’escape `\u2019` est OK)

---

# 5) Requêtes **de base** (exactes à copier/coller)

> Tu peux faire les mêmes dans **Kibana → Dev Tools → Console**

## 5.1. Voir 5 docs

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "size": 5,
  "_source": ["date","category","headline","authors"]
}'
```

## 5.2. `match` (plein texte sur le titre)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": { "match": { "headline": "Trump summit North Korea" } },
  "highlight": { "fields": { "headline": {} } },
  "_source": ["date","category","headline"]
}'
```

## 5.3. `match_phrase` (expression exacte)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": { "match_phrase": { "headline": "President Obama" } },
  "_source": ["headline","date","category"]
}'
```

## 5.4. `minimum_should_match` (précision/rappel)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": {
    "match": {
      "headline": {
        "query": "obama trump clinton",
        "minimum_should_match": 2
      }
    }
  },
  "_source": ["headline","date","category"]
}'
```

## 5.5. `multi_match` (plusieurs champs)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": {
    "multi_match": {
      "query": "President Obama",
      "fields": ["headline^2","short_description","authors"]
    }
  },
  "_source": ["headline","authors","date","category"]
}'
```

## 5.6. `range` sur la date (2018-05-24 → 2018-05-26)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": {
    "range": {
      "date": { "gte": "2018-05-24", "lte": "2018-05-26" }
    }
  },
  "sort": [{ "date": "asc" }],
  "_source": ["date","category","headline"]
}'
```

## 5.7. `bool` (combiner must + must_not + filter)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": {
    "bool": {
      "must": [
        { "match_phrase": { "headline": "President Obama" } }
      ],
      "must_not": [
        { "match": { "category": "SPORTS" } }
      ],
      "filter": [
        { "range": { "date": { "gte": "2018-05-24", "lte": "2018-05-26" } } }
      ]
    }
  },
  "_source": ["date","category","headline"]
}'
```

---

# 6) **Agrégations** puissantes

## 6.1. Comptage par **catégorie**

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' -H 'Content-Type: application/json' -d '{
  "aggs": {
    "par_categorie": { "terms": { "field": "category.keyword", "size": 20 } }
  }
}'
```

## 6.2. Termes **significatifs** (mots saillants dans les titres d’une catégorie)

Ex: extraire les termes caractéristiques des titres en **POLITICS** :

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' -H 'Content-Type: application/json' -d '{
  "query": { "term": { "category.keyword": "POLITICS" } },
  "aggs": {
    "hot_terms": { "significant_text": { "field": "headline" } }
  }
}'
```

## 6.3. **Date histogram** (répartition par jour)

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' -H 'Content-Type: application/json' -d '{
  "aggs": {
    "par_jour": {
      "date_histogram": { "field": "date", "calendar_interval": "day", "min_doc_count": 0 }
    }
  }
}'
```

## 6.4. **Top titres** par catégorie (terms + top_hits)

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' -H 'Content-Type: application/json' -d '{
  "aggs": {
    "cats": {
      "terms": { "field": "category.keyword", "size": 10 },
      "aggs": {
        "top_titles": {
          "top_hits": { "_source": ["headline","date"], "size": 3, "sort": [{ "date": "desc" }] }
        }
      }
    }
  }
}'
```

---

# 7) **Surlignage** (highlight) & **tri**

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": { "match": { "short_description": "sexual harassment" } },
  "highlight": { "fields": { "short_description": {} } },
  "sort": [{ "date": "desc" }],
  "_source": ["date","headline","short_description","category"]
}'
```

---

# 8) **Suggester** (autocomplétion simple sur `headline`)

On ajoute un champ `suggest` par **update by query** (ou tu peux recréer l’index propre; ici on fait simple) :

### 8.1. Ajouter un champ `headline_suggest` à tous (copie de headline)

```bash
curl -s -X POST 'http://localhost:9200/news/_update_by_query?pretty&conflicts=proceed' -H 'Content-Type: application/json' -d '{
  "script": {
    "source": "ctx._source.headline_suggest = params.v",
    "lang": "painless",
    "params": { "v": "" }
  },
  "query": { "match_all": {} }
}'
# Maintenant remettons vraiment les titres:
curl -s -X POST 'http://localhost:9200/news/_update_by_query?pretty&conflicts=proceed' -H 'Content-Type: application/json' -d '{
  "script": {
    "source": "ctx._source.headline_suggest = ctx._source.headline",
    "lang": "painless"
  },
  "query": { "match_all": {} }
}'
```

### 8.2. Index alias pour le suggester (facultatif), puis requête de complétion « prefix »

*(Sans mapping completion avancé, on simule un autocomplete par prefix sur `headline.keyword`)*

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "size": 5,
  "query": {
    "prefix": { "headline.keyword": "Harvey" }
  },
  "_source": ["headline","date","category"]
}'
```

*(Une vraie completion utiliserait un champ `type: completion` et un reindex — on garde simple pour débuter.)*

---

# 9) **Kibana** – visualisations express

Dans **Kibana → Stack Management → Index Patterns** :

1. Create data view → Name: `news` → Index pattern: `news` → Timestamp field: `date`
2. **Discover** → explore data
3. **Visualize Library** → Create:

   * **Pie** → Buckets: Split slices → `Terms` on `category.keyword` → Size 10
   * **Bar** → X-axis: `date histogram` (field `date`, daily) ; Split series by `category.keyword`
   * **Tag cloud** → Field: `headline` (ou utilise `significant_text` via Lens si dispo)
4. **Dashboard** → ajoute tes visualisations.

---

# 10) Recherches « style TP » (exactement celles de ton guide)

### 10.1. Rappel « track_total_hits »

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "track_total_hits": true,
  "query": { "match_all": {} }
}'
```

### 10.2. `match_phrase` vs `match`

```bash
# match_phrase - ordre et proximité exacts
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": { "match_phrase": { "headline": "President Obama" } }
}'

# match - plus souple
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": { "match": { "headline": "President Obama" } }
}'
```

### 10.3. `multi_match` type `phrase`

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": {
    "multi_match": {
      "query": "President Obama",
      "fields": ["headline^2","short_description"],
      "type": "phrase"
    }
  }
}'
```

### 10.4. `bool` combinée (must / should / filter / must_not)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": {
    "bool": {
      "must": [
        { "match_phrase": { "headline": "President Obama" } }
      ],
      "should": [
        { "match": { "category": "POLITICS" } }
      ],
      "must_not": [
        { "match": { "category": "SPORTS" } }
      ],
      "filter": [
        { "range": { "date": { "gte": "2018-01-01", "lte": "2018-12-31" } } }
      ]
    }
  }
}'
```

---

# 11) Bonnes pratiques & astuces

* **NDJSON** : une action + un document par ligne (pas de `[]`).
* **Types** : `text` pour recherche, `keyword` pour agrégations/tri/filtre exact.
* **Dates** : indique le `format` dans le mapping.
* **Analyzers** : `standard` suffit ici; pour français, tu peux créer un analyzer `french`.
* **Reindex** (si tu changes le mapping d’un champ) : crée un nouvel index, puis `_reindex`.
* **Sauvegarde** : snapshot (S3/FS) *ou* export NDJSON (via `_scroll` + `_bulk` inverse).
* **Perf** : `terms` sur `.keyword`, jamais sur `text`.

---

# 12) Export / Backup rapide (logiciel)

Pour exporter tous les documents :

```bash
# Scroll (simple): récupère tout en JSON
curl -s -H 'Content-Type: application/json' -X POST 'http://localhost:9200/news/_search?scroll=1m&size=500' -d '{"query":{"match_all":{}}}' > page1.json
# (pour un vrai export complet, on enchaîne avec _scroll_id ; pour un jeu court, pas nécessaire)
```

---

# 13) Nettoyer / Repartir

```bash
# compter docs
curl -s http://localhost:9200/news/_count?pretty
# supprimer index
curl -s -X DELETE http://localhost:9200/news?pretty
# arrêter la stack
docker compose down
# (pour tout effacer, y compris les données ES)
docker compose down -v
```

---

# 14) Mini-FAQ (débutant)

* **Kibana ne se connecte pas ?**
  Attends que `es-news` soit `healthy` (`docker compose ps`) ; la variable `ELASTICSEARCH_HOSTS` pointe vers `http://elasticsearch:9200` (réseau interne Docker) — OK.

* **Bulk renvoie `400` ?**
  Souvent : NDJSON mal formé (ligne vide, JSON non valide, guillemets manquants). Vérifie `news.bulk.ndjson` : chaque action + chaque doc sur **deux lignes**, pas de virgules finales.

* **Aucune donnée retournée ?**
  Vérifie `/_count`, puis tes filtres (`date` gte/lte, `category.keyword` vs `category`).

