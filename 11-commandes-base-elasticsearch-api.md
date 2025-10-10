---------------------------------
# Commandes de base :
---------------------------------

### 1. **Vérifier les informations du cluster**
   Obtenir les informations générales sur le cluster, comme la version d'Elasticsearch et les détails du nœud.
   ```bash
   curl -k -XGET "https://localhost:9200/"
   ```

### 2. **Lister tous les indices**
   Afficher tous les indices présents dans le cluster avec leurs statuts.
   ```bash
   curl -k -XGET "https://localhost:9200/_cat/indices?v"
   ```

### 3. **Vérifier l’état des nœuds**
   Obtenir une vue d'ensemble des nœuds du cluster, y compris leur nom, leur utilisation de la mémoire, et d'autres détails importants.
   ```bash
   curl -k -XGET "https://localhost:9200/_cat/nodes?v"
   ```

### 4. **Obtenir les statistiques des indices**
   Afficher des statistiques détaillées pour tous les indices, comme le nombre de documents et l'espace de stockage.
   ```bash
   curl -k -XGET "https://localhost:9200/_stats"
   ```

### 5. **Rechercher dans un index**
   Effectuer une recherche simple dans un index pour des documents qui correspondent à un terme donné.
   ```bash
   curl -k -XGET "https://localhost:9200/nom_index/_search?q=champ:valeur"
   ```
   Remplacez `nom_index` par le nom de votre index et `champ:valeur` par le champ et la valeur que vous souhaitez rechercher.

### 6. **Obtenir les informations de mappage d'un index**
   Afficher le mappage d'un index spécifique, qui montre la structure des champs et types de données.
   ```bash
   curl -k -XGET "https://localhost:9200/nom_index/_mapping"
   ```

### 7. **Ajouter un document à un index**
   Créer un document dans un index spécifique.
   ```bash
   curl -k -XPOST "https://localhost:9200/nom_index/_doc/1" -H 'Content-Type: application/json' -d'
   {
     "champ": "valeur"
   }'
   ```
   Remplacez `nom_index` par le nom de votre index, `1` par l’ID du document, et `champ: valeur` par vos données JSON.

### 8. **Mettre à jour un document existant**
   Mettre à jour un document en utilisant une requête partielle pour modifier seulement certains champs.
   ```bash
   curl -k -XPOST "https://localhost:9200/nom_index/_update/1" -H 'Content-Type: application/json' -d'
   {
     "doc": {
       "champ": "nouvelle_valeur"
     }
   }'
   ```
   Remplacez `nom_index` et `1` par l'index et l’ID du document que vous souhaitez mettre à jour.

### 9. **Supprimer un document**
   Supprimer un document spécifique d’un index.
   ```bash
   curl -k -XDELETE "https://localhost:9200/nom_index/_doc/1"
   ```

### 10. **Supprimer un index**
   Supprimer un index complet.
   ```bash
   curl -k -XDELETE "https://localhost:9200/nom_index"
   ```

Ces commandes vous permettent de gérer de nombreux aspects d'un cluster Elasticsearch. Assurez-vous de remplacer les valeurs de `localhost`, `nom_index`, `champ`, `valeur` et `ID` par celles spécifiques à votre configuration et à vos données.
