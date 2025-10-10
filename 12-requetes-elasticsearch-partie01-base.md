------------
# Partie 01 - Écriture des requêtes sans (user:password) !
------------

## Prérequis : Annexe 01

-----------------------------
#### Introduction:
-----------------------------

*L'objectif de ce document est de vous fournir un ensemble détaillé de commandes `curl` pour interagir facilement avec Elasticsearch.* 
> Vous y trouverez des explications claires et des exemples concrets utilisant des noms d'index et des données, afin de vous guider dans l'exploration et la manipulation des données au sein d'Elasticsearch.


---

### 1. **Vérifier les informations générales du cluster**
   Cette commande permet de vérifier si Elasticsearch fonctionne et d'obtenir des informations de base sur le cluster (nom, version, UUID, etc.).
   ```bash
   curl -k -XGET "https://localhost:9200/"
   ```
   **Explication :** Cette commande envoie une requête GET à Elasticsearch sans préciser de chemin spécifique, ce qui renvoie des informations générales sur le cluster.

---

### 2. **Vérifier l'état de santé du cluster**
   Utilisez cette commande pour voir l'état de santé global du cluster (`green`, `yellow`, ou `red`).
   ```bash
   curl -k -XGET "https://localhost:9200/_cluster/health"
   ```
   **Explication :** L'état `green` signifie que le cluster est entièrement opérationnel, `yellow` signifie que certaines répliques manquent, et `red` indique que des données ou des shards ne sont pas disponibles.

---

### 3. **Lister tous les indices dans le cluster**
   Cette commande affiche tous les indices disponibles avec des informations comme l'état, le nombre de documents, et l'espace disque utilisé.
   ```bash
   curl -k -XGET "https://localhost:9200/_cat/indices?v"
   ```
   **Exemple de résultat :**
   ```
   health status index      uuid                   pri rep docs.count store.size pri.store.size
   green  open   products   xaZV1...aV            1   1       10000       1.2gb           600mb
   green  open   customers  AaQ20...2v            1   1       5000        300mb           150mb
   ```

---

### 4. **Créer un index nommé `products`**
   Cette commande crée un index vide nommé `products` pour y stocker des documents de produits.
   ```bash
   curl -k -XPUT "https://localhost:9200/products"
   ```
   **Explication :** Les indices sont similaires aux tables en bases de données relationnelles. Cet index `products` pourra contenir des documents sur différents produits.

---

### 5. **Vérifier les informations de mappage d’un index (structure des données)**
   Cette commande affiche le mappage d’un index, montrant comment les champs et les types de données sont structurés.
   ```bash
   curl -k -XGET "https://localhost:9200/products/_mapping"
   ```
   **Explication :** Le mappage détermine la structure des documents stockés dans un index, comme les types de données (texte, nombre, etc.) et les paramètres de champ.

---

### 6. **Ajouter un document dans l’index `products`**
   Ajoutons un produit de test dans l'index `products` avec un ID unique.
   ```bash
   curl -k -XPOST "https://localhost:9200/products/_doc/1" -H 'Content-Type: application/json' -d'
   {
     "name": "Laptop",
     "category": "Electronics",
     "price": 1200,
     "stock": 50
   }'
   ```
   **Explication :** Ce document contient les informations d'un produit (`Laptop`) avec des champs tels que `name`, `category`, `price`, et `stock`.

---

### 7. **Rechercher des documents dans l’index `products`**
   Effectuons une recherche pour tous les produits de la catégorie `Electronics`.
   ```bash
   curl -k -XGET "https://localhost:9200/products/_search?q=category:Electronics"
   ```
   **Explication :** Cette commande utilise une recherche basée sur une requête (`q`) pour trouver des documents où le champ `category` est égal à `Electronics`.

---

### 8. **Lister les documents de l’index `products` avec un filtre par prix**
   Cette commande recherche les produits ayant un prix inférieur à 1500.
   ```bash
   curl -k -XGET "https://localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
   {
     "query": {
       "range": {
         "price": {
           "lt": 1500
         }
       }
     }
   }'
   ```
   **Explication :** Utilisation d’une requête de `range` pour filtrer les documents dont le champ `price` est inférieur (`lt`) à 1500.

