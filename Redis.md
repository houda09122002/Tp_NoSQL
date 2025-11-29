Rémi GOMEZ 12102211\
Houda KHIATI 12306375

## Introduction au NoSQL

Le **NoSQL** (“Not Only SQL”) désigne une famille de systèmes de gestion de bases de données qui ne suivent pas le modèle relationnel classique basé sur les **tables, lignes et colonnes**.

Ces systèmes ont été conçus pour répondre à des besoins de :

- **Scalabilité**
- **Haute performance**
- **Flexibilité des données**

Ils sont particulièrement adaptés aux grandes applications **web**, **mobiles**, **IoT**, et plus généralement aux environnements générant de très grands volumes de données.

Les bases SQL traditionnelles deviennent parfois limitées lorsque :

- les volumes de données augmentent fortement ;
- les données sont peu ou mal structurées ;
- le système doit supporter **des millions d’utilisateurs simultanés**.

Le NoSQL apporte alors plusieurs avantages :

- **Structure flexible** (pas de schéma fixe)
- **Gestion efficace des gros volumes de données**
- **Scalabilité horizontale** (on ajoute des serveurs plutôt que d’améliorer un seul)
- **Très hautes performances** en lecture/écriture

# Partie 1

## Qu'est-ce que Redis ?

**Redis** (*REmote DIctionary Server*) est une base de données **NoSQL en mémoire**, rapide et **open source**.  
Utilisée principalement comme :

- cache,
- courtier de messages,
- base de données **clé-valeur**,

elle est réputée pour ses **performances élevées** et sa **simplicité d'utilisation**.

---

### Lancer le serveur Redis

Pour démarrer le serveur Redis, exécuter simplement :

```bash
redis-server
```
---
### 1. Commande SET (Redis)
```bash
127.0.0.1:6379> SET demo "Bonjour"
```
 
J’utilise la commande SET dans Redis pour créer une nouvelle paire clé/valeur.
La commande SET demo "Bonjour" enregistre la clé demo avec la valeur "Bonjour".
Le serveur renvoie OK, indiquant que l’opération s’est bien déroulée.
Cette manipulation illustre le fonctionnement de base d'une base NoSQL de type clé/valeur, où chaque donnée est accessible uniquement via une clé unique.

---
### 2. Commande SET avec clé structurée

```bash
127.0.0.1:6379> SET user:1234 "Samir"
```
  
La clé `user:1234` suit la convention Redis “prefix:id”.  
La commande enregistre la valeur `"Samir"` sous cette clé.  
`OK` confirme que l’écriture a réussi.

---
### 3. Commande GET

```bash
127.0.0.1:6379> GET user:1234
``` 
La commande `GET user:1234` récupère la valeur associée à la clé `user:1234`.  
Redis renvoie `"Samir"`, ce qui confirme que la donnée a bien été stockée.

---
### 4. Commande DEL

```bash
127.0.0.1:6379> DEL user:1234
(integer) 1
127.0.0.1:6379> DEL user:1234
(integer) 0
``` 
- `DEL user:1234` supprime la clé `user:1234`.  
- Le retour **(integer) 1** signifie que la clé existait et a été supprimée.  
- Le deuxième appel renvoie **(integer) 0**, indiquant que la clé n’existe plus.
---
### 5. Commande SET

```bash
127.0.0.1:6379> SET 27nov 0
```
 
La commande crée la clé `27nov` et lui associe la valeur `0`.  
Le message `OK` confirme que l’écriture a réussi.

---
###  6. Incrémentation d’une clé

