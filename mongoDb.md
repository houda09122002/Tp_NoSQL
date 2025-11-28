Rémi GOMEZ 12102211\
Houda KHIATI 12306375
# Partie 2 
## Introduction

Dans ce TP, nous utilisons MongoDB, une base de données NoSQL orientée documents, afin d'explorer différentes opérations essentielles : 
- la manipulation de documents (find, projection, filtres),
- l'agrégation (pipeline, groupements, tris),
- la mise à jour et la suppression de données,
- ainsi que l’indexation pour optimiser les performances.

La base utilisée provient d’un fichier BSON restauré avec `mongorestore`.  
Elle contient plusieurs collections importantes, notamment :

- **movies** : informations détaillées sur des films  
- **comments** : commentaires des utilisateurs  
- **users** : données des comptes utilisateurs  

L’objectif de ce TP est de comprendre comment interroger, transformer et optimiser les données dans MongoDB à travers des requêtes simples, complexes ou agrégées.

---
## Partie 1 – Requêtes simples

### 1. Afficher les 5 films sortis depuis 2015

```js
db.movies.find({ year: { $gte: 2015 } }).limit(5)
```
{ year: { $gte: 2015 } } → filtre : retourne les films dont l’année (year) est ≥ 2015

.limit(5) → limite le nombre de résultats à 5
### 2. Trouver tous les films dont le genre est "Comedy"
```js
db.movies.find({ genres: "Comedy" })
```
{ genres: "Comedy" } → filtre simple : sélectionne les films contenant "Comedy" dans le tableau genres
### 3. Afficher les films sortis entre 2000 et 2005
```js
db.movies.find(
  { year: { $gte: 2000, $lte: 2005 } },
  { title: 1, year: 1 }
).pretty()
```
1er paramètre → filtre
{ year: { $gte: 2000, $lte: 2005 } }
→ garde les films avec 2000 ≤ year ≤ 2005

2ᵉ paramètre → projection
{ title: 1, year: 1 }
→ n’affiche que title et year

.pretty() → mise en forme lisible
### 4. Afficher les films de genres “Drama” ET “Romance”
```js
db.movies.find(
  { genres: { $all: ["Drama", "Romance"] } },
  { title: 1, genres: 1 }
)
```
Filtre
{ genres: { $all: ["Drama", "Romance"] } }
→ $all : le tableau genres doit contenir les deux valeurs

Projection
{ title: 1, genres: 1 }
→ n’affiche que le titre et les genres
### 5. Afficher les films sans champ rated
```js
db.movies.find(
  { rated: { $exists: false } },
  { title: 1 }
)
```
Filtre
{ rated: { $exists: false } }
→ retourne les films où rated n’existe pas

Projection
{ title: 1 }
→ affiche seulement le titre
## Partie 2 – Agrégation


### 6. Afficher le nombre de films par année

```js
db.movies.aggregate([
  { $group: { _id: "$year", total: { $sum: 1 } } },
  { $sort: { _id: 1 } }
])
```
$group

  _id: "$year" → regroupe par année

  total: { $sum: 1 } → compte le nombre de films par année

$sort: { _id: 1 } → tri croissant par année

### 7. Afficher la moyenne des notes IMDb par genre
```js
db.movies.aggregate([
  { $unwind: "$genres" },
  { $group: { _id: "$genres", moyenne: { $avg: "$imdb.rating" } } },
  { $sort: { moyenne: -1 } }
])
```
$unwind: "$genres" → sépare le tableau genres en éléments individuels

$group :

  _id: "$genres" → groupe par genre

  moyenne: { $avg: "$imdb.rating" } → calcule la moyenne des notes

$sort: { moyenne: -1 } → tri par moyenne décroissante
### 8. Afficher le nombre de films par pays
```js
db.movies.aggregate([
  { $unwind: "$countries" },
  { $group: { _id: "$countries", total: { $sum: 1 } } },
  { $sort: { total: -1 } }
])
```
$unwind → éclate le tableau countries

$group → groupe par pays (_id) et compte les films

$sort: { total: -1 } → du pays le plus productif au moins productif

### 9. Afficher les top 5 réalisateurs
```js
db.movies.aggregate([
  { $unwind: "$directors" },
  { $group: { _id: "$directors", total: { $sum: 1 } } },
  { $sort: { total: -1 } },
  { $limit: 5 }
])
```
$unwind: "$directors" → un réalisateur = une ligne

