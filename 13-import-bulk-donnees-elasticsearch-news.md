# 🚀 Tutoriel pour Importer un Jeu de Données en Bulk dans Elasticsearch

Avant de commencer, assurez-vous d'avoir le fichier **`News_Category_Dataset_v2.json`** dans le même dossier où vous allez exécuter les commandes.

---

# 🔍 Commandes Initiales

1. **🛠️ Vérifier l'état d'Elasticsearch :**

    ```bash
    curl -k -X GET 'https://localhost:9200/'
    ```

    Cette commande vérifie que votre instance Elasticsearch fonctionne. Le paramètre `-k` ignore les erreurs de certificat SSL, utile pour une configuration en local.

2. **📁 Créer un index "news" :**

    ```bash
    curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X PUT "https://localhost:9200/news"
    ```

    Ici, vous créez un nouvel index nommé `news`. Le paramètre `-u elastic:c+vdv5FUzys5hft5*8Fs` indique les informations de connexion pour le compte `elastic`.

3. **⚡ Tenter un Import Direct avec la Commande Bulk :**

    ```bash
    curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X POST "https://localhost:9200/news/_bulk" -H "Content-Type: application/json" --data-binary "@News_Category_Dataset_v2.json"
    ```

    Vous tentez ici d'importer directement le fichier JSON dans Elasticsearch. Si vous rencontrez une erreur, cela signifie probablement que le fichier **`News_Category_Dataset_v2.json`** n'est pas dans le format requis par l'API Bulk d'Elasticsearch.

---

# ❗ Erreur de Format avec l'API Bulk d'Elasticsearch

- L'API Bulk nécessite une ligne d'action avant chaque document JSON pour indiquer l'opération à effectuer. 
- Dans notre cas, nous ajoutons la ligne `{"index": {}}` pour chaque document, ce qui indique qu'il s'agit d'une opération d'indexation.

---

# 📝 Étape 1 : Préparer le Fichier `News_Category_Dataset_v2.json` pour l'API Bulk

##### 🐍 Option 1 : Utiliser un Script Python

Voici un script Python simple pour ajouter la ligne d'action nécessaire avant chaque document JSON :

```python
# Script Python pour préparer le fichier Bulk
input_file = 'News_Category_Dataset_v2.json'
output_file = 'News_Category_Dataset_bulk.json'

with open(input_file, 'r') as infile, open(output_file, 'w') as outfile:
    for line in infile:
        # Écrire la ligne d'action pour chaque document
        outfile.write('{"index": {}}\n')
        # Écrire le document JSON lui-même
        outfile.write(line)

print(f"Fichier {output_file} prêt pour importation en bulk.")
```

1. Sauvegardez ce script sous le nom **`prepare_bulk.py`**.
2. Exécutez-le avec la commande suivante :

    ```bash
    nano prepare_bulk.py
    python3 prepare_bulk.py
    ```

Cela créera un fichier **`News_Category_Dataset_bulk.json`** qui est maintenant compatible avec l'API Bulk d'Elasticsearch.

---

##### 🖥️ Option 2 : Utiliser la Commande `sed` dans le Terminal

Si vous préférez une méthode rapide dans le terminal, utilisez `sed` pour ajouter la ligne d'index avant chaque document JSON :

```bash
sed 's/^/{\"index\": {}}\n&/' News_Category_Dataset_v2.json > News_Category_Dataset_bulk.json
```

Cette commande créera également un fichier **`News_Category_Dataset_bulk.json`** avec les lignes d'action appropriées.

---

# 📥 Étape 2 : Importer le Fichier Transformé dans Elasticsearch

Une fois le fichier **`News_Category_Dataset_bulk.json`** prêt, vous pouvez l'importer dans Elasticsearch avec la commande suivante :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X POST "https://localhost:9200/news/_bulk" -H "Content-Type: application/json" --data-binary "@News_Category_Dataset_bulk.json"
```

- **`{"index": {}}`** : Cette ligne d'action informe Elasticsearch qu'il s'agit d'une opération d'indexation pour chaque document.
- **`News_Category_Dataset_bulk.json`** : Ce fichier contient désormais les lignes d'action nécessaires.

---

# ✅ Étape 3 : Vérifier l'Importation des Données

Pour vérifier que les documents ont été importés, exécutez une requête de recherche simple :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET "https://localhost:9200/news/_search?pretty"
```

