# Guide pas à pas — ETL Novaria dans Power BI

Ce guide reprend, dans l'ordre, les étapes réalisées dans `Novaria_ETL.pbix` et indique pour chacune **où se trouve le bouton**. Les noms sont ceux de Power BI Desktop en français.

Notation : `Onglet > Bouton` désigne un bouton du ruban. « PQ » désigne l'éditeur Power Query, ouvert par `Accueil > Transformer les données`.

## Repères dans l'éditeur Power Query

| Zone | Où | À quoi elle sert |
|---|---|---|
| Volet Requêtes | À gauche | Liste des requêtes et des groupes |
| Barre de formule | Au-dessus des données (`Affichage > Barre de formule` si elle est masquée) | Voir et modifier le code de l'étape |
| Bouton `fx` | À gauche de la barre de formule | Ajouter une étape écrite à la main |
| Étapes appliquées | À droite | Historique des transformations ; la roue dentée rouvre la fenêtre de l'étape |
| En-tête de colonne | Icône à gauche du nom : type de données. Flèche à droite : tri et filtre | Typer, trier, filtrer |

Gestes utilisés partout :

| Action | Où cliquer |
|---|---|
| Créer une requête qui dépend d'une autre | Clic droit sur la requête > `Référence` |
| Renommer une requête | Double-clic sur son nom, ou volet de droite, champ `Nom` |
| Créer un groupe | Clic droit dans le volet Requêtes > `Nouveau groupe` |
| Ranger une requête | Clic droit sur la requête > `Déplacer vers le groupe` |
| Ne pas charger une requête dans le modèle | Clic droit sur la requête > décocher `Activer le chargement` |
| Valider le travail | `Accueil > Fermer & appliquer` |

## Étape 1 — Connexion aux sources

Dans PQ, tout part de `Accueil > Nouvelle source` (ou `Accueil > Obtenir les données` depuis la fenêtre principale).

| Source | Où cliquer | Réglages |
|---|---|---|
| SQL Server | `Nouvelle source > SQL Server` | Serveur `localhost\SQLEXPRESS`, base `Novaria_NRJ`, mode `Importer`. Cocher la vue `VW_ConsommationsEnergetiques`, puis les tables `Batiment`, `Quartier`, `TypeEnergie` |
| Excel | `Nouvelle source > Classeur Excel` | Fichier `MeteoQuartier_Brut.xlsx`, cocher le tableau `MeteoQuartierBrut` |
| CSV | `Nouvelle source > Texte/CSV` | Fichier `TarifsEnergie_Brut.csv`, origine `65001 : Unicode (UTF-8)`, délimiteur `Point-virgule` |
| JSON | `Nouvelle source > JSON` | Fichier `AuditsEnergetiques_Brut.json` |

Pour le JSON, trois clics après l'ouverture :

1. `Transformer > Vers la table`, puis `OK`.
2. Double flèche dans l'en-tête de `Column1` > cocher les champs > `OK`.
3. Double flèche dans l'en-tête de `audit` > cocher `priorite`, `recommandation`, `coutEstimeEuro` > `OK`.

Ensuite, créer par `Référence` les requêtes `Stg_Consommations`, `Stg_Meteo`, `Stg_Tarifs`, `Stg_Audits` et les ranger dans `02_staging`. Les trois tables SQL deviennent `Ref_Batiment`, `Ref_Quartier`, `Ref_TypeEnergie` dans `01_Referentiels`.

## Étape 2 — Contrôle de la qualité

**Profilage.** `Affichage` > cocher `Qualité de la colonne`, `Distribution des colonnes` et `Profil de colonne`. Pour analyser toutes les lignes, cliquer en bas à gauche sur `Profilage de la colonne en fonction des 1000 premières lignes` et choisir l'ensemble du jeu de données.

**Lignes à clé vide (`Ctl_Vides_`).**

1. Clic droit sur la requête `Stg_` > `Référence`.
2. `Ajouter une colonne > Colonne personnalisée`, nom `RaisonControle`, formule du type `if [CodeQuartier] = null then "code quartier absent" else null`.
3. Flèche de l'en-tête `RaisonControle` > décocher `(null)`.

**Doublons (`Ctl_Doublons_`).**

1. Clic droit sur la requête `Stg_` > `Référence`.
2. Sélectionner les colonnes de la clé avec `Ctrl`, clic droit > `Supprimer les autres colonnes`.
3. `Accueil > Regrouper par` > `Avancé` > regrouper sur les colonnes de la clé, nouvelle colonne `NombreOccurrences`, opération `Compter les lignes`.
4. Flèche de l'en-tête `NombreOccurrences` > `Filtres numériques > Supérieur à` > `1`.

