# Partie 1 : importer les données

- https://drive.google.com/drive/folders/17vTaNQFnTyhEcizFYofLpdfd-f792utk?usp=sharing (News_Category_Dataset_v2.json)
  
## Méthode 1 (sans terminal) — via l’UI Kibana

1. Dans Kibana, ouvre le menu (☰ en haut à gauche)
2. va dans **Machine Learning → Data Visualizer → File** (ou “Upload a file”).
3. **Glisse ton fichier** dans la zone (ou clique *Select file* et prends-le depuis `~/Téléchargements`).

   * Kibana accepte directement le **NDJSON** (ton format: 1 objet JSON par ligne).
4. **Index name** : mets par ex. `news`.
5. Time field → choisis **`date`**.
6. Vérifie que `category` et `authors` sont en **keyword** (Kibana le propose souvent tout seul).
7. Clique **Import**.
8. Une fois fini, va dans **Discover**, crée une **Data View** `news*` et tu vois tes documents.

> Si ton fichier est *vraiment* gros (>100 Mo), l’UI peut galérer. Dans ce cas, prends la méthode terminal ci-dessous.



## Méthode 2 (terminal) — rapide & robuste

Supposons que ton fichier est dans `~/Téléchargements/news.ndjson`. On l’ingère à chaud.

1. Place-le dans ton dossier de travail :

```bash
mkdir -p ~/elk-dev/data
cp ~/Téléchargements/news.ndjson ~/elk-dev/data/
cd ~/elk-dev
# charge les variables de .env pour ne pas taper le mot de passe
set -a; source .env; set +a
```

2. (Optionnel) Template pour bien typer `date`, `authors`, `category` :

```bash
curl -u elastic:$ELASTIC_PASSWORD -H 'Content-Type: application/json' \
  -X PUT http://localhost:9200/_index_template/news-template -d '{
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
}}}}'
```

3. Envoi en **_bulk** (ton fichier est déjà NDJSON) :

```bash
# On ajoute la ligne d’action _bulk avant chaque doc
awk '{print "{\"index\":{\"_index\":\"news\"}}"; print}' data/news.ndjson > data/bulk.ndjson

# Si le fichier est gros, envoie par paquets de 10k lignes (~5k docs)
cd data
split -l 10000 bulk.ndjson bulk_part_

for f in bulk_part_*; do
  echo "Envoi de $f…"
  curl -s -u elastic:$ELASTIC_PASSWORD \
       -H 'Content-Type: application/x-ndjson' \
       -X POST 'http://localhost:9200/_bulk?refresh=false' \
       --data-binary "@$f" | jq '.errors'
done

# refresh (optionnel)
curl -u elastic:$ELASTIC_PASSWORD -X POST 'http://localhost:9200/news/_refresh'
```

4. Vérifie :

```bash
curl -u elastic:$ELASTIC_PASSWORD 'http://localhost:9200/news/_count?pretty'
```

Puis dans Kibana → **Discover** → crée une **Data View** `news*`.



## Où est le fichier ?

- https://drive.google.com/drive/folders/17vTaNQFnTyhEcizFYofLpdfd-f792utk?usp=sharing

* Si tu l’as téléchargé dans la **VM** via Firefox : il est probablement dans `~/Téléchargements` (ou `~/Downloads`). 
* S’il est sur **Windows (hôte)** : plus simple → re-télécharge-le **dans la VM** avec Firefox, ou configure un *dossier partagé* VirtualBox (mais ce n’est pas nécessaire si tu peux le retélécharger dans la VM).






<br/>

# Partie 2


Une fois que nos docs sont dans `news`, nous avons 3 façons principales de les interroger :

* **Discover** (Kibana) avec **KQL** – super simple pour filtrer/chercher.
* **Dev Tools → Console** avec le **DSL `_search`** (JSON).
* **ES|QL** (nouveau langage style SQL) dans **Discover → Query** ou **Dev Tools**.

Je vous fais un mémo ultra-pratique avec des exemples pour notre schéma (`category`, `authors`, `date`, `headline`, `short_description`, `link`).


# 1) KQL (barre de recherche dans Discover)


- **KQL (Kibana Query Language)** est le langage de requête intégré à la barre de recherche de **Discover** dans Kibana.
Il permet d’explorer et de filtrer rapidement les documents d’un index Elasticsearch à l’aide d’une syntaxe simple et lisible.
KQL est conçu pour les **recherches interactives**, sans avoir à écrire de JSON ni de code complexe.
Il fonctionne sur les champs des documents et gère à la fois les **filtres exacts** (sur les champs *keyword*) et les **recherches plein texte** (sur les champs *text*).
C’est la méthode la plus intuitive pour interroger visuellement des données avant de passer à des requêtes plus avancées en **ES|QL** ou en **DSL JSON**.



* Tous les articles de la catégorie “POLITICS” :

```
category : "POLITICS"
```

* Articles entre deux dates :

```
date >= "2018-05-20" and date < "2018-06-01"
```

* Recherche plein texte dans le titre et la description :

```
headline : "Trump" or short_description : "Trump"
```

* Plusieurs filtres combinés :

```
category : ("WORLD NEWS" or "POLITICS") and authors : "Ron Dicker" and date >= "2018-05-25"
```

💡 Astuces

* Si tu vois des champs **`field.keyword`**, utilise la version **`keyword`** pour les filtres exacts (KQL gère souvent ça tout seul).
* Tu peux sauvegarder ta requête comme “Saved Search”.


<br/>


