# MODULE 05 — Confidentialité & Classification de Sécurité

## Projet : Système d'Archivage Numérique des Documents
**Version :** 1.0  
**Dépendances :** MODULE 01 — Architecture | MODULE 02 — Authentification | MODULE 03 — Taxonomie  
**Statut :** Référentiel de sécurité — le MODULE 06 (Documents) et le MODULE 13 (Accès & Partage) dépendent directement de ce module

---

## 1. Présentation du module

La classification de sécurité est un mécanisme fondamental de protection de l'information sensible. Elle définit qui peut accéder à quoi, dans quelles conditions, et avec quelles restrictions. Dans une administration publique, la fuite d'un document confidentiel peut avoir des conséquences juridiques, financières, politiques ou atteindre l'intégrité des personnes.

Ce module définit les niveaux de confidentialité, les règles d'accès associées, les procédures de classification et de déclassification, et garantit que chaque document possède un niveau de confidentialité explicite dès sa création. Il est essentiel de bien comprendre que **la confidentialité est une dimension de sécurité**, distincte de la priorité de traitement qui est une dimension opérationnelle (Urgent/Normal/Faible).

Aucun document ne peut exister sans niveau de confidentialité. C'est une contrainte d'intégrité non négociable.

---

## 2. Les quatre niveaux de confidentialité

### 2.1 Hiérarchie des niveaux

| Niveau | Code | Couleur | Description | Exemple de documents |
|---|---|---|---|---|
| **Public** | `public` | 🟢 Vert | Accessible à tous sans restriction | Rapports publics, circulaires publiques, organigrammes |
| **Interne** | `internal` | 🟡 Jaune | Réservé aux employés de l'administration | Notes de service, procédures internes, comptes-rendus de réunion |
| **Confidentiel** | `confidential` | 🟠 Orange | Accès limité à certains services ou responsables autorisés | Contrats, dossiers RH, rapports financiers sensibles, données personnelles |
| **Secret** | `secret` | 🔴 Rouge | Accès strictement limité aux responsables autorisés nominativement | Documents stratégiques, contentieux en cours, enquêtes internes, données hautement sensibles |

### 2.2 Règles d'accès par niveau

| Niveau | Qui peut accéder ? |
|---|---|
| **Public** | Tout le monde, y compris le grand public si le document est publié |
| **Interne** | Tous les employés de l'administration ayant un compte actif |
| **Confidentiel** | Uniquement les utilisateurs ayant le rôle approprié OU appartenant au service propriétaire du document OU ayant reçu une autorisation explicite |
| **Secret** | Uniquement les utilisateurs explicitement autorisés nominativement par le propriétaire ou par un administrateur. Liste blanche obligatoire. |

---

## 3. Séparation entre Confidentialité et Priorité

### 3.1 Pourquoi cette séparation est critique

Le document original d'analyse du projet mentionnait "Urgent" comme un niveau de classification. C'était une **erreur conceptuelle grave**. La confidentialité et la priorité sont deux dimensions orthogonales :

- **Confidentialité** : qui peut voir ce document ?
- **Priorité** : à quelle vitesse ce document doit-il être traité ?

Un document peut être :
- Public et Urgent (ex: communiqué de presse à diffuser rapidement)
- Secret et Normal (ex: rapport stratégique sans urgence particulière)
- Confidentiel et Urgent (ex: contrat à valider rapidement)

### 3.2 Niveaux de priorité (dimension séparée)

| Priorité | Code | Description |
|---|---|---|
| Urgent | `urgent` | Traitement prioritaire requis |
| Normal | `normal` | Traitement standard |
| Faible | `low` | Peut être traité plus tard |

Cette dimension est gérée dans le MODULE 06 (Documents) comme une métadonnée distincte, jamais confondue avec la confidentialité.

---

