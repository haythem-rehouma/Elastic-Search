# Elasticsearch et un Kibana fonctionnels en version 8.19.5 via Docker Compose, sans avoir à saisir de mot de passe dans chaque commande.


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

## 1. Préparer le dossier et le fichier `.env`

Créez votre répertoire et le fichier `.env` en veillant bien à **citer** les options mémoire :

```bash
mkdir -p elk-dev && cd elk-dev

cat > .env <<'ENV'
STACK_VERSION=8.19.5
# mot de passe du super‑utilisateur 'elastic'
ELASTIC_PASSWORD=changeme123!
# Les guillemets simples sont OBLIGATOIRES pour éviter l'erreur -Xmx1g: command not found
ES_JAVA_OPTS='-Xms1g -Xmx1g'

# Clés d’encryption pour Kibana (32 caractères mini chacune)
KIBANA_ENCRYPTION_KEY1=change_me_to_a_very_long_random_string_key_1_________
KIBANA_ENCRYPTION_KEY2=change_me_to_a_very_long_random_string_key_2_________
KIBANA_ENCRYPTION_KEY3=change_me_to_a_very_long_random_string_key_3_________

# Le jeton de service Kibana sera ajouté automatiquement plus bas :
# ELASTICSEARCH_SERVICEACCOUNTTOKEN=
ENV
```

Vous n’aurez plus à taper le mot de passe dans vos commandes si vous chargez le `.env` :

```bash
set -a
source .env
set +a
```

<br/>

## 2. Créer un `docker-compose.yml` propre

Assurez‑vous que votre `docker-compose.yml` :

* **Ne mentionne pas** de nom d’utilisateur/mot de passe pour Kibana.
* Active la sécurité mais **désactive les seuils disque** pour éviter que les indices système passent en lecture seule (ce qui casse l’authentification).

Voici un exemple prêt à l’emploi :

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    container_name: es01
    restart: unless-stopped
    environment:
      - node.name=es01
      - discovery.type=single-node
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=false
      - xpack.security.transport.ssl.enabled=false
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - ES_JAVA_OPTS=${ES_JAVA_OPTS}
      # désactive le blocage flood-stage (utile en VM avec peu d'espace)
      - cluster.routing.allocation.disk.threshold_enabled=false
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
    restart: unless-stopped
    depends_on:
      elasticsearch:
        condition: service_healthy
    environment:
      - SERVER_NAME=kibana
      - SERVER_HOST=0.0.0.0
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      # Jeton de service ajouté dans .env après génération
      - ELASTICSEARCH_SERVICEACCOUNTTOKEN=${ELASTICSEARCH_SERVICEACCOUNTTOKEN}
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

**Important :** Kibana ne doit plus utiliser l’utilisateur `elastic` pour se connecter à Elasticsearch. Le message d’erreur « value of "elastic" is forbidden » indique que l’utilisateur super‑admin ne peut pas écrire dans les indices système et qu’il faut un service token.

<br/>

## 3. Réinitialiser complètement et démarrer Elasticsearch

Pour que le nouveau mot de passe soit pris en compte et que l’index `.security` soit sain, supprimez l’ancien volume et redémarrez :

```bash
# Arrête et supprime les conteneurs + volumes
docker compose down -v

# S'assurer qu'il ne reste aucun processus sur 9200/5601 (optionnel)
sudo ss -ltnp | egrep ':(9200|5601)\b' || echo "OK: ports libres"

# Démarre seulement Elasticsearch
docker compose up -d elasticsearch

# Attendre qu'Elasticsearch soit healthy
echo "Attente qu'Elasticsearch soit healthy…"
until [ "$(docker inspect -f '{{.State.Health.Status}}' es01)" = "healthy" ]; do sleep 2; done
```

<br/>

## 4. Générer et enregistrer le service token

1. Générez un jeton unique pour le compte de service `elastic/kibana` **une fois qu’Elasticsearch est opérationnel** :

   ```bash
   NEW_TOKEN=$(docker exec es01 bash -lc \
     "/usr/share/elasticsearch/bin/elasticsearch-service-tokens create elastic/kibana kibana-$(date +%s)" \
     | awk -F'= ' '/= /{print $2}' | tr -d '\r')

   echo "Jeton généré : ${NEW_TOKEN:0:15}..."
   ```

