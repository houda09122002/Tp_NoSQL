

# Introduction

Avec l’augmentation continue des volumes de données et du nombre d’utilisateurs, les bases de données doivent être capables de **passer à l’échelle** tout en conservant de bonnes performances. Les bases de données NoSQL, et en particulier MongoDB, proposent des mécanismes adaptés à ces contraintes, notamment le **partitionnement des données**, appelé *sharding*.

Ce TP a pour objectif de **mettre en place et comprendre le fonctionnement du sharding sous MongoDB**, à travers la construction d’un cluster shardé composé de plusieurs composants (config server, routeur et shards). Il permet d’observer concrètement comment les données sont réparties sur plusieurs serveurs et comment MongoDB assure l’équilibrage de la charge et la transparence pour le client.

---

# 1. Le partitionnement des données (Sharding)

Le **partitionnement des données**, ou *sharding*, est une technique qui consiste à **diviser horizontalement une collection de données** et à répartir ces données sur plusieurs serveurs appelés *shards*. Chaque shard stocke **une partie seulement** des données, ce qui permet de gérer des volumes très importants sans surcharger un seul serveur.

Dans MongoDB, le sharding est utilisé principalement pour :

* améliorer la **scalabilité horizontale** (ajout de nouveaux serveurs),
* répartir la **charge de stockage**,
* répartir la **charge des requêtes** (lecture et écriture),
* éviter les limites matérielles d’un serveur unique.

La répartition des données repose sur une **clé de partitionnement** (*shard key*). Cette clé détermine sur quel shard un document sera stocké. MongoDB découpe les données en blocs appelés **chunks**, chacun correspondant à une plage de valeurs de la clé de sharding. Ces chunks peuvent être déplacés automatiquement entre les shards afin de maintenir un bon équilibre, grâce à un processus appelé le **balancer**.

Il est important de distinguer le sharding de la réplication :

* la **réplication** vise la haute disponibilité en dupliquant les mêmes données,
* le **sharding** vise la montée en charge en répartissant des données différentes.

Dans ce TP, le sharding est mis en œuvre à l’aide d’un cluster MongoDB composé de plusieurs éléments interconnectés.

---

# 2. Mise en place de l’environnement de travail : ouverture des six terminaux

Avant de démarrer les différents composants du cluster MongoDB shardé, il est nécessaire de préparer un environnement de travail clair et organisé. Pour cela, six terminaux distincts sont ouverts, chacun étant dédié à un rôle précis dans l’architecture du cluster. Cette séparation permet de mieux comprendre le fonctionnement distribué du système et de suivre plus facilement l’exécution de chaque composant.

Les six terminaux ouverts correspondent aux rôles suivants :

* **Terminal 1 – Config Server** :
  Il est dédié au serveur de configuration, qui stocke les métadonnées du cluster shardé, notamment la localisation des chunks et la configuration du sharding.

* **Terminal 2 – Shard 1** :
  Ce terminal est utilisé pour lancer le premier shard, qui stockera une partie des données de la base.

* **Terminal 3 – Shard 2** :
  Ce terminal est utilisé pour lancer le second shard, chargé de stocker une autre partie des données.

* **Terminal 4 – Router (mongos)** :
  Il correspond au routeur MongoDB (*mongos*), qui agit comme point d’entrée du cluster et redirige les requêtes vers les shards appropriés.

* **Terminal 5 – Initialisation des replica sets** :
  Ce terminal est utilisé pour initialiser les replica sets des différents composants (config server et shards).

* **Terminal 6 – Client** :
  Ce terminal permet de se connecter au cluster en tant que client afin d’exécuter des commandes MongoDB et d’observer le comportement du sharding.

Cette organisation en plusieurs terminaux reflète l’architecture distribuée d’un cluster MongoDB shardé et facilite la compréhension du rôle de chaque composant dans le fonctionnement global du système.


# 3. Mise en place du cluster MongoDB shardé

## 3.1 Création des répertoires de stockage

### Commandes exécutées

```js
mkdir configsvrdb
mkdir serv1
mkdir serv2
```

### Explication 

Avant de démarrer les différents composants du cluster MongoDB, des répertoires de stockage distincts ont été créés. Le répertoire `configsvrdb` est destiné au serveur de configuration, tandis que les répertoires `serv1` et `serv2` sont respectivement utilisés par les deux shards. Cette séparation est nécessaire car MongoDB impose qu’un seul serveur utilise un répertoire de données donné, afin d’éviter toute corruption des données.

---

## 3.2 Démarrage du serveur de configuration (Config Server)

### Terminal utilisé :  **ConfigServer**

### Commande exécutée

