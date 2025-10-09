
# Introduction du lab 

Vous allez bâtir une mini-plateforme de recherche d’actualités avec **Elasticsearch** et **Kibana** à partir d’un dataset JSON (catégorie, titre, auteur, lien, description, date).

À l’issue du lab, vous saurez :

* **Déployer** Elasticsearch + Kibana via Docker et vérifier la santé des services.
* **Modéliser** un index `news` avec un **mapping propre** (`text`/`keyword`, `date`, analyzers).
* **Ingerer** des documents en **NDJSON** via l’API `_bulk`.
* **Interroger** efficacement (match, match_phrase, multi_match, bool, range, minimum_should_match) et **expliquer** la pertinence (highlight).
* **Explorer** avec des **agrégations** (terms, date_histogram, significant_text, top_hits).
* **Visualiser** dans Kibana (graphiques simples, tableau de bord).
* Bonus : **autocomplétion** (suggester), analyzers FR/EN, export/backup.

Pré-requis : Docker/Compose, `curl` (et idéalement `jq`), 3 Go RAM libres, ports 9200/5601 disponibles.


<br/>
<br/>

# 0) Pré-requis rapides (Annexe 0)

* Docker et Docker Compose installés
  Vérifier : `docker --version` et `docker compose version`
* Ports disponibles : **9200** (Elasticsearch) et **5601** (Kibana)
  Vérifier : `ss -lnt | grep -E ':9200|:5601'` (aucune ligne ne doit s’afficher)
* Terminal opérationnel sur la VM (accès root/sudo si besoin)
* Outils conseillés : `curl` (et `jq` pour lire les réponses JSON)
  Installer si nécessaire : `sudo apt-get install -y curl jq`



<br/>
<br/>



# Partie 1 — Démarrage complet d’ELK en Docker (avec persistance)

## A) Pré-requis rapides

```bash
# 1) Docker + Compose plugin
docker --version
docker compose version

# 2) Ports libres (9200 pour ES, 5601 pour Kibana)
ss -lnt | grep -E ':9200|:5601' || echo "OK: ports libres"
```

Si un port est occupé, libère-le :

```bash
# Voir qui écoute
sudo ss -ltnp | grep -E ':9200|:5601'
# Exemple de kill (PID à adapter)
sudo kill -9 <PID>
# Si c'est un conteneur Docker
docker ps | grep -E '9200|5601'
docker stop <CONTAINER_ID>
```

Optionnel mais recommandé (compatibilité) :

```bash
# Souvent inutile en single-node, mais utile selon l’hôte
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-elasticsearch.conf
sudo sysctl --system
```

## B) Arborescence de travail

```bash
mkdir -p ~/elk-news && cd ~/elk-news
```

## C) docker-compose.yml (persistance via volume nommé)

```bash
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
    restart: unless-stopped

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
    restart: unless-stopped

volumes:
  esdata:
    name: elk_esdata
YAML
```

## D) Lancer, vérifier, logs

```bash
docker compose up -d
docker compose ps
docker compose logs -f elasticsearch   # Ctrl+C pour quitter
```

## E) Tests rapides (API Elasticsearch)

```bash
# 1) Ping/version
curl -s http://localhost:9200 | jq '.'

# 2) Santé du cluster
curl -s http://localhost:9200/_cluster/health?pretty

# 3) Infos noeud
curl -s http://localhost:9200/_nodes?pretty | jq '.nodes | keys'
```

