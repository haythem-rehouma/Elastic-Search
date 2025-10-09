
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

# Partie 0 - Pré-requis rapides (Annexe 0)

* Docker et Docker Compose installés
  Vérifier : `docker --version` et `docker compose version`
* Ports disponibles : **9200** (Elasticsearch) et **5601** (Kibana)
  Vérifier : `ss -lnt | grep -E ':9200|:5601'` (aucune ligne ne doit s’afficher)
* Terminal opérationnel sur la VM (accès root/sudo si besoin)
* Outils conseillés : `curl` (et `jq` pour lire les réponses JSON)
  Installer si nécessaire : `sudo apt-get install -y curl jq`



<br/>
<br/>



# Partie 1 — Démarrage complet d’ELK en Docker, avec persistance (Annexe 1)

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


<br/>
<br/>





# Partie 2 - Préparer ton dataset et le convertir en NDJSON (Annexe 1 et Annexe 2)

### 2.1. Mettre tes lignes JSON brutes dans un fichier

Chaque article sur **une ligne** (JSON valide, UTF-8) :

```bash
cd ~/elk-news
cat > raw.jsonl <<'JSON'
{"category":"POLITICS","headline":"Ryan Zinke Looks To Reel Back...","authors":"Chris D'Angelo","link":"https://...","short_description":"The interior secretary...","date":"2018-05-26"}
{"category":"POLITICS","headline":"Trump's Scottish Golf Resort...","authors":"Mary Papenfuss","link":"https://...","short_description":"And there are four times...","date":"2018-05-26"}
# ... colle toutes tes lignes ici (une par article)
JSON
```

### 2.2. Transformer en NDJSON pour `_bulk`

Le bulk exige **une ligne “action”** avant **chaque document** :

```bash
awk '{print "{\"index\":{\"_index\":\"news\"}}"; print}' raw.jsonl > news.bulk.ndjson
wc -l raw.jsonl news.bulk.ndjson
# news.bulk.ndjson doit avoir 2x plus de lignes (action + source)
```

Astuce (gros fichier) : compresser

```bash
gzip -c news.bulk.ndjson > news.bulk.ndjson.gz
```

<br/>
<br/>

# Partie 3 - Créer l’index `news` avec un mapping propre

D'abord, un **shard** (éclat) est un « morceau » d’un index Elasticsearch. Au lieu de stocker tous tes documents dans un seul gros bloc, Elasticsearch découpe l’index en plusieurs blocs indépendants appelés shards. Chaque shard est comme une petite base Lucene complète : il peut être placé sur n’importe quel nœud du cluster, cherché en parallèle, puis les résultats sont fusionnés.

Points clés :

* **Primaires vs répliques**

  * **Shard primaire** : contient les données « officielles ».
  * **Shard de réplication** : copie du primaire pour la **tolérance aux pannes** et plus de **débit en lecture**. Si un nœud tombe, une réplique peut devenir primaire.

* **Pourquoi on met `number_of_shards: 1` dans un lab**

  * Un seul nœud + petit volume de données = inutile de se fragmenter.
  * Plus simple à gérer, moins de surcoût, pas de distribution à orchestrer.

* **Quand augmenter le nombre de shards**

  * Très gros index (beaucoup de documents/volume) ou besoin d’**évolutivité** : plusieurs shards se répartissent sur plusieurs nœuds, les recherches s’exécutent en parallèle.
  * Attention : trop de shards = overhead (mémoire/CPU) et latence.

* **Répliques (`number_of_replicas`)**

  * `0` en machine perso si tu veux économiser des ressources.
  * `1` (ou plus) en prod pour **HA** et **lectures plus rapides**.

* **Taille et limites pratiques**

  * Viser des shards « sains » (souvent quelques Go à quelques dizaines de Go). Des milliers de petits shards nuisent aux performances.
  * Le **nombre de shards primaires est fixé à la création** de l’index (tu ne peux pas le changer après, sauf en reindexant vers un nouvel index avec la bonne configuration).

* **Résumé**
  Un shard est une brique de stockage/recherche distribuable ; tu en mets **peu** pour un petit labo (1 primaire, 0 réplique), **plus** quand tu dois répartir la charge et assurer la haute disponibilité.


Dans cette étape, on crée l’index `news` en définissant dès le départ un **mapping maîtrisé** pour obtenir des recherches pertinentes et des agrégations fiables. On fixe `number_of_shards` à 1 (lab local) et on ajoute un **normalizer** `lowercase_normalizer` afin d’avoir des valeurs « keyword » comparables sans sensibilité à la casse. Les champs textuels importants (`headline`, `short_description`, `authors`, `category`) sont déclarés en **multi-champs** : une version `text` pour le plein-texte (analyzers) et une sous-clé `keyword` pour les tris/agrégations/exact matches (avec `ignore_above` pour éviter des termes trop longs), plus `category.keyword_lower` pour des filtres insensibles à la casse. Le champ `date` est typé en **`date`** avec le format `yyyy-MM-dd` pour permettre les **filtres et histogrammes temporels**, et `link` est en **`keyword`** (valeur exacte non analysée). La première commande `PUT` crée l’index avec cette configuration; la seconde (`GET /news`) sert à **vérifier** que l’index existe bien avec les réglages attendus.



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
      "date": { "type": "date", "format": "yyyy-MM-dd" },
      "category": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword" },
          "keyword_lower": { "type": "keyword", "normalizer": "lowercase_normalizer" }
        }
      },
      "headline": { "type": "text", "fields": { "keyword": { "type": "keyword", "ignore_above": 256 } } },
      "authors":  { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
      "short_description": { "type": "text", "fields": { "keyword": { "type": "keyword", "ignore_above": 256 } } },
      "link": { "type": "keyword" }
    }
  }
}'
curl -s http://localhost:9200/news | jq '.'
```

<br/>
<br/>

# Partie 4 - Ingestion — 7 façons différentes (Annexe 1)

Cette section présente, de façon opérationnelle, toutes les voies d’**ingestion** possibles vers Elasticsearch autour de la même idée clé : utiliser l’**API `_bulk`** qui attend du **NDJSON** (une ligne « action » puis une ligne « document », répété). Selon ton contexte, tu peux 1) pousser un fichier NDJSON complet via **`curl`** (simple et scriptable), 2) faire la même chose en **gzip** pour gagner en débit sur les gros jeux de données, 3) coller/valider tes blocs directement dans **Kibana → Dev Tools** (pratique pour tester et corriger), 4) automatiser par code avec **Python/Node.js** (clients HTTP, gestion d’erreurs, reprise), 5) passer par **Logstash** (lecture de fichiers streamée, robustesse, transformations), 6) utiliser **elasticdump** (outil « prêt à l’emploi » pour charger/exporter), ou 7) **découper en lots** et paralléliser pour accélérer tout en évitant les erreurs de type **413/429**. Au-delà de la tuyauterie, retiens les bonnes pratiques : **batchs raisonnables** (ex. 5–20 Mo par requête), **compression** quand c’est gros, **contrôle d’erreurs `_bulk`** (rejouer uniquement les items en échec), gestion du **`refresh`** (désactiver ou espacer pour la vitesse), optionnellement fixer **`_id`** (idempotence), et envisager un **ingest pipeline** si tu dois normaliser/enrichir à l’entrée. Résultat : une ingestion fiable, reproductible et adaptée à la taille de tes données.



## NDJSON, c’est quoi ?

**NDJSON** = **N**ewline **D**elimited **JSON**
C’est juste **une suite d’objets JSON**, **un par ligne**, séparés par des **retours à la ligne**.
Chaque ligne est un JSON complet. On **ne met pas** de crochets `[` `]` autour de l’ensemble.

### Exemple (NDJSON)

```
{"id":1,"titre":"A"}
{"id":2,"titre":"B"}
{"id":3,"titre":"C"}
```

Trois lignes = trois objets.

### En quoi c’est différent du JSON « classique » ?

JSON « classique » pour une liste, c’est un **tableau** :

```
[
  {"id":1,"titre":"A"},
  {"id":2,"titre":"B"},
  {"id":3,"titre":"C"}
]
```

En NDJSON, **pas de tableau** et **pas de virgules** entre les objets, juste **une ligne = un objet**.

## Pourquoi Elasticsearch aime le NDJSON ?

L’API **`_bulk`** lit le fichier **ligne par ligne**.
Elle attend des **paires de lignes** :

1. **Ligne d’action** (ex. `{"index":{"_index":"news"}}`)
2. **Ligne document** (ton objet JSON)

Exemple minimal pour `_bulk` :

```
{"index":{"_index":"news"}}
{"id":1,"headline":"Titre A"}
{"index":{"_index":"news"}}
{"id":2,"headline":"Titre B"}
```

## Règles à mémoriser

* **Une ligne = un JSON valide** (UTF-8, guillemets doubles `"`, pas de virgules en fin de ligne).
* **Pas de crochets** autour de l’ensemble.
* Pour `_bulk`, **action** et **document** vont **par deux lignes**, répétées.
* La **dernière ligne se termine par un retour à la ligne** (important pour certains parseurs).

