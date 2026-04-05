# Data Engineering & Machine Learning avec Snowflake

## Table des matières
1. Vue d'ensemble du projet
2. Environnement technique
3. Dataset
4. Pipeline
5. Résultats des modèles
6. Optimisation par GridSearchCV
7. Conclusions
8. Structure du dépôt

---

## 1. Vue d'ensemble du projet

Ce projet implémente un pipeline Machine Learning complet et natif dans Snowflake, de l'ingestion des données brutes depuis S3 jusqu'à l'exposition du modèle via une application Streamlit, sans exporter aucune donnée vers un système externe.

**Objectif :** prédire le prix de vente d'une maison à partir de ses caractéristiques structurelles et de confort, en utilisant Snowpark, Snowflake ML et le Model Registry.

**Source des données :** s3://logbrain-datalake/datasets/house_price/Housing_Price_Data.json (bucket public, sans credentials)

---

## 2. Environnement technique

| Composant | Valeur |
|---|---|
| Snowflake Database | SNOWFLAKE_LEARNING_DB |
| Schema | PUBLIC |
| Warehouse | COMPUTE_WH |
| pandas | 2.3.1 |
| numpy | 2.2.5 |
| scikit-learn | 1.7.2 |
| xgboost | 3.2.0 |
| matplotlib | 3.10.8 |
| seaborn | 0.13.2 |

---

## 3. Dataset

Le dataset contient 1 090 lignes et 13 colonnes décrivant des biens immobiliers et leur prix de vente.

### Variables

| Feature | Type | Description |
|---|---|---|
| PRICE | Numérique | Cible : prix de vente (87 500 à 665 000) |
| AREA | Numérique | Surface totale en m² (33 à 324, moy. 102,6) |
| BEDROOMS | Numérique | Nombre de chambres (1 à 6) |
| BATHROOMS | Numérique | Nombre de salles de bain (1 à 4) |
| STORIES | Numérique | Nombre d'étages (1 à 4) |
| PARKING | Numérique | Places de parking (0 à 3) |
| MAINROAD | Binaire | Route principale : yes 83%, no 17% |
| GUESTROOM | Binaire | Chambre d'amis : no 82%, yes 18% |
| BASEMENT | Binaire | Sous-sol : no 64%, yes 36% |
| HOTWATERHEATING | Binaire | Chauffage eau chaude : no 95%, yes 5% |
| AIRCONDITIONING | Binaire | Climatisation : no 66%, yes 34% |
| PREFAREA | Binaire | Zone privilégiée : no 76%, yes 24% |
| FURNISHINGSTATUS | Catégorielle | semi-furnished 39%, unfurnished 34%, furnished 27% |

**Qualité des données :** 0 valeur manquante sur l'ensemble des 13 colonnes, aucune imputation nécessaire.

---

## 4. Pipeline

Le pipeline suit 22 étapes organisées autour de 6 grandes phases :

**Ingestion :** chargement du fichier JSON depuis S3 dans Snowflake via un stage externe, puis transformation en table relationnelle typée. 1 090 lignes chargées avec succès.

**Exploration (EDA) :** analyse de la distribution des prix, matrice de corrélations entre variables numériques, et visualisation des distributions catégorielles. La distribution de PRICE est asymétrique à droite, avec une médiane autour de 220 000 et quelques outliers dépassant 500 000.

**Préparation :** encodage des variables catégorielles en valeurs numériques, normalisation des features numériques, et division du dataset en jeu d'entraînement (80% soit 872 lignes) et jeu de test (20% soit 218 lignes).

**Entraînement :** trois modèles de régression comparés : Régression Linéaire, Random Forest et XGBoost.

**Optimisation :** recherche des meilleurs hyperparamètres pour XGBoost via GridSearchCV (3-fold cross-validation, 24 configurations testées).

**Déploiement :** enregistrement du pipeline final dans le Snowflake Model Registry et exposition via une application Streamlit.

---

## 5. Résultats des modèles

### Comparaison baseline (test set sur 218 observations)

| Modèle | MAE | RMSE | R² |
|---|---|---|---|
| Linear Regression | 40 253 | 53 985 | 0.6732 |
| Random Forest | 19 587 | 32 513 | 0.8815 |
| XGBoost | 12 757 | 27 517 | 0.9151 |

XGBoost domine sur les trois métriques. Avec un R² de 0.9151, il explique 91,5% de la variance du prix sur le jeu de test, et une erreur absolue moyenne de seulement 12 757, soit environ 5% du prix médian.

### Corrélations avec le prix

| Feature | Corrélation | Interprétation |
|---|---|---|
| AREA | 0.55 | Plus fort prédicteur numérique |
| BATHROOMS | 0.53 | Reflète la qualité globale du bien |
| STORIES | 0.44 | Les maisons multi-étages sont plus chères |
| PARKING | 0.38 | Les places de parking ajoutent de la valeur |
| BEDROOMS | 0.35 | Corrélation positive modérée |

### Predicted vs Actual

Le nuage de points prédit/réel sur les 218 observations du test set montre des points très proches de la diagonale d'alignement parfait, sur l'ensemble de la plage de prix (100 000 à 615 000). Une légère dispersion est observable en milieu de gamme, ce qui est typique pour des données immobilières hétérogènes.

---

## 6. Optimisation par GridSearchCV

Une recherche automatique des meilleurs paramètres a été lancée sur un pipeline complet (StandardScaler + XGBoost) avec validation croisée à 3 folds.

### Meilleurs paramètres trouvés

| Paramètre | Valeur optimale |
|---|---|
| learning_rate | 0.1 |
| max_depth | 5 |
| n_estimators | 200 |
| Meilleur R² (CV) | 0.8286 |

### Comparaison avant/après optimisation

| Modèle | MAE | RMSE | R² |
|---|---|---|---|
| XGBoost baseline | 12 757 | 27 517 | 0.9151 |
| XGBoost optimisé (pipeline) | 18 054 | 29 531 | 0.9022 |
| Écart | | | -1.29% |

La légère baisse après optimisation est attendue et reflète une meilleure généralisation. Le pipeline optimisé a été évalué par validation croisée, une méthode plus rigoureuse qui évite le surapprentissage. C'est ce modèle qui a été choisi pour la production, car il est auto-suffisant : le scaler est embarqué, rendant l'inférence sur données brutes sûre et reproductible.

---

## 7. Conclusions

### Ce qui détermine le prix d'une maison

La surface est le facteur dominant : chaque m² supplémentaire augmente le prix de manière significative. Le nombre de salles de bain et le nombre d'étages reflètent la qualité globale du bien. La zone privilégiée ajoute une prime de localisation importante. La climatisation et le statut d'ameublement ont un impact positif mesurable. Le chauffage à eau chaude est le facteur le moins discriminant, car seulement 5% des biens en sont équipés.

### Conclusions sur les modèles

XGBoost est le meilleur modèle pour ce dataset : il capture des relations entre les variables que les modèles plus simples ne voient pas (R² de 0.9151 contre 0.6732 pour la régression linéaire). Le pipeline optimisé (R² de 0.9022) est prêt pour la production, auto-suffisant, versionné et enregistré avec ses métriques. La légère baisse post-optimisation est attendue et signe d'un modèle qui généralisera mieux sur de nouvelles données.

---

## 8. Structure du dépôt

```
.
├── Projet DATA ENGINEER MLOPS.ipynb   # Notebook principal
├── environment.yml                    # Fichier d'environnement
└── README.md                          # Ce fichier
```

---

## Auteurs

ABED MERAIM Abdelbassit  
EL ABASS Sidi Mohamed
