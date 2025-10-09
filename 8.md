# 0) Dossier de projet

```bash
mkdir elk-dev && cd elk-dev
```

# 1) `.env` — variables (modifiez le mot de passe si vous voulez)

```bash
cat > .env << 'ENV'
STACK_VERSION=8.19.5
ELASTIC_PASSWORD=changeme123!
# Limites mémoire confortables pour une petite VM
ES_JAVA_OPTS=-Xms1g -Xmx1g
KIBANA_ENCRYPTION_KEY1=change_me_to_a_very_long_random_string_key_1_________
KIBANA_ENCRYPTION_KEY2=change_me_to_a_very_long_random_string_key_2_________
KIBANA_ENCRYPTION_KEY3=change_me_to_a_very_long_random_string_key_3_________
ENV
```

# 2) `docker-compose.yml` — stack minimal (sécurité ON, HTTP sans TLS)

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    container_name: es01
    environment:
      - node.name=es01
      - discovery.type=single-node
      - xpack.security.enabled=true
      # HTTP en clair (dev) pour éviter les certificats
      - xpack.security.http.ssl.enabled=false
      # Transport en clair (OK pour single-node en dev)
      - xpack.security.transport.ssl.enabled=false
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - ES_JAVA_OPTS=${ES_JAVA_OPTS}
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    healthcheck:
      test: [ "CMD-SHELL", "curl -s -u elastic:${ELASTIC_PASSWORD} http://localhost:9200 >/dev/null" ]
      interval: 10s
      timeout: 5s
      retries: 30

  kibana:
    image: docker.elastic.co/kibana/kibana:${STACK_VERSION}
    container_name: kib01
    depends_on:
      elasticsearch:
        condition: service_healthy
    environment:
      - SERVER_NAME=kibana
      - SERVER_HOST=0.0.0.0
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      # Connexion directe avec l'admin "elastic" (simple pour dev)
      - ELASTICSEARCH_USERNAME=elastic
      - ELASTICSEARCH_PASSWORD=${ELASTIC_PASSWORD}
      # Clés d’encryption (obligatoires pour certaines features)
      - XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY=${KIBANA_ENCRYPTION_KEY1}
      - XPACK_REPORTING_ENCRYPTIONKEY=${KIBANA_ENCRYPTION_KEY2}
      - XPACK_SECURITY_ENCRYPTIONKEY=${KIBANA_ENCRYPTION_KEY3}
    ports:
      - "5601:5601"
    healthcheck:
      test: [ "CMD-SHELL", "curl -s http://localhost:5601/api/status | grep -q 'available' || exit 1" ]
      interval: 10s
      timeout: 5s
      retries: 30

volumes:
  es-data:
```

> Pourquoi **sans TLS** : en dev/local, ça supprime les problèmes de certificats. La sécurité (auth) reste activée.
> En prod, activez TLS (http + transport) et n’utilisez pas le compte `elastic` dans Kibana.

# 3) Démarrage

```bash
docker compose up -d
docker compose ps
# Attendre "healthy" sur les 2 services :
docker compose logs -f
```

---

# 4) URLs

* Elasticsearch : `http://localhost:9200`
* Kibana : `http://localhost:5601`
  Identifiants : `elastic` / **${ELASTIC_PASSWORD}** (défini dans `.env`)

---

# 5) Cheat-sheet des commandes (version Docker/dev, **auth obligatoire**, **HTTP**)

> Remplacez `<nom_index>`, `<id>`, `<champ>`, `<valeur>`.
> Utilise le mot de passe défini dans `.env` (ici `changeme123!`).

### 1) Infos cluster

```bash
curl -u elastic:changeme123! http://localhost:9200/
```

### 2) Lister les indices

```bash
curl -u elastic:changeme123! "http://localhost:9200/_cat/indices?v"
```

### 3) État des nœuds

```bash
curl -u elastic:changeme123! "http://localhost:9200/_cat/nodes?v"
```

### 4) Statistiques des indices (tous)

```bash
curl -u elastic:changeme123! "http://localhost:9200/_stats"
```

### 5) Rechercher dans un index

```bash
curl -u elastic:changeme123! "http://localhost:9200/<nom_index>/_search?q=<champ>:<valeur>"
```

### 6) Mapping d’un index

```bash
curl -u elastic:changeme123! "http://localhost:9200/<nom_index>/_mapping"
```

### 7) Ajouter / indexer un document (ID fixé)

```bash
curl -u elastic:changeme123! -X PUT "http://localhost:9200/<nom_index>/_doc/<id>" \
  -H 'Content-Type: application/json' -d '{"<champ>":"<valeur>"}'
```

*(Variante ID auto : `POST http://localhost:9200/<nom_index>/_doc`)*

### 8) Mettre à jour partiellement un document

```bash
curl -u elastic:changeme123! -X POST "http://localhost:9200/<nom_index>/_update/<id>" \
  -H 'Content-Type: application/json' -d '{"doc":{"<champ>":"<nouvelle_valeur>"}}'
```