---

### 9. **Ajouter un document dans un nouvel index `customers`**
   Ajoutons un document client avec des informations de base dans un nouvel index nommé `customers`.
   ```bash
   curl -k -XPOST "https://localhost:9200/customers/_doc/1" -H 'Content-Type: application/json' -d'
   {
     "name": "Alice",
     "age": 30,
     "membership": "gold",
     "email": "alice@example.com"
   }'
   ```
   **Explication :** Le document contient des informations de base sur un client nommé `Alice` avec des champs comme `name`, `age`, `membership`, et `email`.

---

### 10. **Mettre à jour un document existant dans `customers`**
   Mettons à jour l'âge d'un client existant.
   ```bash
   curl -k -XPOST "https://localhost:9200/customers/_update/1" -H 'Content-Type: application/json' -d'
   {
     "doc": {
       "age": 31
     }
   }'
   ```
   **Explication :** La commande met à jour uniquement le champ `age` d'un document existant, en laissant les autres champs intacts.

---

### 11. **Supprimer un document spécifique d’un index**
   Supposons que nous voulons supprimer le document avec l'ID `1` dans l'index `products`.
   ```bash
   curl -k -XDELETE "https://localhost:9200/products/_doc/1"
   ```
   **Explication :** Cette commande supprime le document identifié par `1` dans l'index `products`.

---

### 12. **Supprimer un index complet**
   Pour supprimer complètement l'index `products` (attention, cela supprimera tous les documents dans cet index).
   ```bash
   curl -k -XDELETE "https://localhost:9200/products"
   ```
   **Explication :** Supprimer un index signifie supprimer tous les documents et le mappage de cet index dans Elasticsearch.

---

### 13. **Obtenir les statistiques d'un index**
   Cette commande fournit des statistiques comme le nombre de documents, la taille de l'index, et d'autres informations de stockage pour l’index `customers`.
   ```bash
   curl -k -XGET "https://localhost:9200/customers/_stats"
   ```
   **Explication :** Les statistiques d’index donnent un aperçu de la quantité de données et de l’utilisation de l’espace disque pour un index particulier.

---

### 14. **Lister les informations des shards pour un index**
   Cette commande affiche des informations détaillées sur les shards utilisés par l'index `customers`.
   ```bash
   curl -k -XGET "https://localhost:9200/_cat/shards/customers?v"
   ```
   **Explication :** Chaque index est divisé en shards, qui sont répartis entre les nœuds du cluster. Cette commande permet de voir la distribution des shards pour un index spécifique.

---

### 15. **Vérifier l'état de récupération de tous les indices**
   Utilisez cette commande pour obtenir des informations sur les processus de récupération en cours pour tous les indices.
   ```bash
   curl -k -XGET "https://localhost:9200/_recovery?pretty"
   ```
   **Explication :** Lors de la restauration de snapshots ou après une panne, il est possible de suivre la progression de la récupération des shards.

---

