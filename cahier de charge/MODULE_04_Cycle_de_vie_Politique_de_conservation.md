# MODULE 04 — Cycle de Vie & Politique de Conservation

## Projet : Système d'Archivage Numérique des Documents
**Version :** 1.0  
**Dépendances :** MODULE 01 — Architecture | MODULE 02 — Authentification | MODULE 03 — Taxonomie  
**Statut :** Référentiel de conformité — le MODULE 06 (Documents) dépend directement de ce module

---

## 1. Présentation du module

Le cycle de vie documentaire est le principe fondamental de l'archivistique moderne. Chaque document traverse plusieurs phases d'existence, depuis sa création jusqu'à son élimination ou son archivage définitif à valeur patrimoniale. Ces phases sont juridiquement encadrées : chaque type de document administratif possède une durée légale de conservation minimale qu'il est interdit de violer.

Ce module définit les statuts du cycle de vie, les politiques de conservation par type de document, les règles d'élimination et les procédures de versement aux archives nationales. Il garantit la conformité légale de l'organisation et automatise les alertes d'expiration documentaire.

Sans ce module, l'organisation accumule indéfiniment tous les documents sans jamais pouvoir les éliminer légalement, ce qui conduit à une saturation du stockage et à une impossibilité de gérer les obligations légales de conservation.

---

## 2. Principes du cycle de vie documentaire

### 2.1 Les trois âges des archives

La théorie archivistique définit trois âges pour un document :

**1. Archives courantes (Actif)**
> Documents nécessaires à l'activité quotidienne de l'administration. Consultation fréquente. Stockés dans les bureaux ou systèmes actifs.

**2. Archives intermédiaires (Semi-actif)**
> Documents dont l'utilité administrative diminue mais qui doivent être conservés pour des raisons juridiques, fiscales ou probatoires. Consultation rare. Peuvent être stockés dans un espace de préarchivage.

**3. Archives définitives (Inactif → Archivé définitivement)**
> Documents qui ont perdu toute utilité administrative mais possèdent une valeur historique, patrimoniale ou de recherche. Conservation permanente. Versement aux archives nationales ou destruction selon la décision de tri.

---

### 2.2 Statuts du cycle de vie dans l'application

| Statut | Code | Description | Transitions autorisées |
|---|---|---|---|
| **Actif** | `active` | Document en usage courant | → Semi-actif |
| **Semi-actif** | `semi_active` | Document de référence occasionnelle | → Inactif |
| **Inactif** | `inactive` | Document sans utilité administrative | → À éliminer / Archivé définitivement |
| **À éliminer** | `to_be_destroyed` | Document en attente de destruction | → Éliminé |
| **Éliminé** | `destroyed` | Document physiquement détruit | Aucune (état final) |
| **Archivé définitivement** | `permanently_archived` | Document à valeur patrimoniale | Aucune (état final) |
| **En cours de versement** | `transfer_pending` | Document en attente de versement aux archives nationales | → Archivé définitivement |

---