```js
mongod --configsvr --replSet replicaconfig --dbpath configsvrdb --port 27019
```

### Explication

Cette commande permet de démarrer le serveur de configuration du cluster shardé. L’option `--configsvr` indique que le serveur joue le rôle de config server. Celui-ci stocke les métadonnées du cluster, notamment les informations relatives à la répartition des données et à la localisation des chunks. Le serveur de configuration est configuré sous forme de replica set (`replicaconfig`), conformément aux exigences de MongoDB, même si un seul nœud est utilisé dans ce TP.

---

## 3.3 Initialisation du replica set du serveur de configuration

### Terminal utilisé : **InitReplicaSet**

### Commandes exécutées

```js
mongosh --port 27019
```

```js
rs.initiate()
```

### Explication

Après le démarrage du serveur de configuration, son replica set a été initialisé à l’aide de la commande `rs.initiate()`. Cette étape est indispensable, car MongoDB exige que les config servers soient déployés sous forme de replica set afin d’assurer la cohérence et la disponibilité des métadonnées du cluster shardé.

---

## 3.4 Démarrage des shards

### 3.4.1 Démarrage du premier shard

### Terminal utilisé : **Shard1**

### Commande exécutée

```js
mongod --replSet replicashard1 --dbpath serv1 --shardsvr --port 20004
```

### Explication

Cette commande démarre le premier shard du cluster. L’option `--shardsvr` indique que le serveur participe à un cluster shardé. Le shard est configuré comme un replica set nommé `replicashard1` et utilise son propre répertoire de stockage (`serv1`). Chaque shard stocke une partie des données de la base.

---

### 3.4.2 Démarrage du second shard

### Terminal utilisé : **Shard2**

### Commande exécutée

```js
mongod --replSet replicashard2 --dbpath serv2 --shardsvr --port 20005
```

### Explication 

Le second shard est démarré de manière similaire au premier. Il possède son propre replica set et son propre répertoire de données. La présence de plusieurs shards permet de répartir les données et la charge sur plusieurs serveurs.

---

## 3.5 Initialisation des replica sets des shards

### Terminal utilisé : **InitReplicaSet**

### Commandes exécutées

* Pour le shard 1 :

```js
mongosh --port 20004
```

```js
rs.initiate()
```

* Pour le shard 2 :

```js
mongosh --port 20005
```

```js
rs.initiate()
```

### Explication 

Les replica sets des deux shards ont été initialisés séparément. Cette configuration permet à chaque shard de fonctionner comme une unité tolérante aux pannes et constitue une exigence de MongoDB dans une architecture shardée.

---

## 3.6 Lancement du routeur MongoDB (mongos)

### Terminal utilisé : **Mongos**

### Commande exécutée

```js
mongos --configdb replicaconfig/localhost:27019
```

### Explication 

Le routeur MongoDB (`mongos`) a été lancé afin d’assurer la communication entre le client et les différents shards. Le mongos agit comme un point d’entrée unique du cluster shardé. Il consulte les serveurs de configuration pour déterminer l’emplacement des données et redirige automatiquement les requêtes vers les shards appropriés, de manière totalement transparente pour le client.

---

# 4. Ajout des shards au cluster

## 4.1 Connexion au routeur MongoDB

### Terminal utilisé : **Client**

### Commande exécutée

```js
mongosh
```

Cette connexion se fait **via le routeur mongos** (port par défaut 27017).

---

### Explication 

La connexion au cluster est effectuée via le routeur MongoDB (`mongos`). Le client ne communique jamais directement avec les shards. Le routeur agit comme un point d’entrée unique et masque la distribution des données au client.

---

## 4.2 Ajout des shards au cluster

### Commandes exécutées

```js
sh.addShard("replicashard1/localhost:20004")
```

```js
sh.addShard("replicashard2/localhost:20005")
```

---

### Explication 

Ces commandes permettent d’ajouter les deux shards au cluster MongoDB. Chaque shard est identifié par le nom de son replica set ainsi que par l’adresse et le port du serveur. Une fois ajoutés, les shards deviennent disponibles pour stocker les données de la base shardée.

---

# 5. Activation du sharding sur la base de données

## 5.1 Activation du sharding sur la base

### Commande exécutée

```js
sh.enableSharding("mabasefilms")
```

---

### Explication 

Par défaut, les bases de données MongoDB ne sont pas shardées. Cette commande active explicitement le sharding pour la base de données `mabasefilms`. À partir de ce moment, les collections de cette base peuvent être partitionnées entre les différents shards.

---

# 6. Activation du sharding sur la collection

## 6.1 Choix de la clé de sharding