## 4. Cas d'utilisation — Vue d'ensemble

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-CONF-01 | Classifier un document à la création | Archiviste / Responsable / Agent (selon permissions) |
| UC-CONF-02 | Modifier le niveau de confidentialité d'un document existant | Archiviste / Responsable |
| UC-CONF-03 | Consulter l'historique des classifications d'un document | Tout utilisateur ayant accès au document |
| UC-CONF-04 | Définir les règles d'accès par défaut pour une catégorie | Super Admin / Admin / Archiviste |
| UC-CONF-05 | Accorder un accès explicite à un document confidentiel | Propriétaire du document / Responsable |
| UC-CONF-06 | Révoquer un accès explicite | Propriétaire du document / Responsable |
| UC-CONF-07 | Consulter la liste des accès explicites d'un document | Propriétaire du document / Archiviste / Admin |
| UC-CONF-08 | Demander l'accès à un document confidentiel (traité dans MODULE 13) | Tout utilisateur authentifié |
| UC-CONF-09 | Déclassifier un document (passage à un niveau inférieur) | Archiviste / Responsable avec justification |
| UC-CONF-10 | Surclassifier un document (passage à un niveau supérieur) | Archiviste / Responsable avec justification |
| UC-CONF-11 | Consulter les documents par niveau de confidentialité | Tout utilisateur (filtrés selon ses droits) |
| UC-CONF-12 | Définir une politique de déclassification automatique | Super Admin / Admin |
| UC-CONF-13 | Consulter les alertes de classification incohérente | Archiviste / Admin |
| UC-CONF-14 | Hériter automatiquement du niveau de confidentialité du parent | Système (automatique) |
| UC-CONF-15 | Marquer un document comme contenant des données personnelles (RGPD) | Archiviste / Responsable |
| UC-CONF-16 | Consulter les documents contenant des données personnelles | DPO (Délégué à la Protection des Données) / Admin |
| UC-CONF-17 | Configurer les règles d'accès par rôle et niveau | Super Admin / Admin |
| UC-CONF-18 | Exporter la liste des documents classifiés (audit de sécurité) | Admin / Auditeur |
| UC-CONF-19 | Appliquer un marquage visuel selon le niveau de confidentialité | Système (automatique) |
| UC-CONF-20 | Bloquer le partage externe d'un document secret | Système (automatique) |

---

## 5. Description détaillée des cas d'utilisation

### UC-CONF-01 — Classifier un document à la création

**Acteur :** Archiviste, Responsable ou Agent (selon permissions)  
**Préconditions :** L'acteur crée un nouveau document. Le type de document possède un niveau de confidentialité par défaut (MODULE 03)  
**Postconditions :** Le document possède un niveau de confidentialité explicite

**Scénario principal :**
1. L'acteur remplit le formulaire de création de document
2. Le système suggère le niveau de confidentialité par défaut du type de document
3. L'acteur peut accepter la suggestion ou choisir un niveau différent
4. Si l'acteur choisit "Confidentiel" ou "Secret", le système demande une justification textuelle obligatoire
5. Le système enregistre le niveau de confidentialité et la justification
6. Le système crée une entrée dans l'historique de classification
7. Le système applique les règles d'accès correspondantes
8. La classification est journalisée

**Scénarios alternatifs :**
- **3a.** L'acteur choisit un niveau supérieur au niveau suggéré → justification obligatoire
- **3b.** L'acteur n'a pas la permission de classifier à un niveau donné (ex: un Agent ne peut pas classifier en Secret) → le système bloque l'action et affiche un message d'erreur

**Règles métier :**
- Un document ne peut jamais être créé sans niveau de confidentialité — si aucun n'est spécifié, le niveau par défaut du type est appliqué automatiquement
- La justification est obligatoire pour les niveaux "Confidentiel" et "Secret"
- Seuls les Super Admin, Admin, Archiviste et Responsable peuvent classifier en "Secret"

---

### UC-CONF-02 — Modifier le niveau de confidentialité d'un document existant

**Acteur :** Archiviste ou Responsable  
**Préconditions :** Le document existe. L'acteur a la permission de modifier la classification  
**Postconditions :** Le niveau de confidentialité est modifié et l'événement est tracé

**Scénario principal :**
1. L'acteur consulte le document
2. L'acteur accède à la fonction "Modifier la classification"
3. Le système affiche le niveau actuel et propose les niveaux disponibles selon les permissions de l'acteur
4. L'acteur sélectionne le nouveau niveau
5. Le système demande une justification obligatoire du changement
6. L'acteur saisit la justification
7. Le système met à jour le niveau de confidentialité
8. Le système crée une entrée dans l'historique de classification avec l'ancien et le nouveau niveau
9. Le système recalcule les règles d'accès et révoque les accès devenus incompatibles (si passage à un niveau supérieur)
10. Une notification est envoyée aux utilisateurs dont l'accès a été révoqué
11. La modification est journalisée

