# Brief coach — carnet d'entraînement

Ce document est destiné à Claude Code. Il décrit la base, les règles de
progression, et le travail à faire chaque semaine.

## Contexte athlète

- 23 ans, 125 kg, objectif 100 kg.
- Sportif depuis l'enfance, musculation niveau intermédiaire solide, surtout en poussée.
- Reprise de la muscu il y a 2 mois après une pause. Course à pied depuis 1 mois.
- Objectif course : 5 km confortable, puis chercher le chrono, puis 10 km.
- FC max estimée 192 bpm. Sorties actuelles à 155-162 bpm, soit 81-84 % — trop haut
  pour de l'endurance fondamentale.
- Références course : 3,75 km en 27'39 puis 4 km en 28'35 (≈ 7'09/km).

**Contraintes médicales à ne jamais ignorer :**
- Épaule gauche limitée. Développé militaire en **prise neutre uniquement**.
  Renforcement des coiffes des rotateurs à chaque séance de haut du corps.
- Brûlure à l'extérieur de la paume : vérifier périodiquement si les prises
  barre restent douloureuses.

**Planning hebdomadaire :** lundi Push, mardi Pull + fractionné, mercredi Push,
jeudi Jambes + course facile, vendredi piscine, samedi repos ou foot,
dimanche Pull + course facile. Tennis 3×/semaine, hors programme.

## Base de données

Projet Supabase. Deux tables.

### `carnet` — ce qu'il a fait

```
user_id    uuid  primary key
data       jsonb
updated_at timestamptz
```

`data` contient tout l'historique :

```json
{
  "seances": [
    {"id":"a1b2","date":"2026-07-28","type":"pull",
     "exos":[{"n":"Tirage horizontal V-bar","sets":[{"w":79,"r":12},{"w":79,"r":10}]}],
     "note":"épaule ok"}
  ],
  "cardio": [{"id":"c1","date":"2026-07-28","kind":"course","dist":4,"secs":1715,"fc":158}],
  "poids":  [{"id":"p1","date":"2026-07-28","kg":124.2}],
  "reglages":{"age":23}
}
```

`type` vaut `push`, `pull` ou `legs`. `kind` vaut `course` ou `piscine`.
`secs` est la durée totale en secondes, `dist` en kilomètres.

### `prescriptions` — ce qu'il doit faire

À créer :

```sql
create table prescriptions (
  user_id  uuid not null references auth.users on delete cascade,
  date     date not null,
  contenu  jsonb not null,
  cree_le  timestamptz not null default now(),
  primary key (user_id, date)
);

alter table prescriptions enable row level security;

create policy "chacun ses donnees" on prescriptions
  for all
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);
```

Format de `contenu` :

```json
{
  "muscu": {
    "exos": [
      {"n":"Tirage horizontal V-bar","cible":"79 kg × 12-14",
       "note":"tu as tenu 12 la dernière fois, vise 14 avant de monter"}
    ]
  },
  "cardio": {
    "kind":"course",
    "titre":"Fractionné court",
    "blocs":[["Échauffement","10 min très lent"],
             ["Corps de séance","6 × (1 min rapide / 2 min marche)"]],
    "dist":"4 à 4,5 km",
    "allure":"5'50 à 6'20/km sur les blocs",
    "fc":[0.85,0.92],
    "fcRecup":0.75,
    "cle":"une phrase, l'idée directrice de la séance"
  }
}
```

Le champ `n` doit correspondre **exactement** au nom de l'exercice dans l'app,
sinon l'objectif ne s'affiche pas. `fc` est un couple de pourcentages de FC max.
Les deux blocs `muscu` et `cardio` sont facultatifs.

L'app lit la ligne dont la `date` est celle du jour et affiche les objectifs.
En l'absence de ligne, elle retombe sur ses valeurs par défaut.

## Règles de progression

### Musculation — double progression

Chaque exercice a une fourchette de répétitions (8-12 sur les gros mouvements,
12-18 sur l'isolation et la coiffe).

1. Tant qu'il n'atteint pas le haut de la fourchette sur toutes les séries,
   la charge ne bouge pas : il ajoute des répétitions.
2. Quand il atteint le haut sur **deux séances consécutives**, monter la charge
   du plus petit incrément disponible et redescendre en bas de fourchette.
3. Trois séances sans progression, ni en charge ni en répétitions : réduire la
   charge de 10 % et remonter. La stagnation vient presque toujours de la
   récupération ou du sommeil, pas du programme.

Cas particuliers :
- **Traction assistée** : la charge est l'assistance, donc progresser signifie
  **descendre**. Ne jamais l'inverser dans les calculs.
- **Coiffe des rotateurs** (rotation externe, face pull) : ne jamais chercher la
  charge. Rester léger, viser la qualité. Progression par répétitions seulement.
- **Développé militaire machine** : prise neutre, sans exception.

### Course — priorité à la base aérobie

Il est en surpoids et court depuis un mois : le risque principal est la blessure
par surcharge, pas le manque d'intensité.

1. Volume hebdomadaire : +10 % maximum d'une semaine à l'autre.
2. Une seule séance intense par semaine (le fractionné du mardi). Le reste en
   endurance fondamentale, 65-75 % de FC max, soit 125-144 bpm.
3. S'il ne parvient pas à rester sous 144 bpm en courant, prescrire de
   l'alternance course/marche. C'est une méthode, pas un échec.
4. Ne pas travailler le chrono sur 5 km tant qu'il ne tient pas 5 km continus
   en restant dans sa zone.
5. Toute douleur articulaire signalée dans les notes : remplacer la course par
   la piscine ou le vélo pour la semaine.

### Poids

Objectif 125 → 100 kg. Une perte de 0,5 à 0,75 kg par semaine est la bonne
fourchette. Raisonner sur la moyenne mobile de trois semaines, jamais sur une
pesée isolée. Si la perte dépasse 1 kg par semaine sur trois semaines, le
signaler : le déficit est trop agressif et la masse musculaire trinque.

## Travail hebdomadaire attendu

1. Lire `carnet.data` et prendre les 3 à 4 dernières semaines.
2. Pour chaque exercice, appliquer la double progression et calculer la cible.
3. Analyser les sorties : allure, FC, écart aux zones prescrites.
4. Lire les notes de séance — c'est là que se trouvent les signaux d'épaule,
   de main et de fatigue. Ils priment sur les chiffres.
5. Écrire une ligne `prescriptions` par jour d'entraînement de la semaine à venir.
6. Résumer en quelques phrases ce qui a changé et pourquoi.

Ne jamais monter les charges et le volume de course la même semaine.
Si les notes mentionnent une douleur d'épaule, aucune progression sur les
mouvements de poussée, quelles que soient les performances chiffrées.

## Accès

La clé `service_role` de Supabase est nécessaire pour écrire. Elle reste dans un
fichier `.env` local, jamais dans le dépôt, jamais dans `index.html`.

```
SUPABASE_URL=https://tjajybrpsyqnyhfgcjzt.supabase.co
SUPABASE_SERVICE_KEY=...
```
