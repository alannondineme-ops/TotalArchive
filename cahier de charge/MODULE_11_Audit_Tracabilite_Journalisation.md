# MODULE 11 — Audit, Traçabilité & Journalisation

## Projet : Système d'Archivage Numérique des Documents
**Version :** 1.0  
**Dépendances :** MODULE 01 — Architecture | MODULE 02 — Authentification  
**Statut :** Module de conformité — CRITIQUE pour la sécurité et la réglementation

---

## 1. Présentation du module

Dans une administration publique, **la traçabilité n'est pas une option, c'est une obligation légale**. Chaque action effectuée sur un document sensible doit être enregistrée de manière **immuable, horodatée et attribuable**. En cas d'audit de sécurité, de litige juridique ou d'enquête interne, le système doit pouvoir répondre à la question : "Qui a fait quoi, quand, et pourquoi ?"

Ce module constitue le **journal de bord universel** de l'application. Chaque module (02 à 18) génère des événements d'audit qui sont tous centralisés ici. Le journal d'audit est **append-only** (ajout uniquement) : aucune entrée ne peut jamais être modifiée ou supprimée, même par un Super Admin. Cette garantie d'intégrité est essentielle pour la valeur probante du journal.

Au-delà de la simple journalisation, ce module fournit :
- Des **tableaux de bord de sécurité** en temps réel
- Des **alertes automatiques** sur comportements suspects
- Des **rapports d'audit** exportables pour les autorités de contrôle
- Un **moteur de recherche** dans les logs avec filtres avancés
- Une **détection d'anomalies** comportementales

---

## 2. Principes fondamentaux de l'audit

### 2.1 Les 5W+1H de l'audit

Chaque événement enregistré répond aux questions fondamentales :

| Question | Champ correspondant | Exemple |
|---|---|---|
| **Who** (Qui ?) | `user_id` | Jean Dupont (Archiviste) |
| **What** (Quoi ?) | `event_type`, `action`, `description` | Téléchargement de document |
| **When** (Quand ?) | `timestamp` | 2026-02-18 14:35:22.123 |
| **Where** (Où ?) | `ip_address`, `location`, `device` | 192.168.1.50, Paris, Chrome/Windows |
| **Which** (Lequel ?) | `resource_type`, `resource_id` | Document "Contrat-2026-001" |
| **How** (Comment ?) | `interface`, `method` | Web interface, API REST |

### 2.2 Niveaux de sévérité

Chaque événement possède un niveau de sévérité qui détermine l'urgence de réaction :

| Niveau | Code | Description | Exemples | Action automatique |
|---|---|---|---|---|
| **INFO** | `INFO` | Opération normale | Connexion réussie, Document créé | Aucune |
| **WARNING** | `WARNING` | Attention requise | Tentative de connexion échouée (3x), Modification sensible | Surveillance |
| **ALERT** | `ALERT` | Action sensible | Suppression de document, Export massif de données | Notification admin |
| **CRITICAL** | `CRITICAL` | Incident de sécurité | Accès non autorisé détecté, Intégrité compromise | Alerte immédiate + blocage possible |

### 2.3 Catégories d'événements

| Catégorie | Code | Exemples d'événements |
|---|---|---|
| Authentification | `AUTH` | Connexion, Déconnexion, Échec de connexion, TOTP activé |
| Documents | `DOCUMENT` | Création, Modification, Suppression, Téléchargement, Consultation |
| Confidentialité | `CONFIDENTIALITY` | Classification modifiée, Accès accordé/révoqué |
| Workflow | `WORKFLOW` | Soumission, Validation, Rejet |
| Cycle de vie | `LIFECYCLE` | Transition de statut, Élimination, Versement |
| Recherche | `SEARCH` | Recherche effectuée, Export de résultats |
| Administration | `ADMIN` | Création utilisateur, Modification de rôle, Changement de configuration |
| Sécurité | `SECURITY` | Tentative d'accès non autorisé, Anomalie détectée |

---

## 3. Cas d'utilisation — Vue d'ensemble