**Scénarios alternatifs :**
- **9a.** Passage d'un niveau supérieur à un niveau inférieur (déclassification) → le système envoie une alerte aux archivistes pour validation si le document contient des données sensibles
- **9b.** Passage à "Secret" → le système demande de définir immédiatement la liste blanche des utilisateurs autorisés

---

### UC-CONF-05 — Accorder un accès explicite à un document confidentiel

**Acteur :** Propriétaire du document ou Responsable  
**Préconditions :** Le document est classifié "Confidentiel" ou "Secret". L'acteur a la permission d'accorder des accès  
**Postconditions :** Un utilisateur spécifique obtient l'accès au document

**Scénario principal :**
1. L'acteur consulte le document
2. L'acteur accède à la section "Gestion des accès"
3. L'acteur recherche l'utilisateur à autoriser (par nom, email ou service)
4. L'acteur sélectionne l'utilisateur
5. L'acteur définit les droits accordés : lecture seule, lecture + téléchargement, lecture + modification (si applicable)
6. L'acteur peut définir une date d'expiration optionnelle de l'accès
7. L'acteur saisit une justification obligatoire
8. Le système crée l'autorisation d'accès
9. Une notification est envoyée à l'utilisateur autorisé
10. L'événement est journalisé

**Règles métier :**
- Un accès explicite ne peut jamais donner plus de droits que ceux du rôle de l'utilisateur
- Un accès temporaire expire automatiquement à la date définie
- L'utilisateur autorisé ne peut pas redistribuer cet accès à d'autres personnes

---

### UC-CONF-09 — Déclassifier un document

**Acteur :** Archiviste ou Responsable  
**Préconditions :** Le document existe et est classifié à un niveau supérieur à "Public"  
**Postconditions :** Le document passe à un niveau de confidentialité inférieur

**Scénario principal :**
1. L'acteur consulte le document
2. L'acteur déclenche la déclassification
3. Le système affiche un avertissement sur les implications (plus d'utilisateurs auront accès)
4. L'acteur sélectionne le nouveau niveau (inférieur)
5. L'acteur saisit une justification détaillée obligatoire
6. Si le document contient des données personnelles (flag RGPD activé), le système demande confirmation supplémentaire et alerte le DPO
7. Le système met à jour le niveau de confidentialité
8. Le système élargit les accès selon les nouvelles règles
9. Une notification est envoyée aux archivistes et au propriétaire
10. La déclassification est journalisée

**Règles métier :**
- La déclassification d'un document "Secret" vers "Public" nécessite une double validation (Archiviste + Admin)
- Un document contenant des données personnelles ne peut jamais être déclassifié en "Public" sans anonymisation préalable

---

### UC-CONF-12 — Définir une politique de déclassification automatique

**Acteur :** Super Admin ou Admin  
**Préconditions :** L'acteur est authentifié avec les permissions appropriées  
**Postconditions :** Une politique de déclassification automatique est active

**Scénario principal :**
1. L'acteur accède à la gestion des politiques de confidentialité
2. L'acteur crée une nouvelle politique de déclassification automatique
3. L'acteur définit les paramètres :
   - Type de document concerné (ou catégorie)
   - Niveau de confidentialité actuel
   - Niveau cible après déclassification
   - Délai avant déclassification (en années depuis la création du document)
   - Conditions d'application (ex: uniquement si le document n'a pas été consulté depuis X mois)
4. Le système enregistre la politique
5. Une tâche Celery planifiée appliquera cette politique automatiquement selon le calendrier défini

**Exemple concret :**
> "Tous les rapports d'activité classifiés 'Confidentiel' passent automatiquement à 'Interne' après 5 ans, sauf s'ils contiennent des données personnelles."

---

### UC-CONF-15 — Marquer un document comme contenant des données personnelles (RGPD)

**Acteur :** Archiviste ou Responsable  
**Préconditions :** Le document existe  
**Postconditions :** Le document est marqué comme contenant des données personnelles et soumis aux obligations RGPD

**Scénario principal :**
1. L'acteur consulte le document
2. L'acteur active le flag "Contient des données personnelles"
3. Le système demande de préciser les catégories de données personnelles présentes (liste prédéfinie : identité, santé, données financières, opinions politiques, etc.)
4. Le système demande la base légale du traitement (consentement, contrat, obligation légale, intérêt légitime)
5. Le système demande la durée de conservation spécifique RGPD
6. Le système applique automatiquement le niveau de confidentialité minimum "Confidentiel" si le document était à un niveau inférieur
7. Le système active les protections RGPD : interdiction de partage externe sans anonymisation, droit à l'effacement activé, journalisation renforcée
8. Une notification est envoyée au DPO
9. Le marquage est journalisé

