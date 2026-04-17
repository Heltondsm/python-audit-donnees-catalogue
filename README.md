# 🍷 Audit données catalogue — E-commerce vins

![Python](https://img.shields.io/badge/Python-3776AB?style=flat-square&logo=python&logoColor=white)
![Pandas](https://img.shields.io/badge/Pandas-150458?style=flat-square&logo=pandas&logoColor=white)
![Plotly](https://img.shields.io/badge/Plotly-3F4F75?style=flat-square&logo=plotly&logoColor=white)
![Seaborn](https://img.shields.io/badge/Seaborn-3776AB?style=flat-square&logo=python&logoColor=white)
![Jupyter](https://img.shields.io/badge/Jupyter-F37626?style=flat-square&logo=jupyter&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-22c55e?style=flat-square)

3 fichiers sources. 9 anomalies. 712 produits réconciliés. Une boutique de vins gérait son stock dans un ERP et son catalogue en ligne dans WooCommerce — **sans jamais vérifier que les deux systèmes se parlaient.**

---

## 📖 Contexte

Une boutique de vins en ligne utilise deux systèmes distincts : un **ERP interne** (stocks, prix d'achat, références produits) et un **catalogue WEB WooCommerce** (fiches produits, données de vente). Ces deux sources ne partagent pas de clé commune directe — une table de liaison assure la correspondance via deux identifiants séparés.

**La question :** les données sont-elles fiables ? Où sont les incohérences, et quel est leur impact financier réel ?

---

## 🎯 Objectif de l'analyse

À partir des trois fichiers sources, l'objectif est de :
- Identifier toutes les anomalies dans chaque source indépendamment
- Réconcilier les trois fichiers via la jointure ERP × Liaison × WEB
- Analyser le CA, les stocks et les marges sur le mois d'octobre
- Formuler des recommandations priorisées à partir des résultats

---

## 🔍 Résultats clés

### 1️⃣ 9 anomalies identifiées dans les données brutes

Les données ne sont pas fiables tel quel. Les anomalies sont réparties sur les deux sources :

| # | Anomalie | Source | Traitement |
|---|---|---|---|
| 1 | 3 prix négatifs (product_id 4233, 5017, 6594) | ERP | Suppression |
| 2 | 2 stocks négatifs (product_id 4973, 5700) | ERP | Remis à 0 |
| 3 | 2 incohérences stock_status / stock_quantity | ERP | Correction |
| 4 | 88 produits ERP sans correspondance WEB | Liaison | Signalement |
| 5 | 2 SKU non conformes (`bon-cadeau-25-euros`, `13127-1`) | WEB | Signalement |
| 6 | 85 lignes sans SKU (dont 83 totalement vides) | WEB | Suppression |
| 7 | 2 ventes négatives dans les lignes sans SKU | WEB | Conservées |
| 8 | 4 colonnes 100% vides | WEB | Suppression |
| 9 | 714 lignes "attachment" (médias orphelins) | WEB | Exclues |

```python
# Détection des prix négatifs dans l'ERP
print("Prix minimum :", df_erp["price"].min())
print("\nArticles avec prix négatif :")
df_erp[df_erp["price"] < 0][["product_id", "price"]]

# Résultats :
# Prix minimum : -20.0
# 3 articles avec prix négatif → supprimés du dataset
```

### 2️⃣ 88 produits actifs en stock sans visibilité en ligne

88 références présentes dans l'ERP avec un stock disponible n'ont aucune correspondance dans le catalogue WEB. Ces produits ne sont pas commandables en ligne — du cash immobilisé sans aucune visibilité.

```python
# Identification des produits ERP sans correspondance WEB
sans_correspondance = df_liaison[df_liaison["id_web"].isnull()]
print(f"Articles sans correspondance web : {len(sans_correspondance)}")
# Résultat : 88 articles
```

### 3️⃣ 1 produit vendu à -549% de marge

Parmi les 712 produits réconciliés, un article est vendu en dessous de son prix d'achat.

**Champagne Egly-Ouriet Grand Cru Blanc de Noirs**
- Prix de vente TTC : **12,65 €**
- Prix d'achat : **77,48 €**
- Taux de marge : **-549 %**

Probable erreur de saisie dans l'ERP — le prix de vente réel de ce champagne grand cru est bien supérieur.

```python
# Calcul du taux de marge pour chaque produit
df_merge["prix_ht"] = df_merge["price"] / 1.2
df_merge["taux_marge"] = (
    df_merge["prix_ht"] - df_merge["purchase_price"]
) / df_merge["purchase_price"]

# Identification des articles à marge négative
marge_negative = df_merge[df_merge["taux_marge"] < 0]
print(f"Articles vendus à perte : {len(marge_negative)}")
marge_negative[["post_title", "price", "purchase_price", "taux_marge"]]

# Résultat : 1 article — Champagne Egly-Ouriet, taux de marge -549%
```

### 4️⃣ 419 références génèrent 80% du CA — Loi de Pareto vérifiée

Sur 712 produits référencés, seulement 419 (58,8%) génèrent 80% du chiffre d'affaires d'octobre. Le même phénomène se vérifie côté quantités : 423 références représentent 80% des unités vendues.

**CA total octobre : 153 353,90 €**

```python
# Calcul du CA par article
df_merge["ca_par_article"] = df_merge["price"] * df_merge["total_sales"]
ca_total = df_merge["ca_par_article"].sum()
print(f"CA total : {ca_total:,.2f} €")  # 153 353,90 €

# Tri décroissant + cumul pour le Pareto
df_top_ca = df_merge.sort_values("ca_par_article", ascending=False).reset_index(drop=True)
df_top_ca["part_ca"] = df_top_ca["ca_par_article"] / ca_total
df_top_ca["cumul_ca"] = df_top_ca["part_ca"].cumsum()

articles_80 = df_top_ca[df_top_ca["cumul_ca"] <= 0.8]
print(f"Articles représentant 80% du CA : {len(articles_80)}")
print(f"Proportion du catalogue : {len(articles_80) / len(df_merge) * 100:.1f}%")
# Résultat : 419 articles → 58,8% du catalogue
```

### 5️⃣ 276 859€ immobilisés en stocks

La valorisation des stocks (prix de vente × quantités en stock) atteint **276 859,09 €** — soit environ **1,8× le CA mensuel**.

16 711 unités en stock. Certains articles présentent une rotation quasi nulle : des bouteilles stockées en grande quantité sans ventes enregistrées sur la période.

```python
# Valorisation des stocks
df_merge["valorisation_stock"] = (
    df_merge["stock_quantity"] * df_merge["purchase_price"]
)
print(f"Valorisation totale : {df_merge['valorisation_stock'].sum():,.2f} €")
print(f"Unités en stock : {df_merge['stock_quantity'].sum()}")
# Résultat : 276 859,09 € | 16 711 unités
```

---

## 💡 Conclusion

> **Les données ne sont pas fiables dans leur état brut. Sur 3 sources, chacune présente des anomalies qui biaisent toute analyse directe.**

**Les 3 problèmes prioritaires :**
1. Des prix incorrects dans l'ERP (négatifs, ou manifestement faux comme le champagne à 12,65 €) faussent le CA et la valorisation des stocks
2. 88 références actives en stock sont absentes du catalogue en ligne — du chiffre d'affaires potentiel invisible
3. La valorisation des stocks (276 K€) représente 1,8× le CA mensuel — la rotation est trop lente sur une partie du catalogue

---

## 📋 Recommandations

### 🚨 Court terme (0-4 semaines)

**1. Corriger les prix dans l'ERP**
- 3 prix négatifs à corriger manuellement (product_id 4233, 5017, 6594)
- 1 prix manifestement faux → vérifier avec l'équipe achat (Champagne Egly-Ouriet)
- Impact direct : CA recalculé + valorisation stock fiable

### 🎯 Moyen terme (1-3 mois)

**2. Synchroniser l'ERP et le catalogue WEB**
- Créer les 88 fiches produits manquantes sur WooCommerce
- Ou décider consciemment de ne pas les mettre en ligne (et identifier pourquoi)
- Établir un process de synchronisation ERP → WEB à chaque nouvelle référence

**3. Prioriser les 419 références Pareto**
- Concentrer les efforts de vérification de stock sur ces 419 articles
- Ce sont eux qui portent le CA — toute rupture sur ces références a un impact immédiat

### 🌱 Long terme (3-12 mois)

**4. Traiter les stocks à rotation lente**
- Identifier les articles avec stock > 12 mois de ventes → actions commerciales (promotions, déréférencement)
- La valorisation à 276 K€ pour 153 K€ de CA mensuel indique un surstockage structurel sur certaines références

**5. Mettre en place un monitoring qualité données**
- Contrôle automatique des prix négatifs, SKU non conformes et ruptures de correspondance ERP/WEB
- Un outil simple (script Python hebdomadaire) éviterait de découvrir ces anomalies lors d'un audit annuel

---

## 🛠️ Technologies utilisées

- **Python 3.9+** : langage de programmation
- **Pandas** : manipulation, nettoyage et jointure des données
- **Plotly Express** : visualisations interactives (boxplots)
- **Seaborn / Matplotlib** : heatmap de corrélations
- **Jupyter Notebook** : environnement d'analyse et documentation

---

## 📂 Structure du projet

```
.
├── README.md                  # Documentation du projet
├── audit_catalogue.ipynb      # Notebook Jupyter complet
└── data/
    ├── erp.xlsx               # 825 lignes — stocks, prix d'achat, références
    ├── web.xlsx               # 1 513 lignes — catalogue WooCommerce (ventes, métadonnées)
    └── liaison.xlsx           # 825 lignes — table de correspondance ERP ↔ WEB
```

---

## 🚀 Installation et utilisation

### Prérequis

```bash
Python 3.9+
```

### Installation des dépendances

```bash
pip install pandas plotly seaborn matplotlib openpyxl jupyter
```

### Lancer le notebook

```bash
git clone https://github.com/Heltondsm/python-audit-donnees-catalogue.git
cd python-audit-donnees-catalogue
jupyter notebook audit_catalogue.ipynb
```

---

## 📊 Aperçu du code

### Jointure des trois sources

```python
# Étape 1 : ERP × Liaison (clé : product_id)
df_merge = pd.merge(df_erp, df_liaison, on="product_id", how="left")
print(f"Lignes après jointure ERP + Liaison : {len(df_merge)}")

# Étape 2 : résultat × WEB (clé : id_web / sku)
df_merge = pd.merge(df_merge, df_web, left_on="id_web", right_on="sku", how="inner")
print(f"Produits réconciliés : {len(df_merge)}")
# Résultat : 712 produits avec données complètes
```

### Détection des prix atypiques — méthode IQR

```python
# Calcul du seuil outliers par l'intervalle interquartile
Q1 = df_merge["price"].quantile(0.25)
Q3 = df_merge["price"].quantile(0.75)
IQR = Q3 - Q1
seuil_haut = Q3 + 1.5 * IQR

print(f"Seuil outliers (Q3 + 1.5×IQR) : {seuil_haut:.2f} €")
# Résultat : 84,01 €

outliers_iqr = df_merge[df_merge["price"] > seuil_haut]
print(f"Articles outliers : {len(outliers_iqr)} ({len(outliers_iqr)/len(df_merge)*100:.1f}%)")
# Résultat : 31 articles (4,4%) — champagnes millésimés et grands crus
```

### Analyse Pareto — concentration du CA

```python
# Tri par CA décroissant + cumul
df_top_ca = df_merge.sort_values("ca_par_article", ascending=False).reset_index(drop=True)
df_top_ca["part_ca"] = df_top_ca["ca_par_article"] / ca_total
df_top_ca["cumul_ca"] = df_top_ca["part_ca"].cumsum()

# Articles représentant les premiers 80% du CA
articles_80 = df_top_ca[df_top_ca["cumul_ca"] <= 0.8]
print(f"Articles pour 80% du CA : {len(articles_80)} sur {len(df_merge)}")
print(f"Proportion : {len(articles_80)/len(df_merge)*100:.1f}% du catalogue")
# Résultat : 419 articles → 58,8% du catalogue porte 80% du CA
```

---

## 📈 Compétences démontrées

### Techniques Data
- ✅ Nettoyage et correction de données multi-sources (ERP + WEB + Liaison)
- ✅ Jointures Pandas avec clés non directes (triple jointure via table intermédiaire)
- ✅ Détection d'outliers par méthode statistique (IQR) et z-score
- ✅ Analyse Pareto (cumsum sur CA et quantités)
- ✅ Calcul de taux de marge, valorisation de stocks, corrélations multivariées
- ✅ Visualisations interactives (Plotly boxplot) et heatmap de corrélations (Seaborn)

### Business acumen
- ✅ Traduction d'une question métier en pipeline d'analyse reproductible
- ✅ Quantification de l'impact financier de chaque anomalie
- ✅ Identification d'un produit vendu à perte malgré un catalogue de 700+ références
- ✅ Recommandations priorisées et actionnables (court / moyen / long terme)
- ✅ Communication structurée des résultats (présentation CODIR)

---

## 📧 Contact

**Helton Dos Santos Moreira**
Data Analyst | 10 ans d'expérience Business (retail + e-commerce) → Reconversion Data

- 📧 Email : heltonmail8@gmail.com
- 💼 LinkedIn : [in/helton-dsm-data](https://linkedin.com/in/helton-dsm-data)
- 🐙 GitHub : [Heltondsm](https://github.com/Heltondsm)

---

## 🔗 Autres projets

- [Étude sous-nutrition mondiale — FAO](https://github.com/Heltondsm/etude-sante-publique-fao) — 4 datasets ONU, paradoxe production/répartition, recommandations politiques
- [Exploration SQL — Portefeuille assurances habitation](https://github.com/Heltondsm/sql-assurances-habitation) — 50K+ contrats, segmentation géographique, opportunités de croissance
- [Performance e-commerce — Prévision SARIMA](https://github.com/Heltondsm/ecommerce-sales-analysis-sarima) — Séries temporelles, grid search sur 64 modèles, RMSE ±12%

---

**Projet réalisé en avril 2026**
