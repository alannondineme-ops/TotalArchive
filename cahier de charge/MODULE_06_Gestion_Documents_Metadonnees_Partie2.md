# MODULE 06 — Gestion des Documents & Métadonnées (Partie 2)

## Suite du MODULE 06 — Séquences, Endpoints, Règles métier

---

## 6. Séquences détaillées des flux principaux

### 6.1 Séquence — Création complète d'un document avec fichier

```
Acteur              Frontend React         API Django            Base de données       Celery Queue         Système Fichiers
  |                       |                     |                       |                       |                       |
  |-- sélectionne type -->|                     |                       |                       |                       |
  |                       |-- GET /taxonomy/ --->|                      |                       |                       |
  |                       |   document-types/{id}/|-- SELECT type ------>|                      |                       |
  |                       |                     |<-- type + schema -----|                       |                       |
  |<-- formulaire --------|   + default_conf     |   metadata           |                       |                       |
  |   dynamique           |                     |                       |                       |                       |
  |                       |                     |                       |                       |                       |
  |-- remplit metadata -->|                     |                       |                       |                       |
  |-- uploade fichier ---->|                    |                       |                       |                       |
  |                       |-- POST /documents/ ->|                       |                       |                       |
  |                       |   (multipart/form)   |-- valider format ---->|                      |                       |
  |                       |                     |-- valider taille ----->|                      |                       |
  |                       |                     |-- calculer SHA-256     |                       |                       |
  |                       |                     |-- vérifier doublon --->|                      |                       |
  |                       |                     |<-- hash unique --------|                       |                       |
  |                       |                     |-- générer UUID filename|                      |                       |
  |                       |                     |-- stocker fichier --------------------------------->                       |
  |                       |                     |-- générer thumbnail ----------------------------->                       |
  |                       |                     |-- INSERT document ---->|                      |                       |
  |                       |                     |-- INSERT version 1.0 ->|                      |                       |
  |                       |                     |-- INSERT history ----->|                      |                       |
  |                       |                     |-- INSERT classification>|                     |                       |
  |                       |                     |-- calculer expiry_date>|                      |                       |
  |                       |                     |-- créer tâche OCR ----------------------------->|                      |
  |                       |                     |-- créer tâche IA ------------------------------>|                      |
  |                       |                     |-- INSERT audit log --->|                      |                       |
  |                       |<-- 201 + document ---|                       |                       |                       |
  |<-- document créé ------|                     |                       |                       |                       |
```

### 6.2 Séquence — Consultation d'un document avec vérification des permissions

```
Utilisateur         Frontend React         API Django            Base de données       Audit
    |                     |                     |                       |                 |
    |-- clique document -->|                     |                       |                 |
    |                     |-- GET /documents/ -->|                       |                 |
    |                     |   {id}/              |-- SELECT document ---->|                 |
    |                     |                     |<-- document ------------|                 |
    |                     |                     |-- vérifier permission  |                 |
    |                     |                     |   utilisateur vs       |                 |
    |                     |                     |   confidentiality_level|                 |
    |                     |                     |-- vérifier accès       |                 |
    |                     |                     |   explicite si besoin  |                 |
    |                     |                     |-- autorisation OK      |                 |
    |                     |                     |-- INSERT document_view>|                 |
    |                     |                     |-- UPDATE view_count -->|                 |
    |                     |                     |-- UPDATE last_viewed_at>|                |
    |                     |                     |-- INSERT audit ----------------------->|
    |                     |<-- 200 + document ---|                       |                 |
    |<-- affichage --------|                     |                       |                 |
```

### 6.3 Séquence — Création d'une nouvelle version