---

## 6. Modèles de données

### 6.1 Modèle `ConfidentialityLevel` (Niveau de confidentialité — référentiel système)

**Table :** `confidentiality_levels`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `code` | VARCHAR(30) | UNIQUE, NOT NULL | Code technique (`public`, `internal`, `confidential`, `secret`) |
| `name` | VARCHAR(100) | NOT NULL | Nom affiché |
| `description` | TEXT | NULL | Description détaillée |
| `level_rank` | SMALLINT | UNIQUE, NOT NULL | Rang hiérarchique (1=Public, 2=Interne, 3=Confidentiel, 4=Secret) |
| `color_hex` | VARCHAR(7) | NOT NULL | Couleur d'affichage |
| `icon` | VARCHAR(50) | NULL | Icône associée |
| `is_system` | BOOLEAN | NOT NULL, DEFAULT TRUE | Niveau système (non modifiable) |
| `requires_justification` | BOOLEAN | NOT NULL, DEFAULT FALSE | Justification obligatoire à l'assignation |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |

**Règles métier :**
- Les 4 niveaux système sont créés à l'initialisation et ne peuvent pas être supprimés
- Le `level_rank` permet les comparaisons : un utilisateur autorisé à accéder au niveau 3 peut accéder aux niveaux 1, 2 et 3, mais pas 4

---

### 6.2 Modèle `DocumentClassification` (Classification d'un document)

**Table :** `document_classifications`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_id` | UUID | FK → documents.id, NOT NULL, INDEX | Document classifié |
| `confidentiality_level_id` | UUID | FK → confidentiality_levels.id, NOT NULL | Niveau de confidentialité |
| `classification_date` | TIMESTAMP | NOT NULL, AUTO | Date de la classification |
| `classified_by_id` | UUID | FK → users.id, NOT NULL | Qui a classifié |
| `justification` | TEXT | NULL | Justification (obligatoire pour Confidentiel/Secret) |
| `is_current` | BOOLEAN | NOT NULL, DEFAULT TRUE | Classification actuelle (seule une ligne par document avec TRUE) |
| `automatic_declassification_date` | DATE | NULL | Date de déclassification automatique prévue |

**Index :**
- `idx_document_classifications_document_id` sur `document_id`
- `idx_document_classifications_is_current` sur `is_current`

**Contrainte :** Un seul enregistrement avec `is_current = TRUE` par document (`UNIQUE (document_id) WHERE is_current = TRUE`)

---

### 6.3 Modèle `DocumentAccessGrant` (Accès explicite à un document)

**Table :** `document_access_grants`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_id` | UUID | FK → documents.id, NOT NULL, INDEX | Document |
| `user_id` | UUID | FK → users.id, NOT NULL, INDEX | Utilisateur autorisé |
| `granted_by_id` | UUID | FK → users.id, NOT NULL | Qui a accordé l'accès |
| `access_level` | VARCHAR(20) | NOT NULL | Niveau d'accès : `read`, `read_download`, `read_write` |
| `justification` | TEXT | NOT NULL | Justification obligatoire |
| `granted_at` | TIMESTAMP | NOT NULL, AUTO | Date d'octroi |
| `expires_at` | TIMESTAMP | NULL | Date d'expiration (NULL = permanent) |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Accès actif |
| `revoked_at` | TIMESTAMP | NULL | Date de révocation |
| `revoked_by_id` | UUID | FK → users.id, NULL | Qui a révoqué |

**Index :**
- `idx_document_access_grants_document_id` sur `document_id`
- `idx_document_access_grants_user_id` sur `user_id`
- `idx_document_access_grants_is_active` sur `is_active`

**Contrainte :** `UNIQUE (document_id, user_id) WHERE is_active = TRUE` — un utilisateur ne peut avoir qu'un seul accès actif par document

---

### 6.4 Modèle `CategoryDefaultConfidentiality` (Niveau par défaut par catégorie)

**Table :** `category_default_confidentiality`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `category_id` | UUID | FK → document_categories.id, UNIQUE, NOT NULL | Catégorie |
| `confidentiality_level_id` | UUID | FK → confidentiality_levels.id, NOT NULL | Niveau par défaut |
| `inherit_to_subcategories` | BOOLEAN | NOT NULL, DEFAULT TRUE | Héritage aux sous-catégories |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `created_by_id` | UUID | FK → users.id, NULL | Créateur |