## Annexe 1 pour plus d'explications et détails :

## 1) KQL (barre de recherche dans Discover)

**KQL (Kibana Query Language)** est un langage de recherche intuitif utilisé directement dans la barre de recherche de Kibana (section **Discover**).
Il te permet d’interroger les données de ton index sans écrire de code JSON. Chaque requête KQL filtre dynamiquement les documents affichés dans la vue Discover, ce qui en fait un outil idéal pour l’analyse exploratoire.



### ➤ Tous les articles de la catégorie “POLITICS”

```kql
category : "POLITICS"
```

Cette requête affiche uniquement les documents où le champ `category` est exactement égal à **POLITICS**.
Le champ `category` est de type **keyword**, ce qui signifie que la correspondance doit être **exacte** (sensible à la casse si l’index n’est pas configuré pour la normalisation).
KQL comprend automatiquement que tu veux un filtre **exact**, sans scoring de pertinence.



### ➤ Articles entre deux dates

```kql
date >= "2018-05-20" and date < "2018-06-01"
```

Cette requête sélectionne les documents dont le champ `date` se situe entre le **20 mai 2018** et le **31 mai 2018** (le 1er juin est exclu).
KQL comprend les comparateurs (`>=`, `<`, `>`, `<=`) pour les champs de type **date** ou **numérique**.
C’est une forme de **filtrage temporel** très utile pour créer des fenêtres d’analyse ou des comparaisons sur une période donnée.



### ➤ Recherche plein texte dans le titre et la description

```kql
headline : "Trump" or short_description : "Trump"
```

Ici, KQL recherche le mot **"Trump"** dans les deux champs textuels `headline` et `short_description`.
Ces champs sont de type **text**, donc Elasticsearch effectue une **analyse linguistique** (tokenisation, casse ignorée, etc.) avant de comparer les mots.
L’opérateur **OR** combine les deux champs : la requête retourne les documents contenant “Trump” dans **au moins un** des deux.



### ➤ Plusieurs filtres combinés

```kql
category : ("WORLD NEWS" or "POLITICS") and authors : "Ron Dicker" and date >= "2018-05-25"
```

Cette requête combine plusieurs conditions logiques :

* `category : ("WORLD NEWS" or "POLITICS")` → articles appartenant à **l’une** de ces deux catégories,
* `authors : "Ron Dicker"` → écrits par **Ron Dicker**,
* `date >= "2018-05-25"` → publiés après le **25 mai 2018**.
  Les opérateurs `and` et `or` peuvent être regroupés avec des parenthèses pour clarifier la logique.
  Ce type de combinaison permet de construire des **filtres complexes** en quelques mots seulement.



###  Astuces pratiques KQL

* Si Kibana affiche des champs du type `field.keyword`, choisis la version **`keyword`** pour les **filtres exacts** et les agrégations.
  Exemple : `authors.keyword : "Ron Dicker"` est plus précis que `authors : "Ron Dicker"`.
* Pour les recherches **plein texte**, utilise la version **textuelle** du champ (sans `.keyword`).
* Tu peux **sauvegarder ta requête** avec le bouton **Save Search** et la réutiliser dans **Visualize → Lens** ou dans un **Dashboard**.
* KQL est **non sensible à la casse** pour les champs `text`, mais il peut l’être pour certains `keyword` selon la configuration de l’index.
* KQL n’utilise **pas de guillemets** pour les valeurs simples sans espaces : `category : POLITICS` fonctionne aussi.





<br/>
<br/>

# 2) ES|QL (style SQL) – super lisible


**ES|QL (Elasticsearch Query Language)** est un langage introduit récemment par Elastic pour interroger les index à la manière du **SQL**, tout en tirant parti de la puissance d’Elasticsearch.
Contrairement à KQL, qui se limite à la recherche et au filtrage dans Kibana, ES|QL permet de **combiner, agréger, transformer et ordonner** les données directement dans le moteur.
Il fonctionne sur la syntaxe `FROM index | commande1 | commande2 | ...`, chaque commande transformant le flux de données (principe de *pipeline*).
C’est un langage **déclaratif et lisible**, parfait pour les analyses statistiques, les regroupements et les agrégations temporelles.
Tu peux exécuter ces requêtes dans **Discover** (en activant le mode ES|QL) ou dans **Dev Tools → Console**, avec un rendu tabulaire très clair.




Dans **Discover** (switch “ES|QL”) ou **Dev Tools** :

* Compter les docs par catégorie :

```sql
FROM news
| STATS count() BY category
| ORDER BY count DESC
| LIMIT 10
```

* Top auteurs sur une période :

```sql
FROM news
| WHERE date BETWEEN "2018-05-20" AND "2018-05-31"
| STATS count() BY authors
| ORDER BY count DESC
| LIMIT 10
```

* Lignes (docs) contenant “Trump” :

```sql
FROM news
| WHERE MATCH(headline, "Trump") OR MATCH(short_description, "Trump")
| LIMIT 20
```

* Histogramme temporel (agrégé par jour) :

```sql
FROM news
| WHERE category == "POLITICS"
| EVAL day = DATE_TRUNC(1 days, date)
| STATS count() BY day
| ORDER BY day
```


<br/>
<br/>



### ➤ Compter les documents par catégorie

```sql
FROM news
| STATS count() BY category
| ORDER BY count DESC
| LIMIT 10
```