```
Utilisateur         Frontend React         API Django            Base de données       Celery
    |                     |                     |                       |                 |
    |-- nouvelle version ->|                     |                       |                 |
    |                     |-- GET /documents/ -->|                       |                 |
    |                     |   {id}/versions/     |-- SELECT versions ---->|                |
    |                     |                     |<-- current version -----|                |
    |<-- formulaire -------|                     |                       |                 |
    |   + version actuelle |                     |                       |                 |
    |                       |                     |                       |                 |
    |-- uploade fichier -->|                     |                       |                 |
    |-- type version ------>|                    |                       |                 |
    |-- notes ------------>|                     |                       |                 |
    |                     |-- POST /documents/ ->|                       |                 |
    |                     |   {id}/versions/     |-- calculer nouveau    |                 |
    |                     |                     |   numéro version      |                 |
    |                     |                     |-- calculer SHA-256     |                 |
    |                     |                     |-- stocker nouveau file |                 |
    |                     |                     |-- UPDATE old version -->|                |
    |                     |                     |   is_current = FALSE   |                 |
    |                     |                     |-- INSERT new version -->|                |
    |                     |                     |   is_current = TRUE    |                 |
    |                     |                     |-- UPDATE document ----->|                |
    |                     |                     |   file_path, hash      |                 |
    |                     |                     |   current_version_num  |                 |
    |                     |                     |   version_count++      |                 |
    |                     |                     |-- INSERT history ------>|                |
    |                     |                     |-- créer tâche OCR ------------------------>|
    |                     |                     |-- INSERT audit -------->|                |
    |                     |<-- 201 + version ----|                       |                 |
    |<-- confirmation -----|                     |                       |                 |
```

### 6.4 Séquence — Fusion de documents doublons

```
Archiviste          Frontend React         API Django            Base de données
    |                     |                     |                       |
    |-- sélectionne 2 docs>|                    |                       |
    |                     |-- POST /documents/ ->|                       |
    |                     |   merge/preview/     |-- SELECT doc1 -------->|
    |                     |                     |-- SELECT doc2 -------->|
    |                     |                     |-- comparer metadata    |
    |                     |<-- preview + diff ---|                       |
    |<-- choix metadata ---|                     |                       |
    |   + versions         |                     |                       |
    |   + commentaires     |                     |                       |
    |                       |                     |                       |
    |-- valide fusion ----->|                    |                       |
    |                     |-- POST /documents/ ->|                       |
    |                     |   merge/confirm/     |-- BEGIN TRANSACTION -->|
    |                     |                     |-- UPDATE doc1 metadata>|
    |                     |                     |-- TRANSFER versions --->|
    |                     |                     |   (doc2 → doc1)        |
    |                     |                     |-- TRANSFER comments --->|
    |                     |                     |-- TRANSFER attachments>|
    |                     |                     |-- UPDATE relations ---->|
    |                     |                     |   (doc2 → doc1)        |
    |                     |                     |-- INSERT history ------>|
    |                     |                     |-- UPDATE doc2 --------->|
    |                     |                     |   is_deleted = TRUE    |
    |                     |                     |   merged_into_id = doc1|
    |                     |                     |-- COMMIT TRANSACTION -->|
    |                     |                     |-- INSERT audit -------->|
    |                     |<-- 200 OK -----------|                       |
    |<-- confirmation -----|                     |                       |
```

---

## 7. Endpoints API du module

### 7.1 Documents — CRUD de base

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/documents/` | Liste paginée avec filtres | Oui |
| POST | `/api/v1/documents/` | Créer un document | Oui + Permission |
| GET | `/api/v1/documents/{id}/` | Détail d'un document | Oui + Permission |
| PATCH | `/api/v1/documents/{id}/` | Modifier métadonnées | Oui + Permission |
| DELETE | `/api/v1/documents/{id}/` | Suppression logique | Oui + Permission |
| POST | `/api/v1/documents/{id}/restore/` | Restaurer (annuler suppression) | Oui + Permission |
| POST | `/api/v1/documents/{id}/duplicate/` | Dupliquer | Oui + Permission |
| POST | `/api/v1/documents/{id}/move/` | Déplacer vers catégorie | Oui + Permission |

### 7.2 Téléchargement et fichiers

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/documents/{id}/download/` | Télécharger le fichier actuel | Oui + Permission |
| GET | `/api/v1/documents/{id}/thumbnail/` | Récupérer la miniature | Oui + Permission |
| GET | `/api/v1/documents/{id}/preview/` | Prévisualiser (iframe si possible) | Oui + Permission |