---

### 6.5 Modèle `DeclassificationPolicy` (Politique de déclassification automatique)

**Table :** `declassification_policies`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `name` | VARCHAR(200) | NOT NULL | Nom de la politique |
| `description` | TEXT | NULL | Description |
| `source_level_id` | UUID | FK → confidentiality_levels.id, NOT NULL | Niveau actuel |
| `target_level_id` | UUID | FK → confidentiality_levels.id, NOT NULL | Niveau cible |
| `trigger_years_after_creation` | SMALLINT | NOT NULL | Délai avant déclassification (années) |
| `applies_to_category_id` | UUID | FK → document_categories.id, NULL | Catégorie concernée (NULL = tous) |
| `applies_to_document_type_id` | UUID | FK → document_types.id, NULL | Type concerné (NULL = tous) |
| `exclude_if_contains_personal_data` | BOOLEAN | NOT NULL, DEFAULT TRUE | Exclure si données personnelles |
| `exclude_if_accessed_within_months` | SMALLINT | NULL | Exclure si consulté récemment (mois) |
| `requires_manual_validation` | BOOLEAN | NOT NULL, DEFAULT FALSE | Validation manuelle avant application |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Politique active |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `created_by_id` | UUID | FK → users.id, NULL | Créateur |

---

### 6.6 Modèle `PersonalDataMarking` (Marquage RGPD)

**Table :** `personal_data_markings`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_id` | UUID | FK → documents.id, UNIQUE, NOT NULL | Document marqué |
| `data_categories` | ARRAY VARCHAR | NOT NULL | Catégories de données (ex: `['identity', 'health']`) |
| `legal_basis` | VARCHAR(50) | NOT NULL | Base légale : `consent`, `contract`, `legal_obligation`, `legitimate_interest` |
| `retention_period_months` | SMALLINT | NOT NULL | Durée de conservation RGPD (mois) |
| `anonymization_required` | BOOLEAN | NOT NULL, DEFAULT FALSE | Anonymisation requise avant partage externe |
| `right_to_erasure_enabled` | BOOLEAN | NOT NULL, DEFAULT TRUE | Droit à l'effacement activé |
| `notes` | TEXT | NULL | Notes complémentaires |
| `marked_at` | TIMESTAMP | NOT NULL, AUTO | Date du marquage |
| `marked_by_id` | UUID | FK → users.id, NOT NULL | Qui a marqué |

**Catégories de données personnelles prédéfinies :**
- `identity` : Nom, prénom, adresse, téléphone, email
- `health` : Données de santé
- `financial` : Données bancaires, revenus
- `biometric` : Empreintes, reconnaissance faciale
- `political_opinions` : Opinions politiques, syndicales
- `ethnic_origin` : Origine ethnique
- `religious_beliefs` : Convictions religieuses
- `sexual_orientation` : Orientation sexuelle

---

## 7. Matrice des permissions du module

| Permission code | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `confidentiality.classify.public` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `confidentiality.classify.internal` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `confidentiality.classify.confidential` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `confidentiality.classify.secret` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `confidentiality.modify_classification` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `confidentiality.grant_access` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `confidentiality.revoke_access` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `confidentiality.view_access_grants` | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| `confidentiality.declassify` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `confidentiality.create_policy` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `confidentiality.mark_personal_data` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `confidentiality.view_personal_data_list` | ✅ | ✅ | ✅ (DPO) | ❌ | ❌ | ✅ |
| `confidentiality.export_audit` | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |

---

## 8. Séquences détaillées des flux principaux

### 8.1 Séquence — Classification d'un document à la création

```
Acteur              Frontend React         API Django            Base de données
  |                       |                     |                       |
  |-- crée document ----->|                     |                       |
  |-- remplit formulaire ->|                    |                       |
  |                       |-- GET /taxonomy/ --->|                      |
  |                       |   document-types/{id}/|-- SELECT type ----->|
  |                       |                     |<-- type + default_conf|
  |<-- suggestion level --|                     |                       |
  |   (ex: Confidentiel)  |                     |                       |
  |                       |                     |                       |
  |-- accepte ou modifie ->|                    |                       |
  |-- saisit justification>|                    |                       |
  |                       |-- POST /documents/ ->|                      |
  |                       |                     |-- vérifier permission |
  |                       |                     |   selon level choisi  |
  |                       |                     |-- INSERT document --->|
  |                       |                     |-- INSERT classification>|
  |                       |                     |   (is_current=TRUE)   |
  |                       |                     |-- INSERT audit ------->|
  |                       |<-- 201 + document ---|                      |
  |<-- document créé ------|                     |                       |
