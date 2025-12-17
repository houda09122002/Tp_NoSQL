# Introduction

Dans le contexte des bases de données NoSQL, le traitement de grands volumes de données nécessite des mécanismes adaptés permettant d’effectuer des calculs distribués et efficaces. MongoDB propose pour cela le paradigme **MapReduce**, inspiré des travaux de Google, qui permet de réaliser des traitements d’agrégation complexes directement au niveau de la base de données.

L’objectif de ce TP est de se familiariser avec l’écriture et l’utilisation de fonctions **MapReduce dans MongoDB**, à partir d’une collection de films déjà utilisée lors du TP précédent. À travers une série de requêtes, nous mettons en œuvre différents traitements statistiques (comptages, moyennes, classements) afin de mieux comprendre le fonctionnement et les possibilités offertes par MapReduce.

---

## 1. Présentation de MapReduce dans MongoDB

Le modèle **MapReduce** repose sur deux fonctions principales :

* **Map** : parcourt chaque document de la collection et émet des paires *(clé, valeur)* à l’aide de la fonction `emit(key, value)`.
* **Reduce** : agrège toutes les valeurs associées à une même clé afin de produire un résultat global (somme, moyenne, maximum, etc.).

Dans MongoDB, l’exécution MapReduce se fait via la méthode :

```js
db.collection.mapReduce(mapFunction, reduceFunction, options)
```
### Explication des options importantes

* `out` : indique où MongoDB écrit le résultat :

  * `out: { replace: "nom" }` : remplace (ou crée) la collection résultat.
  * `out: { inline: 1 }` : renvoie le résultat directement (petits résultats).
* `query` : filtre les documents traités (ex : seulement un genre).
* `finalize` : applique une fonction après reduce.
* Résultat stocké sous forme :

  ```json
  { "_id": <clé>, "value": <résultat> }
  ```


Le résultat peut être stocké dans une collection temporaire ou permanente. Ce mécanisme est particulièrement utile lorsque les agrégations sont trop complexes pour les requêtes classiques.

---


Avant de lancer MapReduce :

```js
use mabasefilms
show collections
db.films.countDocuments()
```

## **Explication :**

* `use mabasefilms` : sélectionne la base.
* `show collections` : liste les collections.
* `countDocuments()` : vérifie que `films` contient des documents (utile avant les traitements).

---

# 1) Compter le nombre total de films

## Commande

```js
var map1 = function () { emit("total", 1); };

var reduce1 = function (key, values) { return Array.sum(values); };

db.films.mapReduce(map1, reduce1, { out: { replace: "q1_total_films" } });
db.q1_total_films.find();
```

## Explication 
* `emit("total", 1)` : chaque film produit la paire (“total”, 1).
* `Array.sum(values)` : additionne toutes les valeurs 1 → nombre total.
* `out: { replace: ... }` : stocke le résultat dans `q1_total_films`.
* `db.q1_total_films.find()` : affiche le résultat.

---

# 2) Compter le nombre de films par genre

```js
var map2 = function () {
  if (this.genre) emit(this.genre, 1);
};

var reduce2 = function (key, values) {
  return Array.sum(values);
};

db.films.mapReduce(map2, reduce2, { out: { replace: "q2_films_par_genre" } });
db.q2_films_par_genre.find().sort({ value: -1 });
```

## **Explication :**

* La clé = `genre`. Tous les films du même genre sont regroupés.
* `sort({value:-1})` : affiche les genres les plus fréquents en premier.

---

# 3) Films par réalisateur

```js
var map3 = function () {
  if (this.director) emit(this.director, 1);
};

var reduce3 = function (key, values) {
  return Array.sum(values);
};

db.films.mapReduce(map3, reduce3, { out: { replace: "q3_films_par_realisateur" } });
db.q3_films_par_realisateur.find().sort({ value: -1 });
```

---

# 4) Nombre d’acteurs uniques (tous films confondus)