## Avantages

* Très **facile à streamer** (on traite une ligne à la fois).
* Parfait pour les **gros fichiers** (on peut découper par blocs de lignes).
* Compatible avec **`curl`**, **Logstash**, et les **clients** (Python/Node).

## Erreurs classiques (et corrections)

* ❌ Vous mettez `[` `]` autour de tout → **retirez-les** en NDJSON.
* ❌ Vous mettez des virgules entre les objets → **enlevez les virgules**.
* ❌ Vous oubliez la ligne d’action pour `_bulk` → **ajoutez-la** avant chaque document.
* ❌ Vous collez des retours à la ligne à l’intérieur d’un objet → **un objet doit tenir sur une seule ligne** (échapper les sauts de ligne en `\n` si besoin).

## Astuces pratiques

* Transformer JSON « tableau » → NDJSON (jq) :

```bash
jq -c '.[]' tableau.json > data.ndjson
```

* Préparer `_bulk` (ajouter l’action avant chaque ligne) :

```bash
awk '{print "{\"index\":{\"_index\":\"news\"}}"; print}' data.ndjson > bulk.ndjson
```

En résumé : **NDJSON = des objets JSON, chacun sur sa propre ligne**. C’est ce format ligne-par-ligne qui permet à Elasticsearch de **charger vite et par paquets** via l’API `_bulk`.


## Méthode A — `curl` (bulk NDJSON classique)

```bash
curl -s -H 'Content-Type: application/x-ndjson' \
     -X POST 'http://localhost:9200/_bulk?pretty' \
     --data-binary @news.bulk.ndjson
# Vérif
curl -s 'http://localhost:9200/news/_count?pretty'
```

### Variante A.1 — avec gzip (plus rapide sur gros fichiers)

```bash
curl -s -H 'Content-Type: application/x-ndjson' \
     -H 'Content-Encoding: gzip' \
     -X POST 'http://localhost:9200/_bulk?pretty' \
     --data-binary @news.bulk.ndjson.gz
```

## Méthode B — Kibana Dev Tools (colle et envoie)

1. Ouvre Kibana → **Dev Tools** → **Console**.
2. Crée l’index (bloc du §3).
3. Dans la Console, exécute :

   ```json
   POST _bulk
   {"index":{"_index":"news"}}
   {"category":"POLITICS","headline":"Ryan Zinke Looks To Reel Back...","authors":"Chris D'Angelo","link":"https://...","short_description":"The interior secretary...","date":"2018-05-26"}
   {"index":{"_index":"news"}}
   {"category":"POLITICS","headline":"Trump's Scottish Golf Resort...","authors":"Mary Papenfuss","link":"https://...","short_description":"And there are four times...","date":"2018-05-26"}
   ```

   Tu peux coller tout le contenu de `news.bulk.ndjson` (attention, **une requête peut être volumineuse**; en cas d’erreur 413, split en plusieurs paquets).

## Méthode C — Python (requests)

```bash
python3 - <<'PY'
import requests, sys
bulk = open('news.bulk.ndjson','rb').read()
r = requests.post('http://localhost:9200/_bulk',
                  headers={'Content-Type':'application/x-ndjson'},
                  data=bulk)
print(r.status_code, r.text[:500])
PY
```

Variante gzip :

```bash
python3 - <<'PY'
import requests, gzip
with open('news.bulk.ndjson.gz','rb') as f:
    data=f.read()
r = requests.post('http://localhost:9200/_bulk?pretty',
                  headers={'Content-Type':'application/x-ndjson','Content-Encoding':'gzip'},
                  data=data)
print(r.status_code)
print(r.text[:500])
PY
```

## Méthode D — Node.js (axios ou fetch)

```bash
node - <<'JS'
import fs from 'fs';
import axios from 'axios';
const data = fs.readFileSync('news.bulk.ndjson');
const r = await axios.post('http://localhost:9200/_bulk?pretty', data, {
  headers: {'Content-Type':'application/x-ndjson'}
});
console.log(r.status, r.data.items?.length);
JS
```

## Méthode E — Logstash (lecture de fichier → ES)

Installe Logstash dans un conteneur dédié (simple pour un test) :

```bash
mkdir -p ~/elk-news/logstash && cd ~/elk-news/logstash
cat > pipeline.conf <<'CONF'
input {
  file {
    path => "/data/raw.jsonl"
    codec => json_lines
    start_position => "beginning"
    sincedb_path => "/data/.sincedb"
  }
}
filter { }
output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "news"
  }
  stdout { codec => dots }
}
CONF
```

Compose minimal à côté d’ELK (même réseau) :

```bash
cat > docker-compose.logstash.yml <<'YAML'
services:
  logstash:
    image: docker.elastic.co/logstash/logstash:8.14.3
    container_name: ls-news
    depends_on:
      - elasticsearch
    volumes:
      - ../raw.jsonl:/data/raw.jsonl:ro
      - ./pipeline.conf:/usr/share/logstash/pipeline/logstash.conf:ro
      - lsdata:/data
    environment:
      - LS_JAVA_OPTS=-Xms512m -Xmx512m
    network_mode: "service:elasticsearch"
volumes:
  lsdata:
YAML
```