### 16. **Effectuer une sauvegarde de snapshot d’un index dans un dépôt de snapshots**
   (Nécessite une configuration préalable d'un dépôt de snapshot).
   ```bash
   curl -k -XPUT "https://localhost:9200/_snapshot/my_backup/products_snapshot" -H 'Content-Type: application/json' -d'
   {
     "indices": "products",
     "ignore_unavailable": true,
     "include_global_state": false
   }'
   ```
   **Explication :** Sauvegarde l’index `products` dans un snapshot nommé `products_snapshot` dans un dépôt de snapshots appelé `my_backup`.

------------
# Partie 02 - Ré-écriture des requêtes avec le user:password !
------------

---------------------
##### Exemple:
---------------------

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET 'https://localhost:9200/'
```

---------------------
##### Introduction:
---------------------

*L'objectif de ce document est de vous fournir un ensemble détaillé de commandes `curl` pour interagir facilement avec Elasticsearch.* 
> Vous y trouverez des explications claires et des exemples concrets utilisant des noms d'index et des données, afin de vous guider dans l'exploration et la manipulation des données au sein d'Elasticsearch.


---

### 1. **Vérifier les informations générales du cluster**
   Cette commande permet de vérifier si Elasticsearch fonctionne et d'obtenir des informations de base sur le cluster (nom, version, UUID, etc.).
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XGET "https://localhost:9200/"
   ```
   **Explication :** Cette commande envoie une requête GET à Elasticsearch sans préciser de chemin spécifique, ce qui renvoie des informations générales sur le cluster.

---

### 2. **Vérifier l'état de santé du cluster**
   Utilisez cette commande pour voir l'état de santé global du cluster (`green`, `yellow`, ou `red`).
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XGET "https://localhost:9200/_cluster/health"
   ```
   **Explication :** L'état `green` signifie que le cluster est entièrement opérationnel, `yellow` signifie que certaines répliques manquent, et `red` indique que des données ou des shards ne sont pas disponibles.

---

### 3. **Lister tous les indices dans le cluster**
   Cette commande affiche tous les indices disponibles avec des informations comme l'état, le nombre de documents, et l'espace disque utilisé.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XGET "https://localhost:9200/_cat/indices?v"
   ```
   **Exemple de résultat :**
   ```
   health status index      uuid                   pri rep docs.count store.size pri.store.size
   green  open   products   xaZV1...aV            1   1       10000       1.2gb           600mb
   green  open   customers  AaQ20...2v            1   1       5000        300mb           150mb
   ```

---

### 4. **Créer un index nommé `products`**
   Cette commande crée un index vide nommé `products` pour y stocker des documents de produits.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XPUT "https://localhost:9200/products"
   ```
   **Explication :** Les indices sont similaires aux tables en bases de données relationnelles. Cet index `products` pourra contenir des documents sur différents produits.

---

### 5. **Vérifier les informations de mappage d’un index (structure des données)**
   Cette commande affiche le mappage d’un index, montrant comment les champs et les types de données sont structurés.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XGET "https://localhost:9200/products/_mapping"
   ```
   **Explication :** Le mappage détermine la structure des documents stockés dans un index, comme les types de données (texte, nombre, etc.) et les paramètres de champ.

---

### 6. **Ajouter un document dans l’index `products`**
   Ajoutons un produit de test dans l'index `products` avec un ID unique.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XPOST "https://localhost:9200/products/_doc/1" -H 'Content-Type: application/json' -d'
   {
     "name": "Laptop",
     "category": "Electronics",
     "price": 1200,
     "stock": 50
   }'
   ```
   **Explication :** Ce document contient les informations d'un produit (`Laptop`) avec des champs tels que `name`, `category`, `price`, et `stock`.

---

### 7. **Rechercher des documents dans l’index `products`**
   Effectuons une recherche pour tous les produits de la catégorie `Electronics`.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XGET "https://localhost:9200/products/_search?q=category:Electronics"
   ```
   **Explication :** Cette commande utilise une recherche basée sur une requête (`q`) pour trouver des documents où le champ `category` est égal à `Electronics`.

---

### 8. **Lister les documents de l’index `products` avec un filtre par prix**
   Cette commande recherche les produits ayant un prix inférieur à 1500.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XGET "https://localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
   {
     "query": {
       "range": {
         "price": {
           "lt": 1500
         }
       }
     }
   }'
   ```
   **Explication :** Utilisation d’une requête de `range` pour filtrer les documents dont le champ `price` est inférieur (`lt`) à 1500.

---

### 9. **Ajouter un document dans un nouvel index `customers`**
   Ajoutons un document client avec des informations de base dans un nouvel index nommé `customers`.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XPOST "https://localhost:9200/customers/_doc/1" -H 'Content-Type: application/json' -d'
   {
     "name": "Alice",
     "age": 30,
     "membership": "gold",
     "email": "alice@example.com"
   }'
   ```
   **Explication :** Le document contient des informations de base sur un client nommé `Alice` avec des champs comme `name`, `age`, `membership`, et `email`.

---

### 10. **Mettre à jour un document existant dans `customers`**
   Mettons à jour l'âge d'un client existant.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XPOST "https://localhost:9200/customers/_update/1" -H 'Content-Type: application/json' -d'
   {
     "doc": {
       "age": 31
     }
   }'
   ```
   **Explication :** La commande met à jour uniquement le champ `age` d'un document existant, en laissant les autres champs intacts.

---

### 11. **Supprimer un document spécifique d’un index**
   Supposons que nous voulons supprimer le document avec l'ID `1` dans l'index `products`.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XDELETE "https://localhost:9200/products/_doc/1"
   ```
   **Explication :** Cette commande supprime le document identifié par `1` dans l'index `products`.

---

### 12. **Supprimer un index complet**
   Pour supprimer complètement l'index `products` (attention, cela supprimera tous les documents dans cet index).
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XDELETE "https://localhost:9200/products"
   ```
   **Explication :** Supprimer un index signifie supprimer tous les documents et le mappage de cet index dans Elasticsearch.

---

### 13. **Obtenir les statistiques d'un index**
   Cette commande fournit des statistiques comme le nombre de documents, la taille de l'index, et d'autres informations de stockage pour l’index `customers`.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XGET "https://localhost:9200/customers/_stats"
   ```
   **Explication :** Les statistiques d’index donnent un aperçu de la quantité de données et de l’utilisation de l’espace disque pour un index particulier.

---

### 14. **Lister les informations des shards pour un index**
   Cette commande affiche des informations détaillées sur les shards utilisés par l'index `customers`.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XGET "https://localhost:9200/_cat/shards/customers?v"
   ```
   **Explication :** Chaque index est divisé en shards, qui sont répartis entre les nœuds du cluster. Cette commande permet de voir la distribution des shards pour un index spécifique.

---

### 15. **Vérifier l'état de récupération de tous les indices**
   Utilisez cette commande pour obtenir des informations sur les processus de récupération en cours pour tous les indices.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XGET "https

://localhost:9200/_recovery?pretty"
   ```
   **Explication :** Lors de la restauration de snapshots ou après une panne, il est possible de suivre la progression de la récupération des shards.

---

### 16. **Effectuer une sauvegarde de snapshot d’un index dans un dépôt de snapshots**
   (Nécessite une configuration préalable d'un dépôt de snapshot).
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XPUT "https://localhost:9200/_snapshot/my_backup/products_snapshot" -H 'Content-Type: application/json' -d'
   {
     "indices": "products",
     "ignore_unavailable": true,
     "include_global_state": false
   }'
   ```
   **Explication :** Sauvegarde l’index `products` dans un snapshot nommé `products_snapshot` dans un dépôt de snapshots appelé `my_backup`.


### 17. **Restaurer un snapshot dans un dépôt de snapshots**
   (Assurez-vous que le dépôt de snapshots est déjà configuré et accessible pour cette opération).
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XPOST "https://localhost:9200/_snapshot/my_backup/products_snapshot/_restore"
   ```
   **Explication :** Cette commande restaure l’index `products` à partir du snapshot `products_snapshot` stocké dans le dépôt `my_backup`. Elle est utile pour récupérer des données en cas de suppression accidentelle ou de corruption de l’index.

---

### 18. **Créer un alias pour un index**
   Un alias permet de créer un "nom alternatif" pour un index, facilitant l’accès et la gestion des index sans avoir à modifier directement le nom des index.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XPOST "https://localhost:9200/_aliases" -H 'Content-Type: application/json' -d'
   {
     "actions": [
       {
         "add": {
           "index": "products",
           "alias": "current_products"
         }
       }
     ]
   }'
   ```
   **Explication :** Cet alias `current_products` permet de faire référence à l’index `products` de manière flexible. Vous pouvez rediriger les requêtes vers un autre index sans changer le nom dans votre application.

---

### 19. **Supprimer un alias d’un index**
   Pour supprimer un alias créé précédemment pour un index.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XPOST "https://localhost:9200/_aliases" -H 'Content-Type: application/json' -d'
   {
     "actions": [
       {
         "remove": {
           "index": "products",
           "alias": "current_products"
         }
       }
     ]
   }'
   ```
   **Explication :** Cette commande supprime l’alias `current_products` de l’index `products`, ce qui peut être nécessaire pour des raisons de maintenance ou pour réaffecter l’alias à un autre index.

---

### 20. **Ajouter un pipeline d'ingestion pour traitement des données**
   Elasticsearch permet de définir des pipelines pour transformer les données avant de les indexer.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XPUT "https://localhost:9200/_ingest/pipeline/my_pipeline" -H 'Content-Type: application/json' -d'
   {
     "description": "Pipeline de test pour modifier les champs",
     "processors": [
       {
         "set": {
           "field": "category",
           "value": "unknown"
         }
       }
     ]
   }'
   ```
   **Explication :** Ce pipeline `my_pipeline` modifie le champ `category` d'un document, en le définissant à `unknown` si aucune valeur n’est spécifiée. Il peut être appliqué lors de l’indexation de documents pour assurer la cohérence des données.

---

### 21. **Indexer un document avec un pipeline d'ingestion**
   Vous pouvez appliquer un pipeline d’ingestion directement lors de l’indexation de nouveaux documents.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XPUT "https://localhost:9200/products/_doc/2?pipeline=my_pipeline" -H 'Content-Type: application/json' -d'
   {
     "name": "Smartphone",
     "price": 800,
     "stock": 30
   }'
   ```
   **Explication :** Ce document `Smartphone` est ajouté à l’index `products` en passant par le pipeline `my_pipeline`, ce qui garantit que le champ `category` sera défini, même s’il n’est pas inclus dans les données initiales.

---

### 22. **Afficher les pipelines d'ingestion existants**
   Pour obtenir la liste de tous les pipelines d'ingestion configurés dans Elasticsearch.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XGET "https://localhost:9200/_ingest/pipeline"
   ```
   **Explication :** Cette commande renvoie tous les pipelines d’ingestion existants avec leurs configurations, permettant de vérifier les pipelines disponibles et de comprendre leur fonctionnement.

---

### 23. **Supprimer un pipeline d'ingestion**
   Si un pipeline d’ingestion n’est plus nécessaire, vous pouvez le supprimer.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XDELETE "https://localhost:9200/_ingest/pipeline/my_pipeline"
   ```
   **Explication :** La suppression du pipeline `my_pipeline` libère des ressources et évite l’application de transformations indésirables sur les nouveaux documents.

---

### 24. **Exécuter une requête `match_all` pour récupérer tous les documents**
   La requête `match_all` renvoie tous les documents d'un index.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XGET "https://localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
   {
     "query": {
       "match_all": {}
     }
   }'
   ```
   **Explication :** La requête `match_all` est utile pour examiner tous les documents d'un index sans appliquer de filtres spécifiques.

---

### 25. **Utiliser une requête de filtrage booléen**
   Exemple de requête booléenne pour filtrer les documents selon plusieurs critères.
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -XGET "https://localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
   {
     "query": {
       "bool": {
         "must": [
           { "match": { "category": "Electronics" }},
           { "range": { "price": { "lte": 1000 }}}
         ]
       }
     }
   }'
   ```
   **Explication :** Cette requête combine deux conditions avec `bool` : elle sélectionne les documents où `category` est `Electronics` et où `price` est inférieur ou égal à 1000, permettant des recherches plus complexes.

---

*Ces exemples couvrent un éventail de fonctionnalités de base et avancées d'Elasticsearch, offrant une base pour interagir efficacement avec les indices, documents, et pipelines de données. Chaque commande, accompagnée de ses explications, vous aidera à maîtriser les opérations essentielles et à mieux comprendre le rôle de chaque élément dans la gestion et l’interrogation des données au sein d’Elasticsearch.*




------------
------------
------------
------------
------------
------------
------------
------------
------------
------------

# Annexe 01 - Comment éviter le problème de spécification (user:password) dans les requêtes ?
------------

### Annexe 01 - Solution 1 :Utiliser le fichier `.curlrc`


######  Étape 1 : Créer le fichier `.curlrc`

1. Utilisez `nano` pour créer le fichier `.curlrc` dans le répertoire de l’utilisateur `eleve` :
   ```bash
   nano /home/eleve/.curlrc
   ```


2. Ajoutez les informations d'identification dans le fichier `.curlrc` sous la forme suivante :
   ```bash
   user = "elastic:c+vdv5FUzys5hft5*8Fs"
   ```

   - **user** : Le nom d’utilisateur pour Elasticsearch (`elastic`).
   - **c+vdv5FUzys5hft5*8Fs** : Le mot de passe associé à l’utilisateur `elastic`.
   - Assurez-vous de remplacer ces informations si vos identifiants changent.

## *correction: il faut le créer sous /root/.curlrc*
- *Assurez-vous que le fichier `.curlrc` est bien créé dans le répertoire de l'utilisateur qui exécute `curl`.* 
- *Si vous utilisez `root`, placez le fichier dans le répertoire `/root/` au lieu de `/home/eleve/`.*

   ```bash
   nano /root/.curlrc
   cat /root/.curlrc
   ```


3. **Enregistrez et quittez** `nano` :
   - Appuyez sur `Ctrl + X`, puis `Y` pour confirmer et `Enter` pour sauvegarder.

### Étape 2 : Protéger le fichier `.curlrc`

Pour sécuriser le fichier et s'assurer qu'il est accessible uniquement par l'utilisateur `eleve`, définissez les permissions avec la commande suivante :
```bash
chmod 600 /home/eleve/.curlrc
(ou correction ) chmod 600 /root/.curlrc
```

### Étape 3 : Tester la configuration

Maintenant, chaque fois que vous utiliserez `curl` sans spécifier d’utilisateur, `curl` ajoutera automatiquement les informations d'identification à partir du fichier `.curlrc`. 

Par exemple, pour tester la connexion à Elasticsearch avec `curl`, vous pouvez exécuter la commande suivante :
```bash
curl -k -X GET 'https://localhost:9200/'
```

L'option `-k` est utilisée ici pour ignorer les erreurs SSL, car Elasticsearch utilise un certificat auto-signé.

---

En suivant ces étapes, vous éviterez de saisir le mot de passe à chaque requête `curl`, et le fichier `.curlrc` stockera les informations d'identification de manière pratique et sécurisée.



2. **Vérifiez les permissions du fichier** :
   Assurez-vous que le fichier `.curlrc` a bien les permissions `600` :
   ```bash
   chmod 600 /root/.curlrc
   ```



### Annexe 01 - Solution 2 : Utiliser directement l'option `-u` avec `curl`

Si le fichier `.curlrc` ne fonctionne toujours pas, utilisez l'option `-u` pour spécifier manuellement les informations d'identification dans la commande `curl` :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET 'https://localhost:9200/'
```

### Annexe 01 - Solution 3 : Vérifiez la configuration Elasticsearch

Si vous continuez à rencontrer des problèmes, il est possible qu'il y ait une configuration erronée dans `elasticsearch.yml`. Assurez-vous que la sécurité est bien activée et que les paramètres sont corrects. Vous devriez avoir dans `/etc/elasticsearch/elasticsearch.yml` :

```yaml
xpack.security.enabled: true
xpack.security.http.ssl.enabled: true
```

Après chaque modification dans `elasticsearch.yml`, redémarrez Elasticsearch :
```bash
sudo systemctl restart elasticsearch
```

### Annexe 01 - Solution 4 : Testez l'authentification avec un autre outil (comme `Postman`)

Pour vérifier si le problème vient de `curl` ou de la configuration Elasticsearch, essayez de vous connecter à `https://localhost:9200/` avec les identifiants via un outil comme **Postman** (ou un autre client HTTP) en utilisant l’authentification de base.


