# MODULE 03 — Taxonomie Documentaire

## Projet : Système d'Archivage Numérique des Documents
**Version :** 1.0  
**Dépendances :** MODULE 01 — Architecture Générale | MODULE 02 — Authentification, Utilisateurs, Rôles & Permissions  
**Statut :** Référentiel de base — les MODULES 04, 05 et 06 dépendent directement de ce module

---

## 1. Présentation du module

La taxonomie documentaire est le squelette intellectuel de tout le système d'archivage. Elle définit comment les documents sont organisés, classés et retrouvés. Sans une taxonomie solide, cohérente et bien structurée, le système dégénère en un entrepôt désorganisé qui reproduit exactement le problème que le projet cherche à résoudre.

Ce module définit l'ensemble des référentiels de classification : les catégories, les sous-catégories, les types de documents, les tags, et le plan de classement hiérarchique. Ces référentiels sont administrés de manière centralisée et servent de fondation à tous les modules qui manipulent des documents.

Un document ne peut pas exister dans le système sans être rattaché à au moins une catégorie et un type documentaire. C'est une contrainte d'intégrité non négociable.

---

## 2. Cas d'utilisation — Vue d'ensemble

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-TAX-01 | Créer une catégorie documentaire | Super Admin / Admin / Archiviste |
| UC-TAX-02 | Modifier une catégorie documentaire | Super Admin / Admin / Archiviste |
| UC-TAX-03 | Désactiver une catégorie documentaire | Super Admin / Admin |
| UC-TAX-04 | Consulter l'arbre des catégories | Tout utilisateur connecté |
| UC-TAX-05 | Créer une sous-catégorie | Super Admin / Admin / Archiviste |
| UC-TAX-06 | Modifier une sous-catégorie | Super Admin / Admin / Archiviste |
| UC-TAX-07 | Désactiver une sous-catégorie | Super Admin / Admin |
| UC-TAX-08 | Créer un type de document | Super Admin / Admin / Archiviste |
| UC-TAX-09 | Modifier un type de document | Super Admin / Admin / Archiviste |
| UC-TAX-10 | Désactiver un type de document | Super Admin / Admin |
| UC-TAX-11 | Consulter la liste des types de documents | Tout utilisateur connecté |
| UC-TAX-12 | Créer un tag | Super Admin / Admin / Archiviste / Responsable |
| UC-TAX-13 | Modifier un tag | Super Admin / Admin / Archiviste |
| UC-TAX-14 | Fusionner deux tags | Super Admin / Admin / Archiviste |
| UC-TAX-15 | Supprimer un tag | Super Admin / Admin |
| UC-TAX-16 | Consulter la liste des tags | Tout utilisateur connecté |
| UC-TAX-17 | Créer un plan de classement | Super Admin / Admin / Archiviste |
| UC-TAX-18 | Modifier un plan de classement | Super Admin / Admin / Archiviste |
| UC-TAX-19 | Consulter le plan de classement complet | Tout utilisateur connecté |
| UC-TAX-20 | Exporter le plan de classement | Super Admin / Admin / Archiviste |
| UC-TAX-21 | Importer un plan de classement (initialisation) | Super Admin uniquement |
| UC-TAX-22 | Consulter les statistiques d'utilisation d'une catégorie | Super Admin / Admin / Archiviste |
| UC-TAX-23 | Déplacer une catégorie dans l'arborescence | Super Admin / Admin |
| UC-TAX-24 | Rechercher dans le référentiel taxonomique | Tout utilisateur connecté |

---

## 3. Description détaillée des cas d'utilisation

### UC-TAX-01 — Créer une catégorie documentaire

**Acteur :** Super Admin, Admin ou Archiviste  
**Préconditions :** L'acteur est authentifié et possède la permission `taxonomy.categories.create`  
**Postconditions :** Une nouvelle catégorie est créée et disponible pour la classification des documents