Cette requête lit tous les documents de l’index `news` et les regroupe par **catégorie**.
L’instruction `STATS count() BY category` agit comme un `GROUP BY` SQL, en calculant le nombre de documents par valeur de `category`.
`ORDER BY count DESC` trie les résultats du plus fréquent au moins fréquent, et `LIMIT 10` affiche seulement les 10 premiers.
C’est une requête idéale pour visualiser les **catégories dominantes** dans le dataset.



### ➤ Top auteurs sur une période donnée

```sql
FROM news
| WHERE date BETWEEN "2018-05-20" AND "2018-05-31"
| STATS count() BY authors
| ORDER BY count DESC
| LIMIT 10
```

Ici, on **filtre d’abord** les documents dont la date est comprise entre le 20 et le 31 mai 2018.
Le mot-clé `WHERE` joue le rôle d’un filtre exact (similaire au `range` en DSL).
Ensuite, on regroupe par champ `authors` pour compter combien d’articles chaque auteur a publiés sur cette période.
`ORDER BY` trie du plus productif au moins productif, et `LIMIT 10` restreint le tableau à un top 10.
C’est une requête typique pour construire un **tableau de classement** des auteurs.



### ➤ Rechercher du texte dans les titres et descriptions

```sql
FROM news
| WHERE MATCH(headline, "Trump") OR MATCH(short_description, "Trump")
| LIMIT 20
```

La fonction `MATCH()` exécute une **recherche plein texte** sur des champs de type `text`.
Elle analyse les mots comme le ferait une recherche classique (`match` en DSL) et renvoie les documents pertinents.
L’opérateur `OR` combine les deux conditions : on récupère tout article mentionnant “Trump” soit dans le titre, soit dans la description.
`LIMIT 20` affiche les 20 premiers résultats triés par score de pertinence interne.
Ce type de requête est parfait pour **explorer les thèmes ou les noms récurrents** dans les articles.



### ➤ Histogramme temporel (agrégé par jour)

```sql
FROM news
| WHERE category == "POLITICS"
| EVAL day = DATE_TRUNC(1 days, date)
| STATS count() BY day
| ORDER BY day
```

Cette requête extrait uniquement les documents de la catégorie **POLITICS**, puis crée un champ virtuel `day` en **tronquant la date au jour** (`DATE_TRUNC`).
Cela permet de regrouper tous les articles d’une même journée dans un seul bucket.
`STATS count() BY day` compte combien de documents sont publiés chaque jour.
Le `ORDER BY day` classe le résultat dans l’ordre chronologique, ce qui permet de générer un **graphique temporel** directement dans Kibana.
C’est la base d’une **analyse de tendance** sur la fréquence des publications politiques dans le temps.



###  Astuces ES|QL

* `STATS` peut calculer plusieurs agrégations à la fois (`count()`, `avg(field)`, `max(field)`, etc.).
* `EVAL` permet de **créer des colonnes calculées** (comme `DATE_TRUNC`, `LOWER`, `CONCAT`, etc.).
* `KEEP` limite les colonnes affichées, ce qui simplifie la lecture des résultats.
* `MATCH()` sert au **plein texte**, tandis que `==` et `!=` servent aux **comparaisons exactes**.
* ES|QL est **case-insensitive** pour les commandes (`FROM`, `WHERE`, `STATS`, etc.) mais sensible pour les **valeurs** de champs.


<br/>
<br/>



# 3) DSL JSON (Dev Tools → Console) + équivalents `curl`


### En résumé 

Le **DSL JSON (Domain Specific Language)** est le langage natif d’Elasticsearch pour interroger, filtrer et agréger les données sous forme d’objets JSON.
Il est plus **puissant et précis** que KQL ou ES|QL, car il donne un contrôle complet sur les champs, le scoring, les filtres et les agrégations.
Les requêtes s’écrivent dans **Kibana → Dev Tools → Console** ou directement en **curl** depuis le terminal.
Chaque requête JSON peut combiner plusieurs blocs (`query`, `aggs`, `sort`, `size`, etc.) qui décrivent **ce que tu veux récupérer** et **comment tu veux le structurer**.
C’est le format utilisé par toutes les applications qui communiquent directement avec Elasticsearch, que ce soit Kibana, Python, ou Node.js.



###  Exemple détaillé : recherche plein texte multi-champs

```http
GET news/_search
{
  "query": {
    "multi_match": {
      "query": "Trump",
      "fields": ["headline^2", "short_description"]
    }
  },
  "size": 20
}
```

#### Explication ligne par ligne :

➡️ **`GET news/_search`**
→ Indique à Elasticsearch de chercher (`_search`) dans l’index nommé **`news`**.
→ C’est l’équivalent d’un `SELECT * FROM news` en SQL.

➡️ **`"query": { ... }`**
→ Le bloc principal de la recherche : il définit les **critères de filtrage**.
→ Sans ce bloc, Elasticsearch renverrait tous les documents (comme un “match_all”).

➡️ **`"multi_match": { ... }`**
→ Type de requête plein texte qui cherche la même expression dans **plusieurs champs**.
→ Idéal quand le contenu est réparti entre `headline` (titre) et `short_description` (texte).

➡️ **`"query": "Trump"`**
→ Le mot-clé recherché. Elasticsearch le passe dans un **analyseur linguistique** : il enlève la casse, découpe les mots, gère les racines.
→ Cela permet de trouver *“Trump’s”*, *“Donald Trump”*, ou *“Trumpian”* selon l’analyseur configuré.

➡️ **`"fields": ["headline^2", "short_description"]`**
→ Liste des champs où chercher. Le symbole **`^2`** indique un **boost** : le champ `headline` compte deux fois plus dans le score.
→ Ainsi, un article où “Trump” apparaît dans le titre aura un score de pertinence plus élevé qu’un article où il n’apparaît que dans la description.

