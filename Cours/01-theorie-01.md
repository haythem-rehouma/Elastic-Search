# 1) Deux modèles différents

## A. Relationnel (SQL)

* **Un modèle tabulaire** : **tables** (Articles, Auteurs, Catégories), **lignes**, **colonnes**.
* **Schéma strict** : types fixés (VARCHAR, DATE…), **contrôles d’intégrité** (PK, FK, UNIQUE).
* **Normalisation** : on **évite la duplication** (ex : une table `authors` + table de jointure `article_authors`).
* **Transactions ACID** : cohérence forte, multi-lignes, multi-tables.
* **Requêtes** en **SQL** (SELECT/INSERT/UPDATE/DELETE), **JOINS**, **GROUP BY**.
* **Cas d’usage** : systèmes métiers (OLTP), finance, commandes, CRM… où la **cohérence** et les **relations** priment.

<details>
  <summary>Vulgarisation</summary>

  Le modèle relationnel fonctionne comme un ensemble de tableaux bien organisés.

  - Une **table** ressemble à une feuille Excel spécialisée.
  - Une **ligne** représente un enregistrement.
  - Une **colonne** représente un type d’information précis.
  - Les **relations** entre tables permettent de relier proprement les données sans les répéter partout.

  Ce modèle est particulièrement utile quand il faut garantir que les données restent correctes, cohérentes et bien structurées.
</details>

## B. Documents (Elasticsearch)

* **Un modèle “document”** : chaque **document JSON** contient les champs nécessaires, souvent **dénormalisé** (on met les auteurs directement dans l’article).
* **Schéma souple** mais **mappé** (text/keyword/date…), pas de FK/PK au sens SQL.
* **Index inversé** pour **recherche plein texte** (analyzers, stemming, fuzzy, scoring BM25).
* **Near-Real-Time** : écritures visibles après un **refresh** (≈1s), pas de transactions multi-docs.
* **Requêtes** via **Query DSL** (JSON), **KQL** (Kibana), **ES|QL** (style SQL) + **agrégations** distribuées.
* **Cas d’usage** : recherche textuelle (“Google-like”), logs/metrics, analytics temps réel, autosuggest, filtrage rapide avec facettes.

<details>
  <summary>Vulgarisation</summary>

  Le modèle document fonctionne davantage comme une collection de fiches complètes.

  Au lieu de répartir l’information dans plusieurs tables reliées entre elles, on place souvent les données utiles directement dans un même document JSON.

  Par exemple, au lieu d’avoir une table des auteurs séparée, les noms des auteurs peuvent être stockés directement dans l’article.

  Cette approche est très efficace pour :
  - rechercher rapidement,
  - filtrer de grandes quantités de données,
  - afficher des résultats sans faire de jointures complexes.
</details>

👉 **À retenir**

* **SQL** = cohérence, relations, transactions, reporting structuré.
* **Elasticsearch** = **recherche** rapide et intelligente, **filtrage** + **agrégations** massives, **scalabilité horizontale**.



# 2) Le dataset “news” : comment le modéliser ?

## A. En relationnel (ex : PostgreSQL)

### Schéma

```sql
CREATE TABLE categories (
  id SERIAL PRIMARY KEY,
  name TEXT UNIQUE NOT NULL
);

CREATE TABLE authors (
  id SERIAL PRIMARY KEY,
  name TEXT UNIQUE NOT NULL
);

CREATE TABLE articles (
  id SERIAL PRIMARY KEY,
  category_id INT REFERENCES categories(id),
  headline TEXT NOT NULL,
  short_description TEXT,
  link TEXT UNIQUE,
  date DATE NOT NULL
);

-- relation N-N : un article peut avoir plusieurs auteurs
CREATE TABLE article_authors (
  article_id INT REFERENCES articles(id),
  author_id INT REFERENCES authors(id),
  PRIMARY KEY (article_id, author_id)
);
````

<details>
  <summary>Vulgarisation</summary>

Ici, les données sont réparties proprement :

* `categories` contient les catégories,
* `authors` contient les auteurs,
* `articles` contient les articles,
* `article_authors` sert à relier plusieurs auteurs à un même article.

Cette structure évite de répéter inutilement les mêmes informations dans plusieurs endroits.

</details>

### Requêtes typiques

* Derniers articles :

```sql
SELECT a.*
FROM articles a
ORDER BY a.date DESC
LIMIT 10;
```

* Articles “POLITICS” entre deux dates :

```sql
SELECT a.*
FROM articles a
JOIN categories c ON c.id = a.category_id
WHERE c.name = 'POLITICS'
  AND a.date >= '2018-05-24' AND a.date < '2018-05-27'