## Étape 3 — Nettoyage (requêtes `Controle_`)

Créer `Controle_Meteo`, `Controle_Tarifs`, `Controle_Audits`, `Controle_Consommations` par `Référence` aux requêtes `Stg_`.

| Besoin | Où cliquer | Réglage |
|---|---|---|
| Retirer les caractères invisibles | Sélectionner la colonne, `Transformer > Format > Nettoyer` | — |
| Retirer les espaces de début et de fin | `Transformer > Format > Supprimer les espaces` | — |
| Mettre en majuscules | `Transformer > Format > MAJUSCULES` | — |
| Corriger un code ou retirer une unité | `Accueil > Remplacer les valeurs` (ou clic droit sur la colonne) | `Valeur à rechercher` : par exemple `°C` ; `Remplacer par` : laisser vide |
| Changer le type | Icône à gauche du nom de la colonne | `Texte` avant un remplacement, `Nombre décimal` après |
| Remplacer le point décimal | `Accueil > Remplacer les valeurs` | Rechercher `.`, remplacer par `,` |
| Renommer une colonne | Double-clic sur l'en-tête | — |
| Ajouter une colonne calculée | `Ajouter une colonne > Colonne personnalisée` | Nom + formule |

Remplacements faits sur `CodeQuartier` (météo) : `-` par rien, `S0U` par `SOU`, `NORD` par `NOR`, `OUEST` par `OUE`.

Remplacements faits sur `DateReleve` (passée en `Texte`) : `1er` par `1`, `sept.` par `septembre`, `mars-24` par `mars 2024`, `août 25` par `août 2025`.

Colonne personnalisée de conversion des dates :

```
if [DateReleve] = null then null
else try Date.From(Number.FromText([DateReleve]))
otherwise try Date.FromText([DateReleve], "fr-FR")
otherwise null
```

Colonne `Mois` : `Ajouter une colonne > Colonne personnalisée`, formule `Date.StartOfMonth([DateReleve])`, puis type `Date`.

## Étape 4 — Rejets et dédoublonnage

**Retrouver l'identifiant officiel.**

1. `Accueil > Fusionner des requêtes` (groupe `Combiner`, à droite du ruban).
2. Cliquer la colonne de code dans la table du haut, choisir le référentiel dans la liste, cliquer la même colonne en bas.
3. `Type de jointure` : `Externe gauche`, puis `OK`.
4. Double flèche dans l'en-tête de la nouvelle colonne > ne cocher que l'identifiant, décocher `Utiliser le nom de la colonne d'origine comme préfixe`.

**Motif de rejet.** `Ajouter une colonne > Colonne personnalisée`, nom `RaisonRejet` :

```
if [DateReleve] = null then "Date de relevé non interprétable"
else if [IdQuartier] = null then "Quartier inconnu"
else if [Pluviometrie] <> null and [Pluviometrie] < 0 then "Mesure négative impossible"
else null
```

**Séparer valides et rejets.**

| Requête | Création | Filtre sur `RaisonRejet` |
|---|---|---|
| `Rejets_...` | `Référence` à `Controle_...` | Flèche de l'en-tête > décocher `(null)` |
| `Int_...` | `Référence` à `Controle_...` | Flèche de l'en-tête > ne garder que `(null)`, puis clic droit sur la colonne > `Supprimer` |

**Garder la ligne la plus récente (dans `Int_`).**

1. Flèche de l'en-tête `DateImport` (ou `DateChargement`) > `Tri décroissant`.
2. Cliquer `fx` à gauche de la barre de formule et écrire `= Table.Buffer(#"Lignes triées")`.
3. Sélectionner les colonnes de la clé avec `Ctrl`, puis `Accueil > Supprimer les lignes > Supprimer les doublons`.

## Étape 5 — Combinaison (`FactEnergie`)

1. Clic droit sur `Int_Consommations` > `Référence`, renommer `FactEnergie`, ranger dans `05_Modele`.
2. **Météo :** `Accueil > Fusionner des requêtes`, sélectionner `IdQuartier` puis `Mois` avec `Ctrl` dans les deux tables et dans le même ordre, jointure `Externe gauche`. Développer `TemperatureMoyenne` et `DJUChauffage`.
3. **Tarifs :** `Accueil > Fusionner des requêtes` avec `Int_Tarifs` sur `IdTypeEnergie`, jointure `Externe gauche`. Ne pas développer la colonne.
4. `Ajouter une colonne > Colonne personnalisée`, nom `PrixApplicableKWh` :