➡️ **`"size": 20`**
→ Nombre maximum de documents renvoyés.
→ Par défaut Elasticsearch renvoie 10 documents ; ici on augmente la limite à 20.



**Résultat attendu :**
La requête renvoie les **20 articles** les plus pertinents contenant le mot “Trump” dans le **titre** ou la **description**, avec un score plus fort pour ceux où il figure dans le titre.




# 3.a) Recherche plein texte

```json
GET news/_search
{
  "query": {
    "multi_match": {
      "query": "Trump",
      "fields": ["headline^2", "short_description"]
    }
  },
  "size": 20
}
```




## 3.a.1. Objectif de la requête

Cette requête :

```json
GET news/_search
{
  "query": {
    "multi_match": {
      "query": "Trump",
      "fields": ["headline^2", "short_description"]
    }
  },
  "size": 20
}
```

cherche les **20 documents les plus pertinents** contenant le mot **“Trump”** dans le champ `headline` (titre) ou `short_description` (texte descriptif).
Le champ `headline` est **boosté** (avec `^2`) : cela signifie que le moteur donnera **plus d’importance** aux occurrences du mot dans le titre qu’à celles dans la description.



## 3.a.2. Structure typique de la réponse JSON

Je vous présente un exemple simplifié et réaliste de la réponse reçue :

```json
{
  "took": 34,
  "timed_out": false,
  "hits": {
    "total": { "value": 129, "relation": "eq" },
    "max_score": 7.412358,
    "hits": [
      {
        "_index": "news",
        "_id": "r8dXo44B9x8S9t3cY1kT",
        "_score": 7.412358,
        "_source": {
          "date": "2018-05-26T14:23:00Z",
          "category": "POLITICS",
          "authors": "Ron Dicker",
          "headline": "Trump Meets With North Korean Envoy At The White House",
          "short_description": "The president met an envoy amid ongoing talks; details on denuclearization remained unclear.",
          "link": "https://www.example.com/politics/trump-meets-envoy"
        },
        "highlight": {
          "headline": [
            "<em>Trump</em> Meets With North Korean Envoy At The White House"
          ]
        }
      }
    ]
  }
}
```



## 3.a.3. Lecture détaillée du résultat:

### Métadonnées globales

* **`"took": 34`**
  Temps d’exécution en millisecondes. Ici, la recherche a pris 34 ms.

* **`"timed_out": false`**
  Indique que la requête s’est terminée normalement. Si `true`, cela veut dire que le temps limite a été dépassé et les résultats peuvent être incomplets.

* **`"hits.total.value": 129`**
  Nombre total de documents correspondant à la requête. Ici, Elasticsearch a trouvé **129 articles** contenant “Trump”.

* **`"hits.max_score": 7.412358`**
  Score de pertinence du document le plus proche du mot recherché.
  Plus le score est élevé, plus le document est jugé pertinent par le moteur.



###  Section `"hits.hits"` : les documents eux-mêmes

Chaque élément de ce tableau représente **un article trouvé**.
Analysons un seul document en détail.

#### a) `_index`

* Valeur : `"news"`.
  Nom de l’index dans lequel se trouve le document.
  Cela confirme que la recherche a bien été effectuée dans la bonne base de données Elasticsearch.

#### b) `_id`

* Valeur : `"r8dXo44B9x8S9t3cY1kT"`.
  Identifiant unique attribué par Elasticsearch au moment de l’ingestion du document.
  Il permet de retrouver, mettre à jour ou supprimer précisément ce document.

#### c) `_score`

* Valeur : `7.412358`.
  C’est le **score de pertinence** calculé automatiquement par le moteur selon l’algorithme **BM25**.
  Plus ce score est haut, plus le moteur estime que le document répond bien à la recherche.
  Le score dépend :

  * du nombre d’occurrences du mot “Trump” dans le texte,
  * du type de champ (ici, `headline` a un poids plus élevé à cause du boost `^2`),
  * de la longueur du texte (les textes courts peuvent obtenir un score plus fort si le mot est fréquent).

#### d) `_source`

C’est le contenu original du document, c’est-à-dire les données que tu as ingérées.
Ici, on retrouve :

* `"date"` : `"2018-05-26T14:23:00Z"`
  → format ISO standard, lisible par Kibana pour les graphiques temporels.
* `"category"` : `"POLITICS"`
  → champ typique de type `keyword`, utile pour les filtres et agrégations.
* `"authors"` : `"Ron Dicker"`
  → auteur de l’article.
* `"headline"` : `"Trump Meets With North Korean Envoy At The White House"`
  → champ `text`, analysé pour la recherche plein texte.
* `"short_description"` : `"The president met an envoy..."`
  → texte complémentaire servant de support pour le plein texte.
* `"link"` : `"https://www.example.com/politics/trump-meets-envoy"`
  → URL vers l’article original, souvent utile pour les visualisations ou tableaux.

#### e) `"highlight"`

* Permet de visualiser les mots-clés **trouvés** dans les champs recherchés.
  Les termes détectés sont entourés de balises `<em>...</em>`.
  Exemple :

  ```html
  "<em>Trump</em> Meets With North Korean Envoy At The White House"
  ```

  Cela indique que le moteur a trouvé une correspondance sur le mot “Trump” dans le titre.
  Cette fonctionnalité est essentielle pour mettre en valeur les résultats dans une interface web ou dans Kibana.