Si l'importation a réussi, vous devriez voir les documents importés dans l'index `news`.

---

# 🔎 Recherche d'un Document Spécifique

Enfin, pour effectuer une recherche spécifique (par exemple, les documents avec le mot "Trump" dans le champ `headline`), utilisez la commande suivante :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET "https://localhost:9200/news/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "headline": "Trump"
    }
  }
}'
```

Cela vous renvoie tous les documents dans l'index **`news`** contenant le mot **"Trump"** dans le champ `headline`.

---

# 📚 Résumé

Ce tutoriel couvre la préparation d'un fichier JSON pour l'importation en bulk dans Elasticsearch, le formatage du fichier pour l'API Bulk et la vérification de l'importation. Utilisez cette méthode pour tout fichier JSON nécessitant une ligne d'action pour chaque document, afin de garantir la compatibilité avec Elasticsearch.


--------------------------
# Supprimer le tous
--------------------------

Pour supprimer l'index et toutes les données associées dans Elasticsearch, vous pouvez utiliser la commande suivante :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X DELETE "https://localhost:9200/news"
```

### Explication
- **`-u elastic:c+vdv5FUzys5hft5*8Fs`** : Indique les identifiants d'utilisateur et mot de passe pour accéder à Elasticsearch.
- **`-X DELETE`** : Spécifie que vous souhaitez effectuer une opération de suppression.
- **`https://localhost:9200/news`** : Indique l'URL de l'index `news` à supprimer.

### Vérification
Après avoir exécuté la commande, vous pouvez vérifier que l’index a bien été supprimé en exécutant la commande suivante pour lister les index existants :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET 'https://localhost:9200/_cat/indices?v'
```

Cette commande affichera tous les index existants dans Elasticsearch. Si l'index `news` a été supprimé correctement, il ne devrait plus apparaître dans la liste.


--------------------
# Annexe 01: Comment importer un fichier JSON volumineux ?
--------------------

## Objectif:  

- *importer un fichier JSON volumineux, tel que `News_Category_Dataset_v2.json`, dans Elasticsearch avec `curl`* 

### Prérequis

1. Assurez-vous que votre fichier `News_Category_Dataset_v2.json` est dans le bon format pour Elasticsearch. Le fichier doit contenir un document JSON par ligne pour chaque enregistrement, par exemple :

   ```json
   {"category": "World", "headline": "Headline text", "authors": "Author name", "link": "URL", "short_description": "Description", "date": "2022-01-01"}
   {"category": "Politics", "headline": "Another headline", "authors": "Author name", "link": "URL", "short_description": "Description", "date": "2022-01-02"}
   ```
   Assurez-vous que chaque ligne est un objet JSON indépendant (format NDJSON ou JSON lignes).

2. **Index Elasticsearch** : Créez un index pour stocker les données, par exemple `news`. Vous pouvez utiliser la commande suivante :
   ```bash
   curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X PUT "https://localhost:9200/news"
   ```

### Étape 1 : Préparer le fichier pour le Bulk API

Elasticsearch propose une **API Bulk** pour importer des données volumineuses. Vous devez préparer le fichier en ajoutant une ligne d'action avant chaque document JSON. Cela pourrait ressembler à ceci :

   ```json
   {"index": {}}
   {"category": "World", "headline": "Headline text", "authors": "Author name", "link": "URL", "short_description": "Description", "date": "2022-01-01"}
   {"index": {}}
   {"category": "Politics", "headline": "Another headline", "authors": "Author name", "link": "URL", "short_description": "Description", "date": "2022-01-02"}
   ```

### Étape 2 : Importer le fichier avec `curl`

Une fois le fichier prêt, utilisez la commande `curl` pour le charger dans Elasticsearch via l’API Bulk :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X POST "https://localhost:9200/news/_bulk" -H "Content-Type: application/json" --data-binary "@News_Category_Dataset_v2.json"
```

### Explications

