# MODULE 06 — Gestion des Documents & Métadonnées

## Projet : Système d'Archivage Numérique des Documents
**Version :** 1.0  
**Dépendances :** MODULE 01 — Architecture | MODULE 02 — Authentification | MODULE 03 — Taxonomie | MODULE 04 — Cycle de vie | MODULE 05 — Confidentialité  
**Statut :** CŒUR MÉTIER — Tous les modules suivants (07 à 18) dépendent de ce module

---

## 1. Présentation du module

Ce module est le **cœur battant** de toute l'application. Il définit ce qu'est un document dans le système, comment il est créé, stocké, modifié, versionné, lié à d'autres documents, et comment ses métadonnées sont gérées. Sans ce module, l'application n'existe pas.

Un document dans ce système n'est pas qu'un simple fichier : c'est une **entité documentaire complexe** qui possède :
- Un fichier physique (ou plusieurs dans le cas de versions successives)
- Des métadonnées riches (standard + personnalisées selon le type de document)
- Un positionnement dans la taxonomie (catégorie + type)
- Un statut dans son cycle de vie
- Un niveau de confidentialité
- Un hash d'intégrité SHA-256
- Un historique complet de toutes les modifications
- Des relations avec d'autres documents
- Des commentaires et annotations
- Des pièces jointes complémentaires

Ce module orchestre l'ensemble de ces dimensions et garantit la cohérence globale du système documentaire.

---

## 2. Cas d'utilisation — Vue d'ensemble

### 2.1 Gestion de base des documents

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-DOC-01 | Créer un document (upload fichier) | Archiviste / Responsable / Agent |
| UC-DOC-02 | Créer un document sans fichier (métadonnées seules) | Archiviste / Responsable |
| UC-DOC-03 | Consulter le détail d'un document | Tout utilisateur (selon permissions) |
| UC-DOC-04 | Modifier les métadonnées d'un document | Archiviste / Responsable / Propriétaire |
| UC-DOC-05 | Télécharger un document | Tout utilisateur (selon permissions) |
| UC-DOC-06 | Remplacer le fichier d'un document (nouvelle version) | Archiviste / Responsable / Propriétaire |
| UC-DOC-07 | Supprimer logiquement un document | Archiviste / Responsable |
| UC-DOC-08 | Restaurer un document supprimé logiquement | Archiviste / Admin |
| UC-DOC-09 | Supprimer physiquement un document | Super Admin uniquement (après élimination légale) |
| UC-DOC-10 | Dupliquer un document | Archiviste / Responsable |
| UC-DOC-11 | Déplacer un document vers une autre catégorie | Archiviste / Responsable |
| UC-DOC-12 | Changer le type d'un document | Archiviste |

### 2.2 Versioning

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-DOC-13 | Consulter l'historique des versions d'un document | Tout utilisateur (selon permissions) |
| UC-DOC-14 | Restaurer une version antérieure | Archiviste / Responsable |
| UC-DOC-15 | Comparer deux versions d'un document | Tout utilisateur (selon permissions) |
| UC-DOC-16 | Télécharger une version spécifique | Tout utilisateur (selon permissions) |
| UC-DOC-17 | Ajouter une note de version | Archiviste / Responsable / Propriétaire |

### 2.3 Métadonnées

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-DOC-18 | Remplir les métadonnées standard | Archiviste / Responsable / Agent |
| UC-DOC-19 | Remplir les métadonnées personnalisées (selon le type) | Archiviste / Responsable / Agent |
| UC-DOC-20 | Modifier les métadonnées personnalisées | Archiviste / Responsable / Propriétaire |
| UC-DOC-21 | Valider automatiquement la complétude des métadonnées | Système (automatique) |
| UC-DOC-22 | Consulter l'historique de modification des métadonnées | Archiviste / Admin / Auditeur |