$group → groupe par réalisateur

$sum: 1 → compte ses films

$sort: { total: -1 } → classe les réalisateurs les plus productifs

$limit: 5 → garde les 5 premiers
### 10. Afficher les films triés par note IMDb
```js
db.movies.aggregate([
  { $sort: { "imdb.rating": -1 } },
  { $project: { title: 1, "imdb.rating": 1 } }
])
```
$sort: { "imdb.rating": -1 } → tri décroissant

$project → n’affiche que title et imdb.rating
## Partie 3 – Mises à jour
### 11. Ajouter un champ etat
```js
db.movies.updateOne(
  { title: "Jaws" },
  { $set: { etat: "culte" } }
)
```
Filtre : { title: "Jaws" } → cible le film

Update : $set → ajoute ou modifie le champ etat
### 12. Incrémenter les votes IMDb de 100
```js
db.movies.updateOne(
  { title: "Inception" },
  { $inc: { "imdb.votes": 100 } }
)
```
Filtre : { title: "Inception" }

$inc : incrémente imdb.votes de +100

### 13. Supprimer le champ poster pour tous les documents
```js
db.movies.updateMany(
  {},
  { $unset: { poster: "" } }
)
```
Filtre {} → tous les documents

$unset: { poster: "" } → supprime le champ poster
### 14. Modifier le réalisateur de “Titanic”
```js
db.movies.updateOne(
  { title: "Titanic" },
  { $set: { directors: ["James Cameron"] } }
)
```
Filtre : { title: "Titanic" }

$set : remplace totalement le tableau directors
## Partie 4 – Requêtes complexes
### 15. Afficher les films les mieux notés par décennie
```js
db.movies.aggregate([
  { $match: { "imdb.rating": { $exists: true } } },
  {
    $project: {
      title: 1,
      decade: { $subtract: ["$year", { $mod: ["$year", 10] }] },
      "imdb.rating": 1
    }
  },
  { $group: { _id: "$decade", maxRating: { $max: "$imdb.rating" } } },
  { $sort: { _id: 1 } }
])
```
$match → filtre : films ayant une note

$project → crée un champ decade (années arrondies à la décennie)

$group → récupère la meilleure note par décennie

$sort → tri par décennie
### 16. Afficher les films dont le titre commence par “Star”
```js
db.movies.find(
  { title: /^Star/ },
  { title: 1 }
)
```
Filtre : title: /^Star/ → expression régulière

Projection : { title: 1 }
### 17. Afficher les films avec plus de 2 genres
```js
db.movies.find(
  { $where: "this.genres.length > 2" },
  { title: 1, genres: 1 }
)
```
$where → exécute du JavaScript sur chaque document

Condition : genres.length > 2

Projection : titre + genres
### 18. Afficher les films de Christopher Nolan
```js
db.movies.find(
  { directors: "Christopher Nolan" },
  { title: 1, year: 1, "imdb.rating": 1 }
)
```
Filtre : { directors: "Christopher Nolan" } → cherche dans le tableau directors