### 7.3 Versions

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/documents/{id}/versions/` | Liste des versions | Oui + Permission |
| POST | `/api/v1/documents/{id}/versions/` | Créer nouvelle version | Oui + Permission |
| GET | `/api/v1/documents/{id}/versions/{version_id}/` | Détail d'une version | Oui + Permission |
| GET | `/api/v1/documents/{id}/versions/{version_id}/download/` | Télécharger version spécifique | Oui + Permission |
| POST | `/api/v1/documents/{id}/versions/{version_id}/restore/` | Restaurer comme actuelle | Oui + Permission |
| GET | `/api/v1/documents/{id}/versions/compare/` | Comparer deux versions | Oui + Permission |

### 7.4 Relations

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/documents/{id}/relations/` | Relations du document | Oui + Permission |
| POST | `/api/v1/documents/{id}/relations/` | Créer une relation | Oui + Permission |
| DELETE | `/api/v1/documents/relations/{relation_id}/` | Supprimer une relation | Oui + Permission |
| GET | `/api/v1/documents/{id}/tree/` | Arborescence parent/enfants | Oui + Permission |

### 7.5 Pièces jointes

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/documents/{id}/attachments/` | Liste des pièces jointes | Oui + Permission |
| POST | `/api/v1/documents/{id}/attachments/` | Ajouter pièce jointe | Oui + Permission |
| GET | `/api/v1/documents/attachments/{attachment_id}/download/` | Télécharger | Oui + Permission |
| DELETE | `/api/v1/documents/attachments/{attachment_id}/` | Supprimer | Oui + Permission |

### 7.6 Commentaires

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/documents/{id}/comments/` | Liste des commentaires (arbre) | Oui + Permission |
| POST | `/api/v1/documents/{id}/comments/` | Ajouter commentaire | Oui + Permission |
| PATCH | `/api/v1/documents/comments/{comment_id}/` | Modifier son commentaire | Oui + Propriétaire |
| DELETE | `/api/v1/documents/comments/{comment_id}/` | Supprimer commentaire | Oui + Permission |
| POST | `/api/v1/documents/comments/{comment_id}/reply/` | Répondre à un commentaire | Oui + Permission |

### 7.7 Doublons

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/documents/duplicates/` | Liste des doublons détectés | Oui + Permission |
| POST | `/api/v1/documents/duplicates/{duplicate_id}/confirm/` | Confirmer doublon | Oui + Permission |
| POST | `/api/v1/documents/duplicates/{duplicate_id}/dismiss/` | Rejeter doublon | Oui + Permission |
| POST | `/api/v1/documents/merge/preview/` | Prévisualiser fusion | Oui + Permission |
| POST | `/api/v1/documents/merge/confirm/` | Confirmer fusion | Oui + Permission |

### 7.8 Priorité et assignation

| Méthode | URL | Description | Auth |
|---|---|---|---|
| POST | `/api/v1/documents/{id}/assign/` | Assigner à un utilisateur | Oui + Permission |
| POST | `/api/v1/documents/{id}/priority/` | Changer priorité | Oui + Permission |
| POST | `/api/v1/documents/{id}/status/` | Changer statut de traitement | Oui + Permission |
| GET | `/api/v1/documents/urgent/` | Documents urgents | Oui + Permission |
| GET | `/api/v1/documents/assigned-to-me/` | Documents assignés à moi | Oui |

### 7.9 Favoris et historique

| Méthode | URL | Description | Auth |
|---|---|---|---|
| POST | `/api/v1/documents/{id}/favorite/` | Ajouter aux favoris | Oui |
| DELETE | `/api/v1/documents/{id}/favorite/` | Retirer des favoris | Oui |
| GET | `/api/v1/documents/favorites/` | Mes favoris | Oui |
| GET | `/api/v1/documents/recent/` | Documents récents (globaux) | Oui |
| GET | `/api/v1/documents/my-recent-views/` | Mes consultations récentes | Oui |

### 7.10 Historique et audit

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/documents/{id}/history/` | Historique de modification | Oui + Permission |
| GET | `/api/v1/documents/{id}/views/` | Historique de consultation | Oui + Permission |
| GET | `/api/v1/documents/{id}/audit/` | Journal d'audit complet | Oui + Permission |