## 3. Cas d'utilisation — Vue d'ensemble

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-CYC-01 | Définir une politique de conservation pour un type de document | Super Admin / Admin / Archiviste |
| UC-CYC-02 | Modifier une politique de conservation | Super Admin / Admin / Archiviste |
| UC-CYC-03 | Consulter les politiques de conservation | Tout utilisateur connecté |
| UC-CYC-04 | Calculer automatiquement la date d'échéance d'un document | Système (automatique) |
| UC-CYC-05 | Transition manuelle d'un document vers un nouveau statut | Archiviste / Responsable |
| UC-CYC-06 | Transition automatique d'un document (échéance atteinte) | Système (tâche planifiée) |
| UC-CYC-07 | Consulter les documents arrivant à échéance | Archiviste / Responsable |
| UC-CYC-08 | Planifier l'élimination d'un lot de documents | Archiviste |
| UC-CYC-09 | Confirmer l'élimination d'un lot de documents | Super Admin / Admin |
| UC-CYC-10 | Annuler une procédure d'élimination | Super Admin uniquement |
| UC-CYC-11 | Consulter l'historique du cycle de vie d'un document | Tout utilisateur ayant accès au document |
| UC-CYC-12 | Prolonger la durée de conservation d'un document | Archiviste / Responsable avec justification |
| UC-CYC-13 | Marquer un document pour archivage définitif | Archiviste |
| UC-CYC-14 | Créer un bordereau de versement aux archives nationales | Archiviste |
| UC-CYC-15 | Soumettre un bordereau de versement | Archiviste |
| UC-CYC-16 | Consulter les bordereaux de versement | Archiviste / Admin |
| UC-CYC-17 | Valider la réception par les archives nationales | Archiviste après confirmation externe |
| UC-CYC-18 | Générer un rapport d'échéances | Archiviste / Admin |
| UC-CYC-19 | Exporter la liste des documents à éliminer | Archiviste |
| UC-CYC-20 | Consulter les statistiques du cycle de vie | Admin / Archiviste |
| UC-CYC-21 | Configurer les alertes d'échéance | Admin |
| UC-CYC-22 | Geler un document (suspension de l'échéance) | Archiviste avec justification légale |
| UC-CYC-23 | Dégeler un document | Archiviste |
| UC-CYC-24 | Consulter les documents gelés | Archiviste |

---

## 4. Description détaillée des cas d'utilisation

### UC-CYC-01 — Définir une politique de conservation pour un type de document

**Acteur :** Super Admin, Admin ou Archiviste  
**Préconditions :** L'acteur est authentifié. Le type de document existe (MODULE 03)  
**Postconditions :** Une politique de conservation est définie et active pour ce type

**Scénario principal :**
1. L'acteur accède à la gestion des politiques de conservation
2. L'acteur sélectionne un type de document
3. L'acteur définit les paramètres de la politique :
   - Durée de conservation en phase active (en années)
   - Durée de conservation en phase semi-active (en années)
   - Durée de conservation en phase inactive (en années)
   - Sort final : élimination ou archivage définitif
   - Base légale (référence à la loi ou au texte réglementaire)
   - Notes explicatives
4. Le système calcule la durée totale de conservation (somme des phases)
5. Le système vérifie que la durée totale respecte les minimums légaux (si configurés globalement)
6. Le système crée la politique avec le statut "Active"
7. La création est journalisée

**Scénarios alternatifs :**
- **5a.** La durée totale est inférieure au minimum légal → le système affiche un avertissement et demande confirmation (avec justification obligatoire)
- **3a.** Plusieurs types de documents peuvent hériter de la même politique de base (ex: tous les contrats ont la même durée)

**Règles métier :**
- Une politique de conservation peut définir uniquement la durée totale sans détailler les phases — dans ce cas, le document reste "Actif" jusqu'à l'échéance puis passe directement au sort final
- La base légale est un champ obligatoire pour la traçabilité réglementaire
- Une politique peut définir un sort final mixte : "Tri à effectuer" — dans ce cas, un archiviste doit examiner chaque document à l'échéance pour décider entre élimination et archivage

---

### UC-CYC-05 — Transition manuelle d'un document vers un nouveau statut

**Acteur :** Archiviste ou Responsable  
**Préconditions :** Le document existe. La transition demandée est autorisée par le diagramme d'états  
**Postconditions :** Le document change de statut et l'événement est enregistré

**Scénario principal :**
1. L'acteur consulte le document
2. L'acteur voit le statut actuel et les transitions disponibles
3. L'acteur sélectionne la transition souhaitée (ex: "Actif" → "Semi-actif")
4. Le système demande une justification textuelle obligatoire
5. L'acteur saisit la justification
6. Le système met à jour le statut du document
7. Le système enregistre l'événement dans l'historique du cycle de vie du document
8. Si la transition modifie la date d'échéance (ex: passage en semi-actif), le système recalcule la nouvelle échéance
9. La transition est journalisée

**Règles métier des transitions autorisées :**
- `active` → `semi_active` : toujours possible
- `semi_active` → `inactive` : toujours possible
- `inactive` → `to_be_destroyed` : seulement si le sort final de la politique est "Élimination"
- `inactive` → `permanently_archived` : seulement si le sort final est "Archivage définitif"
- `inactive` → `transfer_pending` : si versement aux archives nationales prévu
- `transfer_pending` → `permanently_archived` : après confirmation de réception
- Aucune transition inverse n'est autorisée sauf gel/dégel

---

### UC-CYC-08 — Planifier l'élimination d'un lot de documents

**Acteur :** Archiviste  
**Préconditions :** Des documents ont le statut "Inactif" avec un sort final "Élimination"  
**Postconditions :** Un bordereau d'élimination est créé et en attente de validation

**Scénario principal :**
1. L'archiviste consulte la liste des documents éligibles à l'élimination (statut "Inactif" + échéance dépassée + sort final = élimination)
2. L'archiviste filtre et sélectionne un ensemble de documents (par catégorie, par service, par période)
3. L'archiviste crée un bordereau d'élimination incluant la liste des documents sélectionnés
4. Le système génère un numéro unique pour le bordereau
5. Le système calcule le volume total (nombre de documents, taille totale)
6. L'archiviste peut ajouter des notes et précisions
7. Le bordereau est sauvegardé avec le statut "En attente de validation"
8. Une notification est envoyée aux administrateurs pour validation

**Règles métier :**
- Un document ne peut figurer que dans un seul bordereau d'élimination à la fois
- Le bordereau d'élimination est un document juridique qui doit être conservé même après la destruction effective des documents

---

### UC-CYC-09 — Confirmer l'élimination d'un lot de documents

**Acteur :** Super Admin ou Admin  
**Préconditions :** Un bordereau d'élimination est en attente de validation  
**Postconditions :** Les documents sont marqués comme éliminés et les fichiers physiques sont supprimés

**Scénario principal :**
1. L'administrateur consulte le bordereau d'élimination
2. L'administrateur vérifie la liste des documents à éliminer
3. L'administrateur vérifie que les durées de conservation ont bien été respectées
4. L'administrateur valide le bordereau
5. Le système demande une confirmation finale avec saisie du mot de passe administrateur (action irréversible)
6. Le système change le statut de tous les documents du bordereau à "Éliminé"
7. Le système marque les fichiers physiques pour suppression différée (48 heures de délai de sécurité)
8. Le système génère un certificat de destruction signé numériquement incluant la liste complète des documents éliminés
9. Le bordereau passe au statut "Exécuté"
10. L'élimination est journalisée avec tous les détails

**Règles de sécurité :**
- La suppression physique des fichiers intervient 48 heures après la validation, permettant une annulation en cas d'erreur détectée rapidement
- Le certificat de destruction est un document juridique conservé indéfiniment
- Les métadonnées des documents éliminés sont conservées dans la base de données pour la traçabilité (seul le fichier physique est supprimé)

---

### UC-CYC-14 — Créer un bordereau de versement aux archives nationales

**Acteur :** Archiviste  
**Préconditions :** Des documents ont un sort final "Archivage définitif" et sont au statut "Inactif"  
**Postconditions :** Un bordereau de versement est créé

**Scénario principal :**
1. L'archiviste consulte les documents éligibles au versement (statut "Inactif" + sort final = archivage définitif)
2. L'archiviste sélectionne les documents à verser
3. L'archiviste remplit les informations du bordereau :
   - Service versant
   - Service d'archives destinataire
   - Période couverte par le versement
   - Description générale du fonds
   - Conditions de communicabilité
4. Le système génère le bordereau au format normalisé (SEDA — Standard d'Échange de Données pour l'Archivage)
5. Le système attribue un numéro unique de versement
6. Le bordereau est sauvegardé avec le statut "Brouillon"
7. Les documents inclus passent au statut "En cours de versement"

**Format du bordereau :**
Le bordereau respecte le standard SEDA (XML) et inclut pour chaque document : identifiant, titre, dates, producteur, description, cote, format de fichier, hash d'intégrité.

---

### UC-CYC-22 — Geler un document (suspension de l'échéance)

**Acteur :** Archiviste avec justification légale  
**Préconditions :** Le document existe et n'est pas déjà gelé  
**Postconditions :** Le document est gelé, son échéance est suspendue

**Scénario principal :**
1. L'archiviste consulte le document
2. L'archiviste déclenche la procédure de gel
3. Le système affiche un formulaire de justification obligatoire demandant :
   - Motif du gel (contentieux en cours, enquête administrative, litige, etc.)
   - Référence légale ou numéro de dossier associé
   - Durée prévisionnelle du gel (optionnelle)
4. L'archiviste remplit et valide le formulaire
5. Le système marque le document comme "Gelé"
6. Le système suspend le calcul d'échéance : le document ne change plus de statut automatiquement
7. Un badge visuel "🔒 Gelé" est affiché sur le document
8. Le gel est journalisé

**Règles métier :**
- Un document gelé ne peut pas être éliminé ni versé tant que le gel n'est pas levé
- Le gel peut être appliqué à n'importe quel statut du cycle de vie
- Un rapport mensuel des documents gelés est automatiquement envoyé aux archivistes pour révision

---

## 5. Modèles de données

### 5.1 Modèle `RetentionPolicy` (Politique de conservation)

**Table :** `retention_policies`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_type_id` | UUID | FK → document_types.id, UNIQUE, NOT NULL | Type de document concerné |
| `active_phase_years` | SMALLINT | NOT NULL, DEFAULT 0 | Durée phase active (années) |
| `semi_active_phase_years` | SMALLINT | NOT NULL, DEFAULT 0 | Durée phase semi-active (années) |
| `inactive_phase_years` | SMALLINT | NOT NULL, DEFAULT 0 | Durée phase inactive (années) |
| `total_retention_years` | SMALLINT | NOT NULL | Durée totale (calculé automatiquement) |
| `final_disposition` | VARCHAR(30) | NOT NULL | Sort final : `destroy`, `archive_permanently`, `review_required` |
| `legal_basis` | TEXT | NOT NULL | Référence légale (loi, décret, circulaire) |
| `legal_minimum_years` | SMALLINT | NULL | Durée légale minimale (si applicable) |
| `notes` | TEXT | NULL | Notes explicatives |
| `triggers_automatic_transition` | BOOLEAN | NOT NULL, DEFAULT TRUE | Transitions automatiques activées |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Politique active |
| `effective_date` | DATE | NOT NULL, DEFAULT NOW() | Date d'entrée en vigueur |
| `expiry_date` | DATE | NULL | Date d'expiration (NULL = indéfinie) |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `updated_at` | TIMESTAMP | NOT NULL, AUTO | Date de mise à jour |
| `created_by_id` | UUID | FK → users.id, NULL | Créateur |

**Contraintes :**
- `total_retention_years` = `active_phase_years` + `semi_active_phase_years` + `inactive_phase_years` (calculé automatiquement via signal Django)
- `final_disposition` : valeur dans `{destroy, archive_permanently, review_required}`
- Si `legal_minimum_years` est défini, alors `total_retention_years` >= `legal_minimum_years`

**Index :**
- `idx_retention_policies_document_type_id` sur `document_type_id`
- `idx_retention_policies_is_active` sur `is_active`

---

### 5.2 Modèle `DocumentLifecycleHistory` (Historique du cycle de vie d'un document)

**Table :** `document_lifecycle_history`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_id` | UUID | FK → documents.id, NOT NULL, INDEX | Document concerné |
| `previous_status` | VARCHAR(30) | NULL | Statut précédent (NULL si création) |
| `new_status` | VARCHAR(30) | NOT NULL | Nouveau statut |
| `transition_date` | TIMESTAMP | NOT NULL, AUTO | Date et heure de la transition |
| `transition_type` | VARCHAR(20) | NOT NULL | Type : `automatic`, `manual`, `forced` |
| `justification` | TEXT | NULL | Justification de la transition (obligatoire si manuelle) |
| `triggered_by_id` | UUID | FK → users.id, NULL | Utilisateur ayant déclenché (NULL si automatique) |
| `expiry_date_before` | DATE | NULL | Date d'échéance avant la transition |
| `expiry_date_after` | DATE | NULL | Date d'échéance après la transition |

**Index :**
- `idx_document_lifecycle_history_document_id` sur `document_id`
- `idx_document_lifecycle_history_transition_date` sur `transition_date`

---

### 5.3 Modèle `DestructionBatch` (Bordereau d'élimination)

**Table :** `destruction_batches`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `batch_number` | VARCHAR(50) | UNIQUE, NOT NULL | Numéro unique du bordereau |
| `title` | VARCHAR(200) | NOT NULL | Titre descriptif |
| `description` | TEXT | NULL | Description du lot |
| `status` | VARCHAR(30) | NOT NULL | Statut : `draft`, `pending_approval`, `approved`, `executed`, `cancelled` |
| `document_count` | INTEGER | NOT NULL, DEFAULT 0 | Nombre de documents |
| `total_size_mb` | DECIMAL(12,2) | NOT NULL, DEFAULT 0 | Taille totale en Mo |
| `date_range_start` | DATE | NULL | Début de la période couverte |
| `date_range_end` | DATE | NULL | Fin de la période couverte |
| `notes` | TEXT | NULL | Notes et observations |
| `created_by_id` | UUID | FK → users.id, NOT NULL | Archiviste créateur |
| `approved_by_id` | UUID | FK → users.id, NULL | Administrateur ayant validé |
| `approved_at` | TIMESTAMP | NULL | Date de validation |
| `executed_at` | TIMESTAMP | NULL | Date d'exécution effective |
| `physical_deletion_scheduled_at` | TIMESTAMP | NULL | Date programmée de suppression physique |
| `destruction_certificate_path` | VARCHAR(500) | NULL | Chemin du certificat de destruction |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `updated_at` | TIMESTAMP | NOT NULL, AUTO | Date de mise à jour |

**Index :**
- `idx_destruction_batches_batch_number` sur `batch_number`
- `idx_destruction_batches_status` sur `status`
- `idx_destruction_batches_created_by_id` sur `created_by_id`

---

### 5.4 Modèle `DestructionBatchDocument` (Liaison Bordereau ↔ Document)

**Table :** `destruction_batch_documents`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `batch_id` | UUID | FK → destruction_batches.id, NOT NULL | Bordereau |
| `document_id` | UUID | FK → documents.id, NOT NULL | Document |
| `added_at` | TIMESTAMP | NOT NULL, AUTO | Date d'ajout au bordereau |

**Contrainte :** `UNIQUE (batch_id, document_id)`

---

### 5.5 Modèle `TransferBatch` (Bordereau de versement)

**Table :** `transfer_batches`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `transfer_number` | VARCHAR(50) | UNIQUE, NOT NULL | Numéro unique de versement |
| `title` | VARCHAR(200) | NOT NULL | Titre du versement |
| `description` | TEXT | NULL | Description du fonds |
| `status` | VARCHAR(30) | NOT NULL | Statut : `draft`, `submitted`, `accepted`, `rejected`, `completed` |
| `transferring_department_id` | UUID | FK → departments.id, NOT NULL | Service versant |
| `receiving_archive_name` | VARCHAR(200) | NOT NULL | Nom du service d'archives destinataire |
| `receiving_archive_address` | TEXT | NULL | Adresse du service d'archives |
| `period_covered_start` | DATE | NULL | Début de la période couverte |
| `period_covered_end` | DATE | NULL | Fin de la période couverte |
| `communicability_conditions` | TEXT | NULL | Conditions de communicabilité |
| `document_count` | INTEGER | NOT NULL, DEFAULT 0 | Nombre de documents versés |
| `seda_xml_path` | VARCHAR(500) | NULL | Chemin du fichier SEDA XML |
| `submission_date` | DATE | NULL | Date de soumission |
| `acceptance_date` | DATE | NULL | Date d'acceptation |
| `rejection_reason` | TEXT | NULL | Motif de rejet (si applicable) |
| `notes` | TEXT | NULL | Notes diverses |
| `created_by_id` | UUID | FK → users.id, NOT NULL | Archiviste créateur |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `updated_at` | TIMESTAMP | NOT NULL, AUTO | Date de mise à jour |

**Index :**
- `idx_transfer_batches_transfer_number` sur `transfer_number`
- `idx_transfer_batches_status` sur `status`

---

### 5.6 Modèle `TransferBatchDocument` (Liaison Bordereau de versement ↔ Document)

**Table :** `transfer_batch_documents`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `batch_id` | UUID | FK → transfer_batches.id, NOT NULL | Bordereau |
| `document_id` | UUID | FK → documents.id, NOT NULL | Document |
| `archive_reference` | VARCHAR(100) | NULL | Cote d'archivage attribuée par le service d'archives |
| `added_at` | TIMESTAMP | NOT NULL, AUTO | Date d'ajout au bordereau |

**Contrainte :** `UNIQUE (batch_id, document_id)`

---

### 5.7 Modèle `DocumentFreeze` (Gel de document)

**Table :** `document_freezes`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_id` | UUID | FK → documents.id, NOT NULL | Document gelé |
| `freeze_reason` | VARCHAR(100) | NOT NULL | Motif du gel |
| `legal_reference` | VARCHAR(200) | NULL | Référence légale (dossier, jugement) |
| `expected_duration_months` | SMALLINT | NULL | Durée prévisionnelle du gel (en mois) |
| `notes` | TEXT | NULL | Notes complémentaires |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Gel actif |
| `frozen_at` | TIMESTAMP | NOT NULL, AUTO | Date du gel |
| `frozen_by_id` | UUID | FK → users.id, NOT NULL | Qui a gelé |
| `unfrozen_at` | TIMESTAMP | NULL | Date du dégel |
| `unfrozen_by_id` | UUID | FK → users.id, NULL | Qui a dégelé |
| `unfrozen_reason` | TEXT | NULL | Raison du dégel |

**Index :**
- `idx_document_freezes_document_id` sur `document_id`
- `idx_document_freezes_is_active` sur `is_active`

**Contrainte métier :** Un document ne peut avoir qu'un seul gel actif à la fois (`UNIQUE (document_id) WHERE is_active = TRUE` — contrainte partielle PostgreSQL)

---

### 5.8 Modèle `RetentionExtension` (Prolongation de conservation)

**Table :** `retention_extensions`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_id` | UUID | FK → documents.id, NOT NULL | Document prolongé |
| `extension_years` | SMALLINT | NOT NULL | Nombre d'années supplémentaires |
| `justification` | TEXT | NOT NULL | Justification obligatoire |
| `legal_basis` | VARCHAR(200) | NULL | Base légale de la prolongation |
| `new_expiry_date` | DATE | NOT NULL | Nouvelle date d'échéance |
| `approved_by_id` | UUID | FK → users.id, NOT NULL | Qui a approuvé |
| `approved_at` | TIMESTAMP | NOT NULL, AUTO | Date d'approbation |

---

## 6. Matrice des permissions du module

| Permission code | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `lifecycle.policies.create` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `lifecycle.policies.update` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `lifecycle.policies.read` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `lifecycle.transition.manual` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `lifecycle.history.read` | ✅ | ✅ | ✅ | ✅ | ✅ (si accès doc) | ✅ |
| `lifecycle.destruction.create_batch` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `lifecycle.destruction.approve` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `lifecycle.destruction.cancel` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `lifecycle.transfer.create_batch` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `lifecycle.transfer.submit` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `lifecycle.transfer.read` | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| `lifecycle.freeze.apply` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `lifecycle.freeze.remove` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `lifecycle.extension.create` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `lifecycle.reports.read` | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| `lifecycle.stats.read` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |

---

## 7. Séquences détaillées des flux principaux

### 7.1 Séquence — Transition automatique d'un document (tâche Celery planifiée)

```
Celery Worker         API Django            Base de données       Notifications
    |                     |                       |                       |
    |-- cron quotidien -->|                       |                       |
    |                     |-- SELECT documents -->|                       |
    |                     |   WHERE expiry_date   |                       |
    |                     |   <= TODAY            |                       |
    |                     |<-- liste documents ---|                       |
    |                     |                       |                       |
    |-- pour chaque doc -->|                      |                       |
    |                     |-- vérifier gel ------->|                      |
    |                     |<-- not frozen --------|                       |
    |                     |-- vérifier politique ->|                      |
    |                     |-- déterminer next     |                       |
    |                     |   status selon sort   |                       |
    |                     |   final               |                       |
    |                     |-- UPDATE status ------>|                      |
    |                     |-- INSERT history ----->|                      |
    |                     |-- calculer nouvelle   |                       |
    |                     |   expiry_date         |                       |
    |                     |-- créer notification ---------------------------->|
    |                     |   (archiviste)        |                       |
```

### 7.2 Séquence — Validation d'un bordereau d'élimination

```
Admin               Frontend React         API Django            Base de données
  |                       |                     |                       |
  |-- consulte bordeaux ->|                     |                       |
  |                       |-- GET /lifecycle/ -->|                       |
  |                       |   destruction-batches/|-- SELECT batches --->|
  |                       |                     |<-- liste --------------|
  |<-- liste affichée ----|                     |                       |
  |                       |                     |                       |
  |-- clique bordereau -->|                     |                       |
  |                       |-- GET /lifecycle/ -->|                       |
  |                       |   destruction-batches/|-- SELECT details --->|
  |                       |   {id}/              |-- SELECT documents -->|
  |                       |<-- détails complets --|                      |
  |<-- affiche docs ------|                     |                       |
  |                       |                     |                       |
  |-- valide ------------>|                     |                       |
  |<-- modal confirm -----|                     |                       |
  |   avec mdp            |                     |                       |
  |-- saisit mdp -------->|                     |                       |
  |                       |-- POST /lifecycle/ ->|                       |
  |                       |   destruction-batches/|-- vérifier mdp ----->|
  |                       |   {id}/approve/      |-- UPDATE batch ------>|
  |                       |                     |   status = approved   |
  |                       |                     |-- UPDATE documents --->|
  |                       |                     |   status = destroyed  |
  |                       |                     |-- générer certificat ->|
  |                       |                     |-- planifier suppression>|
  |                       |                     |   physique +48h        |
  |                       |                     |-- INSERT audit ------->|
  |                       |<-- 200 + certificat --|                      |
  |<-- confirmation ------|                     |                       |
```

---

## 8. Endpoints API du module

### 8.1 Politiques de conservation

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/lifecycle/policies/` | Liste des politiques | Oui |
| POST | `/api/v1/lifecycle/policies/` | Créer une politique | Oui + Permission |
| GET | `/api/v1/lifecycle/policies/{id}/` | Détail d'une politique | Oui |
| PATCH | `/api/v1/lifecycle/policies/{id}/` | Modifier une politique | Oui + Permission |
| GET | `/api/v1/lifecycle/policies/by-document-type/{type_id}/` | Politique pour un type | Oui |

### 8.2 Transitions de cycle de vie

| Méthode | URL | Description | Auth |
|---|---|---|---|
| POST | `/api/v1/lifecycle/documents/{doc_id}/transition/` | Transition manuelle | Oui + Permission |
| GET | `/api/v1/lifecycle/documents/{doc_id}/history/` | Historique du cycle de vie | Oui |
| GET | `/api/v1/lifecycle/documents/expiring/` | Documents arrivant à échéance | Oui + Permission |

### 8.3 Bordereaux d'élimination

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/lifecycle/destruction-batches/` | Liste des bordereaux | Oui + Permission |
| POST | `/api/v1/lifecycle/destruction-batches/` | Créer un bordereau | Oui + Permission |
| GET | `/api/v1/lifecycle/destruction-batches/{id}/` | Détail d'un bordereau | Oui + Permission |
| POST | `/api/v1/lifecycle/destruction-batches/{id}/approve/` | Valider (Admin) | Oui + Permission |
| POST | `/api/v1/lifecycle/destruction-batches/{id}/cancel/` | Annuler (Super Admin) | Oui + Permission |
| GET | `/api/v1/lifecycle/destruction-batches/{id}/certificate/` | Télécharger certificat | Oui + Permission |

### 8.4 Bordereaux de versement

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/lifecycle/transfer-batches/` | Liste des bordereaux | Oui + Permission |
| POST | `/api/v1/lifecycle/transfer-batches/` | Créer un bordereau | Oui + Permission |
| GET | `/api/v1/lifecycle/transfer-batches/{id}/` | Détail | Oui + Permission |
| POST | `/api/v1/lifecycle/transfer-batches/{id}/submit/` | Soumettre | Oui + Permission |
| POST | `/api/v1/lifecycle/transfer-batches/{id}/accept/` | Marquer accepté | Oui + Permission |
| GET | `/api/v1/lifecycle/transfer-batches/{id}/seda-xml/` | Télécharger SEDA | Oui + Permission |

### 8.5 Gel de documents

| Méthode | URL | Description | Auth |
|---|---|---|---|
| POST | `/api/v1/lifecycle/documents/{doc_id}/freeze/` | Geler un document | Oui + Permission |
| POST | `/api/v1/lifecycle/documents/{doc_id}/unfreeze/` | Dégeler | Oui + Permission |
| GET | `/api/v1/lifecycle/freezes/active/` | Liste des documents gelés | Oui + Permission |

### 8.6 Prolongations

| Méthode | URL | Description | Auth |
|---|---|---|---|
| POST | `/api/v1/lifecycle/documents/{doc_id}/extend/` | Prolonger la conservation | Oui + Permission |
| GET | `/api/v1/lifecycle/extensions/` | Liste des prolongations | Oui + Permission |

### 8.7 Rapports et statistiques

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/lifecycle/reports/expiring/` | Rapport d'échéances | Oui + Permission |
| GET | `/api/v1/lifecycle/stats/by-status/` | Statistiques par statut | Oui + Permission |
| GET | `/api/v1/lifecycle/stats/by-category/` | Répartition par catégorie | Oui + Permission |

---

## 9. Règles métier et calculs automatiques

### 9.1 Calcul de la date d'échéance

La date d'échéance d'un document est calculée selon la formule :

```
expiry_date = document.creation_date + policy.active_phase_years (années)
```

À chaque transition de statut, la date d'échéance est recalculée :

- Passage à "Semi-actif" : `expiry_date` = `transition_date` + `policy.semi_active_phase_years`
- Passage à "Inactif" : `expiry_date` = `transition_date` + `policy.inactive_phase_years`

### 9.2 Règle de suspension du gel

Si un document est gelé (`DocumentFreeze.is_active = TRUE`), le système :
- N'effectue aucune transition automatique
- N'envoie aucune alerte d'expiration
- Affiche un badge "Gelé" sur le document
- Exclut le document des rapports d'échéances standards

### 9.3 Durées légales minimales prédéfinies (exemples France)

| Type de document | Durée minimale | Base légale |
|---|---|---|
| Contrats de travail | 5 ans après départ | Code du travail art. L1234-19 |
| Bulletins de paie | Permanente | Code du travail art. R243-1 |
| Factures | 10 ans | Code de commerce art. L123-22 |
| Documents comptables | 10 ans | Code de commerce art. L123-22 |
| Décisions administratives | 5 ans minimum | Loi 79-587 (archives publiques) |
| Contrats conclus | 5 ans après expiration | Code civil art. 2224 |

Ces durées sont configurables par l'administrateur et servent de garde-fou lors de la création d'une politique de conservation.

---

## 10. Tâches Celery planifiées

| Tâche | Fréquence | Description |
|---|---|---|
| `lifecycle.process_expirations` | Quotidien (2h du matin) | Vérifie tous les documents dont l'échéance est atteinte et effectue les transitions automatiques |
| `lifecycle.send_expiration_alerts` | Hebdomadaire (lundi 9h) | Envoie les alertes pour les documents arrivant à échéance dans les 30 jours |
| `lifecycle.execute_scheduled_deletions` | Quotidien (4h du matin) | Supprime physiquement les fichiers des documents éliminés depuis plus de 48h |
| `lifecycle.review_frozen_documents` | Mensuel (1er du mois) | Génère un rapport des documents gelés depuis plus de 6 mois |

---

## 11. Événements journalisés (MODULE 11 — Audit)

| Événement | Niveau | Détails |
|---|---|---|
| Politique de conservation créée | INFO | policy_id, document_type, total_years, created_by |
| Politique modifiée | INFO | policy_id, changes, modified_by |
| Transition manuelle de statut | INFO | document_id, old_status, new_status, justification, triggered_by |
| Transition automatique de statut | INFO | document_id, old_status, new_status, policy_id |
| Bordereau d'élimination créé | INFO | batch_id, document_count, created_by |
| Bordereau d'élimination validé | ALERT | batch_id, document_count, approved_by |
| Document éliminé | ALERT | document_id, batch_id, executed_at |
| Bordereau de versement créé | INFO | batch_id, document_count, created_by |
| Bordereau de versement soumis | INFO | batch_id, submission_date, created_by |
| Document gelé | WARNING | document_id, freeze_reason, frozen_by |
| Document dégelé | INFO | document_id, unfrozen_reason, unfrozen_by |
| Prolongation de conservation | INFO | document_id, extension_years, justification, approved_by |

---

## 12. Notifications générées (MODULE 12)

| Déclencheur | Destinataire | Canal | Message |
|---|---|---|
| Documents expirant dans 30 jours | Archivistes | Email + In-app | [N] documents arrivent à échéance dans 30 jours |
| Transition automatique effectuée | Propriétaire du document | In-app | Votre document [X] est passé au statut [Y] |
| Bordereau d'élimination créé | Admins | In-app | Nouveau bordereau d'élimination à valider |
| Élimination exécutée | Archiviste créateur | Email | Le bordereau [X] a été exécuté ([N] docs éliminés) |
| Document gelé depuis 6 mois | Archiviste ayant gelé | Email | Révision : le document [X] est gelé depuis 6 mois |
| Bordereau de versement accepté | Archiviste créateur | Email + In-app | Le versement [X] a été accepté par les archives |

---

*Fin du MODULE 04 — Cycle de Vie & Politique de Conservation*  
*Prochain module : MODULE 05 — Confidentialité & Classification de sécurité*