```

### 8.2 Séquence — Accorder un accès explicite

```
Propriétaire        Frontend React         API Django            Base de données     Notifications
    |                     |                     |                       |                 |
    |-- consulte doc ----->|                     |                       |                 |
    |-- clique "Accorder  >|                     |                       |                 |
    |   accès"             |                     |                       |                 |
    |<-- recherche user ---|                     |                       |                 |
    |                     |-- GET /users/ ------>|                       |                 |
    |                     |   ?search=...        |-- SELECT users ------>|                 |
    |<-- liste users ------|                     |                       |                 |
    |                     |                     |                       |                 |
    |-- sélectionne user ->|                    |                       |                 |
    |-- définit droits ---->|                    |                       |                 |
    |-- justification ------>|                   |                       |                 |
    |                     |-- POST /confidentiality/>                    |                 |
    |                     |   access-grants/     |-- vérifier permission |                 |
    |                     |                     |-- INSERT grant ------->|                 |
    |                     |                     |-- INSERT audit ------->|                 |
    |                     |                     |-- créer notification --------------------->|
    |                     |<-- 201 + grant ------|                       |                 |
    |<-- confirmation -----|                     |                       |                 |
```

### 8.3 Séquence — Déclassification automatique (tâche Celery)

```
Celery Worker         API Django            Base de données       Notifications
    |                     |                       |                       |
    |-- cron hebdomadaire >|                       |                       |
    |                     |-- SELECT policies ---->|                      |
    |                     |   WHERE is_active=TRUE|                       |
    |                     |<-- liste policies -----|                      |
    |                     |                       |                       |
    |-- pour chaque policy>|                      |                       |
    |                     |-- SELECT documents --->|                      |
    |                     |   éligibles selon     |                       |
    |                     |   critères policy     |                       |
    |                     |<-- liste documents ----|                      |
    |                     |                       |                       |
    |-- pour chaque doc -->|                      |                       |
    |                     |-- vérifier exclusions |                       |
    |                     |   (données perso,     |                       |
    |                     |   accès récent)       |                       |
    |                     |-- si éligible :       |                       |
    |                     |-- UPDATE classification>                      |
    |                     |   (is_current=FALSE)  |                       |
    |                     |-- INSERT classification>                      |
    |                     |   (new level, current)|                      |
    |                     |-- recalculer accès -->|                      |
    |                     |-- INSERT audit ------->|                      |
    |                     |-- créer notification ------------------------->|
    |                     |   (archiviste + proprio)|                    |
```

---

## 9. Endpoints API du module

### 9.1 Niveaux de confidentialité

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/confidentiality/levels/` | Liste des niveaux | Oui |
| GET | `/api/v1/confidentiality/levels/{id}/` | Détail d'un niveau | Oui |

### 9.2 Classification

| Méthode | URL | Description | Auth |
|---|---|---|---|
| POST | `/api/v1/confidentiality/documents/{doc_id}/classify/` | Classifier/reclassifier | Oui + Permission |
| GET | `/api/v1/confidentiality/documents/{doc_id}/classification/` | Classification actuelle | Oui |
| GET | `/api/v1/confidentiality/documents/{doc_id}/classification/history/` | Historique | Oui |

### 9.3 Accès explicites

| Méthode | URL | Description | Auth |
|---|---|---|---|
| POST | `/api/v1/confidentiality/access-grants/` | Accorder un accès | Oui + Permission |
| GET | `/api/v1/confidentiality/documents/{doc_id}/access-grants/` | Liste des accès | Oui + Permission |
| DELETE | `/api/v1/confidentiality/access-grants/{id}/` | Révoquer un accès | Oui + Permission |
| PATCH | `/api/v1/confidentiality/access-grants/{id}/` | Modifier un accès | Oui + Permission |