Lancer :

```bash
cd ~/elk-news/logstash
docker compose -f docker-compose.logstash.yml up
# Stoppe avec Ctrl+C quand c'est terminé
```

## Méthode F — `elasticdump` (outil pratique en conteneur)

```bash
docker run --rm -v "$PWD":/data taskrabbit/elasticsearch-dump \
  --type=bulk --input=/data/news.bulk.ndjson --output=http://host.docker.internal:9200
```

Si `host.docker.internal` n’existe pas sur Linux, utilise l’IP de la VM ou `http://172.17.0.1:9200` (selon ta config Docker).

## Méthode G — Split + ingestion parallèle (gros datasets)

```bash
# Split par paquets de 20 000 lignes (soit 10 000 docs)
split -l 20000 news.bulk.ndjson parts_
for f in parts_*; do
  curl -s -H 'Content-Type: application/x-ndjson' \
       -X POST 'http://localhost:9200/_bulk' \
       --data-binary @"$f" >/dev/null &
done
wait
curl -s 'http://localhost:9200/news/_count?pretty'
```


<br/>
<br/>


# Partie 5 - Contrôles post-ingestion - Requêtes (Annexe 1)



```bash
# 5.1. Compter
curl -s 'http://localhost:9200/news/_count?pretty'

# 5.2. Exemple de recherche
curl -s -H 'Content-Type: application/json' \
  'http://localhost:9200/news/_search?pretty' -d '{
  "size": 5,
  "query": { "match": { "headline": "Trump North Korea" } },
  "_source": ["date","category","headline","authors"]
}'

# 5.3. Vérifier le mapping
curl -s 'http://localhost:9200/news/_mapping?pretty'
```



## Dépannage ingestion

* `errors: true` dans la réponse bulk
  → Une ou plusieurs lignes posent problème (JSON invalide, champ “date” non conforme). Récupère les items en erreur et corrige.

  ```bash
  # Voir les erreurs rapidement
  curl -s -H 'Content-Type: application/x-ndjson' \
       -X POST 'http://localhost:9200/_bulk' \
       --data-binary @news.bulk.ndjson \
  | jq '.items[] | select(.index.error != null) | .index.error'
  ```

* Date non parsée
  → Ton mapping attend `yyyy-MM-dd`. Vérifie la colonne `date`.
  Option : autoriser plusieurs formats : `"format":"yyyy-MM-dd||strict_date_optional_time"`.

* Payload trop gros (413)
  → Utilise gzip (Méthode A.1) ou **split** (Méthode G).

* Timeouts
  → Ingest en paquets plus petits, ou ajoute `?timeout=2m` au bulk.




> Remarque générale
>
> * Ajoute `?pretty` pour une lecture confortable.
> * Quand tu explores, limite le bruit: `"size": 5`, `"_source": [...]`.
> * Pour chaque exemple, tu peux exécuter la version `curl` (CLI) ou coller le JSON dans Kibana Console.

## 5.A — Lecture de base / pagination

### A.1. Parcourir quelques docs

```bash
curl -s 'http://localhost:9200/news/_search?size=5&pretty' \
  -H 'Content-Type: application/json' -d '{
  "_source": ["date","category","headline","authors"]
}'
```

### A.2. Trier par date descendante

```bash
curl -s 'http://localhost:9200/news/_search?size=5&pretty' \
  -H 'Content-Type: application/json' -d '{
  "sort": [{ "date": "desc" }],
  "_source": ["date","category","headline"]
}'
```

### A.3. Pagination classique `from/size`

```bash
curl -s 'http://localhost:9200/news/_search?pretty' \
  -H 'Content-Type: application/json' -d '{
  "from": 10, "size": 10,
  "sort": [{ "date": "desc" }]
}'
```

### A.4. Pagination scalable `search_after` (recommandée)

1. Première page:

```bash
curl -s 'http://localhost:9200/news/_search?size=5&pretty' \
  -H 'Content-Type: application/json' -d '{
  "sort": [{ "date": "desc" }, {"_id":"desc"}],
  "_source": ["date","headline"]
}'
```

2. Récupère les `sort` du dernier hit, puis:

```bash
curl -s 'http://localhost:9200/news/_search?size=5&pretty' \
  -H 'Content-Type: application/json' -d '{
  "search_after": ["2018-05-24","<ID_DERNIER_HIT>"],
  "sort": [{ "date": "desc" }, {"_id":"desc"}],
  "_source": ["date","headline"]
}'
```



## 5.B — Full-text: match, phrase, multi-champs, tolérance

### B.1. `match` (analyse linguistique)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' \
  -H 'Content-Type: application/json' -d '{
  "query": { "match": { "headline": "Trump summit North Korea" } },
  "_source": ["headline","date","category"]
}'
```

### B.2. `match_phrase` (expression exacte)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' \
  -H 'Content-Type: application/json' -d '{
  "query": { "match_phrase": { "headline": "President Obama" } },
  "_source": ["headline","date","category"]
}'
```

### B.3. `minimum_should_match` (contrôler la précision)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' \
  -H 'Content-Type: application/json' -d '{
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

### B.4. `multi_match` sur plusieurs champs (pondération)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' \
  -H 'Content-Type: application/json' -d '{
  "query": {
    "multi_match": {
      "query": "President Obama",
      "fields": ["headline^3","short_description","authors"]
    }
  },
  "_source": ["headline","authors","date","category"]
}'
```

### B.5. Fuzzy (tolérance aux fautes)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' \
  -H 'Content-Type: application/json' -d '{
  "query": {
    "match": {
      "headline": {
        "query": "presdent obmaa",
        "fuzziness": "AUTO"
      }
    }
  }
}'
```

### B.6. `query_string` (syntaxe lucene: AND, OR, guillemets, wildcards)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": {
    "query_string": {
      "query": "\"North Korea\" AND Trump",
      "fields": ["headline","short_description"]
    }
  },
  "_source": ["headline","date"]
}'
```

### B.7. `simple_query_string` (plus permissif, pas d’erreur sur opérateurs)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": {
    "simple_query_string": {
      "query": "\"North Korea\" +Trump -Clinton",
      "fields": ["headline^2","short_description"]
    }
  }
}'
```


## 5.C — Exact match / keywords / préfixes

### C.1. `term` exact (sur sous-champ keyword)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": { "term": { "category.keyword": "POLITICS" } },
  "_source": ["category","headline","date"]
}'
```

### C.2. `terms` (liste)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": { "terms": { "category.keyword": ["POLITICS","WORLD NEWS"] } },
  "_source": ["category","headline","date"]
}'
```