**Scénario principal :**
1. L'acteur accède au module d'administration de la taxonomie
2. L'acteur choisit de créer une nouvelle catégorie racine (sans parent) ou une sous-catégorie (avec parent)
3. L'acteur remplit le formulaire : nom, code, description, icône optionnelle, catégorie parente (si applicable), département(s) concerné(s)
4. Le système vérifie que le code est unique dans tout le référentiel des catégories
5. Le système vérifie que le nom est unique au sein du même niveau hiérarchique
6. Le système calcule automatiquement le chemin complet de la catégorie (ex: `Administratif > Ressources Humaines > Contrats`)
7. Le système crée la catégorie avec le statut "Actif"
8. La création est journalisée

**Scénarios alternatifs :**
- **4a.** Le code existe déjà → le système signale le conflit et propose le code le plus proche disponible
- **5a.** Le nom existe déjà au même niveau → le système alerte et demande confirmation ou modification

**Règles métier :**
- La profondeur maximale de l'arborescence est de 5 niveaux
- Le code d'une catégorie est en majuscules, sans espaces ni caractères spéciaux (ex: `RH_CONTRATS`)
- Une catégorie ne peut pas être supprimée physiquement si elle contient des documents ou des sous-catégories actives

---

### UC-TAX-03 — Désactiver une catégorie documentaire

**Acteur :** Super Admin ou Admin  
**Préconditions :** La catégorie existe et est active  
**Postconditions :** La catégorie est désactivée et ne peut plus accueillir de nouveaux documents

**Scénario principal :**
1. L'acteur sélectionne la catégorie à désactiver
2. Le système affiche un avertissement indiquant le nombre de documents encore rattachés à cette catégorie et à toutes ses sous-catégories
3. Le système affiche un avertissement si des sous-catégories actives existent
4. L'acteur choisit une catégorie de remplacement vers laquelle les nouveaux documents seront orientés (optionnel mais recommandé)
5. L'acteur confirme la désactivation
6. La catégorie passe au statut "Inactif" — les documents existants restent rattachés mais aucun nouveau document ne peut y être assigné
7. L'action est journalisée avec la catégorie de remplacement éventuelle

**Règles métier :**
- La désactivation d'une catégorie parente ne désactive pas automatiquement ses enfants — chaque niveau est géré indépendamment
- Les documents existants dans une catégorie désactivée restent accessibles — seule l'assignation de nouveaux documents est bloquée

---

### UC-TAX-14 — Fusionner deux tags

**Acteur :** Super Admin, Admin ou Archiviste  
**Préconditions :** Les deux tags existent  
**Postconditions :** Un seul tag subsiste, tous les documents rattachés aux deux tags sont désormais rattachés au tag survivant

**Scénario principal :**
1. L'acteur sélectionne le tag source (qui sera absorbé) et le tag cible (qui survivra)
2. Le système affiche le nombre de documents affectés par chaque tag
3. L'acteur confirme la fusion
4. Le système met à jour tous les liens document↔tag pour pointer vers le tag cible
5. Le tag source est supprimé logiquement
6. La fusion est journalisée avec les détails (tag source, tag cible, nombre de documents affectés)

---

### UC-TAX-21 — Importer un plan de classement

**Acteur :** Super Admin uniquement  
**Préconditions :** Aucune catégorie n'existe encore (initialisation du système) ou l'administrateur a activé le mode d'import en complément  
**Postconditions :** Le plan de classement est importé et disponible

