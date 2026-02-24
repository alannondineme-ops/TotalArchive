# MODULE 10 — Intelligence Artificielle & Automatisation

## Projet : Système d'Archivage Numérique des Documents
**Version :** 1.0  
**Dépendances :** MODULE 01 — Architecture | MODULE 02 — Authentification | MODULE 03 — Taxonomie | MODULE 06 — Documents | MODULE 07 — OCR | MODULE 09 — Recherche  
**Statut :** Module innovation — différenciation majeure par rapport aux solutions existantes

---

## 1. Présentation du module

Ce module transforme le système d'archivage d'un simple entrepôt documentaire en un système **intelligent et prédictif**. L'intelligence artificielle n'est pas une couche cosmétique : elle automatise réellement des tâches chronophages, améliore la qualité des métadonnées, détecte des anomalies et apprend des comportements des archivistes pour s'améliorer continuellement.

Les fonctionnalités d'IA implémentées sont **pragmatiques et éprouvées**, pas expérimentales. Elles reposent sur des bibliothèques Python matures (spaCy, scikit-learn) et ne nécessitent pas d'infrastructure GPU complexe. L'IA fonctionne en mode **suggestion assistée** : elle propose, l'archiviste valide, et le système apprend de chaque validation.

### Fonctionnalités principales

1. **Classification automatique de documents** — suggérer la catégorie et le type
2. **Extraction d'entités nommées** — identifier automatiquement les personnes, organisations, lieux, dates, montants
3. **Suggestion de tags** — proposer des tags pertinents basés sur le contenu
4. **Détection de doublons par similarité de contenu** — au-delà du hash SHA-256
5. **Résumé automatique de documents** — générer un résumé de 2-3 phrases
6. **Suggestion de niveau de confidentialité** — basée sur le contenu détecté
7. **Scoring de qualité des métadonnées** — identifier les documents mal renseignés
8. **Prédiction de la durée de traitement** — estimer le temps de validation d'un workflow

---

## 2. Technologies et bibliothèques

| Bibliothèque | Usage | Rôle |
|---|---|---|
| **spaCy** | NLP (Natural Language Processing) | Extraction d'entités, analyse syntaxique, tokenization |
| **scikit-learn** | Machine Learning | Classification de texte, clustering, TF-IDF |
| **numpy** | Calcul numérique | Manipulation de vecteurs et matrices |
| **pandas** | Manipulation de données | Préparation des datasets d'entraînement |
| **joblib** | Sérialisation | Sauvegarde des modèles entraînés |
| **transformers (Hugging Face)** | NLP avancé (optionnel) | Résumé automatique via modèles pré-entraînés |

**Modèles spaCy utilisés :**
- `fr_core_news_md` : Modèle français moyen (taille ~43 MB)
- Entités reconnues : PERSON, ORG, LOC, DATE, MONEY, etc.

---

## 3. Cas d'utilisation — Vue d'ensemble

### 3.1 Classification automatique

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-IA-01 | Suggérer automatiquement la catégorie d'un document | Système (automatique) |
| UC-IA-02 | Suggérer automatiquement le type de document | Système (automatique) |
| UC-IA-03 | Valider ou corriger une suggestion de classification | Archiviste |
| UC-IA-04 | Entraîner le modèle de classification | Système (automatique) |
| UC-IA-05 | Consulter la précision du modèle de classification | Admin / Archiviste |

### 3.2 Extraction d'entités

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-IA-06 | Extraire automatiquement les entités nommées d'un document | Système (automatique) |
| UC-IA-07 | Pré-remplir les métadonnées avec les entités extraites | Système (automatique) |
| UC-IA-08 | Corriger une entité mal extraite | Archiviste |
| UC-IA-09 | Rechercher par entité (tous les documents mentionnant "Société X") | Tout utilisateur |

### 3.3 Suggestions et recommandations

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-IA-10 | Suggérer des tags pertinents basés sur le contenu | Système (automatique) |
| UC-IA-11 | Suggérer un niveau de confidentialité | Système (automatique) |
| UC-IA-12 | Générer un résumé automatique du document | Système (automatique) |
| UC-IA-13 | Suggérer des documents similaires | Système (automatique) |