### C.3. `prefix` (sur keyword; utile pour autocomplétions simples)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": { "prefix": { "authors.keyword": "Mary" } },
  "_source": ["authors","headline"]
}'
```

### C.4. `wildcard` et `regexp` (à utiliser avec parcimonie)

```bash
# wildcard (peut être coûteux)
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": { "wildcard": { "headline.keyword": "*Trump*" } },
  "_source": ["headline","date"]
}'
# regexp
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": { "regexp": { "authors.keyword": "M.*y.*" } },
  "_source": ["authors","headline"]
}'
```



## 5.D — Bool, filtres, ranges, boosting

### D.1. Combiner conditions

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "_source": ["date","category","headline"],
  "query": {
    "bool": {
      "must": [
        { "match": { "headline": "Trump" } }
      ],
      "filter": [
        { "terms": { "category.keyword": ["POLITICS","WORLD NEWS"] } },
        { "range": { "date": { "gte": "2018-05-24","lte": "2018-05-26" } } }
      ],
      "must_not": [
        { "match_phrase": { "short_description": "joke" } }
      ]
    }
  }
}'
```

### D.2. `boost` par champ/filtre (function_score simple)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "query": {
    "function_score": {
      "query": { "match": { "headline": "Trump" } },
      "boost_mode": "multiply",
      "score_mode": "sum",
      "functions": [
        { "filter": { "term": { "category.keyword": "POLITICS" } }, "weight": 2.0 }
      ]
    }
  },
  "_source": ["headline","category","_score"]
}'
```



## 5.E — Highlight (surlignage)

### E.1. Surligner dans le titre et la description

```bash
curl -s 'http://localhost:9200/news/_search?pretty' -H 'Content-Type: application/json' -d '{
  "_source": ["headline","short_description","date"],
  "query": { "match": { "short_description": "North Korea summit" } },
  "highlight": {
    "fields": {
      "headline": {},
      "short_description": {}
    },
    "pre_tags": ["<mark>"], "post_tags": ["</mark>"]
  }
}'
```



## 5.F — Agrégations (KPIs, facettes, stats)

### F.1. Compte par catégorie (`terms`)

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' -H 'Content-Type: application/json' -d '{
  "aggs": {
    "by_category": { "terms": { "field": "category.keyword", "size": 20 } }
  }
}'
```

### F.2. Histogramme par jour (`date_histogram`) + sous-agg `terms`

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' -H 'Content-Type: application/json' -d '{
  "aggs": {
    "per_day": {
      "date_histogram": { "field": "date", "calendar_interval": "day" },
      "aggs": {
        "by_cat": { "terms": { "field": "category.keyword", "size": 5 } }
      }
    }
  }
}'
```

### F.3. `top_hits` (ex: dernier titre par catégorie)

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' -H 'Content-Type: application/json' -d '{
  "aggs": {
    "by_category": {
      "terms": { "field": "category.keyword", "size": 10 },
      "aggs": {
        "latest": {
          "top_hits": {
            "size": 1,
            "sort": [{ "date": "desc" }],
            "_source": ["date","headline","authors"]
          }
        }
      }
    }
  }
}'
```

### F.4. Statistiques cardinalité (nb d’auteurs distincts)

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' -H 'Content-Type: application/json' -d '{
  "aggs": {
    "authors_count": { "cardinality": { "field": "authors.keyword" } }
  }
}'
```

### F.5. Percentiles (longueur titre, si tu stockes un champ numérique dérivé)

> Si tu ajoutes plus tard un champ `headline_len` (entier).

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' -H 'Content-Type: application/json' -d '{
  "aggs": {
    "p_headline": { "percentiles": { "field": "headline_len", "percents": [50,90,95,99] } }
  }
}'
```

### F.6. `significant_text` (mots “significatifs” pour un filtre donné)

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' -H 'Content-Type: application/json' -d '{
  "query": { "term": { "category.keyword": "POLITICS" } },
  "aggs": {
    "sig_terms": {
      "significant_text": {
        "field": "short_description",
        "size": 10
      }
    }
  }
}'
```

### F.7. `rare_terms` (trouver des catégories rares – utile sur gros corpus)

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' -H 'Content-Type: application/json' -d '{
  "aggs": {
    "rare_cats": {
      "rare_terms": { "field": "category.keyword", "max_doc_count": 2 }
    }
  }
}'
```

### F.8. Pipelines (ex: moyenne mobile sur un histogramme)

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' \
  -H 'Content-Type: application/json' -d '{
  "aggs": {
    "per_day": {
      "date_histogram": { "field": "date", "calendar_interval": "day" }
    },
    "moving_avg": {
      "moving_fn": {
        "buckets_path": "per_day._count",
        "window": 2,
        "script": "MovingFunctions.unweightedAvg(values)"
      }
    }
  }
}'
```

### F.9. `bucket_sort` (trier/limiter côté agg)

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' \
  -H 'Content-Type: application/json' -d '{
  "aggs": {
    "cats": {
      "terms": { "field": "category.keyword", "size": 100 },
      "aggs": {
        "by_day": { "date_histogram": { "field": "date", "calendar_interval": "day" } },
        "limit": { "bucket_sort": { "size": 5 } }
      }
    }
  }
}'
```

### F.10. `composite` (pagination d’aggregations)

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' \
  -H 'Content-Type: application/json' -d '{
  "aggs": {
    "cats": {
      "composite": {
        "size": 5,
        "sources": [
          { "cat": { "terms": { "field": "category.keyword" } } }
        ]
      }
    }
  }
}'
```

Ensuite, réutilise `after_key` de la réponse pour la page suivante:

```json
"aggs": {
  "cats": {
    "composite": {
      "size": 5,
      "after": { "cat": "ENTERTAINMENT" },
      "sources": [{ "cat": { "terms": { "field": "category.keyword" } } }]
    }
  }
}
```



## 5.G — Diversification des résultats

### G.1. `field_collapse` (éviter doublons par auteur, garder le plus récent)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' \
  -H 'Content-Type: application/json' -d '{
  "query": { "match": { "headline": "Trump" } },
  "collapse": {
    "field": "authors.keyword",
    "inner_hits": {
      "name": "by_author_latest",
      "size": 1,
      "sort": [{ "date": "desc" }]
    }
  },
  "sort": [{ "date": "desc" }],
  "_source": ["authors","headline","date"]
}'
```

### G.2. `sampler` (échantillonnage rapide avant agg)

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' \
  -H 'Content-Type: application/json' -d '{
  "aggs": {
    "sample": {
      "sampler": { "shard_size": 100 },
      "aggs": {
        "cats": { "terms": { "field": "category.keyword", "size": 10 } }
      }
    }
  }
}'
```



## 5.H — Suggester / Autocomplétion

> Nécessite un champ `completion` si tu veux un vrai suggest. Variante simple: prefix/wildcard sur `headline.keyword`.

### H.1. Autocomplete simple (prefix)

```bash
curl -s 'http://localhost:9200/news/_search?size=10&pretty' \
  -H 'Content-Type: application/json' -d '{
  "query": { "prefix": { "headline.keyword": "Harvey" } },
  "_source": ["headline"]
}'
```

### H.2. Suggesteur orthographique (term suggester)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' \
  -H 'Content-Type: application/json' -d '{
  "suggest": {
    "typos": {
      "text": "Presdent Obmaa",
      "term": { "field": "headline" }
    }
  }
}'
```

