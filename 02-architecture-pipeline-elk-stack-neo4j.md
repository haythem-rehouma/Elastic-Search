# Exemple d'intégraion Pipeline #1

Bonne question !

**ELK** est un acronyme qui désigne une **stack logicielle open-source** très utilisée pour la collecte, l’analyse et la visualisation de données, en particulier les **logs** :

* **E** = **Elasticsearch**

  * Le moteur de recherche et d’indexation distribué.
  * Sert à stocker, indexer et rechercher des données textuelles, numériques ou semi-structurées (par ex. des logs).

* **L** = **Logstash**

  * L’outil de collecte, traitement et transformation de données.
  * Il prend des données de différentes sources (fichiers, bases de données, APIs, etc.) et les envoie vers Elasticsearch.

* **K** = **Kibana**

  * L’interface de visualisation et d’analyse.
  * Permet de créer des tableaux de bord interactifs, des graphiques et des recherches sur les données stockées dans Elasticsearch.

-  Exemple d’utilisation :
Un système d’entreprise génère des **logs applicatifs** et des **événements système**.

* **Logstash** collecte ces logs, les nettoie et les transforme.
* **Elasticsearch** indexe les données pour les rendre consultables rapidement.
* **Kibana** permet aux développeurs, analystes ou équipes de sécurité d’explorer et de visualiser ces données en temps réel.

- Aujourd’hui, on parle aussi beaucoup de la **stack Elastic** (parfois **Elastic Stack**), qui inclut **ELK + Beats** (un autre composant pour collecter les données légères).



Je vous propose une représentation de la stack combinant **Elasticsearch**, **Logstash**, **Kibana** et **Neo4j** :

```plaintext
          +-------------------------------+
          |           Application         |
          +-------------------------------+
                        |
                        |
                        v
          +-------------------------------+
          |          REST API             |
          +-------------------------------+
                        |
          +-----------------+    +-----------------+
          |    Logstash    |    |      Neo4j      |
          +-----------------+    +-----------------+
                   |                       |
                   |                       |
                   v                       v
         +----------------+       +----------------+
         | Elasticsearch  |       |    Cypher      |
         | (Indexation    |       | (Langage de    |
         |  et Recherche) |       |  Requêtes)     |
         +----------------+       +----------------+
                    |
                    v
          +-------------------------------+
          |             Kibana            |
          | (Visualisation et Analyse)    |
          +-------------------------------+
```

# Explications de chaque composant :

- **Application** : Le logiciel ou l’interface que les utilisateurs utilisent pour interagir avec les données.
- **REST API** : Une interface permettant à l'application de communiquer avec Elasticsearch et Neo4j.
- **Logstash** : Un outil de collecte et de transformation de données, qui envoie ensuite les données à Elasticsearch.
- **Neo4j** : La base de données de graphe qui stocke et interroge les relations complexes entre les données via Cypher.
- **Elasticsearch** : Moteur de recherche pour l'indexation et la recherche de données textuelles et semi-structurées.
- **Cypher** : Langage de requête utilisé pour interroger la base de données Neo4j.
- **Kibana** : Outil de visualisation permettant d'afficher et d'analyser les données d’Elasticsearch.

Cette stack permet de gérer les données textuelles (via Elasticsearch) et relationnelles (via Neo4j) avec des visualisations dans Kibana pour une analyse efficace.

<br/>

# Annexe 1



```mermaid
flowchart TD
    %% Orientation
    %% TD = Top -> Down

    A[Application] --> B[REST API]

    %% Branchements vers traitement et graphe
    B --> C[Logstash<br/>(collecte & transformation)]
    B --> N[Neo4j<br/>(Base de données graphe)]

    %% Flux Logstash -> Elasticsearch -> Kibana
    C --> E[Elasticsearch<br/>(Indexation & Recherche)]
    E --> K[Kibana<br/>(Visualisation & Analyse)]

    %% Encadrer les composants Elastic
    subgraph Elastic_Stack[Elastic Stack]
      direction TB
      C
      E
      K
    end

    %% Encadrer Neo4j + langage
    subgraph Graph_DB[Base de graphes]
      direction TB
      N --> CY[Cypher<br/>(Langage de requêtes)]
    end

    %% (Option) Beats en amont de Logstash
    %% Décommentez ces deux lignes si vous utilisez Beats :
    %% BE[Beats<br/>(shippers légers)] --> C
    %% A --> BE
```