### 9.4 Politiques de déclassification

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/confidentiality/declassification-policies/` | Liste des politiques | Oui + Permission |
| POST | `/api/v1/confidentiality/declassification-policies/` | Créer une politique | Oui + Permission |
| PATCH | `/api/v1/confidentiality/declassification-policies/{id}/` | Modifier | Oui + Permission |
| DELETE | `/api/v1/confidentiality/declassification-policies/{id}/` | Supprimer | Oui + Permission |

### 9.5 Marquage RGPD

| Méthode | URL | Description | Auth |
|---|---|---|---|
| POST | `/api/v1/confidentiality/documents/{doc_id}/mark-personal-data/` | Marquer RGPD | Oui + Permission |
| GET | `/api/v1/confidentiality/personal-data/` | Liste docs avec données perso | Oui + Permission |
| DELETE | `/api/v1/confidentiality/documents/{doc_id}/personal-data-marking/` | Retirer marquage | Oui + Permission |

---

## 10. Règles métier critiques

### 10.1 Règle de non-régression de confidentialité

Une fois qu'un document a été classifié à un niveau donné, **il ne peut jamais automatiquement passer à un niveau inférieur sans intervention humaine et justification explicite**. Même les politiques de déclassification automatique nécessitent une validation manuelle si `requires_manual_validation = TRUE`.

### 10.2 Règle d'héritage de confidentialité

Si un document possède un document parent (ex: un contrat avec des avenants), les documents enfants héritent automatiquement du niveau de confidentialité du parent au minimum. Ils peuvent avoir un niveau supérieur, mais jamais inférieur.

### 10.3 Règle RGPD absolue

Un document marqué comme contenant des données personnelles :
- Ne peut **jamais** être classifié en "Public"
- Doit **obligatoirement** être au minimum "Confidentiel"
- Ne peut **jamais** être partagé à l'extérieur de l'organisation sans anonymisation préalable
- Est soumis au droit à l'effacement RGPD (sur demande de la personne concernée)

### 10.4 Règle du marquage visuel obligatoire

Chaque document doit afficher visiblement son niveau de confidentialité :
- Badge coloré dans l'interface
- Mention en en-tête lors de l'impression (si impression autorisée)
- Filigrane sur les PDF téléchargés
- Mention dans les emails de notification concernant le document

---

## 11. Tâches Celery planifiées

| Tâche | Fréquence | Description |
|---|---|---|
| `confidentiality.process_declassifications` | Hebdomadaire (dimanche 2h) | Applique les politiques de déclassification automatique |
| `confidentiality.expire_temporary_grants` | Quotidien (3h) | Révoque les accès explicites temporaires arrivés à expiration |
| `confidentiality.alert_inconsistent_classifications` | Hebdomadaire (lundi 9h) | Détecte les classifications incohérentes (ex: document public contenant des données personnelles) |
| `confidentiality.audit_secret_documents` | Mensuel (1er du mois) | Génère un rapport des documents "Secret" et de leurs accès |

---

## 12. Événements journalisés (MODULE 11 — Audit)

| Événement | Niveau | Détails |
|---|---|---|
| Document classifié | INFO | document_id, level, classified_by, justification |
| Classification modifiée | WARNING | document_id, old_level, new_level, modified_by, justification |
| Accès explicite accordé | INFO | document_id, user_id, granted_by, access_level, justification |
| Accès explicite révoqué | INFO | document_id, user_id, revoked_by |
| Déclassification automatique | INFO | document_id, old_level, new_level, policy_id |
| Document marqué RGPD | WARNING | document_id, data_categories, marked_by |
| Tentative d'accès à document secret refusée | WARNING | document_id, user_id, timestamp, reason |
| Politique de déclassification créée | INFO | policy_id, created_by |

---

## 13. Notifications générées (MODULE 12)

| Déclencheur | Destinataire | Canal | Message |
|---|---|---|
| Accès explicite accordé | Utilisateur autorisé | Email + In-app | Vous avez reçu l'accès au document [X] |
| Accès révoqué | Utilisateur concerné | In-app | Votre accès au document [X] a été révoqué |
| Document déclassifié | Propriétaire + Archivistes | In-app | Le document [X] a été déclassifié en [Niveau] |
| Classification incohérente détectée | Archivistes | In-app | Incohérence détectée : [détails] |
| Document marqué RGPD | DPO | Email | Nouveau document avec données personnelles : [X] |
| Accès temporaire expire dans 7 jours | Utilisateur | In-app | Votre accès au document [X] expire dans 7 jours |

---

*Fin du MODULE 05 — Confidentialité & Classification de Sécurité*  
*Prochain module : MODULE 06 — Gestion des documents & Métadonnées*