### H.3. Phrase suggester (contextuel)

```bash
curl -s 'http://localhost:9200/news/_search?pretty' \
  -H 'Content-Type: application/json' -d '{
  "suggest": {
    "p": {
      "text": "North Koria summit",
      "phrase": {
        "field": "short_description",
        "max_errors": 2
      }
    }
  }
}'
```

> Pour un **vrai autocomplete UX**, ajoute un champ `headline_suggest` de type `completion` au mapping, réindexe, puis:

```json
"suggest": {
  "headline_auto": {
    "prefix": "Har",
    "completion": { "field": "headline_suggest", "size": 10 }
  }
}
```



## 5.I — Analyseurs FR/EN (option bonus)

> Si tu veux de meilleurs résultats multilingues: définis des sous-champs avec analyzers spécifiques.

### I.1. Exemple de mapping (à appliquer avant ingestion)

```json
"mappings": {
  "properties": {
    "headline": {
      "type": "text",
      "fields": {
        "fr": { "type": "text", "analyzer": "french" },
        "en": { "type": "text", "analyzer": "english" },
        "keyword": { "type": "keyword", "ignore_above": 256 }
      }
    }
  }
}
```

### I.2. Requête langue ciblée

```bash
curl -s 'http://localhost:9200/news/_search?pretty' \
  -H 'Content-Type: application/json' -d '{
  "query": { "match": { "headline.en": "election interference" } },
  "_source": ["headline","date"]
}'
```



## 5.J — Contrôle d’indexation / rafraîchissement

### J.1. Forcer un refresh (tests rapides)

```bash
curl -s -X POST 'http://localhost:9200/news/_refresh?pretty'
```

### J.2. Récupérer uniquement `_source` (par ID)

```bash
# suppose que tu connais l'ID
curl -s 'http://localhost:9200/news/_source/<ID>?pretty'
```



## 5.K — Mises à jour / réindexations (utile en TP avancé)

### K.1. `_update_by_query` (ex: normaliser catégorie)

```bash
curl -s -X POST 'http://localhost:9200/news/_update_by_query?pretty' \
  -H 'Content-Type: application/json' -d '{
  "script": {
    "source": "ctx._source.category = ctx._source.category.toUpperCase();",
    "lang": "painless"
  },
  "query": { "match_all": {} }
}'
```

### K.2. `_reindex` (copier dans un nouvel index avec nouveau mapping)

```bash
curl -s -X POST 'http://localhost:9200/_reindex?pretty' \
  -H 'Content-Type: application/json' -d '{
  "source": { "index": "news" },
  "dest":   { "index": "news_v2" }
}'
```



## 5.L — Sélections utiles pour Kibana

### L.1. Top 10 catégories

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' \
  -H 'Content-Type: application/json' -d '{
  "aggs": { "cats": { "terms": { "field": "category.keyword", "size": 10 } } }
}'
```

### L.2. Courbe d’articles par jour

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' \
  -H 'Content-Type: application/json' -d '{
  "aggs": { "per_day": { "date_histogram": { "field": "date", "calendar_interval": "day" } } }
}'
```

### L.3. Derniers titres par catégorie (pour table “Top N”)

```bash
curl -s 'http://localhost:9200/news/_search?size=0&pretty' \
  -H 'Content-Type: application/json' -d '{
  "aggs": {
    "by_category": {
      "terms": { "field": "category.keyword", "size": 10 },
      "aggs": {
        "latest": {
          "top_hits": {
            "size": 3,
            "sort": [{ "date": "desc" }],
            "_source": ["date","headline","authors","link"]
          }
        }
      }
    }
  }
}'
```



### Récap express

* **Recherche**: `match`, `match_phrase`, `multi_match`, `query_string`, `fuzzy`.
* **Filtres**: `term/terms`, `range`, `bool`.
* **Classement**: `sort`, `function_score`, `collapse`.
* **Surlignage**: `highlight`.
* **Facettes/KPIs**: `terms`, `date_histogram`, `top_hits`, `cardinality`, `percentiles`, `significant_text`, `rare_terms`, pipelines (`moving_fn`, `bucket_sort`, `composite`).
* **Suggester**: `term`, `phrase`, completion (si champ dédié).
* **Bonus**: analyzers FR/EN, `search_after`, refresh, `_update_by_query`, `_reindex`.


























<br/>
<br/>




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



<br/>
<br/>





# ANNEXE 1 — Importer des **données volumineuses** dans Elasticsearch

## 0) Préparer l’index pour un gros chargement (accélération)

Avant de charger des millions de lignes NDJSON :

```bash
# 0.1 Désactiver/relaxer le refresh & les replicas pendant l’import
curl -s -X PUT 'http://localhost:9200/news/_settings' -H 'Content-Type: application/json' -d '{
  "index": {
    "number_of_replicas": 0,
    "refresh_interval": "-1"      # pas de refresh pendant l’import
  }
}'
```

Après l’import, **remets des valeurs normales** :

```bash
curl -s -X PUT 'http://localhost:9200/news/_settings' -H 'Content-Type: application/json' -d '{
  "index": {
    "number_of_replicas": 1,
    "refresh_interval": "1s"
  }
}'
curl -s -X POST 'http://localhost:9200/news/_refresh'
```

Rappels pratiques :

* **Taille des bulk** : vise ~5–15 Mo par requête, 1k à 5k actions (paires action+source) par fichier.
* **Limite HTTP** par défaut ≈ 100 Mo (`http.max_content_length`). Reste bien en dessous.



## 1.1) Méthode 1 — **Découper** ton NDJSON puis charger en série (robuste, simple)

Supposons ton gros fichier `news.bulk.ndjson` (déjà formaté en `_bulk` : ligne action, puis ligne source).

```bash
# 1.1 Découper en morceaux de 5 000 lignes (≈ 2 500 docs bulk)
mkdir -p chunks
split -l 5000 --numeric-suffixes=1 --additional-suffix=.ndjson news.bulk.ndjson chunks/part_

# 1.2 Import séquentiel
for f in chunks/part_*; do
  echo "Import: $f"
  curl -s -H 'Content-Type: application/x-ndjson' \
       -X POST 'http://localhost:9200/_bulk' \
       --data-binary @"$f" | jq '.errors' || true
done

# 1.3 Vérifier le décompte
curl -s 'http://localhost:9200/news/_count?pretty'
```

Astuce : si tu n’as **pas** mis l’action dans le NDJSON, ajoute-la avant (transforme `raw.jsonl` → `news.bulk.ndjson` comme montré plus haut).



## 1.2) Méthode 2 — **GZIP streaming** (plus rapide, moins d’I/O)

Elasticsearch accepte le gzip si tu indiques l’entête.