### 3.4 Détection d'anomalies

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-IA-14 | Détecter les doublons par similarité de contenu | Système (automatique) |
| UC-IA-15 | Calculer le score de qualité des métadonnées | Système (automatique) |
| UC-IA-16 | Identifier les documents incohérents (contenu vs catégorie) | Système (automatique) |
| UC-IA-17 | Alerter sur un document potentiellement mal classé | Système (automatique) |

### 3.5 Apprentissage et amélioration

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-IA-18 | Apprendre des corrections de l'archiviste | Système (automatique) |
| UC-IA-19 | Réentraîner périodiquement les modèles | Système (tâche planifiée) |
| UC-IA-20 | Consulter les métriques de performance de l'IA | Admin / Archiviste |

---

## 4. Description détaillée des cas d'utilisation

### UC-IA-01 — Suggérer automatiquement la catégorie d'un document

**Acteur :** Système (déclenché automatiquement après OCR ou upload)  
**Préconditions :** Le document possède du contenu textuel (OCR ou fichier texte)  
**Postconditions :** Une ou plusieurs catégories sont suggérées avec un score de confiance

**Scénario principal :**
1. Le document est créé et son contenu OCR est disponible
2. Une tâche Celery de classification est déclenchée automatiquement
3. Le système extrait le texte complet du document (titre + contenu OCR + métadonnées)
4. Le système prétraite le texte :
   - Tokenization (découpage en mots)
   - Suppression des stop words français
   - Lemmatisation (réduction à la forme canonique)
5. Le système vectorise le texte via TF-IDF (Term Frequency-Inverse Document Frequency)
6. Le système applique le modèle de classification entraîné (RandomForestClassifier ou SVM)
7. Le modèle retourne les 3 catégories les plus probables avec leurs scores de confiance
8. Si le score de confiance de la première catégorie > 85%, le système l'applique automatiquement ET notifie l'archiviste pour validation
9. Si le score < 85%, le système stocke les suggestions sans les appliquer automatiquement
10. Les suggestions sont affichées à l'archiviste lors de la saisie des métadonnées

**Règles métier :**
- L'IA ne remplace **jamais** le choix de l'archiviste, elle suggère
- Si l'archiviste corrige une suggestion, le système enregistre la correction pour apprentissage futur
- Un score de confiance > 85% indique une forte probabilité de correction

---

### UC-IA-06 — Extraire automatiquement les entités nommées d'un document

**Acteur :** Système (déclenché automatiquement après OCR)  
**Préconditions :** Le document possède du contenu textuel  
**Postconditions :** Les entités sont extraites et stockées en base de données

**Scénario principal :**
1. Le contenu OCR du document est disponible
2. Une tâche Celery d'extraction d'entités est déclenchée
3. Le système charge le modèle spaCy français (`fr_core_news_md`)
4. Le système analyse le texte et extrait les entités suivantes :

**Types d'entités extraites :**

| Type d'entité | Code spaCy | Exemple | Usage |
|---|---|---|---|
| Personne | PERSON | "Jean Dupont", "Marie Martin" | Pré-remplir le champ "Auteur" ou "Partie contractante" |
| Organisation | ORG | "Ministère de la Santé", "Société ABC" | Pré-remplir "Service émetteur" ou "Partie contractante" |
| Lieu | LOC | "Brazzaville", "Congo", "Avenue de la Paix" | Métadonnées de localisation |
| Date | DATE | "15 janvier 2026", "2026-01-15" | Pré-remplir "Date du document" |
| Montant | MONEY | "1 500 €", "$ 2,000" | Métadonnées financières |
| Référence | MISC | "Contrat n° 2026-001" | Numéro de référence |

5. Le système calcule la fréquence de chaque entité dans le document
6. Le système stocke les entités dans une table dédiée avec leur type, fréquence et position
7. Le système pré-remplit automatiquement les métadonnées personnalisées correspondantes (si le schéma du type de document les définit)
8. L'archiviste peut valider, corriger ou compléter les entités extraites

**Exemple concret :**
> Texte : "Le contrat de travail entre la société ABC et Monsieur Jean Dupont, signé le 15 janvier 2026 à Paris, prévoit un salaire mensuel de 3 500 €."