```js
var map4 = function () {
  if (this.actors) this.actors.forEach(a => emit(a, 1));
};

var reduce4 = function (key, values) {
  return 1; // présence
};

db.films.mapReduce(map4, reduce4, { out: { replace: "q4_acteurs_uniques" } });
db.q4_acteurs_uniques.countDocuments();
```

## **Explication :**

* On émet une clé par acteur.
* Reduce renvoie 1 (peu importe le nombre d’occurrences).
* Le nombre d’acteurs uniques = nombre de documents dans la collection résultat.

---

# 5) Films par année de sortie

```js
var map5 = function () { if (this.year) emit(this.year, 1); };
var reduce5 = function (k, vals) { return Array.sum(vals); };

db.films.mapReduce(map5, reduce5, { out: { replace: "q5_films_par_annee" } });
db.q5_films_par_annee.find().sort({ _id: 1 });
```

---

# 6) Note moyenne par film (à partir de grades)

```js
var map6 = function () {
  if (!this.grades || this.grades.length === 0) return;
  emit(this.title, { sum: Array.sum(this.grades), count: this.grades.length });
};

var reduce6 = function (key, values) {
  var r = { sum: 0, count: 0 };
  values.forEach(v => { r.sum += v.sum; r.count += v.count; });
  return r;
};

var finalize6 = function (key, reduced) {
  return reduced.count === 0 ? null : (reduced.sum / reduced.count);
};

db.films.mapReduce(map6, reduce6, {
  out: { replace: "q6_moyenne_par_film" },
  finalize: finalize6
});
db.q6_moyenne_par_film.find().sort({ value: -1 }).limit(10);
```

## **Explication :**

* Map émet un objet `{sum,count}` pour pouvoir calculer une moyenne propre.
* Reduce additionne les sommes et les compteurs.
* Finalize transforme `{sum,count}` en moyenne.

---

# 7) Note moyenne par genre


```js
var map7 = function () {
  if (!this.grades || this.grades.length === 0 || !this.genre) return;
  emit(this.genre, { sum: Array.sum(this.grades), count: this.grades.length });
};

var reduce7 = reduce6;
var finalize7 = finalize6;

db.films.mapReduce(map7, reduce7, {
  out: { replace: "q7_moyenne_par_genre" },
  finalize: finalize7
});
db.q7_moyenne_par_genre.find().sort({ value: -1 });
```

---

# 8) Note moyenne par réalisateur

```js
var map8 = function () {
  if (!this.grades || this.grades.length === 0 || !this.director) return;
  emit(this.director, { sum: Array.sum(this.grades), count: this.grades.length });
};

db.films.mapReduce(map8, reduce6, {
  out: { replace: "q8_moyenne_par_realisateur" },
  finalize: finalize6
});
db.q8_moyenne_par_realisateur.find().sort({ value: -1 }).limit(10);
```

---

# 9) Film avec la note maximale la plus élevée (globale)

## Étape 1 : maximum par film

```js
var map9 = function () {
  if (!this.grades || this.grades.length === 0) return;
  emit(this.title, Math.max.apply(null, this.grades));
};

var reduce9 = function (key, values) {
  return Math.max.apply(null, values);
};

db.films.mapReduce(map9, reduce9, { out: { replace: "q9_max_par_film" } });
```

## Étape 2 : récupérer le meilleur film

```js
db.q9_max_par_film.find().sort({ value: -1 }).limit(1);
```

## **Explication :**

* Map/Reduce calcule le max par film.
* Puis un tri décroissant permet de prendre le max global.

---

# 10) Nombre de notes > 70 (tous films confondus)

```js
var map10 = function () {
  if (!this.grades) return;
  this.grades.forEach(g => { if (g > 70) emit("sup70", 1); });
};

var reduce10 = function (k, vals) { return Array.sum(vals); };

db.films.mapReduce(map10, reduce10, { out: { replace: "q10_notes_sup70" } });
db.q10_notes_sup70.find();
```