## 3.a.4. Interprétation des résultats

Voici comment tu peux **interpréter et commenter les résultats** de manière analytique :

1. **Pertinence** :
   Le document avec le score `7.41` est classé en premier car le mot “Trump” apparaît **dans le titre**, un champ boosté.
   Si un autre document ne contenait “Trump” que dans la description, son score aurait été plus faible (par exemple `5.2`).

2. **Cohérence des scores** :
   Les scores varient selon la densité du mot-clé, sa position dans le texte, et la longueur totale du document.
   Un document court contenant deux occurrences de “Trump” peut dépasser un long texte où il n’apparaît qu’une fois.

3. **Nature du tri** :
   Par défaut, les résultats sont triés par `_score` (pertinence).
   Si tu voulais les trier par date, tu devrais ajouter :

   ```json
   "sort": [{ "date": "desc" }]
   ```

   mais cela désactive l’ordre par pertinence.

4. **Utilisation pratique** :
   Dans une interface Kibana ou une application web, tu pourrais afficher :

   * le titre (`headline`),
   * la catégorie (`category`),
   * la date (`date`),
   * et la phrase surlignée (`highlight.short_description`)
     pour permettre à l’utilisateur de comprendre pourquoi cet article a été sélectionné.

5. **Qualité du mapping** :
   Pour obtenir des résultats fiables :

   * `headline` et `short_description` doivent être **de type `text`** ;
   * `category`, `authors`, et `link` doivent être **de type `keyword`** ;
   * `date` doit être **de type `date`**.
     Un mapping mal défini (par exemple `category` en `text`) provoquerait des erreurs ou des incohérences dans les filtres.



## 3.a.5. Exemple d’analyse complète et interprétée

**Requête exécutée :**

> Rechercher les 20 articles contenant “Trump” dans le titre ou la description.

**Résultat :**

> 129 articles trouvés.
> Le premier article s’intitule *“Trump Meets With North Korean Envoy At The White House”*, catégorie *POLITICS*, publié le *26 mai 2018*, auteur *Ron Dicker*.
> Son score de pertinence est **7.41**, le plus élevé, car le mot “Trump” est présent dans le **titre**, champ pondéré doublement (`^2`).
> Le moteur met en évidence le mot dans le texte avec `<em>Trump</em>`, indiquant la zone exacte de correspondance.
> Les autres articles mentionnant “Trump” uniquement dans le texte descriptif obtiennent un score inférieur (entre 5 et 6), car la pondération y est plus faible.


<br/>
<br/>


# 3.b) Filtres exacts + plage de dates

```json
GET news/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "category": "POLITICS" } }
      ],
      "filter": [
        { "range": { "date": { "gte": "2018-05-20", "lt": "2018-06-01" } } }
      ]
    }
  },
  "sort": [{ "date": "desc" }]
}
```




## Ce que fait la requête

* Elle recherche dans l’index `news` tous les documents dont `category` est exactement `POLITICS` (match exact sur un champ de type keyword).
* Elle filtre ensuite ces documents sur une plage de dates: `date >= 2018-05-20` et `date < 2018-06-01` (inclusif à gauche, exclusif à droite).
* Elle renvoie les résultats triés par `date` décroissante (les plus récents en premier).
* Comme le tri est explicite par `date`, l’ordre n’utilise pas la pertinence; le `_score` des hits peut être `null`.
* Si `size` n’est pas précisé, Elasticsearch renvoie 10 documents par défaut; ajoute `"size": N` si tu veux plus de résultats.

## Exemple de réponse (extrait réaliste)

```json
{
  "took": 21,
  "timed_out": false,
  "hits": {
    "total": { "value": 342, "relation": "eq" },
    "max_score": null,
    "hits": [
      {
        "_index": "news",
        "_id": "A1B2C3",
        "_score": null,
        "sort": ["2018-05-31T23:04:55Z"],
        "_source": {
          "date": "2018-05-31T23:04:55Z",
          "category": "POLITICS",
          "authors": "Ron Dicker",
          "headline": "White House Responds To Senate Proposal",
          "short_description": "Officials addressed key points late Thursday.",
          "link": "https://www.example.com/politics/white-house-responds"
        }
      },
      {
        "_index": "news",
        "_id": "D4E5F6",
        "_score": null,
        "sort": ["2018-05-31T20:12:10Z"],
        "_source": {
          "date": "2018-05-31T20:12:10Z",
          "category": "POLITICS",
          "authors": "Lee Moran",
          "headline": "Lawmakers React To Late-Night Briefing",
          "short_description": "Key reactions following a closed-door meeting.",
          "link": "https://www.example.com/politics/lawmakers-react"
        }
      }
    ]
  }
}
```

## Comment analyser le résultat

* `took` et `timed_out` indiquent respectivement la durée d’exécution en millisecondes et l’absence de timeout.
* `hits.total.value` te donne le volume total de documents qui satisfont les deux contraintes (catégorie et période).
* Chaque entrée de `hits.hits` contient les données ingérées dans `_source`; vérifie que `category` est `POLITICS` et que `date` se situe bien dans l’intervalle.
* `_score` est `null` parce que le tri par `date` a été demandé; l’ordre des résultats est déterminé par la valeur de `date`, pas par la pertinence.
* Le tableau `sort` renvoie la clé de tri effective; c’est utile pour paginer proprement avec `search_after` si tu dois parcourir de nombreuses pages de résultats.






<br/>
<br/>

# 3.c) Top N catégories (aggregation)