2. Vérifiez que le jeton fonctionne :

   ```bash
   curl -s -H "Authorization: Bearer $NEW_TOKEN" http://localhost:9200/_security/_authenticate?pretty
   ```

   La réponse doit contenir `"username" : "elastic/kibana"`.

3. Mettez-le dans `.env` :

   ```bash
   if grep -q '^ELASTICSEARCH_SERVICEACCOUNTTOKEN=' .env; then
     sed -i "s|^ELASTICSEARCH_SERVICEACCOUNTTOKEN=.*|ELASTICSEARCH_SERVICEACCOUNTTOKEN=$NEW_TOKEN|" .env
   else
     echo "ELASTICSEARCH_SERVICEACCOUNTTOKEN=$NEW_TOKEN" >> .env
   fi
   ```

Recharger `.env` dans votre shell (si nécessaire) :

```bash
set -a; source .env; set +a
```

<br/>

## 5. Démarrer Kibana

```bash
docker compose up -d kibana
```

Kibana lira automatiquement le jeton pour se connecter à Elasticsearch. Suivez son statut :

```bash
# doit renvoyer "available" après quelques instants
curl -s http://localhost:5601/api/status | jq -r '.status.overall.level'
```

Vous pourrez ensuite ouvrir `http://localhost:5601` et vous connecter avec l’utilisateur `elastic` et le mot de passe `ELASTIC_PASSWORD` (celui du `.env`).

<br/>

## 6. Utiliser les commandes cURL sans mot de passe manuel

Après avoir chargé `.env` dans votre session, utilisez :

```bash
# Infos cluster
curl -u elastic:$ELASTIC_PASSWORD http://localhost:9200/

# Liste des indices (il n’y aura rien immédiatement après l’installation)
curl -u elastic:$ELASTIC_PASSWORD "http://localhost:9200/_cat/indices?v"
```

<br/>

## 7. Résumé des bonnes pratiques

* **Quotez `ES_JAVA_OPTS` dans `.env`** pour éviter l’erreur de commande `-Xmx1g`.
* **Désactivez les seuils disque** en développement si votre disque est presque plein : sans cela, Elasticsearch place les indices en lecture seule à partir de 95 % d’occupation.
* **Utilisez un service token** pour Kibana – l’utilisateur `elastic` est interdit pour cela.
* **Rechargez `.env` dans chaque nouveau shell** (`set -a; source .env; set +a`) pour que `$ELASTIC_PASSWORD` et le jeton soient disponibles dans vos commandes.

En suivant ces étapes, vous obtiendrez un déploiement propre où Elasticsearch et Kibana fonctionnent ensemble, sans avoir à saisir votre mot de passe dans les commandes.





# Annexe 1 - Informations de connexion:

**Username :** `elastic`
**Password :** la valeur de `ELASTIC_PASSWORD` dans ton fichier `.env`.

D’après ce que tu as mis plus haut, c’est **`changeme123!`** (sauf si tu l’as modifiée depuis).

Tu peux vérifier rapidement dans ton dossier `elk-dev` :

```bash
grep '^ELASTIC_PASSWORD=' .env
```


<br/>

# Annexe 2 - troubleshooting


Les problème rencontrées peuvent venir principalement de trois points :

1. **Variable `ES_JAVA_OPTS` mal citée** : sans guillemets, elle est interprétée comme une commande (`-Xmx1g: command not found`).
2. **Absence de désactivation du seuil “flood stage”** : avec un disque presque plein, l’index `.security` devient en lecture seule, ce qui empêche d’authentifier les comptes (erreurs 401).
3. **Utilisation erronée ou jetons invalides** : les service tokens doivent être créés une fois Elasticsearch prêt et stockés dans `.env`.


- Si la connexion échoue :

* assurez-vous que vous n’avez pas d’espace en trop et que la casse est correcte ;
* que Kibana est bien sur `http://localhost:5601` et qu’Elasticsearch répond (`curl -u elastic:$ELASTIC_PASSWORD http://localhost:9200/`).

