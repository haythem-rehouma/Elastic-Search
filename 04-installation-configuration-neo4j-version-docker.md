# Option A — `docker run` (rapide)

```bash
# 1) (facultatif) Nettoyer une ancienne install apt sur la VM hôte
sudo systemctl stop neo4j || true
sudo systemctl disable neo4j || true
sudo apt-get purge -y neo4j || true
sudo apt-get autoremove --purge -y || true

# 2) Créer des volumes (persistance)
docker volume create neo4j_data
docker volume create neo4j_logs
docker volume create neo4j_plugins
docker volume create neo4j_import

# 3) Lancer Neo4j 4.1 (Java 11 inclus)
docker run -d --name neo4j4 \
  --restart unless-stopped \
  -p 7474:7474 -p 7687:7687 \
  -v neo4j_data:/data \
  -v neo4j_logs:/logs \
  -v neo4j_plugins:/plugins \
  -v neo4j_import:/var/lib/neo4j/import \
  -e NEO4J_AUTH=neo4j/Neo4jStrongPass! \
  -e NEO4J_dbms_default__listen__address=0.0.0.0 \
  -e NEO4J_dbms_connector_bolt_enabled=true \
  -e NEO4J_dbms_connector_bolt_listen__address=:7687 \
  -e NEO4J_dbms_connector_http_enabled=true \
  -e NEO4J_dbms_connector_http_listen__address=:7474 \
  -e NEO4JLABS_PLUGINS='["apoc"]' \
  -e NEO4J_apoc_export_file_enabled=true \
  -e NEO4J_apoc_import_file_enabled=true \
  -e NEO4J_apoc_import_file_use__neo4j__config=true \
  neo4j:4.1
```

Tester :

```bash
# attendre ~10-20 s que Bolt démarre
docker logs -f neo4j4 | sed -n '1,200p'   # tu dois voir Bolt enabled on 0.0.0.0:7687
ss -tuln | grep 7687                      # LISTEN sur 0.0.0.0:7687

# cypher-shell depuis l'hôte
docker exec -it neo4j4 cypher-shell -u neo4j -p 'Neo4jStrongPass!'
# ou depuis ta machine: cypher-shell -a bolt://<IP_VM>:7687 -u neo4j -p 'Neo4jStrongPass!'
```

Accès web : http://<IP_VM>:7474 (utilisateur `neo4j`, mot de passe `Neo4jStrongPass!`)

> Remarque 4.x : les variables d’environnement utilisent les **double underscores** `__` pour remplacer les points dans `neo4j.conf`.

---

# Option B — `docker-compose` (plus lisible)

`docker-compose.yaml` :

```yaml
services:
  neo4j:
    image: neo4j:4.1
    container_name: neo4j4
    restart: unless-stopped
    ports:
      - "7474:7474"
      - "7687:7687"
    environment:
      NEO4J_AUTH: "neo4j/Neo4jStrongPass!"
      NEO4J_dbms_default__listen__address: "0.0.0.0"
      NEO4J_dbms_connector_bolt_enabled: "true"
      NEO4J_dbms_connector_bolt_listen__address: ":7687"
      NEO4J_dbms_connector_http_enabled: "true"
      NEO4J_dbms_connector_http_listen__address: ":7474"
      NEO4JLABS_PLUGINS: '["apoc"]'
      NEO4J_apoc_export_file_enabled: "true"
      NEO4J_apoc_import_file_enabled: "true"
      NEO4J_apoc_import_file_use__neo4j__config: "true"
    volumes:
      - neo4j_data:/data
      - neo4j_logs:/logs
      - neo4j_plugins:/plugins
      - neo4j_import:/var/lib/neo4j/import

volumes:
  neo4j_data:
  neo4j_logs:
  neo4j_plugins:
  neo4j_import:
```

Lancer :

```bash
docker compose up -d
docker compose logs -f neo4j
docker exec -it neo4j4 cypher-shell -u neo4j -p 'Neo4jStrongPass!'
```



## Pourquoi éviter apt + systemd « dans Docker »

* L’image officielle **intègre la bonne JVM** (plus de conflit Java 21/11).
* Pas de service systemd à gérer dans un conteneur (Docker attend un **processus unique**).
* Configuration claire via variables d’environnement, données persistées via volumes.