```json
GET news/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category", "size": 10 }
    }
  }
}
```

> Exercices partie 1 : expliquez la requête

### d) Histogramme temporel

```json
GET news/_search
{
  "size": 0,
  "query": { "term": { "category": "POLITICS" } },
  "aggs": {
    "per_day": {
      "date_histogram": { "field": "date", "calendar_interval": "day" }
    }
  }
}
```


<br/>
<br/>

# 3.e) Mettre en avant (highlight)

```json
GET news/_search
{
  "query": {
    "multi_match": {
      "query": "North Korea",
      "fields": ["headline", "short_description"]
    }
  },
  "highlight": {
    "fields": {
      "headline": {},
      "short_description": {}
    }
  }
}
```

> Exercices partie 2 : expliquez la requête

### f) `curl` depuis ta VM (tu as les vars `.env`)

```bash
curl -u elastic:$ELASTIC_PASSWORD 'http://localhost:9200/news/_search?pretty' \
  -H 'Content-Type: application/json' -d '{
  "query": { "term": { "category": "POLITICS" } },
  "size": 5,
  "sort": [{ "date": "desc" }]
}'
```

> Exercices partie 3 : expliquez la requête

<br/>
<br/>

## Choisir le bon champ (text vs keyword)

* **`text`** (ex: `headline`, `short_description`) → recherche plein texte (`match`, `multi_match`, `MATCH()` en ES|QL).
* **`keyword`** (ex: `category`, `authors`, `link`) → filtres exacts, agrégations (`term`, `terms`, `BY category`, etc.).
* **`date`** (`date`) → filtres `range`, histogrammes, ES|QL `DATE_TRUNC`.



## Aller plus loin

* Transforme ta requête **Discover** en visualisation (Lens) → graphes en 2 clics.
* Sauvegarde des **Data Views** (`news*`) et des **Dashboards**.
* Si tu veux **compter par mot** dans les titres, crée un *runtime field* qui tokenize, ou utilise l’agg `significant_text` sur `headline`.

<br/>

# Partie 3

- Allez à **“Dev Tools → Console”**. :'objectif est d'écrire des requêtes prêtes pour notre index `news`.
(collez chaque bloc tel quel dans **Kibana → Dev Tools → Console**, et exécute avec ▶)


-----------------------------------------
# Manipulations pour la Sanity checks
----------------------------------------

# 0) Sanity checks

```http
GET /
GET _cat/indices?v
GET news/_count
GET news/_mapping
```

* `/_mapping` te montre les types de champs (ex: `text`, `keyword`, `date`).
  Pour des **filtres exacts** et des **aggrégations**, privilégie les champs `keyword` (ex: `category`, `authors`).
  Pour le **plein texte**, utilise les champs `text` (ex: `headline`, `short_description`).




## 1) `GET /`

**But** : vérifier que le cluster Elasticsearch répond.
**Ce que ça retourne** : métadonnées du cluster (nom, version, tagline).
**À vérifier** :

* La version d’Elasticsearch (utile si tu suis une doc ciblant une version précise).
* Que tu obtiens un 200 OK.
  **Si problème** : service non démarré, port incorrect, auth manquante.



## 2) `GET _cat/indices?v`

**But** : lister les indices, leur état et leur volumétrie.
**Ce que ça retourne** : un tableau texte (colonnes) avec `health`, `status`, `index`, `uuid`, `pri`, `rep`, `docs.count`, `store.size`, etc.
**À vérifier** :

* Que l’index `news` existe (colonne `index`), son `health` (green/yellow/red), et le `docs.count` attendu après ingestion.
* Si `news` est absent, ta phase d’ingestion n’a pas abouti (reprendre import/bulk).
* Si `health` est `red`, vérifier les logs et la disponibilité des shards.



## 3) `GET news/_count`

**But** : compter rapidement le nombre de documents ingérés dans `news`.
**Ce que ça retourne** : un JSON avec `{"count": <nombre>, ...}`.
**À vérifier** :

* Que `count` correspond à l’ordre de grandeur attendu (ex. après ton `_bulk`).
* Écart significatif = ingestion incomplète (chunk manquant, erreurs `_bulk`) ou mauvais index cible.
  **Astuce** : après gros `_bulk`, un `POST news/_refresh` peut rendre immédiatement visibles tous les docs pour ce count.



## 4) `GET news/_mapping`

**But** : connaître le **mapping** effectif des champs dans l’index (types, analyzers, sous-champs `.keyword`).
**Ce que ça retourne** : un JSON détaillé listant les propriétés de `news`.
**À vérifier (critique)** :

* Champs de recherche plein texte en **`text`** (ex. `headline`, `short_description`).
* Champs pour filtres exacts et agrégations en **`keyword`** (ex. `category`, `authors`, `link`).
* Champs temporels en **`date`** avec un `format` compatible (ex. ISO 8601).
* Présence éventuelle de sous-champs `field.keyword` (créés par certains templates dynamiques) utiles pour agrégations.

**Exemple (extrait) :**

```json
{
  "news": {
    "mappings": {
      "properties": {
        "date": { "type": "date", "format": "strict_date_optional_time||epoch_millis" },
        "category": { "type": "keyword" },
        "authors": { "type": "keyword" },
        "headline": { "type": "text" },
        "short_description": { "type": "text" },
        "link": { "type": "keyword" }
      }
    }
  }
}
```



## Pourquoi ce contrôle de mapping est essentiel