### 7.11 Export et statistiques

| Méthode | URL | Description | Auth |
|---|---|---|---|
| POST | `/api/v1/documents/export/` | Exporter liste (CSV/Excel) | Oui + Permission |
| GET | `/api/v1/documents/stats/by-category/` | Statistiques par catégorie | Oui + Permission |
| GET | `/api/v1/documents/stats/by-type/` | Statistiques par type | Oui + Permission |
| GET | `/api/v1/documents/stats/by-status/` | Statistiques par statut | Oui + Permission |

---

## 8. Règles métier critiques du module

### 8.1 Règle de complétude des métadonnées

Un document ne peut pas être marqué comme "Terminé" (`processing_status = completed`) tant que toutes les métadonnées obligatoires ne sont pas remplies. Les métadonnées obligatoires incluent :
- Métadonnées standard : titre, type, catégorie, date du document, niveau de confidentialité
- Métadonnées personnalisées marquées comme `required` dans le schéma JSONB du type de document

Le système calcule automatiquement un score de complétude (0-100%) qui est affiché sur la fiche du document.

---

### 8.2 Règle d'intégrité du hash SHA-256

Le hash SHA-256 d'un document est calculé à l'upload et ne change jamais (sauf nouvelle version). Il sert de **preuve d'intégrité**. Le système vérifie périodiquement (tâche Celery hebdomadaire) que le hash stocké en base correspond toujours au hash du fichier physique. Si une divergence est détectée :
- Une alerte CRITIQUE est envoyée aux administrateurs
- Le document est automatiquement gelé (MODULE 04)
- Un incident de sécurité est créé dans le journal d'audit
- Le document est marqué visuellement comme "Intégrité compromise"

Cette vérification permet de détecter toute altération malveillante ou corruption de fichier.

---

### 8.3 Règle de verrouillage en modification

Lorsqu'un utilisateur ouvre un document pour le modifier, le système peut le verrouiller (`is_locked = TRUE`, `locked_by_id = user_id`) pour empêcher les modifications concurrentes. Le verrouillage est automatiquement libéré après 15 minutes d'inactivité ou lorsque l'utilisateur sauvegarde ou annule ses modifications.

Un autre utilisateur tentant de modifier un document verrouillé voit un message : "Ce document est en cours de modification par [Nom] depuis [durée]".

---

### 8.4 Règle de cascade des relations parent/enfant

Si un document possède un `parent_document_id` :
- Il hérite automatiquement du niveau de confidentialité **minimum** du parent (il peut avoir un niveau supérieur, jamais inférieur)
- Si le parent est gelé (MODULE 04), tous les enfants sont automatiquement gelés
- Si le parent est supprimé logiquement, tous les enfants sont supprimés logiquement en cascade (avec avertissement préalable)

---

### 8.5 Règle du numéro d'enregistrement séquentiel

Le `registration_number` suit le format `YYYY-NNNNN` où :
- `YYYY` = année de création
- `NNNNN` = numéro séquentiel à 5 chiffres, unique par année, incrémenté automatiquement

Exemples : `2026-00001`, `2026-00002`, `2026-12345`

Le système garantit l'unicité en utilisant une séquence PostgreSQL. Ce numéro ne change jamais, même si le document est modifié ou déplacé. Il sert de référence permanente et peut être cité dans des correspondances officielles.

---

### 8.6 Règle de détection intelligente des doublons

Le système détecte trois types de doublons :

