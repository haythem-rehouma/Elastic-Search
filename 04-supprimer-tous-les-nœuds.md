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


