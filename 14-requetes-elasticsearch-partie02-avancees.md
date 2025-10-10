-----------------------------
# Introduction:
-----------------------------

Pour écrire des requêtes dans Elasticsearch, vous pouvez utiliser l'API de recherche et l'API de requête pour explorer et filtrer les données que vous avez importées. Voici quelques exemples de requêtes pour explorer votre index `news` contenant les données du fichier `News_Category_Dataset_v2.json`.

# 1. Requête de Base : Rechercher Tous les Documents

Pour obtenir tous les documents de l'index `news`, utilisez la requête suivante :

```bash
curl -u elastic:c+vdv5FUzys5hft5*8Fs -k -X GET "https://localhost:9200/news/_search?pretty"
```

# 2. Requête de Filtrage : Rechercher par Catégorie

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

# 3. Requête de Filtrage Multiple : Rechercher par Catégorie et Auteur

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

# 4. Requête de Texte Complet : Rechercher par Titre (Headline)

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

# 5. Requête de Plage : Rechercher par Date

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

# 6. Requête de Texte Complet : Rechercher par Description

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

# 7. Requête de Comptage : Compter le Nombre d’Articles par Catégorie

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

# 8. Trier les Résultats : Rechercher et Trier par Date

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

# 9. Pagination des Résultats : Limiter et Paginer les Résultats

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

- Ces exemples couvrent les requêtes de base pour interagir avec vos données dans Elasticsearch. 
- Vous pouvez adapter ces requêtes pour explorer différentes parties de vos données selon vos besoins.
