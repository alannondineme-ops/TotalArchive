# MODULE 08 — Workflow de Validation Documentaire

## Projet : Système d'Archivage Numérique des Documents
**Version :** 1.0  
**Dépendances :** MODULE 01 — Architecture | MODULE 02 — Authentification | MODULE 06 — Documents  
**Statut :** Module orchestration — le MODULE 12 (Notifications) dépend de ce module

---

## 1. Présentation du module

Dans une administration, tous les documents ne peuvent pas être archivés directement sans validation. Certains types de documents nécessitent un circuit de validation impliquant plusieurs acteurs successifs ou parallèles : vérification de la conformité, validation hiérarchique, approbation financière, validation juridique, etc.

Ce module définit des **workflows de validation configurables** qui orchestrent automatiquement la circulation des documents entre les différents validateurs, gèrent les délais, les escalades, les rejets et les approbations. Un workflow peut être séquentiel (étape par étape), parallèle (plusieurs validateurs simultanément), ou hybride.

Sans ce module, l'archivage de documents sensibles ou stratégiques se fait de manière informelle, par emails dispersés, sans traçabilité ni garantie que toutes les validations requises ont bien été obtenues.

---

## 2. Concepts fondamentaux

### 2.1 Qu'est-ce qu'un workflow ?

Un **workflow** est un circuit de validation prédéfini composé d'une séquence d'étapes. Chaque étape possède :
- Un **nom** (ex: "Validation RH", "Approbation Direction")
- Un **type** : validation obligatoire ou consultation informative
- Un **ou plusieurs validateurs** assignés (par rôle ou nominativement)
- Un **délai maximum** de traitement
- Des **actions possibles** : approuver, rejeter, demander révision

### 2.2 Types d'étapes de workflow

| Type | Description | Actions possibles |
|---|---|---|
| **Validation obligatoire** | Doit être approuvée pour passer à l'étape suivante | Approuver, Rejeter, Demander révision |
| **Consultation informative** | Pour information uniquement, n'empêche pas la progression | Prendre connaissance |
| **Approbation finale** | Dernière étape, valide l'archivage définitif | Approuver définitivement, Rejeter |

### 2.3 Types de workflows

| Type | Description | Exemple |
|---|---|---|
| **Séquentiel** | Les étapes se succèdent une par une | Validation RH → Validation Direction → Approbation Finance |
| **Parallèle** | Plusieurs validateurs en même temps (tous doivent valider) | Validation Service juridique ET Service technique (simultanément) |
| **Hybride** | Combinaison de séquentiel et parallèle | Étape 1 séquentielle → Étape 2 parallèle → Étape 3 séquentielle |

---

## 3. Cas d'utilisation — Vue d'ensemble

### 3.1 Configuration des workflows (Admin)

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-WF-01 | Créer un modèle de workflow | Super Admin / Admin |
| UC-WF-02 | Modifier un modèle de workflow | Super Admin / Admin |
| UC-WF-03 | Désactiver un modèle de workflow | Super Admin / Admin |
| UC-WF-04 | Définir les étapes d'un workflow | Super Admin / Admin |
| UC-WF-05 | Assigner un workflow à un type de document | Super Admin / Admin / Archiviste |
| UC-WF-06 | Consulter les workflows existants | Admin / Archiviste |
| UC-WF-07 | Dupliquer un workflow existant | Admin |

### 3.2 Soumission et suivi

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-WF-08 | Soumettre un document au workflow | Archiviste / Responsable / Agent |
| UC-WF-09 | Consulter le statut du workflow d'un document | Tout utilisateur (selon permissions) |
| UC-WF-10 | Consulter l'historique des étapes passées | Tout utilisateur (selon permissions) |
| UC-WF-11 | Consulter mes documents en attente de validation | Validateur |
| UC-WF-12 | Retirer un document du workflow (annulation) | Créateur / Admin |