### 2.4 Relations entre documents

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-DOC-23 | Créer une relation entre deux documents | Archiviste / Responsable |
| UC-DOC-24 | Supprimer une relation entre documents | Archiviste / Responsable |
| UC-DOC-25 | Consulter les documents liés | Tout utilisateur (selon permissions) |
| UC-DOC-26 | Définir un document parent | Archiviste / Responsable |
| UC-DOC-27 | Consulter l'arborescence parent/enfants | Tout utilisateur (selon permissions) |

### 2.5 Pièces jointes

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-DOC-28 | Ajouter une pièce jointe à un document | Archiviste / Responsable / Propriétaire |
| UC-DOC-29 | Télécharger une pièce jointe | Tout utilisateur (selon permissions) |
| UC-DOC-30 | Supprimer une pièce jointe | Archiviste / Responsable / Propriétaire |
| UC-DOC-31 | Lister les pièces jointes d'un document | Tout utilisateur (selon permissions) |

### 2.6 Commentaires et annotations

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-DOC-32 | Ajouter un commentaire sur un document | Tout utilisateur (selon permissions) |
| UC-DOC-33 | Modifier son propre commentaire | Auteur du commentaire |
| UC-DOC-34 | Supprimer un commentaire | Auteur / Archiviste / Admin |
| UC-DOC-35 | Répondre à un commentaire (thread) | Tout utilisateur (selon permissions) |
| UC-DOC-36 | Consulter les commentaires d'un document | Tout utilisateur (selon permissions) |
| UC-DOC-37 | Mentionner un utilisateur dans un commentaire (@mention) | Tout utilisateur (selon permissions) |

### 2.7 Gestion des doublons

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-DOC-38 | Détecter automatiquement les doublons potentiels | Système (automatique) |
| UC-DOC-39 | Consulter les doublons détectés | Archiviste |
| UC-DOC-40 | Marquer deux documents comme doublons | Archiviste |
| UC-DOC-41 | Fusionner deux documents doublons | Archiviste |
| UC-DOC-42 | Déclarer un document comme version principale | Archiviste |

### 2.8 Priorité et statut de traitement

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-DOC-43 | Définir la priorité de traitement d'un document | Archiviste / Responsable |
| UC-DOC-44 | Modifier le statut de traitement | Archiviste / Responsable |
| UC-DOC-45 | Consulter les documents urgents | Archiviste / Responsable |
| UC-DOC-46 | Assigner un document à un utilisateur | Archiviste / Responsable |

### 2.9 Consultation et liste

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-DOC-47 | Consulter la liste des documents (avec filtres) | Tout utilisateur |
| UC-DOC-48 | Trier les documents par différents critères | Tout utilisateur |
| UC-DOC-49 | Exporter une liste de documents (CSV, Excel) | Archiviste / Admin |
| UC-DOC-50 | Consulter les documents récents | Tout utilisateur |
| UC-DOC-51 | Consulter les documents consultés récemment (historique personnel) | Tout utilisateur |
| UC-DOC-52 | Marquer un document comme favori | Tout utilisateur |
| UC-DOC-53 | Consulter ses documents favoris | Tout utilisateur |

---

## 3. Description détaillée des cas d'utilisation (sélection critique)

### UC-DOC-01 — Créer un document (upload fichier)

**Acteur :** Archiviste, Responsable ou Agent (selon permissions)  
**Préconditions :** L'acteur est authentifié. Le fichier respecte les contraintes de taille et de format  
**Postconditions :** Le document est créé avec toutes ses métadonnées et son fichier est stocké de manière sécurisée

