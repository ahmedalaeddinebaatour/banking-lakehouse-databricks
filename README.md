# 🏦 Banking Lakehouse — Databricks End-to-End Data Engineering Project

![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=flat&logo=databricks&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta%20Lake-00ADD8?style=flat&logo=delta&logoColor=white)
![PySpark](https://img.shields.io/badge/PySpark-E25A1C?style=flat&logo=apachespark&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-4479A1?style=flat&logo=postgresql&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)

## 📖 Vue d'ensemble

Projet Data Engineering **End-to-End** reproduisant une architecture **Lakehouse bancaire réelle**, construit sur **Databricks Free Edition**, avec **Architecture Medallion** (Bronze/Silver/Gold), **Delta Lake**, **Unity Catalog**, orchestration via **Databricks Workflows**, et **Databricks Asset Bundles** pour le déploiement CI/CD.

> 🎯 Ce projet ne se contente pas de "faire fonctionner" un pipeline — il documente honnêtement les **échecs, incidents réels détectés et corrigés**, et les **limites mesurées** de chaque technique d'optimisation, dans une démarche d'ingénierie rigoureuse.

---

## 🗂️ Sources de données (100% réelles)

| Dataset | Source | Volume | Usage |
|---|---|---|---|
| **Bank Customer Churn** | Kaggle (`Churn_Modelling.csv`) | 10 003 clients | Dimension client, SCD Type 2 |
| **Credit Card Fraud Detection** | Kaggle (ULB Machine Learning Group) | 283 726 transactions | Table de faits, détection de fraude |

---

## 🏗️ Architecture Lakehouse

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            SOURCES DE DONNÉES                            │
│         Clients (CSV Kaggle)      |      Transactions (CSV Kaggle)       │
└───────────────────────────────┬────────────────────────────────────────┘
                                 │  Auto Loader (Incremental Load)
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  🥉 BRONZE — banking_lakehouse.bronze                                   │
│  clients_raw | transactions_raw                                          │
│  Schema Evolution démontrée (mergeSchema, retry automatique)            │
└───────────────────────────────┬────────────────────────────────────────┘
                                 │  Data Quality + CDC (MERGE INTO)
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  🥈 SILVER — banking_lakehouse.silver                                   │
│  clients (MERGE/upsert) | transactions (append-only + idempotence)      │
│  Flags qualité non-destructifs | Déduplication déterministe (SHA-256)   │
└───────────────────────────────┬────────────────────────────────────────┘
                                 │  Modélisation dimensionnelle
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  🥇 GOLD — banking_lakehouse.gold                                       │
│  dim_client (SCD Type 2) | dim_date | fact_transactions                 │
│  Star Schema | Window Functions | KPIs métier (fraude, churn)           │
└───────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
                    Databricks SQL Dashboards & Alerts

        ORCHESTRATION : Databricks Workflows (DAG 6 tâches, parallélisme)
        CI/CD          : Databricks Asset Bundles (databricks.yml)
        MONITORING     : System Tables + SQL Alerts + Data Quality tracking
        GOUVERNANCE    : Unity Catalog (Catalog > Schema > Table/Volume)
```

### 🔀 Séparation en 2 Data Domains (pattern professionnel réel)

Le projet reproduit une architecture bancaire réelle où les données **CRM/Risk** (avec PII) et les données de **Fraud Detection** (anonymisées pour conformité RGPD/PCI-DSS) sont gérées comme deux domaines distincts, reliés uniquement par une dimension `dim_date` partagée — reflétant une vraie séparation organisationnelle observée en entreprise.

---

## 🛠️ Stack technique

| Catégorie | Technologies |
|---|---|
| **Plateforme** | Databricks Free Edition (Serverless) |
| **Stockage** | Delta Lake, Unity Catalog Volumes |
| **Traitement** | PySpark, Spark SQL, Structured Streaming (Auto Loader) |
| **Orchestration** | Databricks Workflows, Databricks Asset Bundles (IaC) |
| **CDC** | MERGE INTO manuel + Lakeflow AUTO CDC (comparatif) |
| **Qualité** | Flags DQ custom, System Tables monitoring |
| **Versioning** | Git / GitHub (Databricks Repos) |

---

## 📂 Structure du repository

```
banking-lakehouse-databricks/
├── databricks.yml                    # Asset Bundle config
├── resources/
│   └── banking_lakehouse_pipeline.yml
├── 00_setup/
│   └── 00_create_catalog_schemas
├── 01_ingestion/
│   └── 02_ingest_bronze_autoloader
├── 02_silver/
│   ├── 03_transform_silver_clients
│   └── 04_transform_silver_transactions
├── 03_gold/
│   ├── 05_build_dim_date
│   ├── 06_build_dim_client_scd2
│   ├── 07_build_fact_transactions
│   └── 08_gold_kpis_analytics
├── 06_monitoring/
│   └── 09_pipeline_health_monitoring
├── 07_optimization/
│   └── RAPPORT_PERFORMANCE_PHASE7.md
└── banking_lakehouse_autocdc/         # Pipeline Lakeflow (Phase 3.B)
    └── transformations/dim_client_autocdc.py
```

---

## 🚨 Incidents de production détectés et résolus

Ce projet documente **deux incidents réels** découverts via monitoring, avec méthodologie complète Détection → Diagnostic → Correction → Prévention :

| Table | Bug | Facteur | Root Cause | Correction |
|---|---|---|---|---|
| `silver.transactions` | Duplication | **x5** | `append` sans idempotence sur ré-exécutions de Job | Anti-join (`left_anti`) sur clé technique |
| `gold.fact_transactions` | Duplication | **x15** | Même pattern | Même correction généralisée |

➡️ Détail complet dans [`RAPPORT_PERFORMANCE_PHASE7.md`](./07_optimization/RAPPORT_PERFORMANCE_PHASE7.md)

---

## 📊 Résultats et insights métier obtenus

- **Fraude** : montant moyen des transactions frauduleuses **+40%** vs transactions normales
- **Pattern horaire** : pic de fraude relatif entre **2h-7h du matin** (jusqu'à **9x** le taux moyen)
- **Churn** : l'Allemagne présente un taux de churn **2x supérieur** (32.4% vs ~16%) malgré un profil financier plus solide — insight actionnable pour une investigation qualitative

---

## 🔄 CDC : Manuel vs Déclaratif (comparaison réalisée)

| | SCD2 Manuel (`MERGE INTO`) | Lakeflow AUTO CDC |
|---|---|---|
| Lignes de code | ~80 | **8** |
| Contrôle | Total | Standardisé |
| Cas d'usage | Apprentissage approfondi, cas custom | Production moderne, maintenabilité |

---

## ⚙️ CI/CD — Databricks Asset Bundles

```bash
databricks bundle validate    # Validation de la configuration
databricks bundle deploy      # Déploiement Infrastructure as Code
```

Le pipeline complet (`banking_lakehouse_pipeline`) est défini en YAML versionné, avec DAG de dépendances (parallélisme sur les tâches indépendantes), déployable de façon reproductible sur n'importe quel environnement.

---

## 🎓 Compétences démontrées

- Architecture Lakehouse Medallion complète (Bronze/Silver/Gold)
- Auto Loader, Schema Evolution, Incremental Load
- CDC manuel (MERGE INTO) et déclaratif (Lakeflow AUTO CDC)
- SCD Type 2 avec preuve d'historisation temporelle
- Data Quality Engineering (flags non-destructifs)
- Orchestration DAG avec Databricks Workflows
- Infrastructure as Code (Databricks Asset Bundles)
- Monitoring (System Tables, SQL Alerts)
- Root Cause Analysis d'incidents de production réels
- Optimisation Delta Lake (OPTIMIZE, ZORDER, Partitioning, VACUUM) avec analyse rigoureuse avant/après

---

## 👤 Auteur

**Ahmed Alaeddine Baatour** — Data Engineer
[GitHub](https://github.com/ahmedalaeddinebaatour) 

---

## 📄 Licence

MIT License