# Partie 1 : importer les données

- https://drive.google.com/drive/folders/17vTaNQFnTyhEcizFYofLpdfd-f792utk?usp=sharing
  
## Méthode 1 (sans terminal) — via l’UI Kibana

1. Dans Kibana, ouvre le menu (☰ en haut à gauche)
2. va dans **Machine Learning → Data Visualizer → File** (ou “Upload a file”).
3. **Glisse ton fichier** dans la zone (ou clique *Select file* et prends-le depuis `~/Téléchargements`).

   * Kibana accepte directement le **NDJSON** (ton format: 1 objet JSON par ligne).
4. **Index name** : mets par ex. `news`.
5. Time field → choisis **`date`**.
6. Vérifie que `category` et `authors` sont en **keyword** (Kibana le propose souvent tout seul).
7. Clique **Import**.
8. Une fois fini, va dans **Discover**, crée une **Data View** `news*` et tu vois tes documents.

> Si ton fichier est *vraiment* gros (>100 Mo), l’UI peut galérer. Dans ce cas, prends la méthode terminal ci-dessous.



## Méthode 2 (terminal) — rapide & robuste

Supposons que ton fichier est dans `~/Téléchargements/news.ndjson`. On l’ingère à chaud.

1. Place-le dans ton dossier de travail :

```bash
mkdir -p ~/elk-dev/data
cp ~/Téléchargements/news.ndjson ~/elk-dev/data/
cd ~/elk-dev
# charge les variables de .env pour ne pas taper le mot de passe
set -a; source .env; set +a
```

2. (Optionnel) Template pour bien typer `date`, `authors`, `category` :

```bash
curl -u elastic:$ELASTIC_PASSWORD -H 'Content-Type: application/json' \
  -X PUT http://localhost:9200/_index_template/news-template -d '{
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
}}}}'
```

3. Envoi en **_bulk** (ton fichier est déjà NDJSON) :

```bash
# On ajoute la ligne d’action _bulk avant chaque doc
awk '{print "{\"index\":{\"_index\":\"news\"}}"; print}' data/news.ndjson > data/bulk.ndjson

# Si le fichier est gros, envoie par paquets de 10k lignes (~5k docs)
cd data
split -l 10000 bulk.ndjson bulk_part_

for f in bulk_part_*; do
  echo "Envoi de $f…"
  curl -s -u elastic:$ELASTIC_PASSWORD \
       -H 'Content-Type: application/x-ndjson' \
       -X POST 'http://localhost:9200/_bulk?refresh=false' \
       --data-binary "@$f" | jq '.errors'
done

# refresh (optionnel)
curl -u elastic:$ELASTIC_PASSWORD -X POST 'http://localhost:9200/news/_refresh'
```

4. Vérifie :

```bash
curl -u elastic:$ELASTIC_PASSWORD 'http://localhost:9200/news/_count?pretty'
```

Puis dans Kibana → **Discover** → crée une **Data View** `news*`.



## Où est le fichier ?

- https://drive.google.com/drive/folders/17vTaNQFnTyhEcizFYofLpdfd-f792utk?usp=sharing

* Si tu l’as téléchargé dans la **VM** via Firefox : il est probablement dans `~/Téléchargements` (ou `~/Downloads`). 
* S’il est sur **Windows (hôte)** : plus simple → re-télécharge-le **dans la VM** avec Firefox, ou configure un *dossier partagé* VirtualBox (mais ce n’est pas nécessaire si tu peux le retélécharger dans la VM).






<br/>

# Partie 2


Une fois que nos docs sont dans `news`, nous avons 3 façons principales de les interroger :

* **Discover** (Kibana) avec **KQL** – super simple pour filtrer/chercher.
* **Dev Tools → Console** avec le **DSL `_search`** (JSON).
* **ES|QL** (nouveau langage style SQL) dans **Discover → Query** ou **Dev Tools**.

Je vous fais un mémo ultra-pratique avec des exemples pour notre schéma (`category`, `authors`, `date`, `headline`, `short_description`, `link`).