- `-u elastic:c+vdv5FUzys5hft5*8Fs` : Les informations d'authentification pour Elasticsearch.
- `-k` : Ignorer les erreurs SSL (si vous utilisez un certificat auto-signé).
- `-X POST "https://localhost:9200/news/_bulk"` : Utilise l'API Bulk pour envoyer les données dans l’index `news`.
- `-H "Content-Type: application/json"` : Indique que le contenu est en JSON.
- `--data-binary "@News_Category_Dataset_v2.json"` : Charge le fichier JSON en utilisant `--data-binary` pour envoyer les données sans modifications.

### Étape 3 : Vérifier l'importation

Une fois le chargement terminé, vous pouvez vérifier si les données ont été importées correctement en exécutant une requête de recherche simple :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET "https://localhost:9200/news/_search?pretty"
```

Cela affichera quelques documents de l'index `news` pour confirmer que les données ont été importées.

### Remarque

Si vous avez beaucoup de données, l'importation peut être lente. Pour des volumes très importants, il est recommandé d'utiliser un client Elasticsearch comme `Logstash` ou `Elasticsearch Python client`, qui est mieux optimisé pour l'importation en masse que `curl`.


💀💀💀💀💀💀💀💀💀 ===> **POSSIBILITÉ D'ERREUR LORS DE L'IMPORTATION !!!!**

L'erreur que nous avons rencontré  signifie que le format du fichier `News_Category_Dataset_v2.json` n'est pas conforme à ce qu'attend l'API Bulk d'Elasticsearch. Le fichier JSON que vous avez contient des objets JSON sans les actions nécessaires pour le Bulk API.

Pour que l'API Bulk fonctionne, chaque document doit être précédé d'une ligne d'action (`{"index": {}}`) qui indique qu'il s'agit d'une opération d'indexation. Voici comment préparer le fichier pour un import correct.

### Étape 1 : Préparer le fichier `News_Category_Dataset_v2.json` pour l'importation en bulk

Vous pouvez utiliser un outil comme `sed`, `awk` ou un script Python pour ajouter une ligne d'action avant chaque ligne JSON.

#### Option 1 : Utiliser un script Python pour préparer le fichier

Voici un script Python simple pour transformer votre fichier JSON existant en un format compatible avec l'API Bulk d'Elasticsearch.

```python
# Script Python pour préparer le fichier Bulk
input_file = 'News_Category_Dataset_v2.json'
output_file = 'News_Category_Dataset_bulk.json'

with open(input_file, 'r') as infile, open(output_file, 'w') as outfile:
    for line in infile:
        # Écrire la ligne d'action pour chaque document
        outfile.write('{"index": {}}\n')
        # Écrire le document JSON lui-même
        outfile.write(line)

print(f"Fichier {output_file} prêt pour importation en bulk.")
```

Enregistrez ce script sous le nom `prepare_bulk.py` et exécutez-le :

```bash
python3 prepare_bulk.py
```

Cela créera un nouveau fichier `News_Category_Dataset_bulk.json` qui est prêt pour l'importation avec l'API Bulk d'Elasticsearch.

#### Option 2 : Utiliser `sed` dans le terminal pour ajouter des lignes d'index

Si vous préférez utiliser `sed` dans le terminal (si le fichier est très volumineux, cela pourrait être plus rapide), exécutez cette commande :

```bash
sed 's/^/{\"index\": {}}\n&/' News_Category_Dataset_v2.json > News_Category_Dataset_bulk.json
```

### Étape 2 : Importer le fichier transformé dans Elasticsearch

Une fois le fichier `News_Category_Dataset_bulk.json` prêt, utilisez `curl` pour l'importer dans Elasticsearch :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X POST "https://localhost:9200/news/_bulk" -H "Content-Type: application/json" --data-binary "@News_Category_Dataset_bulk.json"
```

### Explications

- **`{"index": {}}`** : Cette ligne d'action est nécessaire pour chaque document JSON, elle informe Elasticsearch qu'il s'agit d'une opération d'indexation.
- **Fichier JSON** : `News_Category_Dataset_bulk.json` contient maintenant les lignes d'action nécessaires pour l'API Bulk.