Projection → affiche titre, année, note IMDb
## Partie 5 – Indexation
### 19. Créer un index sur year
```js
db.movies.createIndex({ year: 1 })
{ year: 1 } → index croissant sur year

### 20. Vérifier les index existants
```js
db.movies.getIndexes()
```
Aucun → retourne la liste des index
### 21. Comparer deux requêtes avec et sans index
```js
db.movies.find({ year: 1995 }).explain("executionStats")
```
Filtre → { year: 1995 }

.explain("executionStats") → montre le plan d’exécution
### 22. Supprimer l'index sur year
```js
db.movies.dropIndex({ year: 1 })
{ year: 1 } → identifie l’index à supprimer
```
### 23. Créer un index composé sur year et imdb.rating
```js
db.movies.createIndex({ year: 1, "imdb.rating": -1 })
```
year: 1 → tri croissant

"imdb.rating": -1 → tri décroissant

Index composé = accélère les recherches par année + note

## Conclusion

Ce TP a permis de découvrir les principales fonctionnalités de MongoDB à travers des manipulations concrètes :

- recherche et filtrage avancé de documents,
- agrégation de données avec `$group`, `$sort`, `$unwind`, `$project`…
- opérations de mise à jour (`$set`, `$unset`, `$inc`…),
- création et suppression d’index pour améliorer les performances des requêtes.

L’ensemble de ces techniques constitue la base du travail avec MongoDB dans des applications réelles.  
L’utilisation de `mongorestore`, `mongosh` et des outils comme MongoDB Compass facilite grandement l’exploration et la compréhension des données.


# Partie 3 
## Introduction

Ce travail pratique a pour objectif de prendre en main MongoDB, un système de gestion de bases de données NoSQL orienté documents. À travers une série de requêtes simples et progressives, nous explorons la structure d’une collection, les opérations de filtrage, les projections, ainsi que l’utilisation d’opérateurs de comparaison.  
Le TP s’appuie sur une base de données contenant une collection de films, ce qui permet de se familiariser avec les principes fondamentaux de MongoDB : insertion de données, interrogation de documents, extraction ciblée d’attributs, et manipulation de tableaux.  
L’objectif final est de comprendre comment interroger efficacement une base NoSQL, analyser les résultats retournés et maîtriser les commandes essentielles pour le développement d’applications exploitant MongoDB.

 --- 

### 1. Vérification de l’importation des données

La commande suivante permet de vérifier le nombre de documents présents dans la collection `films` :

```js
db.films.countDocuments()
```
Elle retourne :

```
278
```
Compte le nombre total de documents dans la collection films
### 2. Afficher la structure d’un document


```js
db.films.findOne()
```
Elle retourne :

```js
{
  _id: 'movie:33',
  title: 'Impitoyable',
  year: 1992,
  genre: 'Western',
  summary: "Après avoir été un impitoyable tueur, toujours entre deux verres, Bill Munny a raccroché ses colts pour l'amour d'une femme aujourd'hui disparue. Il élève péniblement des cochons dans un enclos boueux, avec pour seuls compagnons ses deux jeunes enfants. Bill reçoit un jour la visite de Schofield Kid, un apprenti desperado qui veut devenir le partenaire de cette légende vivante. Le Kid lui propose de partager les mille dollars offerts par des prostituées de Big Whiskey, une bourgade lointaine, pour l'élimination des deux cow-boys qui ont défiguré l'une d'entre elles. Munny finit par accepter la proposition et rend visite à son vieux complice, Ned Logan...",
  country: 'US',
  director: {
    _id: 'artist:190',
    last_name: 'Eastwood',
    first_name: 'Clint',
    birth_date: 1930
  },
  actors: [
    { last_name: 'Eastwood', first_name: 'Clint', birth_date: 1930 },
    { last_name: 'Freeman', first_name: 'Morgan', birth_date: 1937 },
    { last_name: 'Hackman', first_name: 'Gene', birth_date: 1930 },
    { last_name: 'Harris', first_name: 'Richard', birth_date: 1930 },
    { last_name: 'Woolvett', first_name: 'Jaimz', birth_date: 1967 },
    { last_name: 'Levine', first_name: 'Anna', birth_date: 1955 },
    { last_name: 'Rubinek', first_name: 'Saul', birth_date: 1948 }
  ],
  grades: [
    { note: 40, grade: 'B' },
    { note: 14, grade: 'D' },
    { note: 21, grade: 'F' },
    { note: 71, grade: 'A' }
  ]
}
```
Affiche un seul film pour visualiser sa structure : titre, pays, acteurs, notes, etc.
### 3. Afficher la liste des films d’action

```js
db.films.find({ genre: "Action" })
```
Elle retourne :

```js
[
  {
    _id: 'movie:180',
    title: 'Minority Report',
    year: 2002,
    genre: 'Action',
    summary: "À Washington, en 2054, la société du futur a éradiqué le meurtre en se dotant du système de prévention / détection / répression le plus sophistiqué du monde. Dissimulés au cœur du Ministère de la Justice, trois extra-lucides captent les signes précurseurs des violences homicides et en adressent les images à leur contrôleur, John Anderton, le chef de la « Précrime » devenu justicier après la disparition tragique de son fils. Celui-ci n'a alors plus qu'à lancer son escouade aux trousses du « coupable »... Mais un jour se produit l'impensable : l'ordinateur lui renvoie sa propre image. D'ici 36 heures, Anderton aura assassiné un parfait étranger. Devenu la cible de ses propres troupes, Anderton prend la fuite. Son seul espoir pour déjouer le complot : dénicher sa future victime. Sa seule arme : les visions parcellaires, énigmatiques, de la plus fragile des Pré-Cogs : Agatha.",
    country: 'US',
    director: {
      _id: 'artist:488',
      last_name: 'Spielberg',
      first_name: 'Steven',
      birth_date: 1946
    },
    actors: [
      { last_name: 'Stormare', first_name: 'Peter', birth_date: 1953 },
      { last_name: 'Cruise', first_name: 'Tom', birth_date: 1962 },
      {
        last_name: 'Blake Nelson',
        first_name: 'Tim',
        birth_date: 1964
      },
      { last_name: 'von Sydow', first_name: 'Max', birth_date: 1929 },
      { last_name: 'Morton', first_name: 'Samantha', birth_date: 1977 },
      { last_name: 'Smith', first_name: 'Lois', birth_date: 1930 },
      { last_name: 'Farrell', first_name: 'Colin', birth_date: 1976 }
    ],
    grades: [
      { note: 5, grade: 'F' },
      { note: 51, grade: 'F' },
      { note: 27, grade: 'D' },
      { note: 77, grade: 'D' }
    ]
  }...
```
1er paramètre = filtre → { genre: "Action" }
Sélectionne uniquement les documents dont le champ genre vaut "Action"
### 4. Afficher le nombre de films d’action
```js
db.films.countDocuments({ genre: "Action" })
```
Elle retourne :

```js
36
```
1er paramètre = filtre → { genre: "Action" }
Ne compte que les films du genre Action.

### 5. Films d’action produits en France
```js
db.films.find({ genre: "Action", country: "FR" })
```
Elle retourne :
```js
[
  {
    _id: 'movie:4034',
    title: "L'Homme de Rio",
    year: 1964,
    genre: 'Action',
    summary: "Le deuxième classe Adrien Dufourquet est témoin de l'enlèvement de sa fiancée Agnès, fille d'un célèbre ethnologue. Il part à sa recherche, qui le mène au Brésil, et met au jour un trafic de statuettes indiennes.",
    country: 'FR',
    director: {
      _id: 'artist:34613',
      last_name: 'de Broca',
      first_name: 'Philippe',
      birth_date: 1933
    },
    actors: [
      {
        last_name: 'Belmondo',
        first_name: 'Jean-Paul',
        birth_date: 1933
      },
      { last_name: 'Celi', first_name: 'Adolfo', birth_date: 1922 },
      { last_name: 'Servais', first_name: 'Jean', birth_date: 1910 },
      {
        last_name: 'Dorléac',
        first_name: 'Françoise',
        birth_date: 1942
      },
      { last_name: 'Renant', first_name: 'Simone', birth_date: 1911 }
    ],
    grades: [
      { note: 4, grade: 'D' },
      { note: 30, grade: 'E' },
      { note: 34, grade: 'E' },
      { note: 28, grade: 'F' }
    ]
  }...
```
Filtre → { genre: "Action", country: "FR" }
MongoDB ne garde que les films qui vérifient les deux conditions à la fois.


### 6. Films d’action produits en France en 1963
```js
db.films.find({
  genre: "Action",
  country: "FR",
  year: 1963
})

```
Elle retourne :

```js
[
  {
    _id: 'movie:25253',
    title: 'Les tontons flingueurs',
    year: 1963,
    genre: 'Action',
    summary: "Sur son lit de mort, le Mexicain fait promettre à son ami d'enfance, Fernand Naudin, de veiller sur ses intérêts et sa fille Patricia. Fernand découvre alors qu'il se trouve à la tête d'affaires louches dont les anciens dirigeants entendent bien s'emparer. Mais, flanqué d'un curieux notaire et d'un garde du corps, Fernand impose d'emblée sa loi. Cependant, le belle Patricia lui réserve quelques surprises !",
    country: 'FR',
    director: {
      _id: 'artist:18563',
      last_name: 'Lautner',
      first_name: 'Georges',
      birth_date: 1926
    },
    actors: [
      { last_name: 'Ventura', first_name: 'Lino', birth_date: 1919 },
      { last_name: 'Frank', first_name: 'Horst', birth_date: 1929 },
      { last_name: 'Dalban', first_name: 'Robert', birth_date: 1903 },
      { last_name: 'Blier', first_name: 'Bernard', birth_date: 1916 },
      { last_name: 'Rich', first_name: 'Claude', birth_date: 1929 },
      { last_name: 'Sinjen', first_name: 'Sabine', birth_date: 1942 },
      { last_name: 'Lefebvre', first_name: 'Jean', birth_date: 1920 },
      { last_name: 'Bertin', first_name: 'Pierre', birth_date: 1891 },
      { last_name: 'Blanche', first_name: 'Francis', birth_date: 1919 }
    ],
    grades: [
      { note: 35, grade: 'C' },
      { note: 48, grade: 'F' },
      { note: 64, grade: 'C' },
      { note: 42, grade: 'F' }
    ]
  }
]
```
Filtre → 3 conditions simultanées :

genre: "Action"

country: "FR"

year: 1963"
### 7. Films d’action produits en France sans les grades
```js
db.films.find(
  { genre: "Action", country: "FR" },
  { grades: 0 }
)

```
Elle retourne :
```js
[
  {
    _id: 'movie:4034',
    title: "L'Homme de Rio",
    year: 1964,
    genre: 'Action',
    summary: "Le deuxième classe Adrien Dufourquet est témoin de l'enlèvement de sa fiancée Agnès, fille d'un célèbre ethnologue. Il part à sa recherche, qui le mène au Brésil, et met au jour un trafic de statuettes indiennes.",
    country: 'FR',
    director: {
      _id: 'artist:34613',
      last_name: 'de Broca',
      first_name: 'Philippe',
      birth_date: 1933
    },
    actors: [
      {
        last_name: 'Belmondo',
        first_name: 'Jean-Paul',
        birth_date: 1933
      },
      { last_name: 'Celi', first_name: 'Adolfo', birth_date: 1922 },
      { last_name: 'Servais', first_name: 'Jean', birth_date: 1910 },
      {
        last_name: 'Dorléac',
        first_name: 'Françoise',
        birth_date: 1942
      },
      { last_name: 'Renant', first_name: 'Simone', birth_date: 1911 }
    ]
  },
  {
    _id: 'movie:9322',
    title: 'Nikita',
    year: 1990,
    genre: 'Action',
    summary: "Le braquage d'une pharmacie par une bande de junkies en manque de drogue tourne mal : une fusillade cause la mort de plusieurs personnes dont un policier, abattu par la jeune Nikita. Condamnée à la prison à perpétuité, celle-ci fait bientôt la rencontre de Bob, un homme mystérieux qui contraint la jeune femme à travailler secrètement pour le gouvernement. Après quelques rébellions lors d'un entraînement intensif de plusieurs années, Nikita devient un agent hautement qualifié des services secrets, capable désormais selon Bob d'évoluer seule à l'extérieur. Celui-ci espère d'ailleurs s'en assurer lors d'une terrible mise à l'épreuve, dans laquelle Nikita doit éliminer un pilier de la mafia asiatique au beau milieu d'un restaurant bondé...",
    country: 'FR',
    director: {
      _id: 'artist:59',
      last_name: 'Besson',
      first_name: 'Luc',
      birth_date: 1959
    },
    actors: [
      { last_name: 'Reno', first_name: 'Jean', birth_date: 1948 },
      { last_name: 'Duret', first_name: 'Marc', birth_date: 1957 },
      { last_name: 'Karyo', first_name: 'Tchéky', birth_date: 1953 },
      { last_name: 'Parillaud', first_name: 'Anne', birth_date: 1960 },
      { last_name: 'Fontana', first_name: 'Patrick', birth_date: null },
      { last_name: 'Lathière', first_name: 'Alain', birth_date: null }
    ]
  },
  {
    _id: 'movie:25253',
    title: 'Les tontons flingueurs',
    year: 1963,
    genre: 'Action',
    summary: "Sur son lit de mort, le Mexicain fait promettre à son ami d'enfance, Fernand Naudin, de veiller sur ses intérêts et sa fille Patricia. Fernand découvre alors qu'il se trouve à la tête d'affaires louches dont les anciens dirigeants entendent bien s'emparer. Mais, flanqué d'un curieux notaire et d'un garde du corps, Fernand impose d'emblée sa loi. Cependant, le belle Patricia lui réserve quelques surprises !",
    country: 'FR',
    director: {
      _id: 'artist:18563',
      last_name: 'Lautner',
      first_name: 'Georges',
      birth_date: 1926
    },
    actors: [
      { last_name: 'Ventura', first_name: 'Lino', birth_date: 1919 },
      { last_name: 'Frank', first_name: 'Horst', birth_date: 1929 },
      { last_name: 'Dalban', first_name: 'Robert', birth_date: 1903 },
      { last_name: 'Blier', first_name: 'Bernard', birth_date: 1916 },
      { last_name: 'Rich', first_name: 'Claude', birth_date: 1929 },
      { last_name: 'Sinjen', first_name: 'Sabine', birth_date: 1942 },
      { last_name: 'Lefebvre', first_name: 'Jean', birth_date: 1920 },
      { last_name: 'Bertin', first_name: 'Pierre', birth_date: 1891 },
      { last_name: 'Blanche', first_name: 'Francis', birth_date: 1919 }
    ]
  }
]
```
1er paramètre = filtre
→ { genre: "Action", country: "FR" }

2ᵉ paramètre = projection
→ { grades: 0 }
0 = masquer, 1 = afficher
Ici, le champ grades est caché.
### 8. Films d’action produits en France sans identifiants
```js
db.films.find(
  { genre: "Action", country: "FR" },
  { _id: 0 }
)
```
Elle retourne :
```js
[
  {
    title: "L'Homme de Rio",
    year: 1964,
    genre: 'Action',
    summary: "Le deuxième classe Adrien Dufourquet est témoin de l'enlèvement de sa fiancée Agnès, fille d'un célèbre ethnologue. Il part à sa recherche, qui le mène au Brésil, et met au jour un trafic de statuettes indiennes.",
    country: 'FR',
    director: {
      _id: 'artist:34613',
      last_name: 'de Broca',
      first_name: 'Philippe',
      birth_date: 1933
    },
    actors: [
      {
        last_name: 'Belmondo',
        first_name: 'Jean-Paul',
        birth_date: 1933
      },
      { last_name: 'Celi', first_name: 'Adolfo', birth_date: 1922 },
      { last_name: 'Servais', first_name: 'Jean', birth_date: 1910 },
      {
        last_name: 'Dorléac',
        first_name: 'Françoise',
        birth_date: 1942
      },
      { last_name: 'Renant', first_name: 'Simone', birth_date: 1911 }
    ],
    grades: [
      { note: 4, grade: 'D' },
      { note: 30, grade: 'E' },
      { note: 34, grade: 'E' },
      { note: 28, grade: 'F' }
    ]
  },
```
Filtre → { genre: "Action", country: "FR" }

Projection → { _id: 0 }
Cache uniquement l’identifiant _id.
### 9. Titres + grades des films d’action FR, sans _id
```js
db.films.find(
  { genre: "Action", country: "FR" },
  { _id: 0, title: 1, grades: 1 }
)

```

Elle retourne :

```js
[
  {
    title: "L'Homme de Rio",
    grades: [
      { note: 4, grade: 'D' },
      { note: 30, grade: 'E' },
      { note: 34, grade: 'E' },
      { note: 28, grade: 'F' }
    ]
  },
  {
    title: 'Nikita',
    grades: [
      { note: 88, grade: 'F' },
      { note: 36, grade: 'C' },
      { note: 74, grade: 'D' },
      { note: 62, grade: 'D' }
    ]
  },
  {
    title: 'Les tontons flingueurs',
    grades: [
      { note: 35, grade: 'C' },
      { note: 48, grade: 'F' },
      { note: 64, grade: 'C' },
      { note: 42, grade: 'F' }
    ]
  }
]
```
Filtre → { genre: "Action", country: "FR" }

Projection →

_id: 0 → masquer

title: 1 → afficher

grades: 1 → afficher
### 10. Titres et notes des films d’action FR ayant au moins une note > 10

```js
db.films.find(
  {
    genre: "Action",
    country: "FR",
    "grades.note": { $gt: 10 }
  },
  {
    _id: 0,
    title: 1,
    grades: 1
  }
)
```
Elle retourne :
```js
[
  {
    title: "L'Homme de Rio",
    grades: [
      { note: 4, grade: 'D' },
      { note: 30, grade: 'E' },
      { note: 34, grade: 'E' },
      { note: 28, grade: 'F' }
    ]
  },
  {
    title: 'Nikita',
    grades: [
      { note: 88, grade: 'F' },
      { note: 36, grade: 'C' },
      { note: 74, grade: 'D' },
      { note: 62, grade: 'D' }
    ]
  },
  {
    title: 'Les tontons flingueurs',
    grades: [
      { note: 35, grade: 'C' },
      { note: 48, grade: 'F' },
      { note: 64, grade: 'C' },
      { note: 42, grade: 'F' }
    ]
  }
]
```
Filtre :

genre: "Action"

country: "FR"

"grades.note": { $gt: 10 }
→ $gt = strictly greater than
→ Vérifie si au moins un objet du tableau grades a une note > 10.

Projection :

_id: 0 → masque _id

title: 1 → affiche title

grades: 1 → affiche grades
### 11. Films d’action FR dont toutes les notes sont > 10
```js
db.films.find(
  {
    genre: "Action",
    country: "FR",
    grades: {
      $not: {
        $elemMatch: { note: { $lte: 10 } }
      }
    }
  },
  {
    _id: 0,
    title: 1,
    grades: 1
  }
)

```
Elle retourne :
```js
[
[
  {
    title: 'Nikita',
    grades: [
      { note: 88, grade: 'F' },
      { note: 36, grade: 'C' },
      { note: 74, grade: 'D' },
      { note: 62, grade: 'D' }
    ]
  },
  {
    title: 'Les tontons flingueurs',
    grades: [
      { note: 35, grade: 'C' },
      { note: 48, grade: 'F' },
      { note: 64, grade: 'C' },
      { note: 42, grade: 'F' }
    ]
  }
]
```
Filtre :

genre: "Action"

country: "FR"

grades: { $not: { $elemMatch: { note: { $lte: 10 }}}}
→ $elemMatch cherche une note ≤ 10
→ $not inverse
→ donc : aucune note ≤ 10 → toutes > 10

Projection :

_id: 0 → masque _id

title: 1 → affiche title

grades: 1 → affiche grades
### 12. Afficher les différents genres de la base lesfilms
```js
db.films.distinct("genre")
```
Elle retourne :
```js
[
  'Action',          'Adventure',
  'Aventure',        'Comedy',
  'Comédie',         'Crime',
  'Drama',           'Drame',
  'Fantastique',     'Fantasy',
  'Guerre',          'Histoire',
  'Horreur',         'Musique',
  'Mystery',         'Mystère',
  'Romance',         'Science Fiction',
  'Science-Fiction', 'Thriller',
  'War',             'Western'
]
```
"genre"
→ champ dont on veut extraire toutes les valeurs distinctes.
### 13. Afficher les différents grades attribués
```js
db.films.distinct("grades.grade")
```
Elle retourne :
```js
[ 'A', 'B', 'C', 'D', 'E', 'F' ]
```
"grades.grade"
→ valeur distincte d’une sous-clé dans un tableau.
### 14. Films dans lesquels joue au moins un des artistes ["artist:4","artist:18","artist:11"]

```js
db.films.find({
...   cast: { $in: ["artist:4", "artist:18", "artist:11"] }})
```
Filtre : { cast: { $in: [...] } }
→ $in = le champ doit contenir au moins une valeur de la liste.
### 15. Films qui n’ont pas de résumé
```js
db.films.distinct("grades.grade")
```
Elle retourne :
```js
[
  {
    _id: 'movie:181812',
    title: 'Star Wars, épisode IX',
    year: 2019,
    genre: 'Science-Fiction',
    summary: '',
    country: 'GB',
    director: {
      _id: 'artist:15344',
      last_name: 'Abrams',
      first_name: 'J.J.',
      birth_date: 1966
    },
    actors: [
      { last_name: 'Hamill', first_name: 'Mark', birth_date: 1951 },
      { last_name: 'Fisher', first_name: 'Carrie', birth_date: 1956 },
      {
        last_name: 'Dee Williams',
        first_name: 'Billy',
        birth_date: 1937
      },
      { last_name: 'Isaac', first_name: 'Oscar', birth_date: 1979 },
      { last_name: 'Boyega', first_name: 'John', birth_date: 1992 },
      { last_name: 'Driver', first_name: 'Adam', birth_date: 1983 },
      { last_name: 'Ridley', first_name: 'Daisy', birth_date: 1992 },
      {
        last_name: 'Marie Tran',
        first_name: 'Kelly',
        birth_date: 1989
      }
    ],
    grades: [
      { note: 94, grade: 'B' },
      { note: 28, grade: 'A' },
      { note: 81, grade: 'B' },
      { note: 100, grade: 'B' }
    ]
  }
]
```
Filtre composé :

$exists: false → le champ n’existe pas

"" → chaîne vide

$or → au moins une des deux conditions est vraie
### 16. Films tournés avec Leonardo DiCaprio en 1997
```js
db.films.find({
  year: 1997,
  "actors.first_name": "Leonardo",
  "actors.last_name": "DiCaprio"
})
```
Elle retourne :
```js
[
  {
    _id: 'movie:597',
    title: 'Titanic',
    year: 1997,
    genre: 'Drame',
    summary: 'Southampton, 10 avril 1912. Le paquebot le plus grand et le plus moderne du monde, réputé pour son insubmersibilité, le « Titanic », appareille pour son premier voyage. 4 jours plus tard, il heurte un iceberg. À son bord, un artiste pauvre et une grande bourgeoise tombent amoureux.',
    country: 'US',
    director: {
      _id: 'artist:2710',
      last_name: 'Cameron',
      first_name: 'James',
      birth_date: 1954
    },
    actors: [
      { last_name: 'Winslet', first_name: 'Kate', birth_date: 1975 },
      { last_name: 'Zane', first_name: 'Billy', birth_date: 1966 },
      { last_name: 'Fisher', first_name: 'Frances', birth_date: 1952 },
      {
        last_name: 'DiCaprio',
        first_name: 'Leonardo',
        birth_date: 1974
      },
      { last_name: 'Bates', first_name: 'Kathy', birth_date: 1948 },
      { last_name: 'Stuart', first_name: 'Gloria', birth_date: 1910 }
    ],
    grades: [
      { note: 40, grade: 'C' },
      { note: 91, grade: 'B' },
      { note: 45, grade: 'D' },
      { note: 7, grade: 'A' }
    ]
  }
]
```
year: 1997

"actors.first_name": "Leonardo"

"actors.last_name": "DiCaprio"
→ Filtre sur un acteur dans le tableau actors.
### 17. Films tournés avec Leonardo DiCaprio ou en 1997
```js
db.films.find({
  $or: [
    {
      "actors.first_name": "Leonardo",
      "actors.last_name": "DiCaprio"
    },
    {
      year: 1997
    }
  ]
})
```
Elle retourne :
```js
[
  {
    _id: 'movie:184',
    title: 'Jackie Brown',
    year: 1997,
    genre: 'Crime',
    summary: "Hôtesse de l'air, Jackie Brown arrondit ses fins de mois en convoyant de l'argent liquide pour le compte de Ordell Robbie, trafiquant d'armes. Quand la police et le FBI lui signifient qu'ils comptent sur elle pour faire tomber Robbie, Jackie Brown échafaude un plan audacieux.",
    country: 'US',
    director: {
      _id: 'artist:138',
      last_name: 'Tarantino',
      first_name: 'Quentin',
      birth_date: 1963
    },
    actors: [
      { last_name: 'Tucker', first_name: 'Chris', birth_date: 1971 },
      { last_name: 'De Niro', first_name: 'Robert', birth_date: 1943 },
      { last_name: 'Grier', first_name: 'Pam', birth_date: 1949 },
      {
        last_name: 'L. Jackson',
        first_name: 'Samuel',
        birth_date: 1948
      },
      { last_name: 'Keaton', first_name: 'Michael', birth_date: 1951 },
      { last_name: 'Fonda', first_name: 'Bridget', birth_date: 1964 },
      { last_name: 'Bowen', first_name: 'Michael', birth_date: 1953 },
      { last_name: 'Forster', first_name: 'Robert', birth_date: 1941 }
    ],
    grades: [
      { note: 16, grade: 'E' },
      { note: 28, grade: 'C' },
      { note: 18, grade: 'D' },
      { note: 39, grade: 'B' }
    ]
  }...
```
$or : au moins une expression doit être vraie

1ᵉʳ test = contient DiCaprio

2ᵉ test = année 1997


## Conclusion

Ce TP a permis d’acquérir les bases essentielles de l’utilisation de MongoDB. À travers l’exploration de la collection *films*, nous avons appris à importer des données, à comprendre la structure d’un document, à réaliser des requêtes simples et à utiliser des filtres avancés pour affiner les résultats.  
L'utilisation des projections, des opérateurs ($gt, $in, $exists, $not…), ainsi que l’extraction de valeurs distinctes a mis en évidence la flexibilité et la puissance du modèle documentaire.  


