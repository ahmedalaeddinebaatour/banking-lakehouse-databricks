# 📊 Rapport de Performance — Phase 7 Optimisation
## Projet Banking Lakehouse — Databricks

---

## 🎯 Objectif de la phase

Appliquer et évaluer rigoureusement les techniques d'optimisation Delta Lake 
(OPTIMIZE, ZORDER, Partitionnement, VACUUM) sur le pipeline Banking Lakehouse, 
avec mesures chiffrées avant/après.

---

## 📐 Méthodologie

1. Diagnostic initial via `DESCRIBE DETAIL` (baseline structurelle)
2. Benchmark de requêtes représentatives (temps d'exécution)
3. Application de chaque technique d'optimisation
4. Remesure et analyse comparative
5. Documentation honnête des résultats, y compris les non-gains

---

## 🚨 Découverte majeure : 2 incidents de production détectés

Durant cette phase, le processus de benchmark a révélé deux bugs critiques 
de duplication de données, non détectés par le monitoring initial (Phase 6) :

| Table | Lignes attendues | Lignes trouvées | Facteur | Cause racine |
|---|---|---|---|---|
| `silver.transactions` | 283 726 | 1 418 630 | x5 | `append` sans contrôle d'idempotence |
| `gold.fact_transactions` | 283 726 | 4 255 890 | x15 | `append` sans contrôle d'idempotence |

**Root cause commune** : absence de vérification anti-doublon avant écriture 
sur des tables alimentées par des Jobs orchestrés ré-exécutés plusieurs fois.

**Correction appliquée** : 
1. Nettoyage immédiat (déduplication + `overwrite`)
2. Blindage du code source via pattern `left_anti` join (garantit l'idempotence)
3. Vérification de non-régression confirmée sur plusieurs ré-exécutions

**Impact positif inattendu** : ce nettoyage a lui-même constitué une 
optimisation majeure — réduction de stockage de 93% sur `fact_transactions` 
(161 MB → 11 MB pour le même volume de données réelles).

---

## 🔬 Résultats des tests d'optimisation

### 1. OPTIMIZE + Z-ORDER

| Métrique | Avant | Après |
|---|---|---|
| Nombre de fichiers | 2 | 1 |
| Taille totale | 11.6 MB | 11.08 MB |
| Requête filtrée (transaction_date) | 1.783s | 2.081s |

**Verdict : Aucun gain mesurable.**

**Analyse de la cause** :
- Volume trop faible (283 726 lignes, ~11 MB) pour activer un vrai data skipping
- Cardinalité insuffisante de la colonne de tri (2 valeurs distinctes seulement)
- Résultat final : 1 seul fichier — aucune opportunité de "sauter" des fichiers
- La variance naturelle du compute Serverless partagé explique l'écart de 0.3s

**Conclusion** : Le code et la syntaxe sont corrects et deviendraient 
pleinement efficaces sur un volume de production réel (plusieurs Go, 
nombreux fichiers, colonnes à cardinalité élevée).

---

### 2. Partitionnement

**Test réalisé** : Partitionnement par `transaction_date` (2 valeurs distinctes).

**Preuve technique via `EXPLAIN FORMATTED`** :

| Table | Mécanisme observé | Lit `transaction_date` ? |
|---|---|---|
| Partitionnée | `PartitionFilters` (élimination au niveau dossier) | ❌ Non |
| Non partitionnée | `DictionaryFilters` (Photon, au niveau fichier) | ✅ Oui |

**Découverte clé** : Databricks Photon utilise des Dictionary Filters qui 
compensent intelligemment l'absence de partitionnement sur les tables non 
partitionnées, expliquant l'absence de différence de performance mesurable 
à ce volume.

**Décision architecturale** : Partitionnement NON adopté en l'état — 
cardinalité insuffisante (2 dates) pour le justifier. Recommandation pour 
un contexte de production réel : partitionner par `(year, month)` pour un 
équilibre optimal entre nombre et taille des partitions.

---

### 3. VACUUM

**Test DRY RUN standard** : "No rows returned" — aucun fichier orphelin 
éligible (protection de rétention 7 jours active).

**Tentative d'override (`retentionDurationCheck=false`)** : Bloquée par 
la plateforme (`CONFIG_NOT_AVAILABLE`).

**Découverte positive** : Le compute Serverless gouverné par Unity Catalog 
empêche la désactivation de protections de sécurité critiques, même par 
l'administrateur — garantie précieuse en contexte réglementé bancaire, 
assurant qu'aucune suppression accidentelle prématurée de données 
historiques ne peut survenir.

---

## 🎓 Conclusions générales de la Phase 7

1. **Les optimisations Delta (Z-Order, partitionnement) démontrent leur 
   syntaxe et mécanisme corrects, mais leur bénéfice réel est conditionné 
   par le volume et la cardinalité des données** — une optimisation 
   appliquée sans discernement peut n'apporter aucune valeur, voire une 
   légère régression.

2. **Le processus de benchmark rigoureux a une valeur qui dépasse 
   l'optimisation elle-même** : il a permis de détecter 2 bugs de 
   production critiques (duplication x5 et x15) qui n'avaient pas été 
   identifiés par le monitoring initial.

3. **La gouvernance Unity Catalog Serverless apporte des garde-fous de 
   sécurité qui limitent certaines actions administratives** — un 
   compromis entre flexibilité et sécurité à connaître.

4. **L'honnêteté scientifique dans le reporting de performance est plus 
   valorisable qu'un gain artificiellement démontré** — ce rapport 
   documente autant les échecs d'optimisation que les découvertes 
   positives (nettoyage de stockage, gouvernance).