### 3.3 Validation

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-WF-13 | Approuver une étape de validation | Validateur assigné |
| UC-WF-14 | Rejeter une étape de validation avec motif | Validateur assigné |
| UC-WF-15 | Demander une révision du document | Validateur assigné |
| UC-WF-16 | Ajouter un commentaire sur une étape | Validateur assigné |
| UC-WF-17 | Déléguer sa validation à un autre utilisateur | Validateur assigné |
| UC-WF-18 | Réassigner un validateur sur une étape | Admin / Responsable |
| UC-WF-19 | Consulter le délai restant avant escalade | Validateur assigné |

### 3.4 Escalade et exceptions

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-WF-20 | Escalade automatique en cas de délai dépassé | Système (automatique) |
| UC-WF-21 | Validation d'urgence (court-circuitage) | Super Admin uniquement |
| UC-WF-22 | Relance manuelle d'un validateur inactif | Créateur / Admin |

---

## 4. Description détaillée des cas d'utilisation

### UC-WF-01 — Créer un modèle de workflow

**Acteur :** Super Admin ou Admin  
**Préconditions :** L'acteur est authentifié avec les permissions appropriées  
**Postconditions :** Un nouveau modèle de workflow est créé et disponible

**Scénario principal :**
1. L'acteur accède au module d'administration des workflows
2. L'acteur clique sur "Créer un workflow"
3. L'acteur remplit les informations générales :
   - Nom du workflow (ex: "Validation des contrats de travail")
   - Code unique (ex: `WF_CONTRAT_TRAVAIL`)
   - Description
   - Type de workflow : séquentiel, parallèle ou hybride
4. L'acteur définit les étapes du workflow (voir UC-WF-04)
5. L'acteur configure les règles d'escalade globales :
   - Délai global maximal du workflow (en jours)
   - Action en cas de dépassement : notifier admin, escalader à la direction, approbation automatique forcée
6. L'acteur définit les types de documents auxquels ce workflow s'applique
7. L'acteur active le workflow
8. Le système enregistre le workflow
9. Le workflow devient disponible lors de la création de documents du type concerné

**Règles métier :**
- Un workflow doit avoir au minimum 1 étape
- Un workflow ne peut pas être supprimé s'il est en cours d'utilisation sur des documents actifs
- Le code du workflow est unique et immuable après création

---

### UC-WF-08 — Soumettre un document au workflow

**Acteur :** Archiviste, Responsable ou Agent  
**Préconditions :** Le document existe. Un workflow est défini pour le type de document  
**Postconditions :** Le document entre dans le workflow, la première étape est initialisée

**Scénario principal :**
1. L'acteur crée un document (UC-DOC-01) ou consulte un document existant
2. Si le type de document possède un workflow obligatoire, le système affiche : "Ce document nécessite une validation selon le workflow [Nom]"
3. L'acteur peut consulter le circuit de validation avant de soumettre
4. L'acteur clique sur "Soumettre au workflow"
5. Le système crée une instance de workflow pour ce document
6. Le système initialise la première étape
7. Le système identifie le(s) validateur(s) de la première étape selon les règles d'assignation
8. Le système envoie une notification aux validateurs assignés
9. Le système marque le document avec le statut `processing_status = 'in_workflow'`
10. Le système bloque la modification du document (verrouillage automatique)
11. L'acteur reçoit une confirmation : "Document soumis au workflow [Nom], étape actuelle : [Étape 1]"

**Scénarios alternatifs :**
- **2a.** Le workflow est optionnel → l'acteur peut choisir de soumettre ou non
- **7a.** Aucun validateur n'est disponible pour la première étape → le système alerte l'admin et met le workflow en pause

---

### UC-WF-13 — Approuver une étape de validation

**Acteur :** Validateur assigné  
**Préconditions :** Le document est à l'étape actuelle du workflow. L'acteur est un des validateurs assignés  
**Postconditions :** L'étape est approuvée, le workflow progresse vers l'étape suivante