**1. Doublon exact (hash identique)** → Probabilité 100% que ce soit le même fichier
- Action suggérée : ne pas créer le document, référencer l'existant

**2. Doublon de titre (similarité > 80%)** → Attention, potentiel doublon
- Action suggérée : vérifier manuellement, potentiellement lier les documents

**3. Doublon de contenu (OCR similarité > 90%)** → Très probable doublon avec légères modifications
- Action suggérée : comparer les versions, potentiellement fusionner

Le système ne bloque jamais automatiquement la création — il alerte l'utilisateur qui décide de l'action à prendre.

---

### 8.7 Règle de droit d'accès aux versions antérieures

Toutes les versions d'un document héritent du niveau de confidentialité **actuel** du document, pas du niveau qu'elles avaient au moment de leur création. Cela signifie que si un document est reclassifié "Secret" après coup, toutes ses versions antérieures deviennent également inaccessibles aux utilisateurs non autorisés.

---

### 8.8 Règle de nettoyage automatique des suppressions logiques

Les documents supprimés logiquement (`is_deleted = TRUE`) sont conservés pendant une durée configurable (défaut : 90 jours) avant suppression physique définitive. Une tâche Celery mensuelle envoie des alertes 15 jours avant l'expiration pour permettre la restauration si nécessaire.

Exception : les documents éliminés via le MODULE 04 (élimination légale) suivent leur propre procédure et ne sont pas concernés par cette règle.

---

## 9. Calculs automatiques et champs dénormalisés

### 9.1 Calcul du score de complétude

```
score_completude = (nombre_champs_remplis / nombre_champs_obligatoires) * 100
```

Mis à jour à chaque modification de métadonnées. Affiché visuellement par une barre de progression.

### 9.2 Mise à jour des compteurs

Les champs suivants sont maintenus à jour via des signaux Django :

- `version_count` : incrémenté à chaque nouvelle version
- `view_count` : incrémenté à chaque consultation
- `download_count` : incrémenté à chaque téléchargement
- `last_viewed_at` : mis à jour à chaque consultation

### 9.3 Génération automatique des miniatures

Le système génère automatiquement des miniatures (200x200px) pour :
- Images (JPEG, PNG, GIF, WebP, BMP, TIFF)
- PDFs (première page uniquement)

Les miniatures sont stockées dans `/media/thumbnails/` et servent à l'affichage rapide dans les listes.

---

## 10. Tâches Celery planifiées

| Tâche | Fréquence | Description |
|---|---|---|
| `documents.verify_file_integrity` | Hebdomadaire (samedi 3h) | Vérifie que le hash SHA-256 des fichiers correspond au hash stocké |
| `documents.cleanup_soft_deleted` | Mensuel (1er du mois 4h) | Alerte pour les documents supprimés arrivant en fin de rétention (90j) |
| `documents.unlock_stale_locks` | Toutes les 15 minutes | Libère les verrous de documents inactifs depuis plus de 15 minutes |
| `documents.update_expiry_dates` | Quotidien (2h) | Recalcule les dates d'échéance si les politiques ont changé |
| `documents.generate_missing_thumbnails` | Hebdomadaire (dimanche 1h) | Génère les miniatures manquantes ou corrompues |
| `documents.detect_duplicates` | Quotidien (5h) | Analyse les documents créés dans les 24h pour détecter les doublons |

---

## 11. Événements journalisés (MODULE 11 — Audit)