**Entités extraites :**
- ORG : "société ABC"
- PERSON : "Jean Dupont"
- DATE : "15 janvier 2026"
- LOC : "Paris"
- MONEY : "3 500 €"

**Métadonnées pré-remplies :**
- `custom_metadata.parties` = ["société ABC", "Jean Dupont"]
- `document_date` = 2026-01-15
- `custom_metadata.montant` = 3500

---

### UC-IA-10 — Suggérer des tags pertinents basés sur le contenu

**Acteur :** Système  
**Préconditions :** Le document possède du contenu textuel  
**Postconditions :** Des tags pertinents sont suggérés

**Scénario principal :**
1. Le système analyse le contenu du document
2. Le système extrait les mots-clés les plus significatifs via TF-IDF
3. Le système compare ces mots-clés avec la base de tags existants
4. Le système calcule un score de similarité cosinus entre le contenu et chaque tag
5. Le système retourne les 5-10 tags les plus pertinents
6. Le système suggère également de nouveaux tags potentiels si aucun tag existant n'est pertinent
7. L'archiviste peut accepter, refuser ou modifier les suggestions

**Exemple :**
> Contenu : "Ce rapport présente les résultats financiers du trimestre, incluant l'analyse des ventes et des dépenses. Le chiffre d'affaires est en hausse de 12%."

**Tags suggérés :**
- `rapport` (existant, score 0.92)
- `financier` (existant, score 0.89)
- `trimestriel` (nouveau suggéré)
- `ventes` (existant, score 0.75)
- `analyse` (existant, score 0.70)

---

### UC-IA-12 — Générer un résumé automatique du document

**Acteur :** Système  
**Préconditions :** Le document possède un contenu suffisamment long (>500 mots)  
**Postconditions :** Un résumé de 2-3 phrases est généré

**Scénario principal :**
1. Le système analyse le contenu complet du document
2. Le système applique un algorithme de résumé extractif :
   - Calcul de la fréquence des termes (TF-IDF)
   - Scoring de chaque phrase selon l'importance de ses termes
   - Sélection des 2-3 phrases les plus représentatives
3. Le système génère le résumé
4. Le résumé est stocké dans les métadonnées et affiché en haut de la fiche document
5. L'archiviste peut modifier le résumé si nécessaire

**Exemple :**
> Document original (500 mots) : "Ce contrat de prestation de services..."

**Résumé généré :**
> "Contrat de prestation entre la société ABC et XYZ pour la réalisation d'une mission de conseil. La durée de la mission est fixée à 6 mois. Le montant total s'élève à 50 000 €."

---

### UC-IA-14 — Détecter les doublons par similarité de contenu

**Acteur :** Système  
**Préconditions :** Le document est créé et son contenu OCR est disponible  
**Postconditions :** Les doublons potentiels sont identifiés et signalés

**Scénario principal :**
1. Le document est créé
2. Le système calcule le hash SHA-256 (détection doublon exact — déjà fait dans MODULE 06)
3. Le système vectorise le contenu du document via TF-IDF
4. Le système recherche dans la base les documents de la même catégorie avec un vecteur similaire
5. Le système calcule la **similarité cosinus** entre le nouveau document et chaque document existant
6. Si similarité > 90% : doublon quasi-certain → alerte critique
7. Si similarité entre 70-89% : doublon probable → alerte modérée
8. Le système crée une entrée dans la table `ai_duplicate_detections` (MODULE 06 - DocumentDuplicate est enrichi)
9. L'archiviste est notifié et peut comparer les documents côte à côte

**Différence avec la détection de doublon du MODULE 06 :**
- MODULE 06 : détection par hash (fichiers strictement identiques)
- MODULE 10 : détection par similarité de contenu (documents différents mais très similaires, ex: même contrat avec adresse différente)

---

### UC-IA-15 — Calculer le score de qualité des métadonnées

**Acteur :** Système  
**Préconditions :** Le document existe avec ses métadonnées  
**Postconditions :** Un score de qualité (0-100) est calculé et stocké

**Scénario principal :**
1. Le système analyse les métadonnées du document
2. Le système calcule un score basé sur plusieurs critères :

**Critères du score de qualité :**

