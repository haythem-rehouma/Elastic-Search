# Objectif:

- **supprimer tous les nœuds (et toutes les relations)** d’une base Neo4j, la commande la plus simple est :

```cypher
MATCH (n) DETACH DELETE n
```

* `MATCH (n)` sélectionne tous les nœuds.
* `DETACH DELETE` supprime chaque nœud **ainsi que** toutes les relations qui y sont attachées (sinon la suppression échoue s’il reste des relations).

---

## Variantes utiles

### 1) Supprimer **toutes les relations uniquement** (garder les nœuds)

```cypher
MATCH ()-[r]-() DELETE r
```

### 2) Supprimer **tous les nœuds d’un label** (ex. Person) et leurs relations

```cypher
MATCH (n:Person) DETACH DELETE n
```

### 3) Grosse base (meilleure robustesse via APOC par lots)

Si vous avez le plugin APOC activé :

```cypher
CALL apoc.periodic.iterate(
  'MATCH (n) RETURN n',
  'DETACH DELETE n',
  {batchSize: 5000, parallel: true}
);
```

> Traite par paquets pour éviter de saturer la mémoire.

---

## Depuis `cypher-shell`

### Machine locale

```bash
cypher-shell -u neo4j -p 'votre_mot_de_passe' "MATCH (n) DETACH DELETE n"
```

### Docker

```bash
docker exec -it <container_neo4j> cypher-shell -u neo4j -p 'votre_mot_de_passe' "MATCH (n) DETACH DELETE n"
```

---

## (Optionnel) Réinitialiser complètement la base

Si vous voulez repartir **à zéro** (indexes/contraintes compris), vous pouvez recréer la base (Neo4j 4/5) :

Dans le Browser :

```
:use system
STOP DATABASE neo4j;
DROP DATABASE neo4j;
CREATE DATABASE neo4j;
START DATABASE neo4j;
```

Ou en ligne de commande (service arrêté) :

```bash
neo4j-admin database remove --force neo4j   # (selon version)
neo4j-admin dbms set-initial-password 'nouveau_mdp'  # si besoin
```

---

## (Optionnel) Supprimer indexes & contraintes

Après un `DETACH DELETE`, il peut rester des indexes/contraintes :

Lister et supprimer :

```cypher
SHOW INDEXES;
DROP INDEX <nom_index> IF EXISTS;

SHOW CONSTRAINTS;
DROP CONSTRAINT <nom_contrainte> IF EXISTS;
```







# Annexes 1 - erreurs avec SHOW INDEXES

- Si la version de Neo4j/Cypher ne reconnaît pas `SHOW INDEXES` (ou ton `cypher-shell` est plus ancien que le serveur), utilisez la commande compatible avec les versions 3.5–4.1 :

```cypher
CALL db.indexes();
```

### Résumé par version

* **Neo4j 5.x / 4.4** :

  ```cypher
  SHOW INDEXES;
  SHOW CONSTRAINTS;
  ```
* **Neo4j 4.0–4.1 / 3.5** :

  ```cypher
  CALL db.indexes();
  CALL db.constraints();
  ```

  *(Dans Neo4j Browser, `:schema` marche aussi.)*

### Vérifier ta version (pratique)

```cypher
RETURN version();
-- ou
CALL dbms.components();
```

### Astuce “drop” selon la version

* **Neo4j 5.x / 4.4** :

  ```cypher
  SHOW INDEXES YIELD name;
  -- Puis pour chacun :
  DROP INDEX <nom_index> IF EXISTS;

  SHOW CONSTRAINTS YIELD name;
  DROP CONSTRAINT <nom_contrainte> IF EXISTS;
  ```
* **Neo4j 4.0–4.1 / 3.5** :

  * Les index n’ont souvent pas de nom; on les supprime par la vieille syntaxe :

    ```cypher
    DROP INDEX ON :Person(name);
    ```
  * Pour les contraintes (si nommées) :

    ```cypher
    CALL db.constraints();
    -- puis DROP CONSTRAINT ON (n:Label) ASSERT n.prop IS UNIQUE;
    ```



# Annexe 2



# 1) Lister le schéma (syntaxe 4.1)

```cypher
CALL db.indexes();
CALL db.constraints();
```

# 2) Générer les commandes `DROP` 

> En 4.1, les index/contraintes ne sont généralement **pas nommés**. On les supprime avec la “vieille” syntaxe textuelle.

### a) Index

```cypher
CALL db.indexes()
YIELD description
RETURN 'DROP ' + description AS drop_cmd;
/*
Exemple de sortie :
DROP INDEX ON :Person(name)
DROP INDEX ON :Article(publishedAt)
*/
```

### b) Contraintes

```cypher
CALL db.constraints()
YIELD description
RETURN 'DROP ' + description AS drop_cmd;
/*
Exemple :
DROP CONSTRAINT ON (p:Person) ASSERT p.id IS UNIQUE
*/
```

> Exécute ensuite **chaque** `drop_cmd` retourné.

### (Optionnel) Tout faire automatiquement avec APOC

Si le plugin APOC est activé :

```cypher
CALL apoc.schema.assert({},{});
```

> Cela **supprime** tous les index/contraintes existants (et n’en recrée aucun).

# 3) Supprimer toutes les relations & nœuds

### Simple (si volume raisonnable)

```cypher
MATCH (n) DETACH DELETE n;
```

### Gros volumes (plus robuste, APOC)

```cypher
CALL apoc.periodic.iterate(
  'MATCH (n) RETURN n',
  'DETACH DELETE n',
  {batchSize: 5000, parallel: true}
);
```

# 4) Petits utilitaires utiles

* Compter rapidement :

```cypher
MATCH (n) RETURN count(n) AS nodes;
MATCH ()-[r]->() RETURN count(r) AS rels;
```

* Vérifier après nettoyage :

```cypher
CALL db.indexes();
CALL db.constraints();
MATCH (n) RETURN count(n) AS nodes;
```




