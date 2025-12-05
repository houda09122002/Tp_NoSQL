# Introduction

La réplication est un mécanisme essentiel dans les bases NoSQL modernes, en particulier dans MongoDB et Cassandra, qui cherchent avant tout à offrir disponibilité, résilience et tolérance aux pannes. Dans ce TP, nous avons étudié ces principes à travers la mise en place d’un Replica Set MongoDB, l’analyse de son fonctionnement interne, ainsi que l’observation de comportements en conditions de panne.
Le but était non seulement de reproduire les manipulations vues dans les vidéos, mais aussi de comprendre en profondeur ce qu’est un Replica Set, comment il fonctionne, comment il réagit lors d’un basculement, et quelles garanties de cohérence il offre.
# Comprendre la réplication : principes et architecture
MongoDB repose sur une architecture de Replica Set, un ensemble de nœuds contenant chacun une copie des mêmes données.
Dans une telle architecture, un nœud prend naturellement le rôle de Primary, c’est-à-dire la seule instance autorisée à recevoir les opérations d’écriture. Toutes les autres machines sont des Secondaries, dont le rôle principal est de répliquer en continu les opérations du Primary à partir de l’oplog, garantissant ainsi une copie presque temps réel des données.

Par défaut, MongoDB interdit les écritures sur les Secondaries afin d’éviter toute divergence. Cela permet de maintenir une cohérence forte pour les lectures effectuées sur le Primary.
Il est toutefois possible de lire sur les Secondaries lorsque l’on souhaite décharger le Primary, au prix d’un risque : la donnée peut être légèrement obsolète si le Secondary a un léger retard de réplication.

Certaines configurations vont même plus loin, en ajoutant des Arbitres, des nœuds très légers qui ne stockent aucune donnée mais participent aux votes lors des élections. Cela permet de conserver une majorité même avec un nombre réduit de serveurs, tout en évitant de consommer trop de ressources.

## Mise en place du Replica Set
Nous avons commencé par lancer trois instances MongoDB distinctes sur trois ports différents, chacune avec un répertoire de données séparé. La commande suivante, exécutée sur chaque instance, indique à MongoDB que le serveur appartient à un Replica Set nommé monreplicaset:

```js
mongod --replSet monreplicaset --port 27018 --dbpath data1
```

Une fois les instances lancées, nous nous connectons au premier nœud pour initialiser le Replica Set :

```js
rs.initiate()
```

Dès ce moment, MongoDB élit automatiquement le premier nœud en tant que Primary.
Les deux autres nœuds sont ensuite ajoutés :
```js
rs.add("localhost:27018")
rs.add("localhost:27019")
```
La commande : 
```js
rs.config()
```
est une commande MongoDB qui permet de modifier la configuration d’un Replica Set sans le recréer.

La commande :
```js
rs.isMaster()
```
est une commande MongoDB qui permet de savoir quel est le rôle du nœud sur lequel on est connecté.

À l’aide de la commande :
```js
rs.status()
```
nous pouvons visualiser l’état du Replica Set, vérifier quel nœud est Primary ou Secondary, et constater la synchronisation continue entre les membres.

La commande : 
```js
rs.addArb("localhost:27021")
```
rs.addArb() est une commande MongoDB qui permet d’ajouter un arbitre (Arbiter) dans un Replica Set.

Un Arbiter :

* ne stocke aucune donnée

* ne participe pas à la réplication

* sert uniquement à voter lors des élections

* évite les égalités (utile pour garder une majorité)

La commande : 
```js
db.createCollection("personnes")
```
sert à créer une nouvelle collection vide dans une base MongoDB.
Les cammandes : 
```js
db.personnes.insert({ "nom": "Youcef" })
db.personnes.insert({ "nom": "Godart" })
db.personnes.insert({ "nom": "Charoy" })
```
Chaque commande :

* insère un document dans la collection personnes

* chaque document contient un champ "nom"

La commande : 
```js
db.personnes.find()
```
permet d’afficher tous les documents qui se trouvent dans la collection personnes.

La commande :
```js
rs.slaveOk()
```

est une commande qui permet d’autoriser la lecture sur un nœud Secondary dans un Replica Set MongoDB.
Par défaut :

* un Secondary ne peut pas être lu → pour éviter des lectures de données obsolètes

* avec rs.slaveOk(), on active la lecture sur les Secondaries

La commande :
```js
rs.secondaryOk()
```
est une commande équivalente à rs.slaveOk() dans les versions récentes de MongoDB.
Elle permet d’autoriser la lecture depuis un nœud Secondary dans un Replica Set.

