# Pipeline-House-Price-Prediction

## Membres du groupe
- Thandie DRANÉ-COPHY
- Jephté DUFERMEAU

## Description du projet
Pipeline complet de Data Engineering et Machine Learning développé sur Snowflake pour prédire le prix de vente de maisons à partir de leurs caractéristiques (surface, nombre de pièces, équipements, etc.).

L'ensemble du workflow :ingestion, transformation, entraînement, optimisation et inférence est réalisé directement dans Snowflake sans export des données vers un environnement externe.

---

## Architecture du pipeline
```
S3 (logbrain-datalake)
       |
       v
HOUSE_PRICE_RAW       <- Ingestion via Snowflake Stage (JSON)
       |
       v
HOUSE_PRICE_FEATURES  <- Feature Engineering avec Snowpark
       |
       v
Model Journey         <- Entraînement itératif sur 5 rounds
(Ridge / Random Forest / Gradient Boosting)
       |
       v
GridSearch            <- Optimisation des hyperparamètres
       |
       v
Model Registry        <- Stockage versionné (v1, v2, alias production)
       |
       v
HOUSE_PRICE_PREDICTIONS  <- Inférence et stockage des résultats
       |
       v
Streamlit App         <- Interface utilisateur dans Snowflake
```

---

## Dataset

| Colonne           | Description                                          |
|-------------------|------------------------------------------------------|
| price             | Prix de vente (variable cible)                       |
| area              | Surface totale en m²                                 |
| bedrooms          | Nombre de chambres                                   |
| bathrooms         | Nombre de salles de bain                             |
| stories           | Nombre d'étages                                      |
| mainroad          | Accès route principale (yes/no)                      |
| guestroom         | Chambre d'amis (yes/no)                              |
| basement          | Sous-sol (yes/no)                                    |
| hotwaterheating   | Chauffage eau chaude (yes/no)                        |
| airconditioning   | Climatisation (yes/no)                               |
| parking           | Nombre de places de parking                          |
| prefarea          | Zone privilégiée (yes/no)                            |
| furnishingstatus  | Etat d'ameublement (furnished/semi/unfurnished)      |

---