```
let Candidats = Table.SelectRows([Int_Tarifs], (t) => t[DateEffet] <= [Mois])
in  if Table.IsEmpty(Candidats) then null
    else Table.Max(Candidats, "DateEffet")[PrixKWhEuro]
```

5. Clic droit sur la colonne `Int_Tarifs` > `Supprimer`.
6. Colonne personnalisée `CoutRecalcule` : `if [PrixApplicableKWh] = null then null else Number.Round([QuantiteKWh] * [PrixApplicableKWh], 2)`.
7. **Contrôle :** le nombre de lignes s'affiche en bas à gauche de l'éditeur ; il doit rester à 26 448.

## Étape 6 — Modèle en étoile

**Dimensions (dans PQ).**

1. Clic droit sur `Int_Consommations` > `Référence`, renommer `DimBatiment`.
2. Sélectionner les colonnes descriptives avec `Ctrl`, clic droit > `Supprimer les autres colonnes`.
3. Sélectionner `IdBatiment`, puis `Accueil > Supprimer les lignes > Supprimer les doublons`.
4. Même méthode pour `DimQuartier` et `DimEnergie`.

**Table de dates.** `Accueil > Nouvelle source > Requête vide`, puis `Accueil > Éditeur avancé` pour coller le code de `DimDate`.

**Après `Fermer & appliquer`, dans la fenêtre principale :**

| Action | Où cliquer |
|---|---|
| Ouvrir la vue de modèle | Troisième icône de la barre verticale de gauche |
| Créer une relation | Glisser la clé de la dimension sur la même colonne de la table de faits |
| Vérifier une relation | Double-clic sur le trait, ou `Accueil > Gérer les relations` |
| Marquer la table de dates | Cliquer `DimDate` dans le volet `Données`, puis `Outils de table > Marquer comme table de dates`, colonne `Date` |
| Créer une mesure | Cliquer la table dans le volet `Données`, puis `Accueil > Nouvelle mesure` |
| Créer une colonne calculée | `Outils de table > Nouvelle colonne` |
| Trier `Mois` dans l'ordre du calendrier | Cliquer la colonne `Mois`, puis `Outils de colonne > Trier par colonne > N° mois` |

## Étape 7 — Rapport

| Action | Où cliquer |
|---|---|
| Ajouter un visuel | Cliquer une zone vide de la page, puis une icône du volet `Visualisations` |
| Alimenter un visuel | Glisser les champs du volet `Données` vers les zones du volet `Visualisations` |
| Mettre en forme | Volet `Visualisations`, icône du pinceau |
| Filtrer un seul visuel | Volet `Filtres` > `Filtres sur ce visuel` |
| Top 10 | Volet `Filtres`, champ du bâtiment, `Type de filtre : N premiers` |
| Garder un segment d'une page à l'autre | `Afficher > Synchroniser les segments` |
| Page de détail par bâtiment | Sur la page de détail, glisser `NomBatiment` dans la zone `Extraire` du volet `Visualisations` |
| Renommer ou masquer une page | Clic droit sur l'onglet de la page |

**Diagramme radar.**

1. Volet `Visualisations` > `…` > `Obtenir plus de visuels`, chercher `Radar Chart` (Microsoft Corporation), `Ajouter`.
2. Créer les deux mesures par `Accueil > Nouvelle mesure` :

```
Consommation mensuelle 2025 = CALCULATE([Consommation totale], DimDate[Année] = 2025)
Consommation mensuelle 2026 = CALCULATE([Consommation totale], DimDate[Année] = 2026)
```

3. Cliquer l'icône du radar, glisser `DimDate[Mois]` dans `Category` et les deux mesures dans `Y Axis`.
4. Volet `Filtres` > glisser `N° mois` dans `Filtres sur ce visuel` > `Filtrage avancé` > `est inférieur ou égal à` > `6` > `Appliquer le filtre`.

## Vérifications finales

| Contrôle | Résultat attendu |
|---|---|
| `Accueil > Actualiser` | Aucune erreur, aucune fenêtre de saisie |
| Lignes de `FactEnergie` (vue Table, bas de l'écran) | 26 448 |
| Bilan par source | Départ = valides + rejets + doublons retirés |
| Parcours du rapport | Synthèse > clic droit sur un bâtiment > `Extraire` > retour par la flèche |