ORDER BY a.date DESC
LIMIT 20;
```

* Top catégories (group by) :

```sql
SELECT c.name, COUNT(*) AS n
FROM articles a
JOIN categories c ON c.id = a.category_id
GROUP BY c.name
ORDER BY n DESC
LIMIT 10;
```

* **Plein texte** (il faut configurer le FT de Postgres : `to_tsvector / to_tsquery`) :

```sql
SELECT a.*
FROM articles a
WHERE to_tsvector('english', a.headline || ' ' || a.short_description)
      @@ plainto_tsquery('english', 'Trump')
ORDER BY a.date DESC
LIMIT 20;
```

* **Forces** : cohérence, jointures propres, intégrité.
* **Limites** : la recherche FT est bonne mais **moins riche** que celle d’un moteur dédié (synonymes, fuzziness fine, ranker, highlight, scale-out…).

<details>
  <summary>Vulgarisation</summary>

PostgreSQL peut faire de la recherche textuelle, mais ce n’est pas sa spécialité principale.

Il sait très bien :

* stocker les données,
* relier les tables,
* garantir leur cohérence.

En revanche, pour une recherche très avancée avec classement intelligent, fautes de frappe, surlignage des résultats ou très gros volumes, un moteur spécialisé comme Elasticsearch est généralement plus adapté.

</details>

## B. En Elasticsearch (index “news”)

### Mapping minimal

```json
PUT _index_template/news-template
{
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
      }
    }
  }
}
```

<details>
  <summary>Vulgarisation</summary>

Le **mapping** définit comment Elasticsearch doit comprendre chaque champ.

Par exemple :

* `date` est une date,
* `category` est une valeur exacte,
* `headline` est un texte à analyser pour la recherche.

Sans ce cadrage, Elasticsearch peut interpréter les champs de façon moins précise.

</details>

### Exemple de document

```json
{
  "category": "POLITICS",
  "headline": "Trump Suggests North Korea Summit Could Still Happen",
  "authors": ["Roberta Rampton","Christine Kim"],
  "link": "https://www.huffingtonpost.com/entry/trump-north-korea-summit_us_5b0823c7e4b0568a880aee31",
  "short_description": "He had canceled the planned June 12 summit less than a day earlier.",
  "date": "2018-05-25"
}
```

### Requêtes typiques (Query DSL)

* Derniers articles :

```http
GET news/_search
{
  "size": 10,
  "sort": [{ "date": "desc" }]
}
```

* Filtre exact + dates :

```http
GET news/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "category": "POLITICS" } },
        { "range": { "date": { "gte": "2018-05-24", "lt": "2018-05-27" } } }
      ]
    }
  },
  "sort": [{ "date": "desc" }]
}
```

* **Plein texte + highlight** :

```http
GET news/_search
{
  "query": {
    "multi_match": {
      "query": "Trump",
      "fields": ["headline^2","short_description"]
    }
  },
  "highlight": { "fields": { "headline": {}, "short_description": {} } }
}
```

* **Group by** (agrégations distribuées) :

```http
GET news/_search
{
  "size": 0,
  "aggs": { "by_category": { "terms": { "field": "category", "size": 10 } } }
}
```

* **Forces** : recherche textuelle au top (fuzzy, score, highlight), agrégations rapides à grande échelle, **UX Kibana** (facettes, Discover, dashboards).
* **Limites** : pas de FK/JOINS classiques, **dénormalisation** fréquente, **pas de transactions multi-docs**, cohérence **near-real-time**.

<details>
  <summary>Vulgarisation</summary>

Dans Elasticsearch, chaque article est souvent stocké comme un document autonome.

Cela permet :

* d’effectuer des recherches rapides,
* de filtrer sur des champs précis,
* d’agréger les résultats,
* d’afficher les données sans dépendre de jointures classiques.

Cette logique est très performante pour la recherche, mais moins adaptée à la gestion relationnelle stricte.

</details>

# 3) “Document” vs “Table” : que devient une table ?

* **Table `articles`** → **Index `news`** (1 **document** par article).
* **Table `categories`** → soit **champ `category`** (keyword) dans le document, soit un index à part si un référentiel riche doit être géré.
* **Table `authors` + `article_authors` (N-N)** → en ES, **`authors`** est souvent stocké comme tableau de `keyword`.
  Si une vraie relation doit être conservée, ES propose **`join` parent/child**… mais c’est **coûteux**. En général, en ES, on **dénormalise**.

**Idée clé** :

* SQL = **normalisé** (éviter duplication) → **JOINS**.
* ES = **dénormalisé** (rapide à lire/chercher) → **pas de JOINS**, on **réplique** une partie de l’information.

<details>
  <summary>Vulgarisation</summary>

En SQL, on sépare les informations dans plusieurs tables pour éviter les répétitions.

En Elasticsearch, on préfère souvent regrouper dans le même document ce qui sera utile à la recherche.

Le compromis est simple :

* SQL privilégie la structure et la cohérence,
* Elasticsearch privilégie la vitesse de lecture et de recherche.

</details>

# 4) “DSL”, “DQL”, “KQL”, “ES|QL”… c’est quoi ?

* **DSL (Elasticsearch Query DSL)** : le **langage JSON** officiel des requêtes ES (`_search`, `bool`, `match`, `aggs`, etc.).
* **KQL (Kibana Query Language)** : mini-langage dans **Discover** pour filtrer vite (`category:"POLITICS" AND date >= "2018-05-24"`).
* **ES|QL** : langage **style SQL** d’Elastic pour des requêtes plus lisibles (filter, STATS/GROUP BY, ORDER BY…).
* **DQL** : en SQL “classique”, cela signifie **Data Query Language** (SELECT). **Ce n’est pas un langage Elastic.**
  Dans Elastic, on parle plutôt de **DSL**, **KQL** et **ES|QL**.

<details>
  <summary>Vulgarisation</summary>

Ces sigles désignent simplement différentes façons d’écrire des requêtes.

* **DSL** : format JSON complet, très puissant.
* **KQL** : syntaxe simple pour filtrer dans Kibana.
* **ES|QL** : syntaxe plus proche de SQL.
* **DQL** : terme classique du monde SQL, pas un langage propre à Elastic.

</details>

# 5) Shards, replicas, scaling (simple)

* Un **index** ES est découpé en **shards** (morceaux Lucene).

  * **Primary shard** = copie principale
  * **Replica shard** = copie de secours / lecture en parallèle
* **Pourquoi ?**

  * **Scalabilité horizontale** (répartir les shards sur plusieurs nœuds)
  * **Haute disponibilité** (si un nœud tombe, un replica sert les requêtes)
* En dev **single-node** : **1 primary, 0 replica** suffit.

<details>
  <summary>Vulgarisation</summary>

Un shard peut être vu comme un morceau d’index.

Au lieu de tout stocker dans un seul bloc géant, Elasticsearch découpe les données pour :

* mieux répartir la charge,
* accélérer certaines opérations,
* tolérer la panne d’un nœud grâce aux replicas.

</details>

# 6) Choisir : relationnel, Elasticsearch… ou les deux ?

### Prendre **SQL** si l’objectif principal est :

* **Transactions** (multi-lignes, multi-tables), contraintes d’intégrité.
* Beaucoup de **mises à jour** cohérentes, reporting métier précis.
* Des **JOINS** complexes, référentiels rigides.

### Prendre **Elasticsearch** si l’objectif principal est :

* **Recherche plein texte** (classement, fuzzy, highlight, “like Google”).
* **Filtrage** ultra rapide + **tableaux de bord** (Kibana).
* **Agrégations** massives quasi temps réel (logs, analytics, observability).
* **Scalabilité horizontale** simple.

### Le **pattern courant** (et gagnant)

* **SQL = source canonique** (vérité, transactions).
* **ES = couche de recherche/analytics** (indexation asynchrone depuis SQL).
* Les deux peuvent être combinés sans renoncer à la cohérence.

<details>
  <summary>Vulgarisation</summary>

Dans beaucoup de systèmes modernes, les deux outils coexistent :

* la base SQL garde la vérité métier,
* Elasticsearch sert à rechercher vite et à explorer les données.

Ce modèle évite de demander à un seul outil de tout faire.

</details>

# 7) Les pièges classiques

* Utiliser `text` dans des **agrégations** → erreur du type « not aggregatable » → utiliser la variante **`field.keyword`** ou mapper directement en `keyword`.
* **Dates** au mauvais format → mapper correctement (`yyyy-MM-dd` ici).
* Penser qu’ES remplace un SGBD **transactionnel** → non (ES a une cohérence ACID **par document**, pas sur plusieurs documents).
* Oublier la **dénormalisation** : les auteurs sont souvent stockés directement dans le document sous forme de tableau (`array`) plutôt que dans des structures relationnelles séparées.

<details>
  <summary>Vulgarisation</summary>

Les erreurs fréquentes viennent souvent d’un mauvais réflexe : utiliser Elasticsearch comme une base SQL classique.

Elasticsearch fonctionne différemment :

* il faut penser en documents,
* accepter une certaine dénormalisation,
* distinguer les champs pour la recherche (`text`) et les champs pour le filtrage ou les agrégations (`keyword`).

</details>
```