Ouvre Kibana : [http://localhost:5601](http://localhost:5601)
(Aucun login demandé, sécurité désactivée pour le lab.)

## F) Persistance (volume nommé)

Le volume nommé `elk_esdata` conserve les données entre redéploiements.

```bash
# Redéploiement normal (données conservées)
docker compose down
docker compose up -d

# Attention : ceci SUPPRIME les données
docker compose down -v
```

Inspecter l’emplacement réel sur la VM :

```bash
docker volume inspect elk_esdata | jq -r '.[0].Mountpoint'
sudo ls -lah /var/lib/docker/volumes/elk_esdata/_data
```

## G) Sauvegarde et restauration du volume

Sauvegarde (.tar.gz) :

```bash
mkdir -p ~/elk-news/backup
docker run --rm \
  -v elk_esdata:/vol \
  -v ~/elk-news/backup:/backup \
  alpine sh -c "cd /vol && tar czf /backup/elk_esdata_$(date +%F_%H%M).tar.gz ."
ls -lh ~/elk-news/backup
```

Restauration dans le même volume :

```bash
docker compose down
docker run --rm -v elk_esdata:/vol alpine sh -c "rm -rf /vol/*"
docker run --rm \
  -v elk_esdata:/vol \
  -v ~/elk-news/backup:/backup \
  alpine sh -c "cd /vol && tar xzf /backup/elk_esdata_YYYY-MM-DD_HHMM.tar.gz"
docker compose up -d
```

Clonage vers un nouveau volume :

```bash
docker volume create elk_esdata_clone
docker run --rm \
  -v elk_esdata_clone:/vol \
  -v ~/elk-news/backup:/backup \
  alpine sh -c "cd /vol && tar xzf /backup/elk_esdata_YYYY-MM-DD_HHMM.tar.gz"
docker volume ls
```

## H) Dépannage express

1. Ports 9200/5601 occupés
   – Identifie et stoppe le processus (ss/kill) ou change le mapping (ex. `19200:9200`, `15601:5601`) dans le compose.

2. Permissions en bind-mount
   – Préfère les volumes nommés. En bind-mount, prépare le dossier :

```bash
sudo mkdir -p /srv/elk/esdata
sudo chown -R 1000:1000 /srv/elk/esdata
```

3. ES ne devient pas healthy
   – Regarde les logs :

```bash
docker compose logs -f elasticsearch
```

– Vérifie la RAM allouée via `ES_JAVA_OPTS`.
– Assure `vm.max_map_count` suffisant (section A).
– Si tu as importé des données d’une autre version et que ça bloque, repars proprement (backup, puis `down -v`, puis `up -d`, puis réindexation).

4. Perte de données inattendue
   – Tu as probablement fait `down -v` ou modifié le nom du volume. Vérifie `docker volume ls` et `volumes: { esdata: { name: elk_esdata } }`.

## I) Commandes utiles (récap)

```bash
# cycle
docker compose up -d
docker compose ps
docker compose logs -f db   # ici "db" n'existe pas, utilise "elasticsearch" ou "kibana"
docker compose logs -f elasticsearch
docker compose logs -f kibana
docker compose down          # garde les volumes
docker compose down -v       # supprime aussi les volumes (données perdues)

# volumes
docker volume ls
docker volume inspect elk_esdata
docker run --rm -it -v elk_esdata:/vol alpine sh  # shell pour voir /vol
```

Cette partie 1 met en place une base ELK fiable et persistante sur ta VM. À partir de là, tu peux passer à la Partie 2 du TP : création de l’index `news` avec mapping propre, puis ingestion NDJSON via `_bulk`.


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





<br/>
<br/>

### Annexe 0 - Procédure **rapide et sûre** pour libérer **9200** (Elasticsearch) ou **5601** (Kibana) quand `ss -lnt | grep -E ':9200|:5601'` montre qu’ils sont occupés.

## 0.1) Identifier le processus fautif

```bash
# Voir le PID et le binaire qui écoute
sudo ss -lntp | grep -E ':9200|:5601'

# Alternative (souvent plus verbeux)
sudo lsof -i :9200
sudo lsof -i :5601

# Autre option
sudo fuser 9200/tcp
sudo fuser 5601/tcp
```

## 0.2) Si c’est un **container Docker**

```bash
docker ps --format 'table {{.ID}}\t{{.Names}}\t{{.Ports}}' | grep -E '9200|5601'
# Stopper / supprimer
docker stop <CONTAINER_ID_OR_NAME>
docker rm   <CONTAINER_ID_OR_NAME>

# Si lancé via compose dans ce dossier
docker compose down
# Si lancé ailleurs et tu connais le nom du service
docker stop es-news kb-news 2>/dev/null || true
```

## 0.3) Si c’est un **service système** (install local)

Elasticsearch/Kibana installés hors Docker occupent souvent ces ports.

```bash
# Vérifier l’état
systemctl status elasticsearch kibana

# Arrêter et désactiver (pour libérer définitivement)
sudo systemctl stop elasticsearch kibana
sudo systemctl disable elasticsearch kibana
```

## 0.4) Si c’est un **processus “perdu”** (lancé à la main)

1. Récupérer le **PID** (depuis `ss`/`lsof`), puis :

```bash
# Arrêt propre
sudo kill -15 <PID>
# Si ça ne tombe pas en 3–5 s :
sudo kill -9 <PID>
```

## 0.5) Vérifier que le port est libre

```bash
ss -lnt | grep -E ':9200|:5601' || echo "Ports 9200/5601 libres."
```

## 0.6) En dernier recours (rare)

* Un processus se relance tout seul ? Chercher un **service** associé (systemd, pm2, supervisor, snap) et le désactiver.
* Conflit persistant dans une autre stack Docker ?
  Lister tout : `docker ps -a | grep -E '9200|5601'`, puis `docker stop && docker rm`.



### Astuce si tu **ne peux pas** libérer tout de suite

Tu peux changer les ports dans `docker-compose.yml` :

```yaml
# à la place de 9200:9200 et 5601:5601
ports:
  - "19200:9200"
  ...
  - "15601:5601"
```

Puis relancer :

```bash
docker compose up -d
```

Et tu accéderas via `http://localhost:19200` (ES) et `http://localhost:15601` (Kibana).