```bash
# 2.1 En direct (sans fichiers intermédiaires)
gzip -c news.bulk.ndjson | \
curl -s -H 'Content-Type: application/x-ndjson' \
       -H 'Content-Encoding: gzip' \
       -X POST 'http://localhost:9200/_bulk?pretty' \
       --data-binary @-

# 2.2 Ou chunk + gzip
for f in chunks/part_*; do
  gzip -c "$f" | curl -s \
    -H 'Content-Type: application/x-ndjson' \
    -H 'Content-Encoding: gzip' \
    -X POST 'http://localhost:9200/_bulk' \
    --data-binary @- | jq '.errors'
done
```



## 1.3) Méthode 3 — **Paralléliser** prudemment (GNU parallel)

Convient si ta VM a des cœurs CPU libres. Commence modeste (2 jobs), surveille les erreurs/latences, ajuste.

```bash
# Installer parallel si besoin
sudo apt-get update && sudo apt-get install -y parallel

ls chunks/part_* | parallel -j2 '
  echo "Bulk {}";
  curl -s -H "Content-Type: application/x-ndjson" -X POST "http://localhost:9200/_bulk" --data-binary @{} \
  | jq -r ".errors"
'
```

Règle d’or : si tu vois `429 Too Many Requests` ou des temps de réponse qui grimpent, **réduis `-j`** et/ou **réduis la taille des chunks**.



## 1.4) Méthode 4 — **Python (elasticsearch-py) streaming** avec helpers `bulk`

Idéal pour transformer à la volée, gérer les erreurs, re-tenter, etc.

```bash
# 4.1 Installer
pip install elasticsearch==8.* tqdm

# 4.2 Script import_bulk.py
cat > import_bulk.py <<'PY'
from elasticsearch import Elasticsearch, helpers
from tqdm import tqdm
import json, sys

ES = Elasticsearch("http://localhost:9200")
INDEX = "news"

def gen_actions(path):
    with open(path, "r", encoding="utf-8") as f:
        for line in f:
            doc = json.loads(line)
            # adapte si ton fichier n'a PAS les lignes "action"
            # ici: on part d'un "raw.jsonl" (1 doc/ligne) et on crée l'action Python
            yield {
                "_index": INDEX,
                "_source": doc
            }

if __name__ == "__main__":
    # Exemple 1: importer raw.jsonl (sans lignes action)
    # actions = gen_actions("raw.jsonl")

    # Exemple 2: si tu as déjà un NDJSON "action+source" (news.bulk.ndjson),
    #            passe plutôt par une pré-lecture pour ignorer les lignes "action"
    #            et ne garder que les sources, ou lis le raw.jsonl.

    actions = gen_actions("raw.jsonl")
    # chunk_size ~ 1000–5000
    helpers.bulk(ES, actions, chunk_size=2000, request_timeout=120)
PY

# 4.3 Lancer
python3 import_bulk.py
```

Variantes :

* Si tu **dois** lire `news.bulk.ndjson` (action+source), parse une ligne sur deux et ne construis que `_source`.
* Utilise `tqdm` pour une barre de progression sur un fichier découpé; sinon, ajoute un compteur.


## 1.5) Méthode 5 — **Node.js** (client officiel, flux)

```bash
# 5.1 Installer
npm init -y
npm install @elastic/elasticsearch@8 p-limit

# 5.2 Script import_bulk.mjs
cat > import_bulk.mjs <<'JS'
import { createReadStream } from 'node:fs';
import { createInterface } from 'node:readline';
import { Client } from '@elastic/elasticsearch';

const client = new Client({ node: 'http://localhost:9200' });
const INDEX = 'news';

async function runRawJsonl(path) {
  const rl = createInterface({ input: createReadStream(path), crlfDelay: Infinity });
  const ops = [];
  for await (const line of rl) {
    const doc = JSON.parse(line);
    ops.push({ index: { _index: INDEX } });
    ops.push(doc);
    if (ops.length >= 2000 * 2) {                 // 2000 docs
      await client.bulk({ operations: ops });
      ops.length = 0;
    }
  }
  if (ops.length) await client.bulk({ operations: ops });
}

await runRawJsonl('raw.jsonl');
JS

# 5.3 Lancer
node import_bulk.mjs
```

Ajuste le lot `2000` selon ta VM.



## 1.6) Méthode 6 — **Logstash** (dockerisé) pour très gros flux

Simple pipeline : lit un NDJSON et pousse en bulk automatiquement.

### 6.1 Docker Compose (service logstash optionnel)

```yaml
services:
  logstash:
    image: docker.elastic.co/logstash/logstash:8.14.3
    container_name: ls-news
    volumes:
      - ./pipelines:/usr/share/logstash/pipeline
      - ./data:/data          # place ici tes fichiers NDJSON
    depends_on:
      elasticsearch:
        condition: service_healthy
    environment:
      - LS_JAVA_OPTS=-Xms512m -Xmx512m
```

### 6.2 Pipeline simple `pipelines/news.conf`

```conf
input {
  file {
    path => "/data/news.ndjson"
    codec => json_lines
    start_position => "beginning"
    sincedb_path => "/usr/share/logstash/.sincedb_news"
  }
}
filter { }
output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "news"
    action => "index"
    # template/mapping si besoin
  }
  stdout { codec => dots }
}
```

### 6.3 Lancer

```bash
mkdir -p pipelines data
# place ton fichier /data/news.ndjson
docker compose up -d logstash
docker logs -f ls-news
```



## 1.7) Méthode 7 — **Snapshot/Restore** (si tu as déjà un index prêt ailleurs)

Plutôt que réindexer des millions de lignes, tu peux **restaurer** un snapshot.

1. Déclare un repo snapshot (FS) et copie le snapshot :

```bash
# sur la machine "source", snapshot de l'index news
# sur cette VM, configure un repo FS:
sudo mkdir -p /srv/es-repo
sudo chown 1000:1000 /srv/es-repo

curl -X PUT 'http://localhost:9200/_snapshot/localrepo' -H 'Content-Type: application/json' -d '{
  "type": "fs",
  "settings": { "location": "/srv/es-repo" }
}'
```

2. Restaure :

```bash
curl -X POST 'http://localhost:9200/_snapshot/localrepo/snap_news/_restore' \
  -H 'Content-Type: application/json' -d '{
  "indices": "news",
  "include_global_state": false
}'
```



## 1.8) Suivi & monitoring pendant l’import

```bash
# Santé & docs
curl -s 'http://localhost:9200/_cluster/health?pretty'
curl -s 'http://localhost:9200/news/_count?pretty'

# Voir les tâches en cours (dont _bulk)
curl -s 'http://localhost:9200/_tasks?detailed=true&actions=*bulk*&pretty'

# Stats d’index
curl -s 'http://localhost:9200/news/_stats?pretty'

# Cat API (rapide)
curl -s 'http://localhost:9200/_cat/indices?v'
curl -s 'http://localhost:9200/_cat/thread_pool?v'
```



## 1.9) Erreurs fréquentes & corrections

* **`"errors": true` dans la réponse `_bulk`**
  Inspecte les éléments renvoyés : souvent un JSON mal formé (guillemets, encodage, tab au lieu d’espace).
  Solution : localise la ligne fautive (par dichotomie ou `jq -c`), corrige et rejoue le chunk.