## Matrice de corrélation 
![Image 01-04-2026 à 10 57](https://github.com/user-attachments/assets/c397bec1-fdea-4366-b6c8-4061e9689912)


---

## Modèles entraînés

### Approche retenue : The Model Journey

Plutôt qu'un entraînement unique, nous avons mis en place une boucle d'itération sur 5 rounds progressifs comparant 3 familles d'algorithmes :

| Modèle             | Logique d'itération                                       |
|--------------------|-----------------------------------------------------------|
| Ridge Regression   | Alpha décroissant (régularisation de plus en plus souple) |
| Random Forest      | Nombre d'arbres et profondeur croissants                  |
| Gradient Boosting  | Plus d'estimateurs, learning rate décroissant             |

Le vainqueur du Model Journey est **Gradient Boosting** avec un R² de 0.8987.
![Image 01-04-2026 à 10 56](https://github.com/user-attachments/assets/05148186-6746-4dd9-9f13-aa9ce6c488ce)

---

## Analyse des performances

### Comparaison des modèles après le Model Journey (Round 5)

| Modèle            | MAE    | RMSE   | R²     | Accuracy | Precision | Recall |
|-------------------|--------|--------|--------|----------|-----------|--------|
| Ridge Regression  | —      | —      | —      | —        | —         | —      |
| Random Forest     | —      | —      | —      | —        | —         | —      |
| Gradient Boosting | 18 683 | 30 056 | 0.8987 | 0.8624   | 0.8690    | 0.8624 |

> Les métriques Round 5 de Ridge et Random Forest sont disponibles
> dans la console du notebook.

### Après optimisation GridSearch

| Version  | Modèle                            | MAE    | RMSE   | R²     | Accuracy | Precision | Recall |
|----------|-----------------------------------|--------|--------|--------|----------|-----------|--------|
| v1       | Gradient Boosting (Model Journey) | 18 683 | 30 056 | 0.8987 | 0.8624   | 0.8690    | 0.8624 |
| v2       | Gradient Boosting (GridSearch)    | 18 303 | 28 660 | 0.9079 | 0.8716   | 0.8774    | 0.8716 |
| **prod** | **v2 — GridSearch optimisé**      | **18 303** | **28 660** | **0.9079** | **0.8716** | **0.8774** | **0.8716** |

### Interprétation des métriques

**MAE (Mean Absolute Error)** : erreur moyenne en valeur absolue entre
le prix prédit et le prix réel. Plus il est bas, plus le modèle est précis.

**RMSE (Root Mean Squared Error)** : similaire au MAE mais pénalise
davantage les grandes erreurs. Utile pour détecter les prédictions aberrantes.

**R²** : part de la variance du prix expliquée par le modèle.
Un R² de 0.9079 signifie que le modèle explique 90.79% des variations de prix.

**Accuracy / Precision / Recall** : métriques de classification calculées après discrétisation du prix en 3 segments (bas / moyen / élevé).
Permettent d'évaluer la capacité du modèle à identifier correctementle segment de prix d'un bien.

### Axes d'amélioration identifiés
- Enrichir le dataset avec des variables géographiques (quartier, ville)
- Tester d'autres algorithmes : XGBoost, LightGBM, CatBoost
- Appliquer une transformation logarithmique sur PRICE pour normaliser
  la distribution et réduire l'impact des valeurs extrêmes
- Augmenter le volume de données pour améliorer la généralisation

---

## Hyperparamètres retenus (GridSearch)

| Hyperparamètre      | Valeur testée   |
|---------------------|-----------------|
| n_estimators        | 100 / 200 / 300 |
| max_depth           | 5 / 10 / None   |
| min_samples_split   | 2 / 5 / 10      |
| max_features        | sqrt / log2     |

> La combinaison optimale retenue est affichée dans la console
> du notebook après exécution du GridSearch.

---

## Version de production

Le modèle enregistré sous l'alias **production** dans le Snowflake
Model Registry est la version **v2** (Gradient Boosting optimisé par GridSearch),
qui obtient le meilleur R² de **0.9079** contre 0.8987 pour la v1.

---

## Application Streamlit

L'application permet aux utilisateurs métier d'interagir avec le modèle
de production sans aucune connaissance technique. Elle offre :

- Une interface en 3 colonnes : Superficie & Structure / Pièces & Confort / Équipements & Localisation
- Des toggles interactifs pour les caractéristiques binaires
- Un calcul en temps réel via le Snowflake Model Registry
- 4 métriques affichées en résultat :
  - Prix estimé en roupies indiennes (₹)
  - Prix au m²
  - Segment de marché (Bas / Moyen / Élevé)
  - Budget avec marge de sécurité de 10%
- Un tableau récapitulatif de toutes les caractéristiques saisies

Les prix sont exprimés en roupies indiennes (₹), le dataset étant
issu du marché immobilier indien. Les seuils de segmentation sont :
- Prix bas : moins de 40 000 €
- Prix moyen : entre 40 000 € et 200 000 €
- Prix élevé : au-delà de 200 000 €

## Inteface de l'application Streamlit : ![Image 01-04-2026 à 11 11](https://github.com/user-attachments/assets/df08170a-e998-4f07-bb22-8e474454ffea)

## Resultat de l'estimation de l'application : ![Image 01-04-2026 à 11 11 (1)](https://github.com/user-attachments/assets/1525a1de-310e-41e8-a9c0-dd530a2889c2)



---

## Stack technique

| Outil                  | Usage                                       |
|------------------------|---------------------------------------------|
| Snowflake              | Plateforme de données et d'exécution ML     |
| Snowpark (Python)      | Manipulation des données                    |
| Snowflake Notebook     | Environnement de développement              |
| scikit-learn           | Entraînement et évaluation des modèles      |
| Snowflake ML Registry  | Versioning et stockage des modèles          |
| Streamlit in Snowflake | Application d'inférence interactive         |