| Critère | Poids | Calcul |
|---|---|---|
| Complétude des champs obligatoires | 30% | (champs remplis / champs obligatoires) * 100 |
| Complétude des champs optionnels | 15% | (champs remplis / champs optionnels) * 100 |
| Qualité du titre | 15% | Longueur > 10 caractères et < 150 caractères |
| Présence de description | 10% | Description renseignée et > 50 caractères |
| Cohérence catégorie/contenu | 15% | IA vérifie si le contenu correspond à la catégorie |
| Présence de tags | 10% | Au moins 3 tags |
| Qualité OCR | 5% | Score OCR du MODULE 07 |

3. Le système calcule le score total (0-100)
4. Le système classe les documents par qualité :
   - 90-100 : Excellente qualité
   - 70-89 : Bonne qualité
   - 50-69 : Qualité moyenne (amélioration recommandée)
   - 0-49 : Qualité faible (révision obligatoire)
5. Les documents de qualité faible sont automatiquement signalés aux archivistes
6. Le score est mis à jour à chaque modification des métadonnées

---

### UC-IA-18 — Apprendre des corrections de l'archiviste

**Acteur :** Système (apprentissage continu)  
**Préconditions :** Un archiviste a corrigé une suggestion de l'IA  
**Postconditions :** Le système enregistre la correction pour améliorer les futures prédictions

**Scénario principal :**
1. L'IA suggère une catégorie "Juridique" pour un document
2. L'archiviste corrige et choisit "Financier"
3. Le système enregistre cette correction dans une table d'apprentissage :
   - Contenu du document (vectorisé)
   - Catégorie suggérée par l'IA
   - Catégorie choisie par l'archiviste (vérité terrain)
4. Lorsque suffisamment de corrections sont accumulées (seuil : 100 corrections), une tâche de réentraînement est déclenchée
5. Le modèle est réentraîné avec les nouvelles données
6. La précision du modèle s'améliore progressivement

**Règle d'apprentissage actif :**
- Si un archiviste corrige systématiquement la même erreur (ex: l'IA classe toujours "Contrat de prestation" en Juridique alors que c'est Financier), le système détecte le pattern et ajuste rapidement

---

## 5. Modèles de données

### 5.1 Modèle `AIClassificationSuggestion` (Suggestion de classification)

**Table :** `ai_classification_suggestions`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_id` | UUID | FK → documents.id, NOT NULL, INDEX | Document concerné |
| `suggestion_type` | VARCHAR(30) | NOT NULL | Type : `category`, `document_type`, `confidentiality` |
| `suggested_value_id` | UUID | NULL | ID de la catégorie/type suggéré |
| `suggested_value_text` | VARCHAR(200) | NULL | Texte de la suggestion (si pas d'ID) |
| `confidence_score` | DECIMAL(5,2) | NOT NULL | Score de confiance (0-100) |
| `model_version` | VARCHAR(20) | NOT NULL | Version du modèle utilisé |
| `was_accepted` | BOOLEAN | NULL | NULL = en attente, TRUE = acceptée, FALSE = rejetée |
| `actual_value_id` | UUID | NULL | Valeur réelle choisie par l'archiviste |
| `reviewed_by_id` | UUID | FK → users.id, NULL | Qui a validé/rejeté |
| `reviewed_at` | TIMESTAMP | NULL | Date de validation/rejet |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |

**Index :**
- `idx_ai_classification_suggestions_document_id` sur `document_id`
- `idx_ai_classification_suggestions_was_accepted` sur `was_accepted`

---

### 5.2 Modèle `AIExtractedEntity` (Entité extraite)

**Table :** `ai_extracted_entities`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_id` | UUID | FK → documents.id, NOT NULL, INDEX | Document |
| `entity_type` | VARCHAR(30) | NOT NULL | Type : `PERSON`, `ORG`, `LOC`, `DATE`, `MONEY`, `MISC` |
| `entity_text` | VARCHAR(300) | NOT NULL | Texte de l'entité |
| `normalized_text` | VARCHAR(300) | NULL | Forme normalisée (ex: date ISO) |
| `confidence_score` | DECIMAL(5,2) | NOT NULL | Confiance spaCy (0-100) |
| `frequency` | SMALLINT | NOT NULL, DEFAULT 1 | Nombre d'occurrences dans le document |
| `first_occurrence_position` | INTEGER | NULL | Position du premier caractère |
| `is_validated` | BOOLEAN | NOT NULL, DEFAULT FALSE | Validée par un archiviste |
| `validated_by_id` | UUID | FK → users.id, NULL | Qui a validé |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date d'extraction |

