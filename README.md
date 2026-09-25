# Projet ETL — Navicat + MongoDB

Pipeline ETL simple : nettoyage des ventes d'une boutique en ligne.

```
ventes_brutes.csv --[EXTRACT]--> ventes_brutes (Mongo) --[TRANSFORM/LOAD via $out]--> ventes_propres (Mongo)
```

## Contenu du projet

| Fichier / Collection | Rôle |
|---|---|
| `ventes_brutes.csv` | Données source (25 lignes, avec doublons et valeurs manquantes volontaires) |
| `ventes_brutes` (collection Mongo) | Résultat de l'import brut (Extract) |
| `ventes_propres` (collection Mongo) | Résultat nettoyé (Transform + Load) |

## Prérequis

- Un cluster MongoDB (Atlas M0 gratuit ou local)
- Navicat Premium ou Navicat for MongoDB, connecté au cluster

## Étapes

### 1. Extract
Import Wizard sur `boutique_ecommerce` → cible `ventes_brutes`, à partir de `ventes_brutes.csv` (première ligne = en-têtes).

### 2. Transform + Load
Ouvrir l'**éditeur d'agrégation** (clic droit sur `ventes_brutes` → Agrégation, ou `F4`) et construire le pipeline suivant, une étape à la fois :

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

**Stage 4 — `$out`** (écrit le résultat dans la collection finale — ⚠️ ne saisir que la valeur, pas la clé `$out` autour) :
```json
"ventes_propres"
```

Exécuter (▶). Puis clic droit sur la base → **Refresh** pour voir apparaître `ventes_propres`.

## Vérification

- `ventes_brutes` : 25 documents
- `ventes_propres` : 20 documents attendus (25 − 3 lignes incomplètes/dupliquées − 2 doublons de contenu)
- Sur un document de `ventes_propres` : `montant_total = quantite × prix_unitaire`

## Pièges rencontrés

- Les cellules vides d'un CSV importé par Navicat deviennent `null`, pas `""` → toujours filtrer les deux.
- Deux commandes peuvent avoir un `id_commande` différent tout en étant un doublon de contenu → grouper sur les champs métier, pas sur l'identifiant.
- Dans l'éditeur d'agrégation, chaque étape ne doit contenir que le **corps** de l'opérateur (ou juste sa valeur pour `$out`) : Navicat ajoute déjà la clé de l'opérateur autour.

## Pour aller plus loin

- Automatiser l'import avec un **Batch Job** Navicat (planification).
- Construire un tableau de bord depuis `ventes_propres` avec l'onglet **Chart/BI** de Navicat.