### 3.1 Journalisation

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-AUD-01 | Enregistrer un événement d'audit | Système (automatique) |
| UC-AUD-02 | Garantir l'immuabilité d'une entrée d'audit | Système (automatique) |
| UC-AUD-03 | Enregistrer une tentative d'accès non autorisé | Système (automatique) |

### 3.2 Consultation

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-AUD-04 | Consulter le journal d'audit global | Admin / Auditeur |
| UC-AUD-05 | Consulter l'audit d'un document spécifique | Tout utilisateur (selon permissions) |
| UC-AUD-06 | Consulter l'audit d'un utilisateur | Admin / Auditeur |
| UC-AUD-07 | Rechercher dans les logs avec filtres | Admin / Auditeur |
| UC-AUD-08 | Consulter ses propres actions | Tout utilisateur |

### 3.3 Analyse et rapports

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-AUD-09 | Générer un rapport d'audit pour une période | Admin / Auditeur |
| UC-AUD-10 | Exporter le journal d'audit (CSV, JSON) | Admin / Auditeur |
| UC-AUD-11 | Consulter les statistiques de sécurité | Admin |
| UC-AUD-12 | Consulter les alertes de sécurité | Admin |
| UC-AUD-13 | Visualiser le dashboard de sécurité en temps réel | Admin |

### 3.4 Détection d'anomalies

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-AUD-14 | Détecter un comportement anormal d'un utilisateur | Système (automatique) |
| UC-AUD-15 | Détecter une tentative d'intrusion | Système (automatique) |
| UC-AUD-16 | Détecter un accès suspect à des documents confidentiels | Système (automatique) |
| UC-AUD-17 | Détecter un export massif de données | Système (automatique) |

### 3.5 Alertes

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-AUD-18 | Configurer une règle d'alerte personnalisée | Admin |
| UC-AUD-19 | Recevoir une alerte de sécurité en temps réel | Admin |
| UC-AUD-20 | Accuser réception d'une alerte | Admin |

---

## 4. Description détaillée des cas d'utilisation

### UC-AUD-01 — Enregistrer un événement d'audit

**Acteur :** Système (déclenché automatiquement par tous les modules)  
**Préconditions :** Une action traçable a lieu dans le système  
**Postconditions :** L'événement est enregistré de manière immuable dans le journal d'audit

**Scénario principal :**
1. Un événement se produit dans n'importe quel module (ex: un utilisateur télécharge un document confidentiel)
2. Le module émetteur appelle la fonction d'audit centralisée avec les paramètres de l'événement
3. Le système collecte automatiquement les informations contextuelles :
   - ID de l'utilisateur actuel (depuis le token JWT)
   - Adresse IP de la requête
   - User-Agent (navigateur/appareil)
   - Timestamp précis (millisecondes)
   - Session ID
4. Le système détermine automatiquement le niveau de sévérité selon le type d'événement
5. Le système génère un hash de l'entrée précédente pour chaînage (garantie d'intégrité)
6. Le système insère l'événement dans la table `audit_logs`
7. Si le niveau est ALERT ou CRITICAL : le système déclenche immédiatement une tâche de notification
8. L'événement est également envoyé à un système de logs externes (optionnel : Syslog, ELK Stack)

**Règles métier :**
- Aucune entrée d'audit ne peut être modifiée après insertion (contrainte en base de données)
- Les événements critiques sont dupliqués dans un système externe pour garantir la non-répudiation
- Les données sensibles (ex: mot de passe) ne sont JAMAIS enregistrées dans les logs, même hachées

---

### UC-AUD-02 — Garantir l'immuabilité d'une entrée d'audit

**Acteur :** Système (mécanisme technique permanent)  
**Préconditions :** Le journal d'audit existe  
**Postconditions :** Toute tentative de modification est impossible et détectée

**Mécanisme d'immuabilité :**

**1. Contrainte en base de données :**
```sql
-- Aucune permission UPDATE ou DELETE sur la table audit_logs
REVOKE UPDATE, DELETE ON audit_logs FROM ALL;
```