### Étape 3 : Vérifier l'importation

Vous pouvez vérifier que les données ont été correctement importées avec une requête de recherche simple :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET "https://localhost:9200/news/_search?pretty"
```

*Si tout s'est bien passé, vous devriez voir des documents dans l'index `news`.*


--------------------
# Annexe 02: Comment écrire des requêtes dans Elasticsearch ?
--------------------

Pour écrire des requêtes dans Elasticsearch, vous pouvez utiliser l'API de recherche et l'API de requête pour explorer et filtrer les données que vous avez importées. Voici quelques exemples de requêtes pour explorer votre index `news` contenant les données du fichier `News_Category_Dataset_v2.json`.

### 1. Requête de Base : Rechercher Tous les Documents

Pour obtenir tous les documents de l'index `news`, utilisez la requête suivante :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET "https://localhost:9200/news/_search?pretty"
```

### 2. Requête de Filtrage : Rechercher par Catégorie

Pour rechercher tous les articles de la catégorie `ENTERTAINMENT`, vous pouvez utiliser un filtre :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET "https://localhost:9200/news/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "category": "ENTERTAINMENT"
    }
  }
}'
```

### 3. Requête de Filtrage Multiple : Rechercher par Catégorie et Auteur

Si vous voulez rechercher tous les articles dans la catégorie `ENTERTAINMENT` écrits par `Ron Dicker` :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET "https://localhost:9200/news/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": [
        { "match": { "category": "ENTERTAINMENT" }},
        { "match": { "authors": "Ron Dicker" }}
      ]
    }
  }
}'
```

### 4. Requête de Texte Complet : Rechercher par Titre (Headline)

Pour rechercher des articles contenant un mot spécifique dans le titre (par exemple, `Trump`), utilisez `match` sur le champ `headline` :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET "https://localhost:9200/news/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "headline": "Trump"
    }
  }
}'
```

### 5. Requête de Plage : Rechercher par Date

Si vous souhaitez rechercher des articles publiés après une certaine date (par exemple, après le 1er mai 2018) :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET "https://localhost:9200/news/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "range": {
      "date": {
        "gte": "2018-05-01"
      }
    }
  }
}'
```

### 6. Requête de Texte Complet : Rechercher par Description

Pour rechercher des articles contenant un mot spécifique dans la description courte (`short_description`), utilisez `match` sur ce champ. Par exemple, pour rechercher le mot `song` dans la description :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET "https://localhost:9200/news/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "short_description": "song"
    }
  }
}'
```

### 7. Requête de Comptage : Compter le Nombre d’Articles par Catégorie

Vous pouvez utiliser une agrégation pour compter le nombre d'articles par catégorie :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET "https://localhost:9200/news/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "categories_count": {
      "terms": {
        "field": "category.keyword"
      }
    }
  }
}'
```

### 8. Trier les Résultats : Rechercher et Trier par Date

Pour rechercher des articles dans la catégorie `ENTERTAINMENT` et les trier par date dans l’ordre décroissant (du plus récent au plus ancien) :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET "https://localhost:9200/news/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "category": "ENTERTAINMENT"
    }
  },
  "sort": [
    { "date": "desc" }
  ]
}'
```

### 9. Pagination des Résultats : Limiter et Paginer les Résultats

Pour obtenir les 5 premiers résultats, puis paginer avec `from` et `size` :

- **Première page (documents 0 à 4)** :
  ```bash
  curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET "https://localhost:9200/news/_search?pretty" -H 'Content-Type: application/json' -d'
  {
    "from": 0,
    "size": 5,
    "query": {
      "match_all": {}
    }
  }'
  ```

- **Deuxième page (documents 5 à 9)** :
  ```bash
  curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET "https://localhost:9200/news/_search?pretty" -H 'Content-Type: application/json' -d'
  {
    "from": 5,
    "size": 5,
    "query": {
      "match_all": {}
    }
  }'
  ```

Ces exemples couvrent les requêtes de base pour interagir avec vos données dans Elasticsearch. Vous pouvez adapter ces requêtes pour explorer différentes parties de vos données selon vos besoins.