### 9) Supprimer un document

```bash
curl -u elastic:changeme123! -X DELETE "http://localhost:9200/<nom_index>/_doc/<id>"
```

### 10) Supprimer un index

```bash
curl -u elastic:changeme123! -X DELETE "http://localhost:9200/<nom_index>"
```

---

## (Optionnel) Utiliser un **Service Account Token** dans Kibana (au lieu de `elastic`)

Si vous préférez que Kibana se connecte via un **token** (pratique, scope limité) :

1. Après que **ES est healthy**, créez un token dans le conteneur ES :

```bash
docker exec -it es01 bash -lc "/usr/share/elasticsearch/bin/elasticsearch-service-tokens create elastic/kibana kibana-token"
# Notez la longue valeur retournée après le '='
```

2. Appliquez-le à Kibana (en remplaçant le mot de passe) :

```bash
docker compose down
# Ajoutez dans docker-compose.yml, service kibana -> environment :
#   - ELASTICSEARCH_SERVICEACCOUNTTOKEN=<LE_TOKEN>
#   (et supprimez ELASTICSEARCH_USERNAME/ELASTICSEARCH_PASSWORD)
# Exemple :
#   - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
#   - ELASTICSEARCH_SERVICEACCOUNTTOKEN=AAEAAWVsYXN0aWMv...
docker compose up -d
```

> Avec Kibana 8.x, la variable d’env **ELASTICSEARCH_SERVICEACCOUNTTOKEN** est supportée.
> En prod, laissez TLS activé et chargez votre CA, mais pour ce tuto dev on reste en HTTP.

---

## Maintenance rapide

### Logs live

```bash
docker compose logs -f elasticsearch
docker compose logs -f kibana
```

### Redémarrer

```bash
docker compose restart elasticsearch kibana
```

### Arrêter / supprimer

```bash
docker compose down     # stoppe et garde les volumes (données)
docker compose down -v  # supprime aussi les volumes (réinitialise tout)
```

---

### FAQ ultra-courte

* **Kibana n’est pas “available”** : attendez que `elasticsearch` soit `healthy`.
* **401 sur ES** : c’est normal sans `-u`. Ajoutez `-u elastic:…`.
* **Indices rouges** : voir `/_cluster/health?pretty` et `/_cat/indices?v`.
* **Peu de RAM** : baissez `ES_JAVA_OPTS` à `-Xms512m -Xmx512m`.



### Annexe : libèrer **9200/9300 (Elasticsearch)** et **5601 (Kibana)** proprement avant Docker Compose.

## A) Arrêter tout ce qui peut réoccuper les ports (services & conteneurs)

```bash
# 1) Stopper/empêcher les services système
sudo systemctl stop elasticsearch kibana 2>/dev/null || true
sudo systemctl disable elasticsearch kibana 2>/dev/null || true
sudo systemctl mask elasticsearch kibana 2>/dev/null || true   # évite un redémarrage auto

# 2) Stopper d’éventuels conteneurs existants
docker ps --format 'table {{.Names}}\t{{.Ports}}'
docker compose down 2>/dev/null || true
docker stop $(docker ps -q) 2>/dev/null || true
```

## B) Trouver qui occupe les ports

```bash
# Vue rapide
sudo ss -ltnp | egrep ':(9200|9300|5601)\b' || echo "OK: aucun process trouvé"

# Détail avec lsof (si non installé: sudo apt -y install lsof)
sudo lsof -iTCP:9200 -sTCP:LISTEN -nP || true
sudo lsof -iTCP:9300 -sTCP:LISTEN -nP || true
sudo lsof -iTCP:5601 -sTCP:LISTEN -nP || true
```

## C) Killer proprement (puis fort si besoin)

```bash
# 1) Tentative douce
sudo fuser -k -n tcp 9200 9300 5601 2>/dev/null || true

# 2) Si un PID résiste, kill ciblé (remplacez <PID> par les PIDs vus avec ss/lsof)
# d’abord TERM (gracieux), puis KILL si nécessaire
sudo kill <PID> 2>/dev/null || true
sleep 1
sudo kill -9 <PID> 2>/dev/null || true
```

## D) Vérifier que les ports sont libres

```bash
sudo ss -ltnp | egrep ':(9200|9300|5601)\b' || echo "OK: 9200/9300/5601 sont libres"
```

## E) (Optionnel) Nettoyages utiles

```bash
# PIDs orphelins de services
sudo rm -f /run/elasticsearch/*.pid /run/kibana/kibana.pid 2>/dev/null || true

# Réseau Docker laissé en plan (rarement utile)
docker network prune -f 2>/dev/null || true
```

---

### Ensuite, tu peux lancer la stack Docker Compose que je t’ai donnée :

```bash
docker compose up -d
docker compose logs -f
```