**2. Chaînage cryptographique :**
Chaque entrée d'audit contient le hash SHA-256 de l'entrée précédente, créant une blockchain d'audit :
```
Entry N:
  - id: uuid-N
  - event: "Document downloaded"
  - previous_hash: SHA256(Entry N-1)
  - current_hash: SHA256(Entry N)

Entry N+1:
  - id: uuid-N+1
  - event: "Document modified"
  - previous_hash: SHA256(Entry N)  ← doit correspondre au current_hash de N
  - current_hash: SHA256(Entry N+1)
```

Si une entrée est modifiée, le chaînage est rompu et l'altération est immédiatement détectable.

**3. Vérification périodique d'intégrité :**
Une tâche Celery quotidienne vérifie l'intégrité complète du journal :
- Recalcule tous les hashs
- Vérifie la cohérence du chaînage
- Si une incohérence est détectée → alerte CRITIQUE aux administrateurs

---

### UC-AUD-07 — Rechercher dans les logs avec filtres

**Acteur :** Admin ou Auditeur  
**Préconditions :** L'acteur est authentifié avec les permissions d'audit  
**Postconditions :** Les logs filtrés sont affichés

**Scénario principal :**
1. L'acteur accède au module d'audit
2. L'acteur configure les filtres de recherche :
   - **Période** : date de début et de fin
   - **Utilisateur** : filtrer par utilisateur spécifique
   - **Catégorie** : AUTH, DOCUMENT, SECURITY, etc.
   - **Niveau de sévérité** : INFO, WARNING, ALERT, CRITICAL
   - **Type d'événement** : connexion, téléchargement, modification, etc.
   - **Ressource** : filtrer par document, workflow, utilisateur cible
   - **Adresse IP** : filtrer par plage IP
   - **Texte libre** : recherche dans la description des événements
3. L'acteur applique les filtres
4. Le système exécute la recherche (indexée pour performances)
5. Le système affiche les résultats paginés (20 par page)
6. Pour chaque entrée, l'acteur voit :
   - Horodatage précis
   - Utilisateur
   - Action effectuée
   - Ressource concernée
   - Niveau de sévérité (badge coloré)
   - Adresse IP / Appareil
7. L'acteur peut cliquer sur une entrée pour voir les détails complets (JSON)
8. L'acteur peut exporter les résultats filtrés (CSV, JSON, PDF)

**Filtres avancés disponibles :**
- **Chronologie inversée** : afficher du plus récent au plus ancien
- **Regroupement** : grouper par utilisateur, par jour, par type d'événement
- **Exclusion** : exclure certains types d'événements (ex: connexions réussies)

---

### UC-AUD-14 — Détecter un comportement anormal d'un utilisateur