```bash
127.0.0.1:6379> INCR 27nov
(integer) 1
127.0.0.1:6379> INCR 27nov
(integer) 2
127.0.0.1:6379> INCR 27nov
(integer) 3
127.0.0.1:6379> INCR 27nov
(integer) 4
````

`INCR 27nov` augmente la valeur associée à la clé `27nov` de **+1** à chaque appel.  
Redis renvoie la nouvelle valeur après incrémentation.

---
### 7. Décrémentation d’une clé

```bash
127.0.0.1:6379> DECR 27nov
(integer) 3
127.0.0.1:6379> DECR 27nov
(integer) 2
127.0.0.1:6379> DECR 27nov
(integer) 1
```
`DECR 27nov` diminue la valeur de la clé `27nov` de **–1** à chaque appel.  
Redis affiche la valeur mise à jour après décrémentation.


---
### 8. Commandes SET et TTL

```bash
127.0.0.1:6379> SET macle mavaleur
OK
127.0.0.1:6379> TTL macle
(integer) -1
```
 
- `SET macle mavaleur` crée la clé `macle` avec la valeur `mavaleur`.  
- `TTL macle` renvoie **-1**, ce qui signifie que la clé **n’a pas de durée d’expiration** (elle est permanente).
---
### 9.Vérifier le TTL d’une clé

```bash
127.0.0.1:6379> SET macle mavaleur
OK
127.0.0.1:6379> TTL macle
(integer) -1
```
`TTL macle` renvoie **-1**, ce qui signifie que la clé *n’a pas de durée d’expiration* (elle est permanente).

---
### 10. Définir un TTL (expiration) sur une clé

```bash
127.0.0.1:6379> EXPIRE macle 120
(integer) 1
127.0.0.1:6379> TTL macle
(integer) 100
```
- `EXPIRE macle 120` définit une durée de vie de **120 secondes** pour la clé `macle`.  
- Le retour **1** signifie que l’opération a réussi.  
- `TTL macle` montre le temps restant (ici 100 s), qui diminue chaque seconde.
---
### 11. Suppression avec DEL
```bash
127.0.0.1:6379> DEL macle
(integer) 1
```  
La clé `macle` est supprimée.  
`(integer) 1` = la clé existait et a été effacée.

---

### 12. Liste avec RPUSH et erreur de type
```bash
127.0.0.1:6379> RPUSH mesCours "BDA"
(integer) 1
127.0.0.1:6379> RPUSH mesCours "Services Web"
(integer) 2
127.0.0.1:6379> GET mesCours
(error) WRONGTYPE Operation against a key holding the wrong kind of value
```
- `RPUSH mesCours "..."` crée une **liste** et y ajoute des éléments.  
- `GET mesCours` échoue car `GET` fonctionne uniquement sur les **strings**.  
- Redis renvoie l’erreur **WRONGTYPE** : la clé `mesCours` est une *liste*, pas une chaîne.  
Pour lire la liste, il faut utiliser :

```bash
LRANGE mesCours 0 -1
```
---
### 13. Lecture d’une liste avec LRANGE
```bash
127.0.0.1:6379> LRANGE mesCours 0 0
"BDA"
127.0.0.1:6379> LRANGE mesCours 1 1
"Services Web"
``` 
- `LRANGE mesCours 0 0` retourne le **premier élément** de la liste.  
- `LRANGE mesCours 1 1` retourne le **deuxième élément**.  
Redis renvoie les éléments sous forme de liste ordonnée.
---

### 14. Suppression du premier élément avec LPOP
```bash
127.0.0.1:6379> LPOP mesCours
"BDA"
127.0.0.1:6379> LRANGE mesCours 0 -1
"Services Web"
``` 
- `LPOP mesCours` retire et renvoie le **premier élément** de la liste (`"BDA"`).  
- `LRANGE mesCours 0 -1` affiche les éléments restants de la liste (ici : `"Services Web"`).
---
### 15. Ensemble (SET) avec SADD et SMEMBERS
```bash
127.0.0.1:6379> SADD utilisateur "louis"
(integer) 1
127.0.0.1:6379> SADD utilisateur "jeanne"
(integer) 1
127.0.0.1:6379> SADD utilisateur "jeanne"
(integer) 0
127.0.0.1:6379> SMEMBERS utilisateur

    "louis"

    "jeanne"
