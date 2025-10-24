# 0) (Optionnel) Libérer les ports 9200/9300/5601

```bash
# Stopper d’anciens services/containers
sudo systemctl stop elasticsearch kibana 2>/dev/null || true
sudo systemctl disable elasticsearch kibana 2>/dev/null || true
sudo systemctl mask    elasticsearch kibana 2>/dev/null || true

docker compose down 2>/dev/null || true
docker stop $(docker ps -q) 2>/dev/null || true

# Vérifier et libérer 5601 si besoin
sudo ss -ltnp | egrep ':(9200|9300|5601)\b' || echo "OK: ports libres"
sudo fuser -k -n tcp 5601 2>/dev/null || true
```

---

# 1) Dossier de projet

```bash
mkdir elk-dev && cd elk-dev
```

# 2) `.env` — variables (modifiez le mot de passe si vous voulez)

```bash
cat > .env << 'ENV'
STACK_VERSION=8.19.5
ELASTIC_PASSWORD=changeme123!
# Limites mémoire confortables pour une petite VM
ES_JAVA_OPTS=-Xms1g -Xmx1g
KIBANA_ENCRYPTION_KEY1=change_me_to_a_very_long_random_string_key_1_________
KIBANA_ENCRYPTION_KEY2=change_me_to_a_very_long_random_string_key_2_________
KIBANA_ENCRYPTION_KEY3=change_me_to_a_very_long_random_string_key_3_________
# (sera ajouté automatiquement à l'étape 4)
# ELASTICSEARCH_SERVICEACCOUNTTOKEN=
ENV
```

# 3) `docker-compose.yml` — sécurité ON, HTTP sans TLS

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    container_name: es01
    environment:
      - node.name=es01
      - discovery.type=single-node
      - xpack.security.enabled=true
      # HTTP en clair (dev)
      - xpack.security.http.ssl.enabled=false
      # Transport en clair (OK pour single-node en dev)
      - xpack.security.transport.ssl.enabled=false
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - ES_JAVA_OPTS=${ES_JAVA_OPTS}
      # (facultatif dev) baisser les watermarks disque si peu d'espace :
      # - cluster.routing.allocation.disk.watermark.low=95%
      # - cluster.routing.allocation.disk.watermark.high=97%
      # - cluster.routing.allocation.disk.watermark.flood_stage=98%
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
      # ✅ Utiliser un Service Account Token (sera injecté depuis .env)
      - ELASTICSEARCH_SERVICEACCOUNTTOKEN=${ELASTICSEARCH_SERVICEACCOUNTTOKEN}
      # Clés d’encryption (fortement recommandé)
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

> Pourquoi **sans TLS** : en dev/local, ça supprime les soucis de certificats.
> En prod, activez TLS et n’utilisez pas de variables en clair.

---

# 4) **Auto-setup** : démarrer ES → créer le token → relancer Kibana

Collez ce bloc (il fait tout) :

```bash
# 4.1 Démarre Elasticsearch seul
docker compose up -d elasticsearch
echo "Attente qu'Elasticsearch soit healthy…"
until [ "$(docker inspect -f '{{.State.Health.Status}}' es01)" = "healthy" ]; do sleep 2; done

# 4.2 Crée un service token pour Kibana et l'ajoute à .env
TOKEN=$(docker exec -it es01 bash -lc "/usr/share/elasticsearch/bin/elasticsearch-service-tokens create elastic/kibana kibana-token" | awk -F'= ' '/= /{print $2}' | tr -d '\r')
if ! grep -q '^ELASTICSEARCH_SERVICEACCOUNTTOKEN=' .env; then
  echo "ELASTICSEARCH_SERVICEACCOUNTTOKEN=$TOKEN" >> .env
else
  sed -i "s|^ELASTICSEARCH_SERVICEACCOUNTTOKEN=.*|ELASTICSEARCH_SERVICEACCOUNTTOKEN=$TOKEN|" .env
fi
echo "Token injecté dans .env"

# 4.3 Démarre Kibana avec le token
docker compose up -d kibana
docker compose ps
docker compose logs -f kibana
```

Si le port **5601** est déjà pris, change le mapping en `"5602:5601"` dans `docker-compose.yml`, puis relance `docker compose up -d kibana` et utilise `http://localhost:5602`.

---

# 5) URLs

* Elasticsearch : `http://localhost:9200`
* Kibana : `http://localhost:5601` (ou `5602` si tu as changé)

**Identifiants pour ES (API/cURL) :** `elastic` / **${ELASTIC_PASSWORD}**

> Le **token** est pour **Kibana** → ne l’utilise pas dans le navigateur/cURL pour ES.

---

# 6) Cheat-sheet cURL (auth requise, HTTP)

> Remplace `<nom_index>`, `<id>`, `<champ>`, `<valeur>`.
> Mot de passe = valeur de `ELASTIC_PASSWORD` (ici `changeme123!`).

```bash
# 1) Infos cluster
curl -u elastic:changeme123! http://localhost:9200/

# 2) Lister les indices
curl -u elastic:changeme123! "http://localhost:9200/_cat/indices?v"

# 3) État des nœuds
curl -u elastic:changeme123! "http://localhost:9200/_cat/nodes?v"

# 4) Stats indices
curl -u elastic:changeme123! "http://localhost:9200/_stats"

# 5) Recherche simple
curl -u elastic:changeme123! "http://localhost:9200/<nom_index>/_search?q=<champ>:<valeur>"

# 6) Mapping
curl -u elastic:changeme123! "http://localhost:9200/<nom_index>/_mapping"

# 7) Indexer un document (ID fixé)
curl -u elastic:changeme123! -X PUT "http://localhost:9200/<nom_index>/_doc/<id>" \
  -H 'Content-Type: application/json' -d '{"<champ>":"<valeur>"}'

# 8) Mise à jour partielle
curl -u elastic:changeme123! -X POST "http://localhost:9200/<nom_index>/_update/<id>" \
  -H 'Content-Type: application/json' -d '{"doc":{"<champ>":"<nouvelle_valeur>"}}'

# 9) Supprimer un doc
curl -u elastic:changeme123! -X DELETE "http://localhost:9200/<nom_index>/_doc/<id)"

# 10) Supprimer un index
curl -u elastic:changeme123! -X DELETE "http://localhost:9200/<nom_index>"
```

---

## Dépannage express

* **Kibana refuse de démarrer avec `elastic`**
  → c’est normal en 8.19 : **utilise le Service Account Token**, comme ci-dessus.

* **5601 déjà occupé**

  ```bash
  sudo ss -ltnp | grep :5601 || echo "libre"
  # soit tue le process, soit mappe "5602:5601"
  ```

* **Alerte “flood stage disk watermark 95%” (indices en read-only)**
  Libère de l’espace. En dev, tu peux temporairement :

  ```bash
  curl -u elastic:changeme123! -X PUT "http://localhost:9200/_cluster/settings" \
    -H 'Content-Type: application/json' -d '{"transient":{"cluster.routing.allocation.disk.threshold_enabled":false}}'

  curl -u elastic:changeme123! -X PUT "http://localhost:9200/_all/_settings" \
    -H 'Content-Type: application/json' -d '{"index.blocks.read_only_allow_delete": null}'
  ```

* **Changer le port Kibana** → dans `docker-compose.yml` :

  ```yaml
  ports:
    - "5602:5601"
  ```