* **`413 Request Entity Too Large`**
  Ton payload bulk est trop gros. **Réduis la taille des chunks** (moins de lignes, < 10–20 Mo par POST).

* **`429 Too Many Requests` / Rejections**
  Tu pousses trop vite (trop de jobs ou chunks trop gros). **Réduis `-j`** (parallel) et/ou la taille des lots.
  Attends, puis reprends où tu t’es arrêté.

* **`Timeout`**
  Utilise `request_timeout` plus élevé (clients Python/Node) ou **réduis** la taille des lots.

* **Index “jaune/rouge”** après import
  Vérifie `/_cluster/health`. Reviens à `number_of_replicas` normal, `/_refresh`, puis `/_forcemerge` si besoin :

  ```bash
  curl -s -X POST 'http://localhost:9200/news/_forcemerge?max_num_segments=1&pretty'
  ```

  (à faire hors charge, optionnel, pour réduire le nombre de segments)



## 1.10) Checklist “import massif”

1. Créer l’index avec mapping correct.
2. `number_of_replicas: 0`, `refresh_interval: -1` pour l’import.
3. Choisir une **méthode** (split + curl, gzip, Python, Node, Logstash).
4. **Petits lots**, tester, puis monter progressivement.
5. Surveiller `_tasks`, `_stats`, `_cat/indices`.
6. Après import: remettre `replicas`, `refresh_interval`, `/_refresh`.
7. (Option) `/_forcemerge` hors charge.
8. Sauvegarder un **snapshot** une fois le corpus chargé.



Avec ces méthodes, nous pouvons ingérer de très gros volumes de données sur notre VM Ubuntu sans nous bloquer : commençons simple (split + curl), passons au gzip/parallel si nécessaire, puis au streaming Python/Node ou Logstash pour les très grands jeux, tout en surveillant la santé du cluster et en appliquant les bons réglages d’indexation.



<br/>
<br/>

# Annexe 2 pour l'étape 2 (PARTIE 1) — Préparer le NDJSON pour `_bulk`

## 2.1. Mettre tes **lignes JSON brutes** dans un fichier

Objectif : avoir **un article par ligne**, JSON valide, encodé **UTF-8**, sans virgule finale, sans crochets `[]`.

```bash
cd ~/elk-news
cat > raw.jsonl <<'JSON'
{"category":"POLITICS","headline":"Ryan Zinke Looks To Reel Back Some Critics With 'Grand Pivot' To Conservation","authors":"Chris D'Angelo","link":"https://www.huffingtonpost.com/entry/ryan-zinke-reel-back-critics-grand-pivot-conservation_us_5b086c78e4b0fdb2aa538b3f","short_description":"The interior secretary attempts damage control...","date":"2018-05-26"}
{"category":"POLITICS","headline":"Trump's Scottish Golf Resort Pays Women Significantly Less Than Men: Report","authors":"Mary Papenfuss","link":"https://www.huffingtonpost.com/entry/trump-scottish-golf-resort-pays-women-less-than-men_us_5b08ca29e4b0802d69cb4d37","short_description":"And there are four times as many male as female executives.","date":"2018-05-26"}
# ... colle toutes tes lignes ici (une par article)
JSON
```

### Vérifications de base (fortement recommandé)

1. Chaque ligne est un JSON valide :

```bash
# Affiche les lignes invalides (si rien ne s’affiche, tout va bien)
nl -ba raw.jsonl | while IFS= read -r line; do
  num="${line%%\t*}"; json="${line#*$'\t'}"
  echo "$json" | jq -e . >/dev/null 2>&1 || echo "Ligne $num invalide"
done
```

2. Compter le nombre de lignes :

```bash
wc -l raw.jsonl
# → ce nombre = nombre de documents à indexer
```

3. Aperçu rapide (début/fin) :

```bash
head -n3 raw.jsonl
tail -n3 raw.jsonl
```

4. Supprimer les retours Windows (si tu as copié depuis Windows) :

```bash
sed -i 's/\r$//' raw.jsonl
```



## 2.2. Transformer en **NDJSON pour `_bulk`**

Le format `_bulk` exige **deux lignes par document** :

* une ligne **action** : `{"index":{"_index":"news"}}`
* une ligne **source** : ton JSON

### Option A — Commande la plus simple (awk)

```bash
awk '{print "{\"index\":{\"_index\":\"news\"}}"; print}' raw.jsonl > news.bulk.ndjson
```

Vérifier que tu as bien le **double de lignes** :

```bash
wc -l raw.jsonl news.bulk.ndjson
# news.bulk.ndjson doit avoir 2x plus de lignes que raw.jsonl
```

Aperçu (tu dois voir une alternance action/source) :

```bash
head -n6 news.bulk.ndjson
```

### Option B — Ajouter un **_id** déterministe (ex. hash SHA1 du titre)

Utile pour éviter les doublons si tu rejoues l’import.

```bash
awk '
  {
    cmd = "printf %s \"" $0 "\" | sha1sum | cut -d\" \" -f1";
    cmd | getline h; close(cmd);
    print "{\"index\":{\"_index\":\"news\",\"_id\":\"" h "\"}}";
    print $0
  }
' raw.jsonl > news.bulk.withid.ndjson
```

### Option C — Avec `jq` (si tu veux modifier/nettoyer au passage)

Exemple : forcer la date au format `yyyy-MM-dd` (si déjà bon, ça ne change rien) :

```bash
jq -c '
  .date |= ( . | split("T")[0] )
  | {index:{_index:"news"}}, .
' raw.jsonl > news.bulk.ndjson
```

> `jq` produit déjà la séquence action/source correctement (une paire par document).

### Option D — Fichier très gros : **compression** et **découpage**

1. Compresser :

```bash
gzip -c news.bulk.ndjson > news.bulk.ndjson.gz
ls -lh news.bulk.ndjson*
```

2. Découper en morceaux de 5 000 lignes (≈ 2 500 docs) pour import progressif :

```bash
mkdir -p chunks
split -l 5000 --numeric-suffixes=1 --additional-suffix=.ndjson news.bulk.ndjson chunks/part_
ls -l chunks | head
```

> Règle : vise des fichiers **≤ 10–20 Mo** par envoi (selon ta VM).


## 2.3. Charger le NDJSON (premier test)

Test minimal (sur un petit extrait) :

```bash
# créer un mini-fichier de test (2 docs)
head -n4 news.bulk.ndjson > test.bulk.ndjson

# indexation de test
curl -s -H 'Content-Type: application/x-ndjson' \
     -X POST 'http://localhost:9200/_bulk?pretty' \
     --data-binary @test.bulk.ndjson
```

Vérifier les compteurs :

```bash
curl -s 'http://localhost:9200/news/_count?pretty'
```

Si tout est bon, **charge l’ensemble** :

```bash
curl -s -H 'Content-Type: application/x-ndjson' \
     -X POST 'http://localhost:9200/_bulk?pretty' \
     --data-binary @news.bulk.ndjson | jq '.errors'
# doit afficher: false
```

### Variante gzip (plus rapide, moins d’I/O)