**Scénario principal :**
1. Le validateur reçoit une notification : "Document [X] en attente de votre validation"
2. Le validateur accède à sa liste de documents en attente de validation
3. Le validateur consulte le document
4. Le validateur voit l'étape actuelle et sa responsabilité
5. Le validateur peut télécharger le document, consulter les métadonnées, lire les commentaires des étapes précédentes
6. Le validateur décide d'approuver
7. Le validateur clique sur "Approuver"
8. Le système demande optionnellement un commentaire (recommandé mais pas obligatoire)
9. Le validateur saisit un commentaire : "Document conforme, validation RH accordée"
10. Le validateur confirme l'approbation
11. Le système enregistre l'approbation avec horodatage et identité du validateur
12. Le système vérifie si tous les validateurs de cette étape ont approuvé (en cas d'étape parallèle avec plusieurs validateurs)
13. Si tous ont approuvé : le système passe à l'étape suivante
14. Si l'étape suivante existe : le système notifie les nouveaux validateurs
15. Si c'était la dernière étape : le document est marqué "Validé", déverrouillé, et archivé définitivement
16. Une notification est envoyée au créateur du document

**Règles métier :**
- Une approbation ne peut jamais être annulée (garantie de traçabilité)
- Si l'étape est parallèle, tous les validateurs doivent approuver avant de passer à l'étape suivante
- Un validateur peut approuver même après le délai dépassé (mais une escalade a déjà pu se déclencher)

---

### UC-WF-14 — Rejeter une étape de validation avec motif

**Acteur :** Validateur assigné  
**Préconditions :** Le document est à l'étape actuelle. L'acteur est validateur  
**Postconditions :** L'étape est rejetée, le document retourne au créateur ou à une étape précédente

**Scénario principal :**
1. Le validateur consulte le document
2. Le validateur identifie un problème (document incomplet, erreur, non-conformité)
3. Le validateur clique sur "Rejeter"
4. Le système affiche un formulaire obligatoire de motif de rejet
5. Le validateur sélectionne une raison prédéfinie :
   - Document incomplet
   - Informations erronées
   - Non conforme aux règles
   - Autre (justification libre)
6. Le validateur saisit un commentaire détaillé obligatoire expliquant le rejet
7. Le validateur peut suggérer des corrections
8. Le validateur confirme le rejet
9. Le système enregistre le rejet avec horodatage et identité
10. Le système arrête le workflow
11. Le système retourne le document au créateur avec le statut `processing_status = 'rejected_in_workflow'`
12. Le système déverrouille le document pour permettre les modifications
13. Une notification est envoyée au créateur avec le motif du rejet et les suggestions de correction
14. Le créateur peut corriger le document et le resoumettre (nouveau workflow depuis le début)

**Règles métier :**
- Un rejet arrête immédiatement le workflow, même si d'autres validateurs n'ont pas encore validé
- Le motif du rejet est obligatoire et doit être explicite
- Le workflow rejeté reste dans l'historique pour traçabilité

---

### UC-WF-15 — Demander une révision du document

**Acteur :** Validateur assigné  
**Préconditions :** Le document est à l'étape actuelle  
**Postconditions :** Le document retourne au créateur pour modification sans arrêt définitif du workflow

**Scénario principal :**
1. Le validateur consulte le document
2. Le validateur identifie des ajustements mineurs nécessaires (pas de rejet complet)
3. Le validateur clique sur "Demander une révision"
4. Le validateur saisit les modifications demandées
5. Le validateur confirme
6. Le système met le workflow en pause sur cette étape
7. Le système déverrouille temporairement le document
8. Une notification est envoyée au créateur : "Révision demandée par [Validateur] sur [Étape]"
9. Le créateur modifie le document
10. Le créateur clique sur "Révision effectuée, resoumission"
11. Le système reverrouille le document
12. Le système notifie le même validateur qui avait demandé la révision
13. Le validateur peut alors approuver ou rejeter

**Différence avec le rejet :**
- La révision ne redémarre pas le workflow depuis le début
- Le workflow reprend exactement à la même étape après correction
- C'est une boucle de correction mineure

---

### UC-WF-20 — Escalade automatique en cas de délai dépassé

**Acteur :** Système (tâche Celery automatique)  
**Préconditions :** Un document est dans un workflow. Le délai d'une étape est dépassé  
**Postconditions :** Une escalade est déclenchée selon la configuration du workflow

**Scénario principal :**
1. Une tâche Celery planifiée s'exécute toutes les heures
2. Le système identifie tous les documents en workflow avec une étape dont le délai est dépassé
3. Pour chaque étape en retard, le système vérifie la règle d'escalade configurée :
   - **Niveau 1 (délai dépassé de 0-24h)** : Notification de rappel au validateur + notification au responsable hiérarchique
   - **Niveau 2 (délai dépassé de 24-48h)** : Réassignation automatique au responsable hiérarchique du validateur
   - **Niveau 3 (délai dépassé de >48h)** : Escalade à l'administrateur avec demande de décision manuelle
4. Le système enregistre l'escalade dans l'historique du workflow
5. Le système envoie les notifications appropriées
6. Si l'escalade implique une réassignation, le nouveau validateur est notifié

**Règles métier :**
- L'escalade ne valide jamais automatiquement (sauf configuration explicite rare pour des cas spécifiques)
- Le validateur initial peut toujours valider même après escalade (tant que le nouveau validateur n'a pas encore traité)
- Chaque escalade est journalisée dans l'audit

---

## 5. Modèles de données

### 5.1 Modèle `WorkflowTemplate` (Modèle de workflow)

**Table :** `workflow_templates`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `name` | VARCHAR(200) | NOT NULL | Nom du workflow |
| `code` | VARCHAR(50) | UNIQUE, NOT NULL | Code technique unique |
| `description` | TEXT | NULL | Description détaillée |
| `workflow_type` | VARCHAR(20) | NOT NULL | Type : `sequential`, `parallel`, `hybrid` |
| `is_mandatory` | BOOLEAN | NOT NULL, DEFAULT TRUE | Workflow obligatoire pour les types associés |
| `max_duration_days` | SMALLINT | NULL | Durée maximale globale (jours) |
| `escalation_rule` | VARCHAR(50) | NOT NULL, DEFAULT 'notify_admin' | Règle d'escalade : `notify_admin`, `reassign_manager`, `auto_approve` |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Workflow actif |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `updated_at` | TIMESTAMP | NOT NULL, AUTO | Date de mise à jour |
| `created_by_id` | UUID | FK → users.id, NOT NULL | Créateur |

**Index :**
- `idx_workflow_templates_code` sur `code`
- `idx_workflow_templates_is_active` sur `is_active`

---

### 5.2 Modèle `WorkflowStep` (Étape de workflow)

**Table :** `workflow_steps`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `template_id` | UUID | FK → workflow_templates.id, NOT NULL, INDEX | Workflow parent |
| `step_number` | SMALLINT | NOT NULL | Numéro d'ordre (1, 2, 3...) |
| `name` | VARCHAR(200) | NOT NULL | Nom de l'étape |
| `description` | TEXT | NULL | Description |
| `step_type` | VARCHAR(30) | NOT NULL | Type : `mandatory_validation`, `informative`, `final_approval` |
| `duration_hours` | SMALLINT | NULL | Délai maximum (heures) |
| `is_parallel` | BOOLEAN | NOT NULL, DEFAULT FALSE | Plusieurs validateurs simultanés |
| `requires_all_approvals` | BOOLEAN | NOT NULL, DEFAULT TRUE | Si parallèle, tous doivent approuver (TRUE) ou un seul suffit (FALSE) |
| `validator_assignment_type` | VARCHAR(30) | NOT NULL | Type : `by_role`, `by_user`, `by_department_manager` |
| `validator_role_id` | UUID | FK → roles.id, NULL | Rôle validateur (si by_role) |
| `validator_user_id` | UUID | FK → users.id, NULL | Utilisateur validateur (si by_user) |
| `can_delegate` | BOOLEAN | NOT NULL, DEFAULT TRUE | Le validateur peut déléguer |
| `can_reject` | BOOLEAN | NOT NULL, DEFAULT TRUE | Le validateur peut rejeter |
| `rejection_returns_to_step` | SMALLINT | NULL | Retour à quelle étape en cas de rejet (NULL = début) |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |

**Contrainte :** `UNIQUE (template_id, step_number)`

**Index :**
- `idx_workflow_steps_template_id` sur `template_id`

---

### 5.3 Modèle `WorkflowInstance` (Instance de workflow pour un document)

**Table :** `workflow_instances`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `template_id` | UUID | FK → workflow_templates.id, NOT NULL, INDEX | Modèle utilisé |
| `document_id` | UUID | FK → documents.id, NOT NULL, INDEX | Document concerné |
| `status` | VARCHAR(30) | NOT NULL | Statut : `active`, `completed`, `rejected`, `cancelled`, `on_hold` |
| `current_step_number` | SMALLINT | NULL | Numéro de l'étape actuelle |
| `started_at` | TIMESTAMP | NOT NULL, AUTO | Début du workflow |
| `completed_at` | TIMESTAMP | NULL | Fin du workflow |
| `cancelled_at` | TIMESTAMP | NULL | Date d'annulation |
| `cancelled_by_id` | UUID | FK → users.id, NULL | Qui a annulé |
| `cancellation_reason` | TEXT | NULL | Motif d'annulation |
| `created_by_id` | UUID | FK → users.id, NOT NULL | Qui a soumis le document |

**Contrainte :** Un document ne peut avoir qu'une seule instance de workflow active à la fois

**Index :**
- `idx_workflow_instances_document_id` sur `document_id`
- `idx_workflow_instances_status` sur `status`
- `idx_workflow_instances_current_step_number` sur `current_step_number`

---

### 5.4 Modèle `WorkflowStepExecution` (Exécution d'une étape)

**Table :** `workflow_step_executions`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `instance_id` | UUID | FK → workflow_instances.id, NOT NULL, INDEX | Instance parent |
| `step_id` | UUID | FK → workflow_steps.id, NOT NULL | Étape concernée |
| `step_number` | SMALLINT | NOT NULL | Numéro d'étape (dénormalisé) |
| `status` | VARCHAR(30) | NOT NULL | Statut : `pending`, `in_progress`, `approved`, `rejected`, `revision_requested`, `escalated` |
| `assigned_to_id` | UUID | FK → users.id, NOT NULL, INDEX | Validateur assigné |
| `delegated_to_id` | UUID | FK → users.id, NULL | Validateur délégué (si applicable) |
| `started_at` | TIMESTAMP | NOT NULL, AUTO | Début de l'étape |
| `deadline_at` | TIMESTAMP | NULL | Date limite (calculée depuis duration_hours) |
| `completed_at` | TIMESTAMP | NULL | Date de complétion |
| `action_taken` | VARCHAR(30) | NULL | Action : `approved`, `rejected`, `revision_requested` |
| `comment` | TEXT | NULL | Commentaire du validateur |
| `rejection_reason` | TEXT | NULL | Motif du rejet (si rejeté) |
| `escalation_level` | SMALLINT | NOT NULL, DEFAULT 0 | Niveau d'escalade (0, 1, 2, 3) |
| `escalated_at` | TIMESTAMP | NULL | Date de dernière escalade |
| `completed_by_id` | UUID | FK → users.id, NULL | Qui a complété (validateur ou délégué) |

**Index :**
- `idx_workflow_step_executions_instance_id` sur `instance_id`
- `idx_workflow_step_executions_assigned_to_id` sur `assigned_to_id`
- `idx_workflow_step_executions_status` sur `status`
- `idx_workflow_step_executions_deadline_at` sur `deadline_at` (pour détecter retards)

---

### 5.5 Modèle `WorkflowTemplateDocumentType` (Liaison Workflow ↔ Type de document)

**Table :** `workflow_template_document_types`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `template_id` | UUID | FK → workflow_templates.id, NOT NULL | Workflow |
| `document_type_id` | UUID | FK → document_types.id, NOT NULL | Type de document |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |

**Contrainte :** `UNIQUE (template_id, document_type_id)`

---

## 6. Matrice des permissions du module

| Permission code | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `workflow.templates.create` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `workflow.templates.update` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `workflow.templates.read` | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| `workflow.templates.delete` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `workflow.submit_document` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `workflow.view_status` | ✅ | ✅ | ✅ | ✅ | ✅ (filtré) | ✅ (filtré) |
| `workflow.approve_step` | Validateur assigné uniquement |
| `workflow.reject_step` | Validateur assigné uniquement |
| `workflow.request_revision` | Validateur assigné uniquement |
| `workflow.delegate` | Validateur assigné (si can_delegate) |
| `workflow.reassign_validator` | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| `workflow.cancel_instance` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `workflow.emergency_approve` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `workflow.view_all_pending` | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |

---

## 7. Séquences détaillées des flux principaux

### 7.1 Séquence — Soumission et validation complète (workflow séquentiel)

```
Créateur            Frontend React         API Django            Base de données       Notifications
    |                     |                     |                       |                       |
    |-- crée document ---->|                     |                       |                       |
    |-- soumet workflow -->|                    |                       |                       |
    |                     |-- POST /workflow/ -->|                      |                       |
    |                     |   instances/         |-- INSERT instance --->|                      |
    |                     |                     |-- INSERT step_exec -->|                       |
    |                     |                     |   (step 1, pending)   |                       |
    |                     |                     |-- LOCK document ------>|                      |
    |                     |                     |-- créer notification ---------------------------->|
    |                     |                     |   validateur étape 1  |                       |
    |                     |<-- 201 + instance ---|                       |                       |
    |<-- confirmation -----|                     |                       |                       |
    |                       |                     |                       |                       |
[Validateur 1 reçoit notification]                                       |                       |
    |                       |                     |                       |                       |
Validateur 1        Frontend React         API Django            Base de données       Notifications
    |                     |                     |                       |                       |
    |-- consulte doc ----->|                     |                       |                       |
    |-- approuve ---------->|                    |                       |                       |
    |                     |-- POST /workflow/ -->|                       |                       |
    |                     |   step-executions/   |-- UPDATE step_exec -->|                      |
    |                     |   {id}/approve/      |   status = approved   |                       |
    |                     |                     |-- INSERT step_exec -->|                       |
    |                     |                     |   (step 2, pending)   |                       |
    |                     |                     |-- UPDATE instance ---->|                      |
    |                     |                     |   current_step = 2    |                       |
    |                     |                     |-- créer notification ---------------------------->|
    |                     |                     |   validateur étape 2  |                       |
    |                     |<-- 200 OK -----------|                       |                       |
    |<-- confirmation -----|                     |                       |                       |
    |                       |                     |                       |                       |
[Cycle se répète pour chaque étape jusqu'à la dernière]
    |                       |                     |                       |                       |
[Dernière étape approuvée]                                               |                       |
    |                     |                     |-- UPDATE instance ---->|                      |
    |                     |                     |   status = completed  |                       |
    |                     |                     |-- UNLOCK document ---->|                      |
    |                     |                     |-- UPDATE document ----->|                      |
    |                     |                     |   processing_status   |                       |
    |                     |                     |   = completed         |                       |
    |                     |                     |-- créer notification ---------------------------->|
    |                     |                     |   créateur (succès)   |                       |
```

### 7.2 Séquence — Rejet avec motif

```
Validateur          Frontend React         API Django            Base de données       Notifications
    |                     |                     |                       |                       |
    |-- consulte doc ----->|                     |                       |                       |
    |-- identifie problème>|                    |                       |                       |
    |-- clique "Rejeter" ->|                    |                       |                       |
    |<-- formulaire motif -|                    |                       |                       |
    |-- saisit motif ------>|                   |                       |                       |
    |-- confirme ---------->|                   |                       |                       |
    |                     |-- POST /workflow/ -->|                       |                       |
    |                     |   step-executions/   |-- UPDATE step_exec -->|                      |
    |                     |   {id}/reject/       |   status = rejected   |                       |
    |                     |                     |   rejection_reason    |                       |
    |                     |                     |-- UPDATE instance ---->|                      |
    |                     |                     |   status = rejected   |                       |
    |                     |                     |-- UNLOCK document ---->|                      |
    |                     |                     |-- UPDATE document ----->|                      |
    |                     |                     |   processing_status   |                       |
    |                     |                     |   = rejected_in_wf    |                       |
    |                     |                     |-- INSERT audit ------->|                      |
    |                     |                     |-- créer notification ---------------------------->|
    |                     |                     |   créateur (rejet)    |                       |
    |                     |<-- 200 OK -----------|                       |                       |
    |<-- confirmation -----|                     |                       |                       |
```

---

## 8. Endpoints API du module

### 8.1 Modèles de workflow (Admin)

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/workflow/templates/` | Liste des modèles | Oui + Permission |
| POST | `/api/v1/workflow/templates/` | Créer un modèle | Oui + Permission |
| GET | `/api/v1/workflow/templates/{id}/` | Détail d'un modèle | Oui + Permission |
| PATCH | `/api/v1/workflow/templates/{id}/` | Modifier | Oui + Permission |
| DELETE | `/api/v1/workflow/templates/{id}/` | Supprimer | Oui + Permission |
| POST | `/api/v1/workflow/templates/{id}/duplicate/` | Dupliquer | Oui + Permission |

### 8.2 Étapes de workflow (Admin)

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/workflow/templates/{template_id}/steps/` | Liste des étapes | Oui + Permission |
| POST | `/api/v1/workflow/templates/{template_id}/steps/` | Créer étape | Oui + Permission |
| PATCH | `/api/v1/workflow/steps/{id}/` | Modifier étape | Oui + Permission |
| DELETE | `/api/v1/workflow/steps/{id}/` | Supprimer étape | Oui + Permission |

### 8.3 Instances de workflow

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/workflow/instances/` | Liste des instances | Oui + Permission |
| POST | `/api/v1/workflow/instances/` | Créer instance (soumettre doc) | Oui + Permission |
| GET | `/api/v1/workflow/instances/{id}/` | Détail instance | Oui + Permission |
| POST | `/api/v1/workflow/instances/{id}/cancel/` | Annuler workflow | Oui + Permission |
| GET | `/api/v1/workflow/documents/{doc_id}/instance/` | Instance d'un document | Oui + Permission |

### 8.4 Validation

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/workflow/my-pending-validations/` | Mes documents à valider | Oui |
| GET | `/api/v1/workflow/step-executions/{id}/` | Détail étape | Oui + Permission |
| POST | `/api/v1/workflow/step-executions/{id}/approve/` | Approuver | Oui + Validateur |
| POST | `/api/v1/workflow/step-executions/{id}/reject/` | Rejeter | Oui + Validateur |
| POST | `/api/v1/workflow/step-executions/{id}/request-revision/` | Demander révision | Oui + Validateur |
| POST | `/api/v1/workflow/step-executions/{id}/delegate/` | Déléguer | Oui + Validateur |
| POST | `/api/v1/workflow/step-executions/{id}/reassign/` | Réassigner | Oui + Permission |

---

## 9. Règles métier critiques

### 9.1 Règle de verrouillage automatique

Dès qu'un document entre dans un workflow, il est automatiquement verrouillé (`is_locked = TRUE`, MODULE 06). Aucune modification des métadonnées ni remplacement du fichier n'est autorisé tant que le workflow n'est pas terminé (complété, rejeté ou annulé).

Exception : en cas de demande de révision (UC-WF-15), le document est temporairement déverrouillé.

---

### 9.2 Règle de traçabilité absolue

Toutes les actions sur un workflow sont **immuables** et **traçables** :
- Une approbation ne peut jamais être annulée
- Un rejet ne peut jamais être supprimé
- Toutes les actions sont horodatées avec l'identité de l'acteur
- Les commentaires de validation ne peuvent pas être modifiés après soumission

---

### 9.3 Règle de gestion des étapes parallèles

Si une étape est configurée comme parallèle (`is_parallel = TRUE`) avec plusieurs validateurs :
- **Si `requires_all_approvals = TRUE`** : tous les validateurs doivent approuver avant de passer à l'étape suivante
- **Si `requires_all_approvals = FALSE`** : dès qu'un validateur approuve, l'étape est validée (vote majoritaire à 1)
- **Si un seul validateur rejette** : l'étape est immédiatement rejetée, même si d'autres n'ont pas encore validé

---

### 9.4 Règle de calcul des délais

Le délai d'une étape commence à courir dès que l'étape est créée (`started_at`), pas à partir de la notification. La date limite est calculée : `deadline_at = started_at + duration_hours`.

Les jours ouvrés vs calendaires sont configurables globalement. Par défaut, les délais incluent les week-ends et jours fériés.

---

## 10. Tâches Celery planifiées

| Tâche | Fréquence | Description |
|---|---|---|
| `workflow.check_overdue_steps` | Horaire (toutes les heures) | Détecte les étapes en retard et déclenche les escalades |
| `workflow.send_reminders` | Quotidien (9h) | Envoie des rappels aux validateurs pour les étapes proches de l'échéance |
| `workflow.cleanup_old_instances` | Mensuel (1er du mois) | Archive les instances terminées depuis plus de 1 an |
| `workflow.generate_stats` | Hebdomadaire (dimanche 1h) | Génère les statistiques de performance des workflows |

---

## 11. Événements journalisés (MODULE 11 — Audit)

| Événement | Niveau | Détails |
|---|---|---|
| Workflow soumis | INFO | document_id, template_id, instance_id, submitted_by |
| Étape approuvée | INFO | instance_id, step_number, validator, comment |
| Étape rejetée | WARNING | instance_id, step_number, validator, rejection_reason |
| Révision demandée | INFO | instance_id, step_number, validator, revision_notes |
| Validation déléguée | INFO | instance_id, step_number, delegator, delegated_to |
| Escalade déclenchée | WARNING | instance_id, step_number, escalation_level, reason |
| Workflow complété | INFO | instance_id, document_id, total_duration |
| Workflow annulé | WARNING | instance_id, cancelled_by, reason |
| Validation d'urgence | ALERT | document_id, approved_by, reason |

---

## 12. Notifications générées (MODULE 12)

| Déclencheur | Destinataire | Canal | Message |
|---|---|---|
| Document soumis au workflow | Créateur | In-app | Votre document [X] est entré dans le workflow [Y] |
| Étape assignée à un validateur | Validateur | Email + In-app | Document [X] en attente de votre validation |
| Étape approuvée | Créateur + validateur suivant | In-app | Étape [X] approuvée, passage à l'étape [Y] |
| Étape rejetée | Créateur | Email + In-app | Votre document [X] a été rejeté : [Motif] |
| Révision demandée | Créateur | Email + In-app | Révision demandée sur [Document] : [Détails] |
| Délai proche (24h avant échéance) | Validateur | In-app | Rappel : validation de [Document] à effectuer avant [Date] |
| Délai dépassé | Validateur + Manager | Email | URGENT : Délai dépassé sur [Document] |
| Escalade niveau 2 | Manager | Email | Escalade : validation requise sur [Document] |
| Workflow complété | Créateur | In-app | Votre document [X] a été validé et archivé |

---

*Fin du MODULE 08 — Workflow de Validation Documentaire*  
*Prochain module : MODULE 09 — Moteur de recherche avancée & Indexation*