Dans ce TP, la clé de sharding choisie est le champ **`titre`** des films. Ce champ est présent dans tous les documents et offre une diversité suffisante pour répartir les données entre les shards.

---

## 6.2 Sharding de la collection `films`

### Commande exécutée

```js
sh.shardCollection(
  "mabasefilms.films",
  { "titre": 1 }
)
```

---

### Explication 

Cette commande active le sharding sur la collection `films` de la base `mabasefilms`. La clé de sharding utilisée est le champ `titre`. MongoDB répartit alors les documents de la collection en fonction de cette clé, en les regroupant dans des blocs appelés *chunks*, qui sont ensuite distribués entre les shards.

---

# 7. Insertion des données et observation du sharding

## 7.1 Insertion des données

Une fois le sharding activé, un programme Python est exécuté afin d’insérer un grand nombre de documents dans la collection `films`. Le script génère aléatoirement des films et les insère un par un dans la base de données.

---

###  Explication 

L’insertion des données s’effectue via le routeur MongoDB (`mongos`). Celui-ci se charge de déterminer automatiquement le shard cible pour chaque document en fonction de la clé de sharding. Les documents sont ainsi répartis entre les différents shards sans intervention explicite du client.


## 7.2 Observation de la répartition des données

### Commandes utiles

```js
sh.status()
```

```js
db.films.getShardDistribution()
```

---

### Explication 

Ces commandes permettent d’observer l’état du cluster shardé ainsi que la répartition des données entre les shards. On peut notamment visualiser le nombre de chunks par shard et vérifier que MongoDB équilibre automatiquement la charge à l’aide du balancer.

---

# Réponses aux questions sur le sharding MongoDB


## 1. Qu’est-ce que le sharding dans MongoDB et pourquoi est-il utilisé ?

Le sharding est une technique de **partitionnement horizontal des données** qui consiste à répartir les documents d’une collection sur plusieurs serveurs appelés *shards*. Il est utilisé pour gérer de **grands volumes de données** et un **grand nombre de requêtes**, en améliorant la scalabilité et les performances. Le sharding permet ainsi de dépasser les limites matérielles d’un serveur unique.

---

## 2. Quelle est la différence entre le sharding et la réplication dans MongoDB ?

La réplication consiste à **dupliquer les mêmes données** sur plusieurs serveurs afin d’assurer la **haute disponibilité** et la tolérance aux pannes.
Le sharding, en revanche, consiste à **répartir des données différentes** sur plusieurs serveurs afin d’assurer la **montée en charge** et l’équilibrage de la charge.

---

## 3. Quels sont les composants d’une architecture shardée ?

Une architecture shardée MongoDB repose sur trois composants principaux :

* les **shards**, qui stockent les données,
* les **config servers**, qui stockent les métadonnées du cluster,
* le **routeur mongos**, qui redirige les requêtes des clients vers les shards appropriés.

---

## 4. Quelles sont les responsabilités des config servers (CSRS) ?

Les config servers stockent les **métadonnées du cluster shardé**, notamment :

* la clé de sharding,
* la répartition des chunks,
* l’état du cluster.
  Ils sont indispensables au fonctionnement du sharding, car sans eux, le cluster devient inutilisable.

---

## 5. Quel est le rôle du routeur mongos ?

Le mongos est le **point d’entrée unique** du cluster shardé. Il reçoit les requêtes des clients, consulte les config servers pour déterminer où se trouvent les données, puis redirige les requêtes vers les shards concernés de manière transparente.

---

## 6. Comment MongoDB décide-t-il sur quel shard stocker un document ?

MongoDB utilise la **clé de sharding** pour déterminer dans quel **chunk** se situe un document. Chaque chunk étant associé à un shard, MongoDB sait automatiquement sur quel shard stocker le document.

---

## 7. Qu’est-ce qu’une clé de sharding et pourquoi est-elle essentielle ?

La clé de sharding est un champ (ou un ensemble de champs) utilisé pour répartir les documents d’une collection entre les shards. Elle est essentielle car elle influence directement la **distribution des données**, les **performances** et l’équilibrage de la charge.

---

## 8. Quels sont les critères de choix d’une bonne clé de sharding ?

Une bonne clé de sharding doit :

* avoir une **forte cardinalité**,
* être **bien distribuée**,
* éviter les valeurs monotones,
* être fréquemment utilisée dans les requêtes.

---

## 9. Qu’est-ce qu’un chunk dans MongoDB ?

Un chunk est un **bloc de données** regroupant des documents partageant une plage de valeurs de la clé de sharding. MongoDB répartit les données sous forme de chunks entre les shards.

