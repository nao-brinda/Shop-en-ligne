# Projet ETL — Navicat + MongoDB

Pipeline ETL : nettoyage puis analyse des ventes d'une boutique en ligne.

```
CSV --[EXTRACT]--> ventes_brutes (Mongo) --[TRANSFORM/LOAD via $out]--> ventes_propres (Mongo) --[analyses]--> insights
```

## Fichiers du projet

| Fichier / Collection | Rôle |
|---|---|
| `ventes_brutes.csv` | Échantillon initial (25 lignes) — Partie 1 |
| `ventes_brutes_1000.csv` | Échantillon élargi (~1018 lignes) — Partie 2 |

## Prérequis

- Un cluster MongoDB (Atlas M0 gratuit ou local)
- Navicat Premium ou Navicat for MongoDB, connecté au cluster

---

# Partie 1 — Premier essai (petit échantillon, `ventes_brutes.csv`)

### 1. Extract
Import Wizard sur `boutique_ecommerce` → cible `ventes_brutes`, à partir de `ventes_brutes.csv` (première ligne = en-têtes).

### 2. Transform + Load
Éditeur d'agrégation sur `ventes_brutes` (clic droit → Agrégation, ou `F4`) :

**Stage 1 — `$match`** (écarte les lignes incomplètes ; `null` ET `""` car Navicat importe les cellules vides en `null`) :
```json
{
  "client": { "$nin": [null, ""] },
  "quantite": { "$nin": [null, ""] }
}
```

**Stage 2 — `$group`** (déduplique sur le contenu réel, pas sur `id_commande`) :
```json
{
  "_id": {
    "client": "$client",
    "produit": "$produit",
    "quantite": "$quantite",
    "prix_unitaire": "$prix_unitaire",
    "date_commande": "$date_commande",
    "pays": "$pays"
  },
  "id_commande": { "$min": "$id_commande" }
}
```

**Stage 3 — `$project`** (aplatit les champs + calcule le montant total) :
```json
{
  "id_commande": "$id_commande",
  "client": "$_id.client",
  "produit": "$_id.produit",
  "quantite": "$_id.quantite",
  "prix_unitaire": "$_id.prix_unitaire",
  "date_commande": "$_id.date_commande",
  "pays": "$_id.pays",
  "montant_total": { "$multiply": ["$_id.quantite", "$_id.prix_unitaire"] }
}
```

**Stage 4 — `$out`** (écrit le résultat final ; ⚠️ ne saisir que la valeur, pas la clé `$out` autour) :
```json
"ventes_propres"
```

Exécuter (▶), puis clic droit sur la base → **Refresh**.

### Vérification
- `ventes_brutes` : 25 documents
- `ventes_propres` : 20 documents attendus (25 − 3 lignes incomplètes − 2 doublons de contenu)

### Pièges rencontrés
- Cellule vide dans un CSV importé par Navicat → devient `null`, pas `""` → toujours filtrer les deux.
- Deux commandes peuvent avoir un `id_commande` différent tout en étant un doublon de contenu → grouper sur les champs métier, pas sur l'identifiant.
- Dans l'éditeur d'agrégation, chaque étape ne contient que le **corps** de l'opérateur (ou juste sa valeur pour `$out`) : Navicat ajoute déjà la clé de l'opérateur autour.

---

# Partie 2 — Passage à l'échelle (`ventes_brutes_1000.csv`)

Même structure de colonnes que la Partie 1, donc même logique — juste plus de volume (~1018 lignes, ~4 % de valeurs manquantes, ~5 % de doublons de contenu).

### 1. Extract
Import Wizard → cible `ventes_brutes` (même collection, réimportée avec le fichier `_1000`, ou une nouvelle collection si vous préférez garder les deux jeux séparés) à partir de `ventes_brutes_1000.csv`.

### 2. Transform + Load
Exactement le même pipeline que la Partie 1 :

```javascript
db.getCollection("ventes_brutes").aggregate([
  {$match: {
    "client": { "$nin": [null, ""] },
    "quantite": { "$nin": [null, ""] }
  }},
  {$group: {
    "_id": {
      "client": "$client",
      "produit": "$produit",
      "quantite": "$quantite",
      "prix_unitaire": "$prix_unitaire",
      "date_commande": "$date_commande",
      "pays": "$pays"
    },
    "id_commande": { "$min": "$id_commande" }
  }},
  {$project: {
    "id_commande": "$id_commande",
    "client": "$_id.client",
    "produit": "$_id.produit",
    "quantite": "$_id.quantite",
    "prix_unitaire": "$_id.prix_unitaire",
    "date_commande": "$_id.date_commande",
    "pays": "$_id.pays",
    "montant_total": { "$multiply": ["$_id.quantite", "$_id.prix_unitaire"] }
  }},
  {$out: "ventes_propres"}
]);
```

### 3. Analyses / Insights (sur `ventes_propres`)

À lancer un par un dans l'éditeur d'agrégation sur `ventes_propres` :

**a) Chiffre d'affaires global + panier moyen**
```json
// $group
{
  "_id": null,
  "chiffre_affaires_total": { "$sum": "$montant_total" },
  "nombre_commandes": { "$sum": 1 },
  "panier_moyen": { "$avg": "$montant_total" }
}
```

**b) Ventes par pays**
```json
// Stage 1 - $group
{
  "_id": "$pays",
  "chiffre_affaires": { "$sum": "$montant_total" },
  "nombre_commandes": { "$sum": 1 },
  "panier_moyen": { "$avg": "$montant_total" }
}
```
```json
// Stage 2 - $sort
{ "chiffre_affaires": -1 }
```

**c) Top 5 produits (par chiffre d'affaires)**
```json
// Stage 1 - $group
{
  "_id": "$produit",
  "quantite_totale": { "$sum": "$quantite" },
  "chiffre_affaires": { "$sum": "$montant_total" }
}
```
```json
// Stage 2 - $sort
{ "chiffre_affaires": -1 }
```
```json
// Stage 3 - $limit
5
```

**d) Ventes par mois**
```json
// Stage 1 - $group (extrait "AAAA-MM" de la date)
{
  "_id": { "$substr": ["$date_commande", 0, 7] },
  "chiffre_affaires": { "$sum": "$montant_total" },
  "nombre_commandes": { "$sum": 1 }
}
```
```json
// Stage 2 - $sort
{ "_id": 1 }
```

**e) Top 5 meilleurs clients**
```json
// Stage 1 - $group
{
  "_id": "$client",
  "total_depense": { "$sum": "$montant_total" },
  "nombre_commandes": { "$sum": 1 }
}
```
```json
// Stage 2 - $sort
{ "total_depense": -1 }
```
```json
// Stage 3 - $limit
5
```

### Vérification
- `ventes_brutes` : ~1018 documents
- `ventes_propres` : un peu moins de 1000 documents attendus, selon le nombre exact de lignes incomplètes/dupliquées de ce tirage.

---

## Pour aller plus loin

- Combiner les 5 analyses en une seule requête avec `$facet` (plusieurs sous-pipelines exécutés en parallèle).
- Automatiser l'import avec un **Batch Job** Navicat (planification).
- Construire un tableau de bord depuis `ventes_propres` avec l'onglet **Chart/BI** de Navicat.