| Événement | Niveau | Détails |
|---|---|---|
| Document créé | INFO | document_id, title, type, category, created_by |
| Document modifié | INFO | document_id, fields_changed, modified_by |
| Document supprimé | WARNING | document_id, deleted_by, reason |
| Document restauré | INFO | document_id, restored_by |
| Fichier remplacé (nouvelle version) | INFO | document_id, old_version, new_version, changed_by |
| Version restaurée | WARNING | document_id, restored_version, restored_by |
| Relation créée | INFO | source_doc, target_doc, relation_type, created_by |
| Documents fusionnés | WARNING | doc1_id, doc2_id, survivor_id, merged_by |
| Document téléchargé | INFO | document_id, user_id, ip_address, timestamp |
| Pièce jointe ajoutée | INFO | document_id, attachment_name, uploaded_by |
| Commentaire ajouté | INFO | document_id, comment_id, author |
| Doublon détecté | INFO | doc1_id, doc2_id, duplicate_type, similarity_score |
| Document verrouillé | INFO | document_id, locked_by, timestamp |
| Document déverrouillé | INFO | document_id, unlocked_by, timestamp |
| Intégrité compromise | CRITICAL | document_id, expected_hash, actual_hash, detected_at |
| Document assigné | INFO | document_id, assigned_to, assigned_by |
| Priorité modifiée | INFO | document_id, old_priority, new_priority, changed_by |

---

## 12. Notifications générées (MODULE 12)

| Déclencheur | Destinataire | Canal | Message |
|---|---|---|
| Document créé dans une catégorie suivie | Archivistes | In-app | Nouveau document : [Titre] |
| Document assigné | Utilisateur assigné | Email + In-app | Le document [X] vous a été assigné |
| Mention dans commentaire (@mention) | Utilisateur mentionné | Email + In-app | [User] vous a mentionné dans [Document] |
| Nouvelle version disponible | Propriétaire + followers | In-app | Nouvelle version v[X] du document [Y] |
| Document arrivant à échéance | Propriétaire | Email + In-app | Le document [X] arrive à échéance dans [N] jours |
| Doublon détecté | Créateur du document | In-app | Un doublon potentiel a été détecté : [Document] |
| Intégrité compromise | Admins | Email (urgent) | ALERTE : intégrité compromise du document [X] |
| Document urgent non traité | Assigné | In-app | Document urgent non traité : [X] |
| Commentaire sur document suivi | Propriétaire + followers | In-app | Nouveau commentaire sur [Document] |
| Document restauré après suppression | Propriétaire | In-app | Le document [X] a été restauré |

---

## 13. Intégrations avec les autres modules

### 13.1 MODULE 03 — Taxonomie
- Le `document_type_id` et `category_id` sont obligatoires
- Le schéma de métadonnées personnalisées (`custom_metadata` JSONB) est défini par le type de document
- Les extensions autorisées sont vérifiées selon le type

### 13.2 MODULE 04 — Cycle de vie
- Le champ `lifecycle_status` est géré par le MODULE 04
- La `expiry_date` est calculée automatiquement selon la politique de conservation du type
- Le gel d'un document (`is_frozen`) bloque toutes les modifications

### 13.3 MODULE 05 — Confidentialité
- Le `confidentiality_level_id` est obligatoire
- Les permissions d'accès sont vérifiées à chaque consultation et téléchargement
- Les accès explicites (MODULE 05) sont vérifiés avant autorisation

### 13.4 MODULE 07 — OCR
- À la création d'un document, une tâche OCR est automatiquement déclenchée si le format est supporté
- Le contenu extrait est stocké et indexé par le MODULE 09

### 13.5 MODULE 08 — Workflow
- Le champ `processing_status` peut être piloté par un workflow de validation
- Le champ `assigned_to_id` est utilisé pour router le document dans le workflow

### 13.6 MODULE 10 — Intelligence Artificielle
- À la création, une tâche de classification automatique peut suggérer la catégorie et le niveau de confidentialité
- L'extraction d'entités alimente les métadonnées personnalisées

### 13.7 MODULE 11 — Audit
- Tous les événements documentaires sont journalisés
- L'historique de consultation (`document_views`) alimente les rapports d'audit

### 13.8 MODULE 13 — Accès & Partage
- Les documents peuvent être partagés temporairement via des liens sécurisés
- Les demandes d'accès à un document confidentiel transitent par ce module

---

*Fin du MODULE 06 — Gestion des Documents & Métadonnées*  
*Prochain module : MODULE 07 — Numérisation, OCR & Intégration scanner*