**Scénario principal :**
1. L'acteur accède à la fonction de création de document
2. L'acteur sélectionne le type de document (obligatoire)
3. Le système affiche le formulaire de métadonnées correspondant (métadonnées standard + métadonnées personnalisées du type via le schema JSONB)
4. L'acteur remplit les métadonnées obligatoires :
   - Titre (obligatoire)
   - Catégorie (pré-remplie selon le type, modifiable)
   - Type de document (déjà sélectionné)
   - Date du document (date officielle du document, peut être différente de la date de création)
   - Service émetteur / département
   - Auteur ou créateur du document physique (différent de l'utilisateur qui archive)
   - Langue
   - Niveau de confidentialité (suggéré selon le type, modifiable)
   - Priorité de traitement (Normal par défaut)
5. L'acteur remplit les métadonnées personnalisées selon le schéma du type (ex: pour un contrat → numéro de contrat, parties, montant)
6. L'acteur uploade le fichier principal
7. Le système valide le format du fichier selon les extensions autorisées du type de document
8. Le système valide la taille du fichier (max configuré globalement + max spécifique au type)
9. Le système calcule le hash SHA-256 du fichier pour garantir l'intégrité future
10. Le système détecte automatiquement le type MIME du fichier
11. Le système renomme le fichier en UUID v4 sur le disque (le nom original est conservé en métadonnée)
12. Le système stocke le fichier dans l'arborescence média appropriée (par année/mois)
13. Le système génère automatiquement une miniature (thumbnail) si le fichier est une image ou un PDF
14. Le système calcule automatiquement la date d'échéance selon la politique de conservation du type (MODULE 04)
15. Le système crée le document en base de données avec le statut du cycle de vie "Actif"
16. Le système crée la première version (v1.0) du document
17. Le système crée l'entrée initiale dans l'historique de modification
18. Si le module OCR est activé, le système déclenche une tâche Celery asynchrone de traitement OCR (MODULE 07)
19. Si le module IA est activé, le système déclenche une tâche de classification automatique (MODULE 10)
20. L'événement de création est journalisé (MODULE 11)
21. Le document est retourné à l'utilisateur avec son identifiant unique

**Scénarios alternatifs :**
- **7a.** Le format n'est pas autorisé → le système refuse l'upload et affiche les formats acceptés
- **8a.** La taille dépasse la limite → le système refuse l'upload et affiche la taille maximale
- **9a.** Le hash SHA-256 correspond à un document existant → le système détecte un potentiel doublon et alerte l'utilisateur (UC-DOC-38)
- **5a.** Les métadonnées personnalisées obligatoires ne sont pas remplies → le système bloque la création et affiche les champs manquants
- **18a.** Le fichier n'est pas un format supporté par l'OCR → la tâche OCR est ignorée

**Règles métier :**
- Le titre est obligatoire et doit être unique au sein de la même catégorie (avertissement si doublon, pas de blocage)
- Le niveau de confidentialité ne peut jamais être vide — si non spécifié, le niveau par défaut du type est appliqué
- Le propriétaire du document est automatiquement l'utilisateur qui l'a créé
- Chaque document reçoit automatiquement un numéro d'enregistrement unique séquentiel par année (ex: 2026-00001)

---

### UC-DOC-06 — Remplacer le fichier d'un document (nouvelle version)

**Acteur :** Archiviste, Responsable ou Propriétaire  
**Préconditions :** Le document existe. L'acteur a la permission de modifier ce document  
**Postconditions :** Une nouvelle version du document est créée, l'ancienne version est conservée

**Scénario principal :**
1. L'acteur consulte le document
2. L'acteur déclenche l'action "Nouvelle version"
3. Le système affiche le formulaire de versioning
4. L'acteur sélectionne le nouveau fichier
5. L'acteur choisit le type de version :
   - Version mineure (ex: 1.0 → 1.1) : corrections mineures, ajustements
   - Version majeure (ex: 1.1 → 2.0) : modifications substantielles
6. L'acteur saisit une note de version obligatoire décrivant les changements
7. L'acteur peut optionnellement modifier les métadonnées si nécessaire
8. Le système valide le nouveau fichier (format, taille)
9. Le système calcule le hash SHA-256 du nouveau fichier
10. Le système conserve l'ancien fichier sans le supprimer
11. Le système crée la nouvelle version en incrémentant le numéro de version
12. Le système marque la nouvelle version comme version actuelle
13. Le système met à jour les métadonnées si modifiées
14. Si le fichier a changé de format, le système met à jour le type MIME
15. Le système régénère la miniature si applicable
16. Le système déclenche un nouveau traitement OCR si le contenu a changé
17. L'événement de versioning est journalisé
18. Une notification est envoyée aux utilisateurs qui suivent ce document (si la fonctionnalité de suivi existe)

**Règles métier :**
- Toutes les versions antérieures sont conservées indéfiniment et restent téléchargeables
- Le numéro de version suit la convention Semantic Versioning adaptée : X.Y où X = version majeure, Y = version mineure
- Chaque version possède son propre hash SHA-256
- La date de dernière modification du document est mise à jour
- Le hash du document principal dans le modèle Document est mis à jour avec celui de la dernière version

---

### UC-DOC-23 — Créer une relation entre deux documents

**Acteur :** Archiviste ou Responsable  
**Préconditions :** Les deux documents existent. L'acteur a accès aux deux documents  
**Postconditions :** Une relation bidirectionnelle est créée entre les deux documents

**Scénario principal :**
1. L'acteur consulte le document source
2. L'acteur accède à la section "Relations"
3. L'acteur clique sur "Ajouter une relation"
4. L'acteur recherche le document cible (par titre, référence, numéro)
5. L'acteur sélectionne le document cible
6. L'acteur choisit le type de relation :
   - `related_to` : lié à (relation générique)
   - `replaces` : remplace (le document source remplace le document cible)
   - `replaced_by` : remplacé par (inverse de replaces)
   - `amends` : modifie (le document source est un avenant du document cible)
   - `amended_by` : modifié par (inverse de amends)
   - `references` : référence (le document source cite le document cible)
   - `referenced_by` : référencé par (inverse de references)
   - `annexe_of` : annexe de (le document source est une annexe du document cible)
7. L'acteur peut ajouter une note explicative optionnelle sur la nature de la relation
8. Le système crée la relation dans le sens choisi
9. Le système crée automatiquement la relation inverse (ex: si A replaces B, alors B replaced_by A)
10. La création est journalisée

**Règles métier :**
- Une relation entre deux mêmes documents ne peut exister qu'une seule fois par type
- Les relations sont toujours bidirectionnelles (création automatique de la relation inverse)
- La suppression d'une relation supprime automatiquement son inverse
- Les relations sont affichées dans la fiche du document avec le type et le document lié

---

### UC-DOC-38 — Détecter automatiquement les doublons potentiels

**Acteur :** Système (déclenché automatiquement à la création d'un document)  
**Préconditions :** Un document vient d'être créé ou modifié  
**Postconditions :** Les doublons potentiels sont identifiés et signalés

**Scénario principal :**
1. Le système calcule le hash SHA-256 du fichier uploadé
2. Le système recherche dans la base de données si un autre document possède le même hash
3. Si un hash identique existe : le système crée une alerte de doublon exact (fichier identique)
4. Le système compare également le titre du document avec les titres existants dans la même catégorie (similarité textuelle > 80%)
5. Si des titres similaires existent : le système crée une alerte de doublon potentiel (même titre mais fichier différent)
6. Le système analyse le contenu OCR (si disponible) et compare avec les autres documents de la même catégorie (similarité de contenu > 90%)
7. Si un contenu similaire existe : le système crée une alerte de doublon de contenu
8. Le système affiche toutes les alertes à l'utilisateur et lui demande de confirmer s'il s'agit vraiment d'un document différent
9. Si l'utilisateur confirme qu'il ne s'agit pas d'un doublon, le système enregistre la décision et n'alerte plus sur cette combinaison
10. Si l'utilisateur identifie un doublon, il peut déclencher UC-DOC-41 (Fusionner)

---

### UC-DOC-41 — Fusionner deux documents doublons

**Acteur :** Archiviste  
**Préconditions :** Deux documents sont identifiés comme doublons  
**Postconditions :** Un seul document subsiste, l'autre est supprimé logiquement avec toutes ses données transférées

**Scénario principal :**
1. L'archiviste sélectionne les deux documents doublons
2. L'archiviste déclenche la fusion
3. Le système affiche une prévisualisation de la fusion :
   - Document principal (celui qui survivra)
   - Document secondaire (celui qui sera absorbé)
   - Métadonnées conflictuelles (le système surligne les champs différents)
4. L'archiviste choisit pour chaque métadonnée conflictuelle quelle valeur conserver
5. L'archiviste décide du sort des versions :
   - Conserver toutes les versions des deux documents
   - Conserver uniquement les versions du document principal
6. L'archiviste décide du sort des commentaires et pièces jointes :
   - Transférer tous les commentaires vers le document principal
   - Transférer toutes les pièces jointes vers le document principal
7. L'archiviste valide la fusion
8. Le système met à jour le document principal avec les métadonnées choisies
9. Le système transfère toutes les versions, commentaires, pièces jointes selon les choix
10. Le système transfère toutes les relations du document secondaire vers le document principal
11. Le système crée une entrée dans l'historique indiquant la fusion
12. Le système supprime logiquement le document secondaire
13. Le système crée un lien de redirection : toute tentative d'accès au document secondaire redirige vers le document principal avec un message explicatif
14. La fusion est journalisée avec tous les détails

---

## 4. Modèles de données

### 4.1 Modèle `Document` (Document principal)

**Table :** `documents`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `registration_number` | VARCHAR(20) | UNIQUE, NOT NULL | Numéro d'enregistrement (ex: 2026-00001) |
| `title` | VARCHAR(300) | NOT NULL, INDEX | Titre du document |
| `description` | TEXT | NULL | Description détaillée |
| `document_type_id` | UUID | FK → document_types.id, NOT NULL, INDEX | Type de document |
| `category_id` | UUID | FK → document_categories.id, NOT NULL, INDEX | Catégorie |
| `document_date` | DATE | NOT NULL | Date officielle du document |
| `author` | VARCHAR(200) | NULL | Auteur/créateur du document physique |
| `issuing_department_id` | UUID | FK → departments.id, NULL, INDEX | Service émetteur |
| `language` | VARCHAR(10) | NOT NULL, DEFAULT 'fr' | Code langue ISO 639-1 |
| `confidentiality_level_id` | UUID | FK → confidentiality_levels.id, NOT NULL | Niveau de confidentialité |
| `lifecycle_status` | VARCHAR(30) | NOT NULL, DEFAULT 'active' | Statut du cycle de vie |
| `processing_priority` | VARCHAR(20) | NOT NULL, DEFAULT 'normal' | Priorité : `urgent`, `normal`, `low` |
| `processing_status` | VARCHAR(30) | NOT NULL, DEFAULT 'pending' | Statut traitement : `pending`, `in_progress`, `completed`, `on_hold` |
| `assigned_to_id` | UUID | FK → users.id, NULL | Assigné à (pour traitement) |
| `expiry_date` | DATE | NULL | Date d'échéance (calculée depuis MODULE 04) |
| `file_path` | VARCHAR(500) | NULL | Chemin du fichier principal (version actuelle) |
| `file_name_original` | VARCHAR(255) | NULL | Nom original du fichier uploadé |
| `file_size_bytes` | BIGINT | NULL | Taille du fichier en octets |
| `file_mime_type` | VARCHAR(100) | NULL | Type MIME |
| `file_extension` | VARCHAR(10) | NULL | Extension du fichier |
| `file_hash_sha256` | VARCHAR(64) | NULL, INDEX | Hash SHA-256 du fichier (intégrité) |
| `thumbnail_path` | VARCHAR(500) | NULL | Chemin de la miniature |
| `page_count` | SMALLINT | NULL | Nombre de pages (si applicable) |
| `word_count` | INTEGER | NULL | Nombre de mots (si extrait par OCR) |
| `custom_metadata` | JSONB | NULL | Métadonnées personnalisées selon le schéma du type |
| `tags` | M2M → tags | NULL | Tags associés |
| `parent_document_id` | UUID | FK → documents.id, NULL | Document parent |
| `current_version_number` | VARCHAR(10) | NOT NULL, DEFAULT '1.0' | Numéro de version actuelle |
| `version_count` | SMALLINT | NOT NULL, DEFAULT 1 | Nombre total de versions |
| `has_personal_data` | BOOLEAN | NOT NULL, DEFAULT FALSE | Contient données personnelles (RGPD) |
| `is_favorite_for_users` | M2M → users | NULL | Utilisateurs ayant mis en favori |
| `view_count` | INTEGER | NOT NULL, DEFAULT 0 | Nombre de consultations |
| `download_count` | INTEGER | NOT NULL, DEFAULT 0 | Nombre de téléchargements |
| `last_viewed_at` | TIMESTAMP | NULL | Dernière consultation |
| `owner_id` | UUID | FK → users.id, NOT NULL | Propriétaire du document |
| `is_frozen` | BOOLEAN | NOT NULL, DEFAULT FALSE | Gelé (MODULE 04) |
| `is_locked` | BOOLEAN | NOT NULL, DEFAULT FALSE | Verrouillé en modification |
| `locked_by_id` | UUID | FK → users.id, NULL | Qui a verrouillé |
| `locked_at` | TIMESTAMP | NULL | Date de verrouillage |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | Suppression logique |
| `deleted_at` | TIMESTAMP | NULL | Date de suppression logique |
| `deleted_by_id` | UUID | FK → users.id, NULL | Qui a supprimé |
| `merged_into_id` | UUID | FK → documents.id, NULL | Fusionné dans (si doublon) |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `updated_at` | TIMESTAMP | NOT NULL, AUTO | Date de mise à jour |
| `created_by_id` | UUID | FK → users.id, NOT NULL | Qui a créé |

**Contraintes :**
- `registration_number` : format `YYYY-NNNNN` généré automatiquement et séquentiel par année
- `lifecycle_status` : valeur dans `{active, semi_active, inactive, to_be_destroyed, destroyed, permanently_archived, transfer_pending}`
- `processing_priority` : valeur dans `{urgent, normal, low}`
- `processing_status` : valeur dans `{pending, in_progress, completed, on_hold}`
- `language` : code ISO 639-1 (fr, en, es, de, etc.)
- Si `has_personal_data = TRUE`, alors `confidentiality_level_id` doit être au minimum "Confidentiel"

**Index :**
- `idx_documents_title` sur `title` (full-text search)
- `idx_documents_registration_number` sur `registration_number`
- `idx_documents_document_type_id` sur `document_type_id`
- `idx_documents_category_id` sur `category_id`
- `idx_documents_file_hash_sha256` sur `file_hash_sha256` (détection doublons)
- `idx_documents_lifecycle_status` sur `lifecycle_status`
- `idx_documents_processing_priority` sur `processing_priority`
- `idx_documents_expiry_date` sur `expiry_date`
- `idx_documents_owner_id` sur `owner_id`
- `idx_documents_is_deleted` sur `is_deleted`
- `idx_documents_document_date` sur `document_date`

---

### 4.2 Modèle `DocumentVersion` (Version de document)

**Table :** `document_versions`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_id` | UUID | FK → documents.id, NOT NULL, INDEX | Document parent |
| `version_number` | VARCHAR(10) | NOT NULL | Numéro de version (ex: 1.0, 1.1, 2.0) |
| `version_type` | VARCHAR(10) | NOT NULL | Type : `major`, `minor` |
| `version_notes` | TEXT | NULL | Notes sur les changements |
| `file_path` | VARCHAR(500) | NOT NULL | Chemin du fichier de cette version |
| `file_size_bytes` | BIGINT | NOT NULL | Taille |
| `file_hash_sha256` | VARCHAR(64) | NOT NULL | Hash SHA-256 de cette version |
| `mime_type` | VARCHAR(100) | NOT NULL | Type MIME |
| `is_current` | BOOLEAN | NOT NULL, DEFAULT FALSE | Version actuelle |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création de la version |
| `created_by_id` | UUID | FK → users.id, NOT NULL | Qui a créé cette version |

**Contrainte :** `UNIQUE (document_id, version_number)`

**Index :**
- `idx_document_versions_document_id` sur `document_id`
- `idx_document_versions_is_current` sur `is_current`

---

### 4.3 Modèle `DocumentRelation` (Relation entre documents)

**Table :** `document_relations`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `source_document_id` | UUID | FK → documents.id, NOT NULL, INDEX | Document source |
| `target_document_id` | UUID | FK → documents.id, NOT NULL, INDEX | Document cible |
| `relation_type` | VARCHAR(30) | NOT NULL | Type de relation |
| `notes` | TEXT | NULL | Notes sur la relation |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `created_by_id` | UUID | FK → users.id, NOT NULL | Qui a créé la relation |

**Contraintes :**
- `UNIQUE (source_document_id, target_document_id, relation_type)` — pas de doublon
- `relation_type` : valeur dans `{related_to, replaces, replaced_by, amends, amended_by, references, referenced_by, annexe_of}`

**Index :**
- `idx_document_relations_source` sur `source_document_id`
- `idx_document_relations_target` sur `target_document_id`

---

### 4.4 Modèle `DocumentAttachment` (Pièce jointe)

**Table :** `document_attachments`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_id` | UUID | FK → documents.id, NOT NULL, INDEX | Document parent |
| `file_name` | VARCHAR(255) | NOT NULL | Nom du fichier |
| `file_path` | VARCHAR(500) | NOT NULL | Chemin du fichier |
| `file_size_bytes` | BIGINT | NOT NULL | Taille |
| `file_mime_type` | VARCHAR(100) | NOT NULL | Type MIME |
| `file_hash_sha256` | VARCHAR(64) | NOT NULL | Hash SHA-256 |
| `description` | TEXT | NULL | Description de la pièce jointe |
| `uploaded_at` | TIMESTAMP | NOT NULL, AUTO | Date d'upload |
| `uploaded_by_id` | UUID | FK → users.id, NOT NULL | Qui a uploadé |

**Index :**
- `idx_document_attachments_document_id` sur `document_id`

---

### 4.5 Modèle `DocumentComment` (Commentaire sur document)

**Table :** `document_comments`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_id` | UUID | FK → documents.id, NOT NULL, INDEX | Document commenté |
| `parent_comment_id` | UUID | FK → document_comments.id, NULL | Commentaire parent (si réponse) |
| `content` | TEXT | NOT NULL | Contenu du commentaire |
| `mentioned_users` | ARRAY UUID | NULL | Utilisateurs mentionnés (@mention) |
| `is_edited` | BOOLEAN | NOT NULL, DEFAULT FALSE | Commentaire modifié |
| `edited_at` | TIMESTAMP | NULL | Date de modification |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | Suppression logique |
| `deleted_at` | TIMESTAMP | NULL | Date de suppression |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `created_by_id` | UUID | FK → users.id, NOT NULL | Auteur |

**Index :**
- `idx_document_comments_document_id` sur `document_id`
- `idx_document_comments_parent_comment_id` sur `parent_comment_id`

---

### 4.6 Modèle `DocumentDuplicate` (Doublon détecté)

**Table :** `document_duplicates`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_1_id` | UUID | FK → documents.id, NOT NULL | Premier document |
| `document_2_id` | UUID | FK → documents.id, NOT NULL | Second document |
| `duplicate_type` | VARCHAR(30) | NOT NULL | Type : `exact_hash`, `similar_title`, `similar_content` |
| `similarity_score` | DECIMAL(5,2) | NULL | Score de similarité (0-100) |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT 'pending' | Statut : `pending`, `confirmed`, `dismissed`, `merged` |
| `reviewed_by_id` | UUID | FK → users.id, NULL | Qui a examiné |
| `reviewed_at` | TIMESTAMP | NULL | Date de révision |
| `notes` | TEXT | NULL | Notes de l'archiviste |
| `detected_at` | TIMESTAMP | NOT NULL, AUTO | Date de détection |

**Contrainte :** `UNIQUE (document_1_id, document_2_id)`

**Index :**
- `idx_document_duplicates_status` sur `status`

---

### 4.7 Modèle `DocumentChangeHistory` (Historique de modification)

**Table :** `document_change_history`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_id` | UUID | FK → documents.id, NOT NULL, INDEX | Document modifié |
| `change_type` | VARCHAR(30) | NOT NULL | Type : `created`, `metadata_updated`, `file_replaced`, `status_changed`, `classification_changed` |
| `field_changed` | VARCHAR(100) | NULL | Champ modifié (si applicable) |
| `old_value` | TEXT | NULL | Ancienne valeur |
| `new_value` | TEXT | NULL | Nouvelle valeur |
| `changed_at` | TIMESTAMP | NOT NULL, AUTO | Date du changement |
| `changed_by_id` | UUID | FK → users.id, NOT NULL | Qui a modifié |

**Index :**
- `idx_document_change_history_document_id` sur `document_id`
- `idx_document_change_history_changed_at` sur `changed_at`

---

### 4.8 Modèle `DocumentView` (Traçabilité des consultations)

**Table :** `document_views`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_id` | UUID | FK → documents.id, NOT NULL, INDEX | Document consulté |
| `user_id` | UUID | FK → users.id, NOT NULL, INDEX | Qui a consulté |
| `view_type` | VARCHAR(20) | NOT NULL | Type : `detail_view`, `download`, `preview` |
| `ip_address` | INET | NULL | Adresse IP |
| `user_agent` | TEXT | NULL | User-Agent |
| `viewed_at` | TIMESTAMP | NOT NULL, AUTO | Date et heure de consultation |

**Index :**
- `idx_document_views_document_id` sur `document_id`
- `idx_document_views_user_id` sur `user_id`
- `idx_document_views_viewed_at` sur `viewed_at`

---

## 5. Matrice des permissions du module

| Permission code | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `documents.create` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `documents.read` | ✅ | ✅ | ✅ | ✅ | ✅ (filtré) | ✅ (filtré) |
| `documents.update_own` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `documents.update_any` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `documents.delete` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `documents.delete_physical` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `documents.download` | ✅ | ✅ | ✅ | ✅ | ✅ (filtré) | ✅ (filtré) |
| `documents.version.create` | ✅ | ✅ | ✅ | ✅ | ✅ (own) | ❌ |
| `documents.version.restore` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `documents.relations.create` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `documents.relations.delete` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `documents.attachments.add` | ✅ | ✅ | ✅ | ✅ | ✅ (own) | ❌ |
| `documents.attachments.delete` | ✅ | ✅ | ✅ | ✅ | ✅ (own) | ❌ |
| `documents.comments.create` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `documents.comments.delete_own` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `documents.comments.delete_any` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `documents.duplicates.review` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `documents.duplicates.merge` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `documents.assign` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `documents.priority.set` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `documents.export_list` | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| `documents.view_history` | ✅ | ✅ | ✅ | ✅ | ✅ (own) | ✅ |

---

*Suite du MODULE 06 dans le prochain message...*