**Index :**
- `idx_ai_extracted_entities_document_id` sur `document_id`
- `idx_ai_extracted_entities_entity_type` sur `entity_type`
- `idx_ai_extracted_entities_normalized_text` sur `normalized_text` (pour recherche)

---

### 5.3 Modèle `AIModel` (Modèle d'IA entraîné)

**Table :** `ai_models`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `model_type` | VARCHAR(50) | NOT NULL | Type : `category_classifier`, `type_classifier`, `tag_suggester` |
| `version` | VARCHAR(20) | NOT NULL | Version (ex: v1.2.3) |
| `algorithm` | VARCHAR(50) | NOT NULL | Algorithme : `RandomForest`, `SVM`, `NaiveBayes` |
| `training_data_size` | INTEGER | NOT NULL | Nombre de documents d'entraînement |
| `accuracy` | DECIMAL(5,2) | NULL | Précision globale (0-100) |
| `f1_score` | DECIMAL(5,2) | NULL | Score F1 (0-100) |
| `model_file_path` | VARCHAR(500) | NOT NULL | Chemin du fichier .joblib |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT FALSE | Modèle actif en production |
| `trained_at` | TIMESTAMP | NOT NULL, AUTO | Date d'entraînement |
| `trained_by_id` | UUID | FK → users.id, NULL | Qui a lancé l'entraînement |
| `notes` | TEXT | NULL | Notes sur le modèle |

**Règle :** Un seul modèle peut être actif par type à la fois.

---

### 5.4 Modèle `AITrainingData` (Données d'entraînement)

**Table :** `ai_training_data`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_id` | UUID | FK → documents.id, NOT NULL, INDEX | Document source |
| `training_type` | VARCHAR(50) | NOT NULL | Type : `category_classification`, `type_classification` |
| `features` | TEXT | NOT NULL | Vecteur de features (texte vectorisé sérialisé) |
| `label` | VARCHAR(100) | NOT NULL | Label (catégorie, type...) |
| `is_correction` | BOOLEAN | NOT NULL, DEFAULT FALSE | Issu d'une correction manuelle |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |

---

### 5.5 Modèle `AISimilarityPair` (Paires de documents similaires)

**Table :** `ai_similarity_pairs`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_1_id` | UUID | FK → documents.id, NOT NULL | Premier document |
| `document_2_id` | UUID | FK → documents.id, NOT NULL | Second document |
| `similarity_score` | DECIMAL(5,2) | NOT NULL | Score de similarité cosinus (0-100) |
| `similarity_type` | VARCHAR(30) | NOT NULL | Type : `content`, `metadata`, `combined` |
| `is_duplicate` | BOOLEAN | NULL | NULL = à vérifier, TRUE = doublon confirmé, FALSE = pas doublon |
| `reviewed_by_id` | UUID | FK → users.id, NULL | Qui a vérifié |
| `reviewed_at` | TIMESTAMP | NULL | Date de vérification |
| `detected_at` | TIMESTAMP | NOT NULL, AUTO | Date de détection |

**Contrainte :** `UNIQUE (document_1_id, document_2_id)` avec `document_1_id < document_2_id` pour éviter les doublons inversés

---

### 5.6 Modèle `AIMetadataQualityScore` (Score de qualité métadonnées)

**Table :** `ai_metadata_quality_scores`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_id` | UUID | FK → documents.id, UNIQUE, NOT NULL | Document |
| `overall_score` | DECIMAL(5,2) | NOT NULL | Score global (0-100) |
| `completeness_score` | DECIMAL(5,2) | NOT NULL | Score de complétude |
| `coherence_score` | DECIMAL(5,2) | NOT NULL | Score de cohérence |
| `quality_level` | VARCHAR(20) | NOT NULL | Niveau : `excellent`, `good`, `average`, `poor` |
| `missing_fields` | ARRAY VARCHAR | NULL | Liste des champs manquants importants |
| `recommendations` | TEXT | NULL | Recommandations d'amélioration |
| `calculated_at` | TIMESTAMP | NOT NULL, AUTO | Date de calcul |

---

## 6. Matrice des permissions du module

| Permission code | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `ai.view_suggestions` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `ai.accept_suggestion` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `ai.train_model` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `ai.view_model_metrics` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `ai.manage_models` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `ai.view_extracted_entities` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `ai.validate_entity` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |

---

## 7. Endpoints API du module

### 7.1 Suggestions

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/ai/documents/{doc_id}/suggestions/` | Suggestions IA pour un document | Oui |
| POST | `/api/v1/ai/suggestions/{id}/accept/` | Accepter une suggestion | Oui + Permission |
| POST | `/api/v1/ai/suggestions/{id}/reject/` | Rejeter une suggestion | Oui + Permission |