# 1) KQL (barre de recherche dans Discover)

* Tous les articles de la catégorie “POLITICS” :

```
category : "POLITICS"
```

* Articles entre deux dates :

```
date >= "2018-05-20" and date < "2018-06-01"
```

* Recherche plein texte dans le titre et la description :

```
headline : "Trump" or short_description : "Trump"
```

* Plusieurs filtres combinés :

```
category : ("WORLD NEWS" or "POLITICS") and authors : "Ron Dicker" and date >= "2018-05-25"
```

💡 Astuces

* Si tu vois des champs **`field.keyword`**, utilise la version **`keyword`** pour les filtres exacts (KQL gère souvent ça tout seul).
* Tu peux sauvegarder ta requête comme “Saved Search”.



# 2) ES|QL (style SQL) – super lisible

Dans **Discover** (switch “ES|QL”) ou **Dev Tools** :

* Compter les docs par catégorie :

```sql
FROM news
| STATS count() BY category
| ORDER BY count DESC
| LIMIT 10
```

* Top auteurs sur une période :

```sql
FROM news
| WHERE date BETWEEN "2018-05-20" AND "2018-05-31"
| STATS count() BY authors
| ORDER BY count DESC
| LIMIT 10
```

* Lignes (docs) contenant “Trump” :

```sql
FROM news
| WHERE MATCH(headline, "Trump") OR MATCH(short_description, "Trump")
| LIMIT 20
```

* Histogramme temporel (agrégé par jour) :

```sql
FROM news
| WHERE category == "POLITICS"
| EVAL day = DATE_TRUNC(1 days, date)
| STATS count() BY day
| ORDER BY day
```



# 3) DSL JSON (Dev Tools → Console) + équivalents `curl`

### a) Recherche plein texte

```json
GET news/_search
{
  "query": {
    "multi_match": {
      "query": "Trump",
      "fields": ["headline^2", "short_description"]
    }
  },
  "size": 20
}
```

### b) Filtres exacts + plage de dates

```json
GET news/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "category": "POLITICS" } }
      ],
      "filter": [
        { "range": { "date": { "gte": "2018-05-20", "lt": "2018-06-01" } } }
      ]
    }
  },
  "sort": [{ "date": "desc" }]
}
```

### c) Top N catégories (aggregation)

```json
GET news/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category", "size": 10 }
    }
  }
}
```

### d) Histogramme temporel

```json
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

### e) Mettre en avant (highlight)

```json
GET news/_search
{
  "query": {
    "multi_match": {
      "query": "North Korea",
      "fields": ["headline", "short_description"]
    }
  },
  "highlight": {
    "fields": {
      "headline": {},
      "short_description": {}
    }
  }
}
```

### f) `curl` depuis ta VM (tu as les vars `.env`)

```bash
curl -u elastic:$ELASTIC_PASSWORD 'http://localhost:9200/news/_search?pretty' \
  -H 'Content-Type: application/json' -d '{
  "query": { "term": { "category": "POLITICS" } },
  "size": 5,
  "sort": [{ "date": "desc" }]
}'
```



## Choisir le bon champ (text vs keyword)

* **`text`** (ex: `headline`, `short_description`) → recherche plein texte (`match`, `multi_match`, `MATCH()` en ES|QL).
* **`keyword`** (ex: `category`, `authors`, `link`) → filtres exacts, agrégations (`term`, `terms`, `BY category`, etc.).
* **`date`** (`date`) → filtres `range`, histogrammes, ES|QL `DATE_TRUNC`.



## Aller plus loin (rapide)

* Transforme ta requête **Discover** en visualisation (Lens) → graphes en 2 clics.
* Sauvegarde des **Data Views** (`news*`) et des **Dashboards**.
* Si tu veux **compter par mot** dans les titres, crée un *runtime field* qui tokenize, ou utilise l’agg `significant_text` sur `headline`.

<br/>

# Partie 3

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