---

# 11) Tous les acteurs par genre, sans doublons

On crée une clé composite `genre|acteur` (ça supprime les doublons) :

```js
var map11 = function () {
  if (!this.genre || !this.actors) return;
  this.actors.forEach(a => emit(this.genre + "|" + a, 1));
};

var reduce11 = function () { return 1; };

db.films.mapReduce(map11, reduce11, { out: { replace: "q11_genre_acteur_unique" } });
```

Pour obtenir “liste d’acteurs par genre”, on peut ensuite regrouper côté client, ou faire un 2e MapReduce (bonus). Dans un rapport, tu peux dire :

> Le résultat `q11_genre_acteur_unique` contient des couples uniques (genre, acteur). Une extraction permet ensuite de reconstruire la liste des acteurs par genre sans doublons.

---

# 12) Acteurs apparaissant dans le plus grand nombre de films

```js
var map12 = function () {
  if (!this.actors) return;
  // éviter doublon si un acteur est listé 2 fois dans le même film :
  var seen = {};
  this.actors.forEach(a => { if (!seen[a]) { emit(a, 1); seen[a] = true; }});
};

var reduce12 = function (k, vals) { return Array.sum(vals); };

db.films.mapReduce(map12, reduce12, { out: { replace: "q12_acteurs_nb_films" } });
db.q12_acteurs_nb_films.find().sort({ value: -1 }).limit(10);
```

---

# 13) Classer les films par lettre (A/B/C/…)

On prend la moyenne du film, puis on convertit en lettre :

```js
var map13 = function () {
  if (!this.grades || this.grades.length === 0) return;
  var avg = Array.sum(this.grades) / this.grades.length;
  var letter = (avg >= 90) ? "A" : (avg >= 80) ? "B" : (avg >= 70) ? "C" : (avg >= 60) ? "D" : "E";
  emit(letter, 1);
};

var reduce13 = function (k, vals) { return Array.sum(vals); };

db.films.mapReduce(map13, reduce13, { out: { replace: "q13_films_par_lettre" } });
db.q13_films_par_lettre.find().sort({ _id: 1 });
```

---

# 14) Note moyenne par année

```js
var map14 = function () {
  if (!this.year || !this.grades || this.grades.length === 0) return;
  emit(this.year, { sum: Array.sum(this.grades), count: this.grades.length });
};

db.films.mapReduce(map14, reduce6, {
  out: { replace: "q14_moyenne_par_annee" },
  finalize: finalize6
});
db.q14_moyenne_par_annee.find().sort({ _id: 1 });
```

---

# 15) Réalisateurs dont la moyenne globale > 80

```js
var map15 = function () {
  if (!this.director || !this.grades || this.grades.length === 0) return;
  emit(this.director, { sum: Array.sum(this.grades), count: this.grades.length });
};

db.films.mapReduce(map15, reduce6, {
  out: { replace: "q15_moyenne_realisateur" },
  finalize: finalize6
});
```

Filtrer ceux > 80 :

```js
db.q15_moyenne_realisateur.find({ value: { $gt: 80 } }).sort({ value: -1 });
```

## **Explication :**

* Map/Reduce calcule la moyenne par réalisateur.
* La requête `find({value:{$gt:80}})` extrait ceux qui dépassent 80.

---

# Conclusion

Ce TP a permis de mettre en œuvre des traitements MapReduce dans MongoDB sur une collection de films. Les différentes questions ont illustré des cas classiques d’analyse : comptages, agrégations par attribut (genre, année, réalisateur), déduplication (acteurs uniques) et calculs statistiques (moyennes, maximum). MapReduce met en évidence la logique “clé/valeur + agrégation”, bien que MongoDB recommande souvent aujourd’hui le framework d’agrégation pour des raisons de performance. L’exercice reste néanmoins très formateur pour comprendre les mécanismes fondamentaux de traitement de données dans un SGBD NoSQL.

---