```
- `SADD` ajoute un élément dans un **set** (ensemble sans doublons).  
- Le retour `1` = élément ajouté.  
- Le retour `0` = élément déjà présent.  
- `SMEMBERS utilisateur` affiche tous les membres du set.
---

### 16. Union de deux sets avec SUNION
```bash
127.0.0.1:6379> SUNION utilisateur sutresutilisateur

    "fred"

    "jeanne"

    "louis"
``` 
`SUNION` renvoie l’union des deux ensembles : tous les éléments présents dans **au moins un** des sets, sans doublons.
---

### 17. Sorted Set : ZADD, ZRANGE et ZRANK
```bash
127.0.0.1:6379> ZADD score 11 "jules"
(integer) 1
127.0.0.1:6379> ZADD score 2 "rami"
(integer) 1
127.0.0.1:6379> ZRANGE score 0 -1

    "rami"

    "jules"
    127.0.0.1:6379> ZRANK score "rami"
    (integer) 0

``` 
- `ZADD` ajoute un élément dans un **sorted set** avec un *score* numérique.  
- Redis trie automatiquement par score croissant.  
- `ZRANGE score 0 -1` affiche tous les éléments dans l’ordre : `"rami"` (score 2) puis `"jules"` (score 11).  
- `ZRANK` donne le rang d’un élément dans l’ordre trié (ici, `rami` est en rang **0**).
---

### 18. Hash : HSET et HGETALL
```bash
127.0.0.1:6379> HSET user:1 nom "jules"
(integer) 1
127.0.0.1:6379> HSET user:1 age "21"
(integer) 1
127.0.0.1:6379> HGETALL user:1

    "nom"

    "jules"

    "age"

    "21"
``` 
- `HSET` ajoute ou met à jour un **champ** dans un hash (clé → sous‑clés).  
- Ici, `user:1` contient deux champs : `nom` et `age`.  
- `HGETALL user:1` affiche **tous les champs et leurs valeurs** du hash.


---
### 19. Pub/Sub : SUBSCRIBE et PUBLISH
```bash
127.0.0.1:6379> SUBSCRIBE mesCours

    "subscribe"

    "mesCours"

    (integer) 1
```
Dans un autre terminal :
```bash
127.0.0.1:6379> PUBLISH mesCours "Un nouveau cours sur NoSQL"
(integer) 1
```
 
- `SUBSCRIBE mesCours` met le client en **mode écoute** sur le canal `mesCours`.  
- Dans un autre terminal, `PUBLISH mesCours "Un nouveau cours sur NoSQL"` envoie un message sur ce canal.  
- Le message sera reçu immédiatement par le client abonné.


---
### 20. Commande KEYS *
```bash
127.0.0.1:6379> KEYS *
    "macle"
    "user:1"
    "mesCours"
    ...
```

`KEYS *` affiche **toutes les clés** enregistrées dans Redis.  
Le `*` est un **motif global** (wildcard) qui signifie « n’importe quelle clé ».

---

### 21. Vider une base de données : FLUSHDB

La commande `FLUSHDB` supprime **toutes les clés** de la base de données actuellement sélectionnée.
```bash
127.0.0.1:6379> SET test "données"
OK
127.0.0.1:6379> KEYS *

    "test"
    127.0.0.1:6379> FLUSHDB
    OK
    127.0.0.1:6379> KEYS *
    (empty array)

```

`FLUSHDB` nettoie uniquement **la base active** (par défaut : base 0).

---

### 22. Vider toutes les bases : FLUSHALL

La commande `FLUSHALL` supprime **toutes les clés dans toutes les bases Redis**.

```bash
127.0.0.1:6379> SET test "données"
OK
127.0.0.1:6379> SELECT 1
OK
127.0.0.1:6379[1]> SET exemple "données2"
OK
127.0.0.1:6379[1]> FLUSHALL
OK
``` 
Plus aucune clé dans **aucune** des bases (0, 1, 2, ...).