---

## 10. Comment fonctionne le splitting des chunks ?

Lorsqu’un chunk dépasse une taille prédéfinie, MongoDB le **divise automatiquement** en deux chunks plus petits. Ce mécanisme permet d’éviter qu’un chunk devienne trop volumineux.

---

## 11. Que fait le balancer dans un cluster shardé ?

Le balancer est un processus automatique qui **équilibre les chunks entre les shards** afin d’éviter qu’un shard soit surchargé par rapport aux autres.

---

## 12. Quand et comment le balancer déplace-t-il des chunks ?

Le balancer se déclenche lorsqu’un déséquilibre est détecté entre les shards. Il déplace alors certains chunks d’un shard surchargé vers un shard moins chargé, généralement en arrière-plan.

---

## 13. Qu’est-ce qu’un hot shard et comment l’éviter ?

Un hot shard est un shard qui reçoit une **charge disproportionnée** (écritures ou lectures). On peut l’éviter en choisissant une **clé de sharding bien distribuée** ou en utilisant une clé de sharding hashée.

---

## 14. Quels problèmes une clé de sharding monotone peut-elle engendrer ?

Une clé de sharding monotone (par exemple un identifiant croissant ou une date) peut entraîner :

* une concentration des écritures sur un seul shard,
* la création de hot shards,
* une mauvaise répartition des données.

---

## 15. Comment activer le sharding sur une base et une collection ?

Le sharding s’active avec :

* `sh.enableSharding("nomBase")` pour la base,
* `sh.shardCollection("nomBase.collection", { clé: 1 })` pour la collection.

---

## 16. Comment ajouter un nouveau shard à un cluster MongoDB ?

Un nouveau shard est ajouté à l’aide de la commande :

```js
sh.addShard("nomReplicaSet/hôte:port")
```

Une fois ajouté, MongoDB peut y déplacer des chunks automatiquement.

---

## 17. Comment vérifier l’état du cluster shardé ?

Les commandes les plus utilisées sont :

```js
sh.status()
```

et

```js
db.collection.getShardDistribution()
```

Elles permettent de visualiser l’état du cluster et la répartition des données.

---

## 18. Dans quels cas utiliser une clé de sharding hashée ?

Une clé hashée est utile lorsque les valeurs de la clé sont monotones. Le hachage permet de **répartir uniformément** les données entre les shards.

---

## 19. Dans quels cas privilégier une clé de sharding par plage (ranged) ?

Une clé par plage est préférable lorsque les requêtes portent fréquemment sur des **intervalles de valeurs** (par exemple des dates), afin de limiter le nombre de shards consultés.

---

## 20. Qu’est-ce que le zone sharding et quel est son intérêt ?

Le zone sharding permet d’associer certaines plages de données à des shards spécifiques. Il est utile pour des contraintes géographiques, réglementaires ou de performance.

---

## 21. Comment MongoDB gère-t-il les requêtes multi-shards ?

Pour une requête multi-shards, le mongos envoie la requête à tous les shards concernés, puis **agrège les résultats** avant de les renvoyer au client.

---

## 22. Comment optimiser les performances dans un environnement shardé ?

On peut optimiser les performances en :

* choisissant une bonne clé de sharding,
* utilisant des index adaptés,
* limitant les requêtes multi-shards,
* utilisant des insertions en masse.

---

## 23. Que se passe-t-il lorsqu’un shard devient indisponible ?

Si un shard est indisponible, son replica set peut élire un nouveau primaire. Si aucun nœud n’est disponible, les données stockées sur ce shard deviennent temporairement inaccessibles.

---

## 24. Comment migrer une collection existante vers un schéma shardé ?

Il faut :

* activer le sharding sur la base,
* choisir une clé de sharding,
* exécuter `sh.shardCollection()` sur la collection existante.

---

## 25. Quels outils ou métriques utiliser pour diagnostiquer les problèmes de sharding ?

On peut utiliser :

* `sh.status()`,
* `db.serverStatus()`,
* les logs MongoDB,
* les outils de monitoring (MongoDB Atlas, Prometheus).

---


# Conclusion


Ce TP a permis de mettre en œuvre un cluster MongoDB shardé et de comprendre le fonctionnement du partitionnement des données. La mise en place des config servers, des shards et du routeur mongos illustre l’architecture distribuée de MongoDB. L’activation du sharding et l’insertion d’un grand volume de données ont permis d’observer la création et la migration des chunks, ainsi que l’équilibrage automatique de la charge entre les shards. Le sharding apparaît ainsi comme une solution efficace pour gérer des bases de données de grande taille tout en assurant performance et scalabilité.

---