* **Filtres exacts et agrégations** : ils nécessitent des champs **`keyword`**.
  Exemple : `terms` agg, `term`/`terms` query, `STATS BY category` en ES|QL.
* **Recherche plein texte** : nécessite des champs **`text`** (analyzers, tokenisation).
  Exemple : `match`/`multi_match`, `MATCH(field, "...")` en ES|QL.
* **Séries temporelles** : nécessitent **`date`** (plages `range`, `date_histogram`, `DATE_TRUNC`).



## Erreurs fréquentes et corrections rapides

* **`field [X] of type [text] is not aggregatable`**
  Cause : le champ est `text`.
  Correction : utiliser `X.keyword` si disponible, sinon remapper le champ en `keyword` via un template puis réindexer.

* **Dates non reconnues ou tri impossible par `date`**
  Cause : champ mappé en `text` au lieu de `date`.
  Correction : créer un mapping correct (ou un pipeline d’ingest pour normaliser le format), réindexer.

* **`news` introuvable dans `_cat/indices`**
  Cause : échec d’ingestion ou index name différent.
  Correction : rejouer l’import (UI Kibana “Upload a file” ou `_bulk`) et confirmer l’index cible.


## Ordre conseillé avant d’interroger

1. `GET /` pour la disponibilité.
2. `GET _cat/indices?v` pour vérifier l’existence et la santé de `news`.
3. `GET news/_count` pour valider la volumétrie.
4. `GET news/_mapping` pour confirmer que les types de champs sont adéquats pour les requêtes que tu vas exécuter.

Avec ces vérifications, tu évites la plupart des surprises lors des requêtes KQL, ES|QL et DSL JSON.





<br/>
<br/>



-------------------------------------
# EXERCICES - PARTIE 4
-------------------------------------

> Expliquez ces requêtes

# 1) Lire des documents

## Derniers documents (par date décroissante)

```http
GET news/_search
{
  "size": 10,
  "sort": [{ "date": "desc" }]
}
```

## Filtre exact (catégorie)

```http
GET news/_search
{
  "size": 10,
  "query": { "term": { "category": "POLITICS" } },
  "sort": [{ "date": "desc" }]
}
```

## Plusieurs catégories + plage de dates

```http
GET news/_search
{
  "size": 20,
  "query": {
    "bool": {
      "filter": [
        { "terms": { "category": ["POLITICS", "WORLD NEWS"] } },
        { "range": { "date": { "gte": "2018-05-20", "lt": "2018-06-01" } } }
      ]
    }
  },
  "sort": [{ "date": "desc" }]
}
```

## Plein texte dans titre + description (avec boost du titre)

```http
GET news/_search
{
  "size": 20,
  "query": {
    "multi_match": {
      "query": "Trump",
      "fields": ["headline^2", "short_description"]
    }
  },
  "highlight": {
    "fields": { "headline": {}, "short_description": {} }
  }
}
```

## Combiner plein texte + filtres exacts

```http
GET news/_search
{
  "size": 20,
  "query": {
    "bool": {
      "must": {
        "multi_match": {
          "query": "North Korea",
          "fields": ["headline", "short_description"]
        }
      },
      "filter": [
        { "term": { "category": "WORLD NEWS" } },
        { "range": { "date": { "gte": "2018-05-20", "lt": "2018-06-01" } } }
      ]
    }
  },
  "sort": [{ "date": "desc" }]
}
```



# 2) Tri & pagination

## Pagination simple (jusqu’à ~10k docs)

```http
GET news/_search
{
  "from": 0,
  "size": 10,
  "sort": [{ "date": "desc" }]
}
```

## Pagination “grande échelle” (recommandée) avec `search_after`

1ʳᵉ page :

```http
GET news/_search
{
  "size": 10,
  "sort": [{ "date": "desc" }, { "_id": "asc" }]
}
```

Prends les 2 dernières valeurs de `sort` du dernier hit, puis :

```http
GET news/_search
{
  "size": 10,
  "sort": [{ "date": "desc" }, { "_id": "asc" }],
  "search_after": ["2018-05-26T00:00:00Z", "SOME_ID_HERE"]
}
```



# 3) Aggrégations (stats/group by)

## Compter par catégorie (Top 10)

```http
GET news/_search
{
  "size": 0,
  "aggs": {
    "by_category": { "terms": { "field": "category", "size": 10 } }
  }
}
```

## Histogramme par jour (POLITICS)

```http
GET news/_search
{
  "size": 0,
  "query": { "term": { "category": "POLITICS" } },
  "aggs": {
    "per_day": {
      "date_histogram": { "field": "date", "calendar_interval": "day" }
    }
  }
}
```

## Top auteurs sur une période

```http
GET news/_search
{
  "size": 0,
  "query": { "range": { "date": { "gte": "2018-05-20", "lt": "2018-05-31" } } },
  "aggs": {
    "by_author": {
      "terms": { "field": "authors", "size": 10 }
    }
  }
}
```

## Top N titres par catégorie (ex: 2 titres)

```http
GET news/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category", "size": 5 },
      "aggs": {
        "latest_titles": {
          "top_hits": {
            "size": 2,
            "_source": ["headline","date","link"],
            "sort": [{ "date": "desc" }]
          }
        }
      }
    }
  }
}
```



# 4) ES|QL (option lisible “style SQL”)

Tu peux aussi exécuter ces requêtes dans **Dev Tools** (onglet **ES|QL**) :

## Compter par catégorie

```sql
FROM news
| STATS count() BY category
| ORDER BY count DESC
| LIMIT 10
```