**slaveDelay** est un paramètre de configuration d’un nœud Secondary dans un Replica Set MongoDB.
Il permet d’imposer volontairement un retard de réplication, exprimé en secondes.

Exemple :
Si on met _slaveDelay: 120_, cela signifie que ce Secondary aura toujours 120 secondes de retard par rapport au Primary.
# Réplication et tolérance aux pannes avec MongoDB

## Partie 1 — Compréhension de base
### 1. Qu’est-ce qu’un Replica Set dans MongoDB ?
Un groupe de serveurs MongoDB contenant une copie des mêmes données. Il assure la haute disponibilité.
### 2. Quel est le rôle du Primary dans un Replica Set ?
Gère toutes les écritures et sert de source de réplication pour les Secondary.
### 3. Quel est le rôle essentiel des Secondaries ?
Répliquent les données du Primary et peuvent servir des lectures (selon la configuration).
### 4. Pourquoi MongoDB n’autorise-t-il pas les écritures sur un Secondary ?
Pour éviter la divergence des données et assurer une cohérence stricte.
### 5. Qu’est-ce que la cohérence forte dans le contexte MongoDB ?
Lecture garantie sur des données à jour (lecture depuis le Primary).
### 6. Quelle est la différence entre readPreference : "primary" et "secondary" ?
**primary** : lecture toujours la plus récente

**secondary** : lecture plus rapide mais potentiellement en retard
### 7. Dans quel cas pourrait-on souhaiter lire sur un Secondary malgré les risques ?
Pour décharger le Primary en lecture (analytics, reporting).

## Partie 2 — Commandes & configuration

### 8. Quelle commande permet d’initialiser un Replica Set ?
```js
rs.initiate()
```
### 9. Comment ajouter un nœud à un Replica Set après son initialisation ?
```js
rs.add("host:port")
```
### 10. Quelle commande permet d’afficher l’état actuel du Replica Set ?
```js
rs.status()
```
### 11. Comment identifier le rôle actuel (Primary / Secondary / Arbitre) d’un nœud ?
```js
rs.isMaster()
```
### 12. Quelle commande permet de forcer le basculement du Primary ?
```js
rs.stepDown()
```
### 13. Comment peut-on désigner un nœud comme Arbitre ? Pourquoi le faire ?
```js
rs.addArb("host:port")
```
### 14. Donnez la commande pour configurer un nœud secondaire avec un délai de réplication ( slaveDelay ).
```js
rs.reconfig({
  members: [
    { host: "localhost:27018", slaveDelay: 120 }
  ]
})
```