**Acteur :** Système (tâche automatique d'analyse comportementale)  
**Préconditions :** Suffisamment d'historique existe pour cet utilisateur (>30 jours)  
**Postconditions :** Une anomalie est détectée et signalée

**Scénario principal :**
1. Une tâche Celery s'exécute toutes les heures
2. Le système analyse les patterns de comportement de chaque utilisateur
3. Le système calcule des métriques normales pour chaque utilisateur :
   - Nombre moyen de connexions par jour
   - Heures habituelles de connexion
   - Nombre moyen de documents consultés par session
   - Types de documents habituellement consultés
   - Adresses IP habituelles
4. Le système compare l'activité récente (dernières 24h) aux patterns habituels
5. Le système détecte des anomalies :

**Exemples d'anomalies détectées :**
| Anomalie | Description | Seuil |
|---|---|---|
| Connexion hors horaires | Connexion à 3h du matin alors que l'utilisateur se connecte habituellement entre 9h-18h | Écart >6h |
| Volume anormal de téléchargements | 50 documents téléchargés alors que la moyenne est 5/jour | 10x la moyenne |
| Accès à des documents inhabituels | Accès à des documents Secret alors que l'utilisateur consulte habituellement des documents Publics | Catégorie inhabituelle |
| Connexion depuis IP nouvelle | Connexion depuis un pays étranger alors que l'utilisateur se connecte toujours depuis le bureau | Localisation inhabituelle |
| Export massif de données | Export de 500 documents en une heure | >100 exports/heure |

6. Si une anomalie est détectée, le système :
   - Crée une alerte de sécurité (niveau WARNING ou ALERT)
   - Notifie les administrateurs immédiatement
   - Si l'anomalie est critique (ex: export massif depuis IP étrangère) → peut bloquer temporairement le compte en attente de vérification

**Règles métier :**
- Les faux positifs sont inévitables (ex: un archiviste travaillant un samedi) → l'admin peut marquer une alerte comme "comportement légitime"
- Si un comportement marqué "légitime" se répète, le système apprend et ne l'alerte plus

---

### UC-AUD-18 — Configurer une règle d'alerte personnalisée

**Acteur :** Admin  
**Préconditions :** L'acteur est authentifié avec les permissions d'administration  
**Postconditions :** Une nouvelle règle d'alerte est active

**Scénario principal :**
1. L'admin accède à la configuration des règles d'alerte
2. L'admin clique sur "Créer une règle"
3. L'admin configure la règle :
   - **Nom** : "Accès massif à documents Secret"
   - **Condition** : Si un utilisateur consulte plus de 10 documents Secret en moins de 1 heure
   - **Niveau** : ALERT
   - **Action** : Notifier admins par email + in-app
   - **Seuil** : 10 documents / 1 heure
   - **Exclusions** : Exclure les utilisateurs avec le rôle "Auditeur" (accès légitime)
4. L'admin active la règle
5. Le système surveille en temps réel les événements correspondants
6. Si la condition est remplie, le système déclenche l'alerte configurée

**Exemples de règles prédéfinies disponibles :**
- Tentatives de connexion échouées répétées (>5 en 10 minutes)
- Modification d'un document par un utilisateur autre que le propriétaire
- Téléchargement d'un document Secret en dehors des heures de bureau
- Suppression de plus de 5 documents en moins de 5 minutes
- Accès à un document gelé (MODULE 04)

---

## 5. Modèles de données

### 5.1 Modèle `AuditLog` (Entrée du journal d'audit)

**Table :** `audit_logs`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `sequence_number` | BIGSERIAL | UNIQUE, NOT NULL | Numéro séquentiel (garantit l'ordre) |
| `timestamp` | TIMESTAMP(6) | NOT NULL, INDEX | Horodatage précis (microsecondes) |
| `event_category` | VARCHAR(30) | NOT NULL, INDEX | Catégorie : `AUTH`, `DOCUMENT`, `CONFIDENTIALITY`, etc. |
| `event_type` | VARCHAR(50) | NOT NULL, INDEX | Type précis : `user_login`, `document_downloaded`, etc. |
| `severity_level` | VARCHAR(20) | NOT NULL, INDEX | Niveau : `INFO`, `WARNING`, `ALERT`, `CRITICAL` |
| `user_id` | UUID | FK → users.id, NULL, INDEX | Utilisateur acteur (NULL si système) |
| `impersonated_by_id` | UUID | FK → users.id, NULL | Si action effectuée via impersonation |
| `resource_type` | VARCHAR(50) | NULL | Type de ressource : `document`, `user`, `workflow`, etc. |
| `resource_id` | UUID | NULL, INDEX | ID de la ressource concernée |
| `resource_name` | VARCHAR(300) | NULL | Nom de la ressource (dénormalisé) |
| `action` | VARCHAR(50) | NOT NULL | Action : `created`, `updated`, `deleted`, `viewed`, `downloaded` |
| `description` | TEXT | NOT NULL | Description textuelle de l'événement |
| `old_value` | JSONB | NULL | Valeur avant modification (si applicable) |
| `new_value` | JSONB | NULL | Valeur après modification (si applicable) |
| `ip_address` | INET | NULL, INDEX | Adresse IP source |
| `user_agent` | TEXT | NULL | User-Agent HTTP |
| `device_type` | VARCHAR(20) | NULL | Type : `web`, `desktop`, `mobile` |
| `location` | VARCHAR(200) | NULL | Localisation approximative (ville, pays) |
| `session_id` | VARCHAR(100) | NULL, INDEX | ID de session |
| `request_id` | VARCHAR(100) | NULL | ID de requête (traçabilité technique) |
| `interface` | VARCHAR(20) | NOT NULL | Interface : `web`, `api`, `desktop`, `cli` |
| `http_method` | VARCHAR(10) | NULL | Méthode HTTP : GET, POST, PATCH, DELETE |
| `endpoint` | VARCHAR(500) | NULL | Endpoint API appelé |
| `response_status` | SMALLINT | NULL | Code HTTP de réponse |
| `execution_time_ms` | INTEGER | NULL | Temps d'exécution (ms) |
| `error_message` | TEXT | NULL | Message d'erreur (si échec) |
| `tags` | ARRAY VARCHAR | NULL | Tags personnalisés |
| `previous_entry_hash` | VARCHAR(64) | NULL | Hash SHA-256 de l'entrée précédente (chaînage) |
| `current_entry_hash` | VARCHAR(64) | NOT NULL | Hash SHA-256 de cette entrée |
| `is_anomaly` | BOOLEAN | NOT NULL, DEFAULT FALSE | Marqué comme anomalie comportementale |

**Contraintes :**
- Aucune UPDATE ou DELETE autorisée (contrainte au niveau rôle PostgreSQL)
- `severity_level` : valeur dans `{INFO, WARNING, ALERT, CRITICAL}`
- `event_category` : valeur dans `{AUTH, DOCUMENT, CONFIDENTIALITY, LIFECYCLE, WORKFLOW, SEARCH, ADMIN, SECURITY, SYSTEM}`

**Index :**
- `idx_audit_logs_timestamp` sur `timestamp` DESC
- `idx_audit_logs_user_id` sur `user_id`
- `idx_audit_logs_resource_type_id` sur `(resource_type, resource_id)`
- `idx_audit_logs_severity_level` sur `severity_level`
- `idx_audit_logs_event_category` sur `event_category`
- `idx_audit_logs_ip_address` sur `ip_address`
- `idx_audit_logs_session_id` sur `session_id`
- Index full-text sur `description` pour recherche textuelle

**Partitionnement :**
La table est partitionnée par mois pour maintenir les performances sur de gros volumes :
```sql
CREATE TABLE audit_logs_2026_02 PARTITION OF audit_logs
FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
```

---

### 5.2 Modèle `SecurityAlert` (Alerte de sécurité)

**Table :** `security_alerts`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `alert_type` | VARCHAR(50) | NOT NULL | Type : `anomaly_detected`, `unauthorized_access`, `mass_export`, etc. |
| `severity` | VARCHAR(20) | NOT NULL | Niveau : `WARNING`, `ALERT`, `CRITICAL` |
| `title` | VARCHAR(200) | NOT NULL | Titre de l'alerte |
| `description` | TEXT | NOT NULL | Description détaillée |
| `user_id` | UUID | FK → users.id, NULL | Utilisateur concerné |
| `resource_type` | VARCHAR(50) | NULL | Type de ressource |
| `resource_id` | UUID | NULL | ID de la ressource |
| `related_audit_log_ids` | ARRAY UUID | NULL | IDs des entrées d'audit liées |
| `detection_score` | DECIMAL(5,2) | NULL | Score de confiance (0-100) |
| `detection_rule_id` | UUID | FK → alert_rules.id, NULL | Règle ayant déclenché l'alerte |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT 'pending' | Statut : `pending`, `acknowledged`, `investigating`, `resolved`, `false_positive` |
| `acknowledged_by_id` | UUID | FK → users.id, NULL | Qui a pris connaissance |
| `acknowledged_at` | TIMESTAMP | NULL | Date de prise de connaissance |
| `resolved_by_id` | UUID | FK → users.id, NULL | Qui a résolu |
| `resolved_at` | TIMESTAMP | NULL | Date de résolution |
| `resolution_notes` | TEXT | NULL | Notes de résolution |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |

**Index :**
- `idx_security_alerts_status` sur `status`
- `idx_security_alerts_severity` sur `severity`
- `idx_security_alerts_user_id` sur `user_id`
- `idx_security_alerts_created_at` sur `created_at` DESC

---

### 5.3 Modèle `AlertRule` (Règle d'alerte personnalisée)

**Table :** `alert_rules`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `name` | VARCHAR(200) | NOT NULL | Nom de la règle |
| `description` | TEXT | NULL | Description |
| `rule_type` | VARCHAR(50) | NOT NULL | Type : `event_count`, `time_window`, `pattern_match` |
| `conditions` | JSONB | NOT NULL | Conditions de déclenchement |
| `severity` | VARCHAR(20) | NOT NULL | Niveau d'alerte généré |
| `notification_channels` | ARRAY VARCHAR | NOT NULL | Canaux : `email`, `in_app`, `sms`, `webhook` |
| `notification_recipients` | ARRAY UUID | NULL | IDs des utilisateurs à notifier |
| `excluded_users` | ARRAY UUID | NULL | Utilisateurs exclus de cette règle |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Règle active |
| `trigger_count` | INTEGER | NOT NULL, DEFAULT 0 | Nombre de fois déclenchée |
| `last_triggered_at` | TIMESTAMP | NULL | Dernière fois déclenchée |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `created_by_id` | UUID | FK → users.id, NOT NULL | Créateur |

**Exemple de `conditions` JSONB :**
```json
{
  "event_type": "document_downloaded",
  "resource_type": "document",
  "confidentiality_level": "secret",
  "threshold": {
    "count": 10,
    "time_window_minutes": 60
  },
  "additional_filters": {
    "outside_business_hours": true
  }
}
```

---

### 5.4 Modèle `AuditExport` (Export du journal d'audit)

**Table :** `audit_exports`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `export_type` | VARCHAR(20) | NOT NULL | Format : `csv`, `json`, `pdf` |
| `filters_applied` | JSONB | NOT NULL | Filtres utilisés pour l'export |
| `date_from` | TIMESTAMP | NOT NULL | Début de la période exportée |
| `date_to` | TIMESTAMP | NOT NULL | Fin de la période exportée |
| `entry_count` | INTEGER | NOT NULL | Nombre d'entrées exportées |
| `file_path` | VARCHAR(500) | NOT NULL | Chemin du fichier généré |
| `file_size_bytes` | BIGINT | NOT NULL | Taille du fichier |
| `file_hash_sha256` | VARCHAR(64) | NOT NULL | Hash du fichier (intégrité) |
| `exported_by_id` | UUID | FK → users.id, NOT NULL | Qui a exporté |
| `exported_at` | TIMESTAMP | NOT NULL, AUTO | Date d'export |
| `purpose` | TEXT | NULL | Raison de l'export |

---

## 6. Matrice des permissions du module

| Permission code | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `audit.view_all_logs` | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| `audit.view_own_logs` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `audit.view_document_logs` | ✅ | ✅ | ✅ | ✅ | ✅ (si accès doc) | ✅ |
| `audit.export_logs` | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| `audit.view_security_alerts` | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| `audit.manage_alert_rules` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `audit.acknowledge_alert` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `audit.view_dashboard` | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |

---

## 7. Endpoints API du module

### 7.1 Journal d'audit

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/audit/logs/` | Liste des logs avec filtres | Oui + Permission |
| GET | `/api/v1/audit/logs/{id}/` | Détail d'une entrée | Oui + Permission |
| GET | `/api/v1/audit/documents/{doc_id}/logs/` | Logs d'un document | Oui + Permission |
| GET | `/api/v1/audit/users/{user_id}/logs/` | Logs d'un utilisateur | Oui + Permission |
| GET | `/api/v1/audit/my-activity/` | Mes propres logs | Oui |
| POST | `/api/v1/audit/verify-integrity/` | Vérifier l'intégrité du journal | Oui + Admin |

### 7.2 Alertes de sécurité

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/audit/alerts/` | Liste des alertes | Oui + Permission |
| GET | `/api/v1/audit/alerts/{id}/` | Détail d'une alerte | Oui + Permission |
| POST | `/api/v1/audit/alerts/{id}/acknowledge/` | Accuser réception | Oui + Permission |
| POST | `/api/v1/audit/alerts/{id}/resolve/` | Marquer comme résolu | Oui + Permission |
| POST | `/api/v1/audit/alerts/{id}/mark-false-positive/` | Marquer faux positif | Oui + Permission |

### 7.3 Règles d'alerte

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/audit/alert-rules/` | Liste des règles | Oui + Permission |
| POST | `/api/v1/audit/alert-rules/` | Créer une règle | Oui + Permission |
| PATCH | `/api/v1/audit/alert-rules/{id}/` | Modifier | Oui + Permission |
| DELETE | `/api/v1/audit/alert-rules/{id}/` | Supprimer | Oui + Permission |
| POST | `/api/v1/audit/alert-rules/{id}/test/` | Tester la règle | Oui + Permission |

### 7.4 Export et rapports

| Méthode | URL | Description | Auth |
|---|---|---|---|
| POST | `/api/v1/audit/export/` | Créer un export | Oui + Permission |
| GET | `/api/v1/audit/exports/` | Liste des exports | Oui + Permission |
| GET | `/api/v1/audit/exports/{id}/download/` | Télécharger export | Oui + Permission |
| GET | `/api/v1/audit/dashboard/` | Dashboard de sécurité | Oui + Permission |
| GET | `/api/v1/audit/stats/` | Statistiques d'audit | Oui + Permission |

---

## 8. Tâches Celery planifiées

| Tâche | Fréquence | Description |
|---|---|---|
| `audit.verify_integrity` | Quotidien (4h) | Vérifie l'intégrité complète du journal (chaînage) |
| `audit.detect_anomalies` | Horaire | Détecte les comportements anormaux des utilisateurs |
| `audit.process_alert_rules` | Temps réel (via stream) | Évalue les règles d'alerte en continu |
| `audit.partition_old_logs` | Mensuel (1er du mois 5h) | Crée une nouvelle partition pour le mois en cours |
| `audit.archive_old_logs` | Annuel (janvier) | Archive les logs >5 ans vers stockage froid |
| `audit.cleanup_resolved_alerts` | Hebdomadaire (dimanche 6h) | Archive les alertes résolues >6 mois |
| `audit.generate_compliance_report` | Mensuel (dernier jour du mois) | Génère rapport de conformité mensuel |

---

## 9. Règles métier critiques

### 9.1 Règle d'immuabilité absolue

Aucune UPDATE ou DELETE n'est autorisée sur la table `audit_logs`, même pour un Super Admin. En cas d'erreur dans une entrée, la correction consiste à créer une nouvelle entrée de type `correction` qui référence l'entrée erronée.

### 9.2 Règle de rétention légale

Les logs sont conservés selon les durées légales :
- **Logs d'authentification** : 1 an minimum (réglementation RGPD)
- **Logs de sécurité** : 5 ans minimum
- **Logs documentaires** : durée de conservation du document + 5 ans
- **Logs d'élimination** : conservation permanente (preuve de destruction légale)

### 9.3 Règle de protection des données personnelles

Les logs contiennent des données personnelles (user_id, IP). Conformité RGPD :
- Droit d'accès : un utilisateur peut consulter ses propres logs
- Droit de rectification : N/A (immuabilité du journal)
- Droit à l'effacement : applicable uniquement après expiration légale
- Pseudonymisation : envisageable après archivage vers stockage froid

---

## 10. Dashboard de sécurité — Métriques affichées

Le dashboard temps réel affiche :

**Métriques globales :**
- Nombre d'événements dans les dernières 24h
- Nombre d'alertes en cours (par sévérité)
- Top 10 des utilisateurs les plus actifs
- Top 10 des documents les plus consultés
- Répartition des événements par catégorie (graphique circulaire)

**Métriques de sécurité :**
- Tentatives de connexion échouées (dernières 24h)
- Accès refusés (dernières 24h)
- Anomalies détectées (derniers 7 jours)
- Alertes critiques non résolues

**Graphiques temporels :**
- Évolution du nombre d'événements (derniers 30 jours)
- Heures de pointe d'activité
- Répartition géographique des connexions (carte)

---

*Fin du MODULE 11 — Audit, Traçabilité & Journalisation*  
*Prochain module : MODULE 12 — Notifications & Alertes intelligentes*