### 7.2 Entités extraites

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/ai/documents/{doc_id}/entities/` | Entités extraites | Oui |
| POST | `/api/v1/ai/entities/{id}/validate/` | Valider une entité | Oui + Permission |
| PATCH | `/api/v1/ai/entities/{id}/` | Corriger une entité | Oui + Permission |
| GET | `/api/v1/ai/search/entities/` | Rechercher par entité | Oui |

### 7.3 Modèles (Admin)

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/ai/models/` | Liste des modèles | Oui + Permission |
| POST | `/api/v1/ai/models/train/` | Lancer entraînement | Oui + Permission |
| GET | `/api/v1/ai/models/{id}/metrics/` | Métriques d'un modèle | Oui + Permission |
| POST | `/api/v1/ai/models/{id}/activate/` | Activer un modèle | Oui + Permission |

### 7.4 Qualité et similarité

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/ai/documents/{doc_id}/quality-score/` | Score de qualité | Oui |
| GET | `/api/v1/ai/documents/{doc_id}/similar/` | Documents similaires | Oui |
| GET | `/api/v1/ai/duplicates/potential/` | Doublons potentiels à vérifier | Oui + Permission |

---

## 8. Tâches Celery du module

| Tâche | Fréquence | Description |
|---|---|---|
| `ai.classify_document` | On-demand | Classifier un document (catégorie + type) |
| `ai.extract_entities` | On-demand | Extraire entités nommées |
| `ai.suggest_tags` | On-demand | Suggérer tags |
| `ai.detect_similar_documents` | On-demand | Détecter doublons par similarité |
| `ai.calculate_quality_score` | On-demand | Calculer score qualité métadonnées |
| `ai.retrain_models` | Hebdomadaire (dimanche 2h) | Réentraîner modèles si assez de corrections |
| `ai.update_model_metrics` | Quotidien (1h) | Mettre à jour précision/rappel des modèles |
| `ai.cleanup_old_suggestions` | Mensuel (1er du mois) | Supprimer suggestions >6 mois |

---

## 9. Événements journalisés (MODULE 11 — Audit)

| Événement | Niveau | Détails |
|---|---|---|
| Suggestion IA acceptée | INFO | document_id, suggestion_type, accepted_value, user_id |
| Suggestion IA rejetée | INFO | document_id, suggestion_type, rejected_value, actual_value, user_id |
| Modèle entraîné | INFO | model_type, version, accuracy, training_size |
| Modèle activé | INFO | model_type, version, activated_by |
| Entité validée | INFO | document_id, entity_type, entity_text, validated_by |
| Doublon détecté par IA | WARNING | doc1_id, doc2_id, similarity_score |

---

## 10. Notifications générées (MODULE 12)

| Déclencheur | Destinataire | Canal | Message |
|---|---|---|
| Doublon potentiel détecté (>90%) | Archivistes | In-app | Doublon probable : [Doc1] et [Doc2] (similarité : [X]%) |
| Qualité métadonnées faible (<50%) | Propriétaire | In-app | La qualité des métadonnées de [Doc] est faible (score : [X]) |
| Suggestion haute confiance (>95%) | Archiviste | In-app | Classification suggérée avec haute confiance : [Catégorie] |
| Modèle réentraîné | Admins | In-app | Le modèle [Type] a été réentraîné (précision : [X]%) |

---

*Fin du MODULE 10 — Intelligence Artificielle & Automatisation*  
*Prochain module : MODULE 11 — Audit, Traçabilité & Journalisation*
