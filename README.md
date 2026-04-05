# Pipeline-House-Price-Prediction

## Membres du groupe
- Thandie DRANÉ-COPHY
- Jephté DUFERMEAU

## Description du projet
Pipeline complet de Data Engineering et Machine Learning développé sur Snowflake pour prédire le prix de vente de maisons à partir de leurs caractéristiques (surface, nombre de pièces, équipements, etc.).

L'ensemble du workflow : ingestion, transformation, entraînement, optimisation et inférence est réalisé directement dans Snowflake sans export des données vers un environnement externe.

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

###  Données initiales

- **1090 lignes**
- **13 variables**
  
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

### Nettoyage des données

- **545 doublons supprimés (50%)**
- Dataset final : **545 lignes**
- **0 valeurs manquantes**

 Ce nettoyage a eu un impact majeur sur les performances.


---

## Matrice de corrélation 
<img width="1460" height="1091" alt="image" src="https://github.com/user-attachments/assets/4ae4e027-6f2c-407e-aea6-54209fc0028f" />



---

## Modèles entraînés

### Approche retenue : The Model Journey

Plutôt qu'un entraînement unique, nous avons mis en place une boucle d'itération sur 5 rounds progressifs comparant 3 familles d'algorithmes :

| Modèle             | Logique d'itération                                       |
|--------------------|-----------------------------------------------------------|
| Ridge Regression   | Alpha décroissant (régularisation de plus en plus souple) |
| Random Forest      | Nombre d'arbres et profondeur croissants                  |
| Gradient Boosting  | Plus d'estimateurs, learning rate décroissant             |

### Comparaison globale

| Modèle              | R²    |
|---------------------|------|
| Ridge Regression    | **0.719** |
| Gradient Boosting   | 0.647 |
| Random Forest       | 0.639 |
| XGBoost (benchmark) | ~0.65 |


<img width="1460" height="1078" alt="image" src="https://github.com/user-attachments/assets/baab58da-f5d3-4617-913d-96b6abc641fc" />

---

## Analyse des performances

### Comparaison des modèles après le Model Journey (Round 5)

| Modèle              | MAE     | RMSE    | R²     | Accuracy |
|---------------------|--------|--------|--------|----------|
| Ridge Regression    | 37 593 | 58 646 | **0.719** | 0.8624 |
| Gradient Boosting   | 45 539 | 65 680 | 0.647 | 0.7615 |
| Random Forest       | 44 975 | 66 392 | 0.639 | 0.7431 |


### Après optimisation GridSearch

Un GridSearch a été appliqué sur le modèle Gradient Boosting afin d’optimiser les hyperparamètres.

| Version                  | MAE     | RMSE    | R²     | Accuracy |
|--------------------------|--------|--------|--------|----------|
| Avant optimisation       | 45 539 | 65 680 | 0.647 | 0.7615 |
| Après GridSearch         | 45 083 | 67 022 | 0.633 | 0.7615 |

Contrairement aux attentes, le GridSearch n'améliore pas les performances.

- Le R² diminue (0.647 → 0.633)
- Le RMSE augmente
- L'accuracy reste stable

### Interprétation des résultats

- Les relations sont **plutôt linéaires**
- Ridge Regression capture mieux ces patterns
- Les modèles complexes **sous-performent sur ce dataset**

### Impact du GridSearch

L’optimisation du Gradient Boosting a dégradé les performances :

- R² : **0.647 → 0.633**
- RMSE : augmentation
- Accuracy : stable

Cela montre que :

> **L’optimisation des hyperparamètres ne compense pas un manque de données**
Les performances des modèles sont évaluées à l’aide de plusieurs métriques complémentaires :

### Interprétation des métriques

- **MAE (Mean Absolute Error)** : mesure l’erreur moyenne entre les prix prédits et réels  
- **RMSE (Root Mean Squared Error)** : pénalise davantage les grandes erreurs  
- **R²** : indique la part de variance expliquée par le modèle  

Dans ce projet, le modèle Ridge atteint un R² de **0.719**, ce qui signifie qu’il explique environ 72% des variations du prix.

Des métriques de classification sont également utilisées après segmentation du prix (bas / moyen / élevé) :

- **Accuracy** : proportion de bonnes prédictions  
- **Precision / Recall** : qualité de la classification des segments  

Elles permettent d’évaluer la capacité du modèle à positionner correctement un bien sur le marché.

### Axes d'amélioration identifiés

- Enrichir le dataset avec des variables géographiques (quartier, ville)
- Tester d'autres algorithmes : XGBoost, LightGBM, CatBoost
- Appliquer une transformation logarithmique sur PRICE pour normaliser
  la distribution et réduire l'impact des valeurs extrêmes
- Augmenter le volume de données pour améliorer la généralisation

---

### Hyperparamètres testés (GridSearch)

Un GridSearch a été réalisé sur le modèle Gradient Boosting afin d’explorer différentes combinaisons d’hyperparamètres :

| Hyperparamètre      | Valeurs testées   |
|---------------------|------------------|
| n_estimators        | 100 / 200 / 300 |
| max_depth           | 5 / 10 / None   |
| min_samples_split   | 2 / 5 / 10      |
| max_features        | sqrt / log2     |

Malgré cette exploration, les performances n’ont pas été améliorées,
confirmant que l’optimisation des hyperparamètres ne compense pas
les limites du dataset.

---
## Version de production

Le modèle enregistré sous l'alias **production** dans le Snowflake Model Registry est :

**Ridge Regression (issu du Model Journey)**

### Performances

- **R²** : 0.719  
- **MAE** : 37 593  
- **RMSE** : 58 646  

---

### Pourquoi ce modèle ?

Contrairement aux attentes, les modèles plus complexes (Gradient Boosting, Random Forest, XGBoost) n'ont pas permis d'améliorer les performances.

De plus, l’optimisation via GridSearch sur Gradient Boosting a conduit à une **dégradation des résultats**.

Le modèle Ridge a donc été retenu car :

- il offre les meilleures performances globales  
- il est plus robuste sur un dataset limité  
- il capture efficacement les relations linéaires présentes dans les données  

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

## Interface de l'application Streamlit : ![Image 01-04-2026 à 11 11](https://github.com/user-attachments/assets/df08170a-e998-4f07-bb22-8e474454ffea)

## Résultat de l'estimation de l'application : ![Image 01-04-2026 à 11 11 (1)](https://github.com/user-attachments/assets/1525a1de-310e-41e8-a9c0-dd530a2889c2)


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

## Key Insights

- La qualité des données est plus importante que le choix du modèle
- La présence de doublons peut fortement biaiser les performances
- Les modèles simples peuvent être plus robustes sur des petits datasets
- L’optimisation (GridSearch) n’est pas toujours bénéfique

## Conclusion

Ce projet illustre un principe fondamental en Data Science :

> **La performance dépend avant tout des données disponibles.**

Malgré l’utilisation de modèles avancés, un modèle simple (Ridge)
s’avère ici plus performant, mettant en évidence les limites du dataset.

Ce pipeline constitue une base solide pour des améliorations futures
via l’enrichissement des données.