**Scénario principal :**
1. Le Super Admin prépare un fichier d'import au format JSON ou CSV selon le schéma défini par le système
2. Le Super Admin uploade le fichier via l'interface d'administration
3. Le système analyse le fichier et détecte les erreurs éventuelles (codes en double, profondeur excessive, champs manquants)
4. Le système affiche un rapport de prévisualisation : nombre de catégories, sous-catégories, types de documents détectés, erreurs et avertissements
5. Le Super Admin valide l'import après vérification du rapport
6. Le système importe les éléments valides et ignore les lignes en erreur (rapport d'import final généré)
7. L'import est journalisé

---

### UC-TAX-23 — Déplacer une catégorie dans l'arborescence

**Acteur :** Super Admin ou Admin  
**Préconditions :** La catégorie source et la catégorie destination existent  
**Postconditions :** La catégorie et toutes ses sous-catégories sont déplacées dans le nouvel emplacement

**Scénario principal :**
1. L'acteur sélectionne la catégorie à déplacer via l'interface en arbre (drag-and-drop ou sélection manuelle)
2. L'acteur sélectionne la nouvelle catégorie parente (ou "Racine" pour en faire une catégorie de premier niveau)
3. Le système vérifie que le déplacement ne crée pas de cycle (une catégorie ne peut pas devenir son propre ancêtre)
4. Le système vérifie que la profondeur résultante ne dépasse pas 5 niveaux
5. Le système recalcule les chemins complets de la catégorie déplacée et de toutes ses descendantes
6. Le système met à jour les références dans tous les documents rattachés
7. Le déplacement est journalisé avec l'ancienne et la nouvelle position

---

## 4. Modèles de données

### 4.1 Modèle `DocumentCategory` (Catégorie documentaire)

**Table :** `document_categories`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `name` | VARCHAR(150) | NOT NULL | Nom de la catégorie |
| `code` | VARCHAR(50) | UNIQUE, NOT NULL | Code technique unique (majuscules, sans espaces) |
| `description` | TEXT | NULL | Description détaillée |
| `icon` | VARCHAR(100) | NULL | Nom de l'icône (référence à la bibliothèque d'icônes frontend) |
| `color` | VARCHAR(7) | NULL | Couleur hexadécimale associée (#RRGGBB) |
| `parent_id` | UUID | FK → document_categories.id, NULL | Catégorie parente (NULL = catégorie racine) |
| `path` | TEXT | NOT NULL | Chemin complet calculé (ex: `Administratif/RH/Contrats`) |
| `depth` | SMALLINT | NOT NULL, DEFAULT 0 | Profondeur dans l'arbre (0 = racine) |
| `order_index` | SMALLINT | NOT NULL, DEFAULT 0 | Ordre d'affichage parmi les frères |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Catégorie active |
| `is_system` | BOOLEAN | NOT NULL, DEFAULT FALSE | Catégorie système (non supprimable) |
| `allowed_departments` | M2M → departments | NULL | Départements autorisés à utiliser cette catégorie (NULL = tous) |
| `document_count` | INTEGER | NOT NULL, DEFAULT 0 | Compteur dénormalisé du nombre de documents (mis à jour par signal) |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | Suppression logique |
| `deleted_at` | TIMESTAMP | NULL | Date de suppression logique |
| `deleted_by_id` | UUID | FK → users.id, NULL | Qui a supprimé |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `updated_at` | TIMESTAMP | NOT NULL, AUTO | Date de mise à jour |
| `created_by_id` | UUID | FK → users.id, NULL | Créateur |

**Contraintes supplémentaires :**
- `depth` : entre 0 et 4 (profondeur maximale de 5 niveaux, de 0 à 4)
- `code` : expression régulière `^[A-Z0-9_]+$`
- `color` : expression régulière `^#[0-9A-Fa-f]{6}$`
- Un nœud ne peut pas être son propre ancêtre (contrainte de cycle — vérifiée applicativement)
- `path` est recalculé automatiquement via signal Django à chaque modification de `name` ou `parent_id`

**Index :**
- `idx_document_categories_code` sur `code`
- `idx_document_categories_parent_id` sur `parent_id`
- `idx_document_categories_path` sur `path` (pour les recherches par préfixe)
- `idx_document_categories_is_active` sur `is_active`

---

### 4.2 Modèle `DocumentType` (Type de document)

**Table :** `document_types`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `name` | VARCHAR(150) | NOT NULL | Nom du type |
| `code` | VARCHAR(50) | UNIQUE, NOT NULL | Code technique unique |
| `description` | TEXT | NULL | Description |
| `category_id` | UUID | FK → document_categories.id, NOT NULL | Catégorie de rattachement |
| `allowed_extensions` | ARRAY VARCHAR | NOT NULL | Extensions de fichiers autorisées (ex: `['.pdf', '.docx']`) |
| `max_file_size_mb` | SMALLINT | NOT NULL, DEFAULT 50 | Taille maximale en Mo pour ce type |
| `requires_validation` | BOOLEAN | NOT NULL, DEFAULT FALSE | Nécessite un circuit de validation |
| `default_retention_years` | SMALLINT | NULL | Durée de conservation par défaut (MODULE 04) |
| `default_confidentiality` | VARCHAR(20) | NOT NULL, DEFAULT 'internal' | Niveau de confidentialité par défaut (MODULE 05) |
| `metadata_schema` | JSONB | NULL | Schéma JSON des métadonnées spécifiques à ce type |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Type actif |
| `is_system` | BOOLEAN | NOT NULL, DEFAULT FALSE | Type système (non supprimable) |
| `document_count` | INTEGER | NOT NULL, DEFAULT 0 | Compteur dénormalisé |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | Suppression logique |
| `deleted_at` | TIMESTAMP | NULL | Date de suppression |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `updated_at` | TIMESTAMP | NOT NULL, AUTO | Date de mise à jour |
| `created_by_id` | UUID | FK → users.id, NULL | Créateur |

**Explication du champ `metadata_schema` :**
Ce champ JSONB permet de définir des métadonnées supplémentaires spécifiques à chaque type de document. Par exemple, un contrat peut nécessiter un champ "Numéro de contrat" et "Date de signature", tandis qu'une facture nécessite un "Numéro de facture" et un "Montant". Le schéma définit les champs, leurs types, et s'ils sont obligatoires.

**Index :**
- `idx_document_types_code` sur `code`
- `idx_document_types_category_id` sur `category_id`
- `idx_document_types_is_active` sur `is_active`

---

### 4.3 Modèle `Tag` (Étiquette)

**Table :** `tags`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `name` | VARCHAR(100) | UNIQUE, NOT NULL | Nom du tag (normalisé en minuscules) |
| `slug` | VARCHAR(110) | UNIQUE, NOT NULL | Version URL-safe du nom |
| `description` | TEXT | NULL | Description optionnelle |
| `color` | VARCHAR(7) | NULL | Couleur d'affichage (#RRGGBB) |
| `usage_count` | INTEGER | NOT NULL, DEFAULT 0 | Nombre de documents utilisant ce tag (dénormalisé) |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Tag actif |
| `merged_into_id` | UUID | FK → tags.id, NULL | Si fusionné, pointe vers le tag survivant |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | Suppression logique |
| `deleted_at` | TIMESTAMP | NULL | Date de suppression |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `updated_at` | TIMESTAMP | NOT NULL, AUTO | Date de mise à jour |
| `created_by_id` | UUID | FK → users.id, NULL | Créateur |

**Règles métier :**
- Le `name` est normalisé en minuscules et sans espaces superflus avant stockage
- Le `slug` est généré automatiquement depuis le `name` (caractères accentués convertis, espaces remplacés par des tirets)
- Un tag supprimé logiquement ou fusionné n'apparaît plus dans les suggestions

**Index :**
- `idx_tags_name` sur `name`
- `idx_tags_slug` sur `slug`
- `idx_tags_usage_count` sur `usage_count` (pour le tri par popularité)

---

### 4.4 Modèle `ClassificationPlan` (Plan de classement)

**Table :** `classification_plans`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `name` | VARCHAR(200) | NOT NULL | Nom du plan de classement |
| `code` | VARCHAR(50) | UNIQUE, NOT NULL | Code technique |
| `description` | TEXT | NULL | Description |
| `version` | VARCHAR(20) | NOT NULL, DEFAULT '1.0' | Version du plan |
| `effective_date` | DATE | NOT NULL | Date d'entrée en vigueur |
| `expiry_date` | DATE | NULL | Date d'expiration (NULL = en vigueur indéfiniment) |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Plan actif |
| `is_default` | BOOLEAN | NOT NULL, DEFAULT FALSE | Plan par défaut de l'organisation |
| `applicable_departments` | M2M → departments | NULL | Départements concernés (NULL = tous) |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `updated_at` | TIMESTAMP | NOT NULL, AUTO | Date de mise à jour |
| `created_by_id` | UUID | FK → users.id, NULL | Créateur |

**Contraintes :**
- Un seul plan peut avoir `is_default = True` à la fois (contrainte applicative)

---

### 4.5 Modèle `ClassificationPlanEntry` (Entrée du plan de classement)

**Table :** `classification_plan_entries`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `plan_id` | UUID | FK → classification_plans.id, NOT NULL | Plan de classement |
| `category_id` | UUID | FK → document_categories.id, NOT NULL | Catégorie incluse |
| `document_type_id` | UUID | FK → document_types.id, NULL | Type spécifique (NULL = tous les types de la catégorie) |
| `reference_code` | VARCHAR(30) | NOT NULL | Cote d'archivage (ex: `ADM-RH-001`) |
| `notes` | TEXT | NULL | Notes spécifiques à cette entrée du plan |
| `order_index` | SMALLINT | NOT NULL, DEFAULT 0 | Ordre dans le plan |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |

**Contraintes :**
- `UNIQUE (plan_id, reference_code)` — la cote d'archivage est unique dans un plan
- `UNIQUE (plan_id, category_id, document_type_id)` — pas de doublon dans un plan

---

### 4.6 Modèle `CategoryDepartment` (Table de liaison Catégorie ↔ Département)

**Table :** `category_departments`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `category_id` | UUID | FK → document_categories.id, NOT NULL | Catégorie |
| `department_id` | UUID | FK → departments.id, NOT NULL | Département |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |

**Contrainte :** `UNIQUE (category_id, department_id)`

---

### 4.7 Modèle `TaxonomyChangeLog` (Journal des modifications taxonomiques)

**Table :** `taxonomy_change_logs`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `entity_type` | VARCHAR(50) | NOT NULL | Type d'entité modifiée (`category`, `document_type`, `tag`, `plan`) |
| `entity_id` | UUID | NOT NULL | Identifiant de l'entité |
| `entity_name` | VARCHAR(200) | NOT NULL | Nom de l'entité au moment du changement |
| `action` | VARCHAR(20) | NOT NULL | Action : `created`, `updated`, `deleted`, `activated`, `deactivated`, `moved`, `merged` |
| `previous_state` | JSONB | NULL | État avant modification |
| `new_state` | JSONB | NULL | État après modification |
| `changed_by_id` | UUID | FK → users.id, NOT NULL | Utilisateur ayant effectué le changement |
| `changed_at` | TIMESTAMP | NOT NULL, AUTO | Date du changement |
| `notes` | TEXT | NULL | Notes libres sur le changement |

**Note :** Ce modèle est propre à ce module et complète le journal d'audit global du MODULE 11 avec des informations taxonomiques détaillées.

---

## 5. Référentiel de base — Catégories système prédéfinies

Ces catégories sont créées à l'initialisation du système (`is_system = True`). Elles ne peuvent pas être supprimées mais peuvent être complétées.

### Niveau 1 — Catégories racines

| Code | Nom | Description |
|---|---|---|
| `ADMINISTRATIF` | Administratif | Documents relatifs à la gestion administrative générale |
| `JURIDIQUE` | Juridique | Documents à caractère légal, réglementaire et contractuel |
| `FINANCIER` | Financier | Documents comptables, budgétaires et financiers |
| `TECHNIQUE` | Technique | Documents techniques, projets et spécifications |
| `RH` | Ressources Humaines | Documents liés à la gestion du personnel |
| `COMMUNICATION` | Communication | Documents de communication interne et externe |
| `ARCHIVES` | Archives historiques | Documents archivés définitivement à valeur historique |

### Niveau 2 — Exemples de sous-catégories (extensibles)

| Parent | Code | Nom |
|---|---|---|
| `ADMINISTRATIF` | `ADM_COURRIER` | Courrier entrant et sortant |
| `ADMINISTRATIF` | `ADM_CIRCULAIRES` | Circulaires et notes de service |
| `ADMINISTRATIF` | `ADM_RAPPORTS` | Rapports d'activité |
| `JURIDIQUE` | `JUR_CONTRATS` | Contrats et conventions |
| `JURIDIQUE` | `JUR_DECISIONS` | Décisions et arrêtés |
| `JURIDIQUE` | `JUR_CONTENTIEUX` | Dossiers contentieux |
| `FINANCIER` | `FIN_FACTURES` | Factures et bons de commande |
| `FINANCIER` | `FIN_BUDGETS` | Documents budgétaires |
| `FINANCIER` | `FIN_MARCHES` | Marchés publics |
| `TECHNIQUE` | `TECH_PROJETS` | Dossiers de projets |
| `TECHNIQUE` | `TECH_SPECS` | Spécifications techniques |
| `RH` | `RH_CONTRATS` | Contrats de travail |
| `RH` | `RH_CONGES` | Gestion des congés |
| `RH` | `RH_FORMATIONS` | Dossiers de formation |

---

## 6. Référentiel de base — Types de documents système prédéfinis

| Code | Nom | Catégorie | Extensions | Validation requise |
|---|---|---|---|---|
| `COURRIER_ENTRANT` | Courrier entrant | ADM_COURRIER | .pdf, .jpg, .png, .tiff | Non |
| `COURRIER_SORTANT` | Courrier sortant | ADM_COURRIER | .pdf, .docx | Oui |
| `NOTE_SERVICE` | Note de service | ADM_CIRCULAIRES | .pdf, .docx | Oui |
| `RAPPORT_ACTIVITE` | Rapport d'activité | ADM_RAPPORTS | .pdf, .docx, .xlsx | Oui |
| `CONTRAT` | Contrat | JUR_CONTRATS | .pdf | Oui |
| `DECISION` | Décision | JUR_DECISIONS | .pdf | Oui |
| `ARRETE` | Arrêté | JUR_DECISIONS | .pdf | Oui |
| `FACTURE` | Facture | FIN_FACTURES | .pdf, .jpg, .png | Non |
| `BON_COMMANDE` | Bon de commande | FIN_FACTURES | .pdf, .docx | Oui |
| `CONTRAT_TRAVAIL` | Contrat de travail | RH_CONTRATS | .pdf | Oui |
| `DOSSIER_PERSONNEL` | Dossier personnel | RH | .pdf | Non |

---

## 7. Matrice des permissions du module

| Permission code | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `taxonomy.categories.create` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `taxonomy.categories.update` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `taxonomy.categories.deactivate` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `taxonomy.categories.delete` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `taxonomy.categories.read` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `taxonomy.categories.move` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `taxonomy.document_types.create` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `taxonomy.document_types.update` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `taxonomy.document_types.deactivate` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `taxonomy.document_types.read` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `taxonomy.tags.create` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `taxonomy.tags.update` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `taxonomy.tags.merge` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `taxonomy.tags.delete` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `taxonomy.tags.read` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `taxonomy.plans.create` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `taxonomy.plans.update` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `taxonomy.plans.import` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `taxonomy.plans.export` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `taxonomy.plans.read` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `taxonomy.stats.read` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |

---

## 8. Séquences détaillées des flux principaux

### 8.1 Séquence — Création d'une catégorie avec sous-catégorie

```
Acteur               Frontend React         API Django            Base de données
  |                       |                     |                       |
  |-- accède à l'arbre -->|                     |                       |
  |                       |-- GET /taxonomy/ --->|                       |
  |                       |   categories/tree    |-- SELECT catégories -->|
  |                       |                     |<-- arbre complet ------|
  |<-- arbre affiché -----|                     |                       |
  |                       |                     |                       |
  |-- clique "+ Créer" -->|                     |                       |
  |-- saisit formulaire -->|                    |                       |
  |-- sélectionne parent ->|                    |                       |
  |                       |-- POST /taxonomy/ -->|                       |
  |                       |   categories/        |-- vérifier unicité code>|
  |                       |                     |-- vérifier unicité nom  |
  |                       |                     |-- calculer path        |
  |                       |                     |-- calculer depth       |
  |                       |                     |-- INSERT catégorie --->|
  |                       |                     |-- INSERT change_log -->|
  |                       |<-- 201 + catégorie --|                       |
  |<-- arbre mis à jour --|                     |                       |
```

### 8.2 Séquence — Déplacement d'une catégorie

```
Acteur               Frontend React         API Django            Base de données
  |                       |                     |                       |
  |-- drag catégorie A -->|                     |                       |
  |   vers catégorie B    |                     |                       |
  |                       |-- PATCH /taxonomy/ ->|                      |
  |                       |   categories/{id}/   |-- vérifier cycle ------>|
  |                       |   move/              |-- vérifier profondeur   |
  |                       |                     |-- calculer nouveau path |
  |                       |                     |-- UPDATE catégorie A -->|
  |                       |                     |-- UPDATE enfants A  --->|
  |                       |                     |   (path récursif)       |
  |                       |                     |-- INSERT change_log --->|
  |                       |<-- 200 + arbre mis à jour                    |
  |<-- arbre actualisé ---|                     |                       |
```

### 8.3 Séquence — Fusion de tags

```
Acteur               Frontend React         API Django            Base de données
  |                       |                     |                       |
  |-- sélectionne tag A ->|                     |                       |
  |   + tag B (cible)     |                     |                       |
  |                       |-- POST /taxonomy/ -->|                       |
  |                       |   tags/merge/        |-- compter docs tag A ->|
  |                       |                     |-- compter docs tag B -->|
  |                       |<-- prévisualisation --|                      |
  |<-- affiche impact ----|                     |                       |
  |                       |                     |                       |
  |-- confirme ----------->|                    |                       |
  |                       |-- POST /taxonomy/ -->|                       |
  |                       |   tags/merge/confirm/|-- UPDATE document_tags>|
  |                       |                     |   tag_id A → B         |
  |                       |                     |-- soft delete tag A -->|
  |                       |                     |-- UPDATE usage_count ->|
  |                       |                     |-- INSERT change_log -->|
  |                       |<-- 200 OK -----------|                       |
  |<-- tag A disparu -----|                     |                       |
```

---

## 9. Endpoints API du module

### 9.1 Catégories

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/taxonomy/categories/` | Liste des catégories (filtrée) | Oui |
| GET | `/api/v1/taxonomy/categories/tree/` | Arbre complet des catégories | Oui |
| POST | `/api/v1/taxonomy/categories/` | Créer une catégorie | Oui + Permission |
| GET | `/api/v1/taxonomy/categories/{id}/` | Détail d'une catégorie | Oui |
| PATCH | `/api/v1/taxonomy/categories/{id}/` | Modifier une catégorie | Oui + Permission |
| POST | `/api/v1/taxonomy/categories/{id}/deactivate/` | Désactiver | Oui + Permission |
| POST | `/api/v1/taxonomy/categories/{id}/activate/` | Réactiver | Oui + Permission |
| POST | `/api/v1/taxonomy/categories/{id}/move/` | Déplacer dans l'arbre | Oui + Permission |
| GET | `/api/v1/taxonomy/categories/{id}/stats/` | Statistiques d'utilisation | Oui + Permission |
| GET | `/api/v1/taxonomy/categories/{id}/children/` | Sous-catégories directes | Oui |

### 9.2 Types de documents

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/taxonomy/document-types/` | Liste des types | Oui |
| POST | `/api/v1/taxonomy/document-types/` | Créer un type | Oui + Permission |
| GET | `/api/v1/taxonomy/document-types/{id}/` | Détail d'un type | Oui |
| PATCH | `/api/v1/taxonomy/document-types/{id}/` | Modifier un type | Oui + Permission |
| POST | `/api/v1/taxonomy/document-types/{id}/deactivate/` | Désactiver | Oui + Permission |

### 9.3 Tags

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/taxonomy/tags/` | Liste des tags (avec recherche) | Oui |
| POST | `/api/v1/taxonomy/tags/` | Créer un tag | Oui + Permission |
| GET | `/api/v1/taxonomy/tags/{id}/` | Détail d'un tag | Oui |
| PATCH | `/api/v1/taxonomy/tags/{id}/` | Modifier un tag | Oui + Permission |
| DELETE | `/api/v1/taxonomy/tags/{id}/` | Supprimer logiquement | Oui + Permission |
| POST | `/api/v1/taxonomy/tags/merge/` | Prévisualiser la fusion | Oui + Permission |
| POST | `/api/v1/taxonomy/tags/merge/confirm/` | Confirmer la fusion | Oui + Permission |
| GET | `/api/v1/taxonomy/tags/popular/` | Tags les plus utilisés | Oui |

### 9.4 Plans de classement

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/taxonomy/plans/` | Liste des plans | Oui |
| POST | `/api/v1/taxonomy/plans/` | Créer un plan | Oui + Permission |
| GET | `/api/v1/taxonomy/plans/{id}/` | Détail d'un plan | Oui |
| PATCH | `/api/v1/taxonomy/plans/{id}/` | Modifier un plan | Oui + Permission |
| GET | `/api/v1/taxonomy/plans/{id}/export/` | Exporter en JSON/CSV | Oui + Permission |
| POST | `/api/v1/taxonomy/plans/import/` | Importer un plan | Oui + Super Admin |

---

## 10. Règles de cohérence et d'intégrité

### 10.1 Règles d'intégrité référentielle

- Un document (MODULE 06) doit toujours avoir une `category_id` valide et active
- Un document doit toujours avoir un `document_type_id` valide et actif
- La désactivation d'une catégorie n'est possible que si aucun document actif ne lui est assigné en exclusivité, **ou** si une catégorie de remplacement est désignée
- La suppression logique d'un type de document n'est possible que si aucun document n'y est rattaché

### 10.2 Règles de cohérence des types et catégories

- Un type de document ne peut appartenir qu'à une seule catégorie (relation 1-N)
- Les extensions autorisées d'un type de document sont définitivement validées — tout fichier uploadé qui ne correspond pas aux extensions du type associé est rejeté
- Le `default_confidentiality` d'un type de document est une suggestion — l'archiviste peut toujours choisir un niveau différent lors de l'archivage

### 10.3 Cohérence du compteur `document_count`

Le champ `document_count` des catégories et des types de documents est un compteur dénormalisé pour les performances d'affichage. Il est maintenu à jour via des signaux Django (`post_save` et `post_delete`) sur le modèle Document. En cas de désynchronisation détectée, une tâche Celery planifiée recalcule et resynchronise ces compteurs.

---

## 11. Événements journalisés (MODULE 11 — Audit)

| Événement | Niveau | Détails |
|---|---|---|
| Catégorie créée | INFO | entity_id, name, code, parent, created_by |
| Catégorie modifiée | INFO | entity_id, changes, modified_by |
| Catégorie désactivée | WARNING | entity_id, document_count, deactivated_by |
| Catégorie déplacée | WARNING | entity_id, old_parent, new_parent, moved_by |
| Type de document créé | INFO | entity_id, name, category, created_by |
| Type de document désactivé | WARNING | entity_id, document_count, deactivated_by |
| Tag créé | INFO | entity_id, name, created_by |
| Tags fusionnés | WARNING | source_tag, target_tag, documents_affected, merged_by |
| Plan de classement importé | INFO | plan_id, entries_count, imported_by |
| Plan de classement exporté | INFO | plan_id, format, exported_by |

---

## 12. Notifications générées (MODULE 12)

| Déclencheur | Destinataire | Canal | Message |
|---|---|---|---|
| Catégorie désactivée avec documents | Archivistes | In-app | Alerte : la catégorie [X] a été désactivée avec [N] documents |
| Type de document désactivé | Archivistes | In-app | Alerte : le type [X] n'est plus disponible |
| Plan de classement expiré | Admins + Archivistes | Email + In-app | Le plan de classement [X] a expiré |
| Nouveau plan de classement activé | Tous les agents | In-app | Un nouveau plan de classement est en vigueur |

---

*Fin du MODULE 03 — Taxonomie Documentaire*  
*Prochain module : MODULE 04 — Cycle de vie & Politique de conservation*