## Filtrer par date + top auteurs

```sql
FROM news
| WHERE date BETWEEN "2018-05-20" AND "2018-05-31"
| STATS count() BY authors
| ORDER BY count DESC
| LIMIT 10
```

## Plein texte avec `MATCH`

```sql
FROM news
| WHERE MATCH(headline, "Trump") OR MATCH(short_description, "Trump")
| KEEP headline, category, authors, date, link
| ORDER BY date DESC
| LIMIT 20
```

## Histogramme (agrégé par jour)

```sql
FROM news
| WHERE category == "POLITICS"
| EVAL day = DATE_TRUNC(1 days, date)
| STATS count() BY day
| ORDER BY day
```



# 5) Petits tips

* Erreur `field [X] of type [text] ... not aggregatable` → utilise la variante **`keyword`** du champ si elle existe (ex: `authors.keyword`) OU veille à ingérer `authors` en `keyword` (ce que tu as probablement déjà).
* Dates : assure-toi que `date` est bien mappé en `date`. Si tu as des formats non ISO, ajoute un pipeline d’ingest ou remappe.
* Pour **sauver** une requête Discover, clique **Save** → tu pourras l’utiliser dans un **Dashboard**.



# EXERCICES - PARTIE 5

- Trouvez le *Top 5 catégories entre 2018-05-20 et 2018-05-31 puis listez des 10 titres les plus récents de POLITICS*





<br/>
<br/>


# EXERCICES - PARTIE 6

*(Index : `news` — Champ date : `date`)*

### Règles générales

* Toutes les réponses doivent être des **blocs JSON complets**, prêts à exécuter dans **Kibana → Dev Tools → Console**.
* Spécifier toujours le **tri**, les **champs retournés** (`_source`), et la **limite** (`size`) quand c’est demandé.




## **PARTIE A — Lecture et filtres simples (Niveau 1/5)**

1. **Afficher les 10 derniers documents**, triés par `date` décroissante, en ne retournant que `headline`, `date`, `category`, `authors` et `link`.
2. **Afficher tous les documents** dont `category = "POLITICS"`, triés par `date desc`, `size: 10`.
3. **Afficher les documents** où `category` **n’est pas** `"POLITICS"` **et** où le champ `authors` **existe**.
4. **Afficher les documents** publiés entre le `2018-05-20` et le `2018-06-01`, triés par `date desc`.
5. **Afficher les documents** appartenant à l’une des catégories : `"POLITICS"`, `"WORLD NEWS"`, `"CRIME"`.
   Trier par `date desc`, limiter à 20 résultats.



## **PARTIE B — Recherches plein texte (Niveau 2/5)**

6. **Chercher le mot “Trump”** dans `headline` et `short_description`, avec un **boost** sur `headline`.
7. **Chercher l’expression exacte “North Korea”** dans `headline`, tri `date desc`.
8. Même périmètre que (7) : **ajouter un `highlight`** sur `headline` et `short_description`.
9. **Afficher les documents** dont la `category` **commence par “POL”**, tri `date desc`.
10. **Afficher 20 documents** dont le champ `authors` est **absent ou vide**.



## **PARTIE C — Agrégations et regroupements (Niveau 3/5)**

11. **Top 10 des catégories** : `size: 0`, agrégation `terms` sur `category`, ordre par `_count desc`.
    11-bis. **Afficher toutes les catégories distinctes**, triées **alphabétiquement** (ordre `_key asc`, `size: 0`).
12. **Histogramme par jour** des articles où `category = "POLITICS"`.
13. **Top 10 auteurs** pour la période `2018-05-25` → `2018-05-31`.
14. **Par catégorie**, afficher les **3 titres les plus récents** (`headline, date, link`) : `terms` + `top_hits`.
15. **Détecter les doublons d’URL** : `terms` sur `link` avec `min_doc_count: 2`.



## **PARTIE D — Déduplication et expressions régulières (Niveau 4/5)**

16. **Afficher un seul document par URL** en utilisant `collapse` sur `link`, tri `date desc`.
17. **Afficher 20 documents** dont `authors` **se termine par** “er” (regex).
18. **Requête complète** : plein texte `"election"` (multi-champs `headline` + `short_description` boosté)
     + filtre sur `category ∈ ["POLITICS","WORLD NEWS"]`
     + période `2018-05-20` → `2018-06-01`, tri `date desc`, retourner uniquement `headline, date, category, authors, link`.
19. **Agrégation composite** sur `category` (taille 5) : produire la **1ʳᵉ page** et indiquer comment obtenir la suivante (`after`).
20. **Agrégation multi-niveaux** : `terms` par `category` → sous-`terms` par `authors` (taille 5) → sous-`top_hits` (dernier article, tri `date desc`).



## **PARTIE E — Tri avancé et pagination (Niveau 4–5/5)**

21. **Tri combiné** : résultats classés d’abord par **pertinence**, puis par `date desc`.
22. **Pagination `search_after` — page 1** : trier par `date desc`, puis `_id asc`, `size: 10`.
23. **Pagination `search_after` — page 2** : réémettre la requête précédente avec les **valeurs de tri** de la dernière ligne de la page 1.



## **PARTIE F — Analyse et calculs (Niveau 5/5)**

24. **Extraction de termes significatifs** : `significant_text` sur `headline`, filtrer `category = "POLITICS"`.
25. **Champ calculé (`runtime_mappings`)** : calculer la **longueur en mots** de `short_description`,
     trier les résultats par ce champ décroissant, retourner les 10 plus longs.