## Partie 3 — Résilience et tolérance aux pannes
### 15. Que se passe-t-il si le Primary tombe en panne et qu’il n’y a pas de majorité ?
Le cluster devient read-only.
### 16. Comment MongoDB choisit-il un nouveau Primary ? Quels critères utilise-t-il ?
* priorité
* état de santé
* retard de réplication faible
### 17. Qu’est-ce qu’une élection dans MongoDB ?
Processus automatique permettant de choisir le Primary.
### 18. Que signifie auto-dégradation du Replica Set ? Dans quel cas cela survient-il ?
Un nœud se met en Secondary si conditions réseau → évite deux Primary.
### 19. Pourquoi est-il conseillé d’avoir un nombre impair de nœuds dans un Replica Set ?
Pour faciliter l’obtention d’une majorité.
### 20. Quelles conséquences a une partition réseau sur le fonctionnement du cluster ?
Possibilité de clusters isolés → seulement la partition majoritaire peut avoir un Primary.
## Partie 4 — Scénarios pratiques
### 21. Vous avez 3 nœuds : 27017 (Primary) , 27018 (Secondary) , et 27019 (Arbitre) . Que se passe-t-il si le Primary devient injoignable ?
Le Secondary devient Primary car majorité = 2.
### 22. Vous avez configuré un Secondary avec un slaveDelay de 120 secondes. Quelle est son utilité ? Quels usages peut-on en faire dans la vraie vie ?
Avoir un nœud “retardé” pour protéger contre les suppressions accidentelles.
### 23. Un client exige une lecture toujours à jour, même en cas de bascule. Quelles options de readConcern et writeConcern recommanderiez-vous ?
readConcern: "majority"
writeConcern: { w: "majority" }
### 24. Dans une application critique, vous voulez garantir que l’écriture est confirmée par au moins deux nœuds. Quelle option de writeConcern devez-vous utiliser ?
{ writeConcern: { w: 2 } }
### 25. Un étudiant a lu depuis un Secondary et récupéré une donnée obsolète. Expliquez pourquoi et comment éviter cela.
Cause : retard de réplication.
Solution : readPreference: "primary"
### 26. Montrez la commande pour vérifier quel nœud est actuellement Primary dans votre ReplicaSet.
```js
rs.isMaster()
```
### 27. Expliquez comment forcer une bascule manuelle du Primary sans interruption majeure.
```js
rs.stepDown(60)
```
### 28. Décrivez la procédure pour ajouter un nouveau nœud secondaire dans un Replica Set en fonctionnement.
```js
rs.add("host:port")
```
### 29. Quelle commande permet de retirer un nœud défectueux d’un Replica Set ?
```js
rs.remove("host:port")
```
### 30. Comment configurer un nœud secondaire pour qu’il soit caché (non visible aux clients) ? Pourquoi ferait-on cela ?
```js
rs.reconfig({ members: [{ hidden: true }] })
```
### 31. Montrez comment modifier la priorité d’un nœud afin qu’il devienne le Primary préféré.
```js
rs.reconfig({
  members: [{ priority: 2 }]
})
```
### 32. Expliquez comment vérifier le délai de réplication d’un Secondary par rapport au Primary.
```js
rs.printSecondaryReplicationInfo()
```
### 33. Que fait la commande rs.freeze() et dans quel scénario est-elle utile ?
Empêche un nœud de devenir Primary temporairement.
### 34. Comment redémarrer un Replica Set sans perdre la configuration ?
La configuration est stockée dans la collection local.system.replset.
### 35. Expliquez comment surveiller en temps réel la réplication via les logs MongoDB ou commandes shell.
mongod --verbose
lecture des logs
rs.printReplicationInfo()
## Questions complémentaires
### 37. Qu’est-ce qu’un Arbitre (Arbiter) et pourquoi ne stocke-t-il pas de données ?
Vote uniquement, ne stocke aucune donnée
### 38. Comment vérifier la latence de réplication entre le Primary et les Secondaries ?
```js
rs.printSecondaryReplicationInfo()
```
### 39. Quelle commande MongoDB permet d’afficher le retard de réplication des membres secondaires ?
```js
rs.printReplicationInfo()
```
### 40. Quelle est la différence entre la réplication asynchrone et synchrone ? Quel type utilise MongoDB ?
MongoDB utilise une réplication asynchrone.
### 41. Peut-on modifier la configuration d’un Replica Set sans redémarrer les serveurs ?
Oui → via rs.reconfig().
### 42. Que se passe-t-il si un nœud Secondary est en retard de plusieurs minutes ?
Il rattrape automatiquement ; peut entraîner une lecture obsolète.
### 43. Comment MongoDB gère-t-il les conflits de données lors de la réplication ?
MongoDB choisit la branche la plus récente.
### 44. Est-il possible d’avoir plusieurs Primarys simultanément dans un Replica Set ? Pourquoi ?
Impossible : élection + quorum empêchent ce cas.
### 45. Pourquoi est-il déconseillé d’utiliser un Secondary pour des opérations d’écriture même en lecture préférée secondaire ?
Cause une divergence irrecoverable des données
### 46. Quelles sont les conséquences d’un réseau instable sur un Replica Set ?
Élections fréquentes, pertes de Primary, cluster read-only.

## Commandes essentielles manipulées dans le TP

Les commandes suivantes ont été particulièrement utilisées :

Initialisation : _rs.initiate()_

Ajout de nœuds : _rs.add()_

Retrait d’un nœud : _rs.remove()_

Vérification de l’état : _rs.status()_

Forcer une bascule : _rs.stepDown()_

Diagnostic du retard de réplication :
_rs.printReplicationInfo()_
_rs.printSecondaryReplicationInfo()_

Lecture du rôle actuel :
_rs.isMaster()_

# Conclusion 
Ce TP nous a permis d’acquérir une compréhension complète et pratique du fonctionnement des Replica Sets MongoDB et de leur rôle fondamental dans la tolérance aux pannes. À travers la mise en place d’un cluster, l’observation des élections, la simulation de la panne du Primary et l’utilisation de paramètres avancés (priorité, lecture secondaire, délais, arbitres…), nous avons pu constater que MongoDB met en œuvre des mécanismes robustes pour maintenir la disponibilité des données même dans des environnements instables.