```bash
gzip -c news.bulk.ndjson | \
curl -s -H 'Content-Type: application/x-ndjson' \
       -H 'Content-Encoding: gzip' \
       -X POST 'http://localhost:9200/_bulk?pretty' \
       --data-binary @- | jq '.errors'
```

### Variante par **chunks** (recommandée pour gros volumes)

```bash
for f in chunks/part_*; do
  echo "Import: $f"
  curl -s -H 'Content-Type: application/x-ndjson' \
       -X POST 'http://localhost:9200/_bulk' \
       --data-binary @"$f" | jq '.errors'
done

curl -s 'http://localhost:9200/news/_count?pretty'
```



## 2.4. Contrôles rapides post-import

```bash
# 5 documents au hasard
curl -s 'http://localhost:9200/news/_search' -H 'Content-Type: application/json' -d '{
  "size": 5, "_source": ["date","category","headline","authors"]
}' | jq '.hits.hits[]._source'

# Compte total
curl -s 'http://localhost:9200/news/_count?pretty'
```



## 2.5. Dépannage express

* **`"errors": true` dans la réponse bulk**
  Cherche la première erreur dans la réponse (champ `items`). Souvent : JSON mal formé (guillemets, caractères spéciaux non échappés), champ `date` invalide, ou un retour Windows `\r`.

  * Localiser la ligne fautive par dichotomie : coupe ton fichier en deux et reteste.
  * Vérifier l’encodage : `file -bi raw.jsonl` doit contenir `charset=utf-8`.
  * Nettoyer `\r` : `sed -i 's/\r$//' raw.jsonl` puis régénérer le bulk.

* **Payload trop gros (`413 Request Entity Too Large`)**
  Découpe en **plus petits chunks** (`split -l 5000` ou moins).

* **Trop de requêtes (`429`)**
  Réduis la taille des chunks et/ou fais l’import **séquentiel** (pas de parallélisme).

* **Dates refusées**
  Ton mapping attend `yyyy-MM-dd`. Assure-toi que `date` a ce format (cf. filtre `jq` plus haut).

* **Doublons**
  Utilise l’option **_id déterministe** (Option B) pour écraser/éviter les doublons.



## 2.6. Récap ultra-court

1. `raw.jsonl` = 1 doc JSON valide par ligne.
2. Transforme en `_bulk` : **action + source** (awk ou jq).
3. Teste sur **4 lignes** (`head -n4 …`) puis charge tout.
4. Pour gros fichiers : **split** en chunks, **gzip** possible.
5. Vérifie `_count`, puis passe à la suite (requêtes, aggrégations, Kibana).


<br/>
<br/>




# Annexe 2 pour l'étape 2 (PARTIE 2) —Importer via l’interface Kibana (Upload a file)

## A) Ce qu’il faut préparer

* Utilise **le fichier brut** `raw.jsonl` (une **ligne JSON = 1 article**).
  **Ne pas** utiliser le fichier `_bulk` (avec lignes `{"index":…}`) pour cette méthode.
* Assure-toi que le champ `date` est au format `YYYY-MM-DD`.

## B) Chemin exact dans Kibana

1. Ouvre **[http://localhost:5601](http://localhost:5601)**.
2. Menu latéral → **Analytics** → **Discover** (ou **Machine Learning** → **Data Visualizer** selon ta version)
3. Clique **Upload a file** (ou **File Data Visualizer** → **Select file**).
4. Glisse-dépose `raw.jsonl` ou clique **Select file**.

> Kibana lit **CSV/TSV/NDJSON**. Ton `raw.jsonl` est un **NDJSON** (un JSON par ligne).

## C) Assistant d’import (écran par écran)

1. **Preview & Parse**

   * Vérifie l’aperçu des champs.
   * Si la **date** n’est pas reconnue, choisis **Set time field → date** puis **Format → yyyy-MM-dd**.
2. **Transform (facultatif)**

   * Tu peux renommer des champs ou exclure des colonnes si besoin.
3. **Index settings**

   * **Index name** : `news` (ou `news-raw` si tu veux garder `news` pour un mapping avancé).
   * **Create index pattern** : coche pour créer le data view automatiquement.
   * Laisse les autres options par défaut pour commencer.
4. **Import**

   * Clique **Import** et attends la fin (Kibana crée un index, un data view et un ingest pipeline minimal si nécessaire).

## D) Vérifier immédiatement

* Va dans **Discover** → choisis le data view (souvent `news*`).
* Mets le time picker sur un intervalle large (ex. **Last 15 years**).
* Tu dois voir tes documents, filtrables par `category`, `authors`, etc.



# Variante 2 — Import dans un **index existant** (mapping maîtrisé)

Si tu veux **imposer ton mapping** (analyzers, sous-champs keyword, normalizer…), fais d’abord la création d’index (une seule fois) puis **uploade dedans**.

1. Crée l’index `news` (mapping propre) via **Kibana → Dev Tools → Console** :

```json
PUT news
{
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
          "keyword":       { "type": "keyword" },
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
}
```

2. Reviens à **Upload a file** et, dans l’étape **Index settings**, choisis **Use existing index** → saisis **`news`**.
   Kibana enverra les documents dans **ton** index (avec **tes** types/analyzers).



# Variante 3 — Interface **Logstash** (assistant visuel)

Si tu préfères un pipeline graphique :

1. **Stack Management** → **Ingest Pipelines** → **Create pipeline** (facultatif si tu dois nettoyer la date/texte).
2. **Integrations** → **Logstash** → suis l’assistant (il te donne un `pipelines.conf`).
3. Dans **/etc/logstash/conf.d/news.conf** (sur ta VM), configure un input **file** sur `raw.jsonl` (codec json_lines), output **Elasticsearch** index `news`.
4. Démarre Logstash.
   C’est plus lourd, mais tout par interface/assistants; utile pour **très gros fichiers**.



# Conseils pratiques (interface)

* **Fichier trop gros** via navigateur :
  Passe par **Settings → Advanced Settings** et augmente `xpack.fileUpload.maxFileSize` (si ta version l’expose). Sinon, coupe le fichier (ex. 50–100 MB) avec `split` et uploade en **plusieurs fois**.
* **Dates non reconnues** : dans l’assistant, force **Time field = date** et **format yyyy-MM-dd**.
* **Duplication** : si tu réimportes le même fichier, tu auras des doublons (Kibana ne met pas d’`_id` déterministe). Pour éviter ça, importe dans un index **neuf** (ex. `news_v2`), ou fais du pré-traitement côté VM pour générer un `_id` (méthode API ou Logstash).
* **Encodage** : garde le fichier en **UTF-8** et supprime les `\r` Windows : `sed -i 's/\r$//' raw.jsonl`.


# Check-list ultra-courte

1. Fichier **`raw.jsonl`** = un JSON par ligne.
2. Kibana → **Upload a file** → sélectionne ton fichier.
3. Time field **date**, format **yyyy-MM-dd**.
4. **Index name** = `news` (ou `news-raw`), **Create index pattern** coché.
5. **Import**, puis **Discover** pour vérifier.

