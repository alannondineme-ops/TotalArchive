# MODULE 12 — Notifications & Alertes Intelligentes

## Projet : Système d'Archivage Numérique des Documents
**Version :** 1.0  
**Dépendances :** MODULE 01 — Architecture | MODULE 02 — Authentification  
**Statut :** Module transversal — utilisé par TOUS les autres modules

---

## 1. Présentation du module

Dans un système documentaire où de multiples workflows, validations, échéances et événements de sécurité se produisent simultanément, **les utilisateurs doivent être informés au bon moment, par le bon canal, sans être submergés**. Ce module orchestre l'ensemble du système de notifications de l'application.

Les notifications ne sont pas de simples messages : elles sont **intelligentes, contextuelles et adaptatives**. Le système apprend des préférences de l'utilisateur, groupe les notifications similaires pour éviter le spam, et utilise le canal approprié selon l'urgence.

### Canaux de notification supportés

| Canal | Usage | Délai | Coût |
|---|---|---|---|
| **In-app** | Notifications internes à l'application | Temps réel via WebSocket | Gratuit |
| **Email** | Notifications importantes, résumés quotidiens | 1-5 minutes | Gratuit (SMTP) |
| **SMS** | Alertes critiques uniquement (optionnel) | <1 minute | Payant |
| **Desktop push** | Notifications système (Tauri) | Temps réel | Gratuit |
| **Webhook** | Intégrations externes (Slack, Teams, etc.) | <1 minute | Gratuit |

### Principes de conception

1. **Non-intrusif** : L'utilisateur contrôle ce qu'il reçoit et quand
2. **Intelligent** : Groupement automatique des notifications similaires
3. **Actionnable** : Chaque notification contient un lien direct vers l'action à effectuer
4. **Persistant** : Toutes les notifications sont conservées dans l'historique
5. **Multi-appareil** : Synchronisation entre web, desktop et mobile

---

## 2. Types de notifications

### 2.1 Par niveau de priorité

| Priorité | Description | Canaux par défaut | Exemples |
|---|---|---|---|
| **Critique** | Nécessite action immédiate | In-app + Email + SMS | Intégrité document compromise, Tentative intrusion |
| **Haute** | Action requise rapidement | In-app + Email | Validation workflow urgente, Échéance proche |
| **Normale** | Information importante | In-app + Email (résumé quotidien) | Document assigné, Nouvelle version disponible |
| **Basse** | Information secondaire | In-app uniquement | Document consulté, Tag ajouté |

### 2.2 Par catégorie

| Catégorie | Code | Exemples |
|---|---|---|
| Système | `SYSTEM` | Mise à jour application, Maintenance planifiée |
| Authentification | `AUTH` | Connexion nouveau appareil, TOTP désactivé |
| Document | `DOCUMENT` | Document partagé, Nouvelle version, Commentaire ajouté |
| Workflow | `WORKFLOW` | Validation requise, Workflow rejeté, Étape approuvée |
| Échéance | `DEADLINE` | Document expire dans 30 jours, Délai workflow dépassé |
| Sécurité | `SECURITY` | Accès non autorisé, Anomalie détectée |
| IA | `AI` | Classification suggérée, Doublon détecté |
| Mention | `MENTION` | Vous avez été mentionné dans un commentaire |

---

## 3. Cas d'utilisation — Vue d'ensemble

### 3.1 Réception et consultation

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-NOTIF-01 | Recevoir une notification en temps réel | Tout utilisateur |
| UC-NOTIF-02 | Consulter ses notifications non lues | Tout utilisateur |
| UC-NOTIF-03 | Marquer une notification comme lue | Tout utilisateur |
| UC-NOTIF-04 | Marquer toutes les notifications comme lues | Tout utilisateur |
| UC-NOTIF-05 | Accéder directement à la ressource depuis la notification | Tout utilisateur |
| UC-NOTIF-06 | Consulter l'historique complet des notifications | Tout utilisateur |
| UC-NOTIF-07 | Supprimer une notification | Tout utilisateur |
| UC-NOTIF-08 | Rechercher dans les notifications | Tout utilisateur |

### 3.2 Préférences utilisateur

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-NOTIF-09 | Configurer ses préférences de notification | Tout utilisateur |
| UC-NOTIF-10 | Désactiver un type de notification | Tout utilisateur |
| UC-NOTIF-11 | Choisir le canal préféré par type de notification | Tout utilisateur |
| UC-NOTIF-12 | Configurer les horaires de réception (mode silencieux) | Tout utilisateur |
| UC-NOTIF-13 | Configurer la fréquence des résumés par email | Tout utilisateur |
| UC-NOTIF-14 | S'abonner/se désabonner d'un document ou workflow | Tout utilisateur |

### 3.3 Envoi et gestion (système)

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-NOTIF-15 | Créer une notification système | Système (automatique) |
| UC-NOTIF-16 | Grouper les notifications similaires | Système (automatique) |
| UC-NOTIF-17 | Envoyer un résumé quotidien/hebdomadaire | Système (tâche planifiée) |
| UC-NOTIF-18 | Réessayer l'envoi d'une notification échouée | Système (automatique) |
| UC-NOTIF-19 | Envoyer une notification à un groupe d'utilisateurs | Système (automatique) |
| UC-NOTIF-20 | Annuler une notification non encore lue (si événement annulé) | Système (automatique) |

### 3.4 Administration

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-NOTIF-21 | Créer un modèle de notification personnalisé | Admin |
| UC-NOTIF-22 | Envoyer une notification broadcast à tous les utilisateurs | Admin |
| UC-NOTIF-23 | Consulter les statistiques d'envoi | Admin |
| UC-NOTIF-24 | Consulter les notifications échouées | Admin |
| UC-NOTIF-25 | Gérer les templates d'email | Admin |

---

## 4. Description détaillée des cas d'utilisation

### UC-NOTIF-01 — Recevoir une notification en temps réel

**Acteur :** Tout utilisateur  
**Préconditions :** L'utilisateur est connecté  
**Postconditions :** La notification est reçue et affichée

**Scénario principal :**
1. Un événement déclencheur se produit (ex: un document est assigné à l'utilisateur)
2. Le module émetteur (ex: MODULE 06) appelle le service de notification
3. Le service de notification crée l'entrée dans la base de données
4. Le service vérifie les préférences de l'utilisateur pour ce type de notification
5. Le service détermine les canaux d'envoi selon la priorité et les préférences

**Canal In-app (WebSocket) :**
6. Le backend envoie un message WebSocket au client connecté
7. Le frontend React affiche une notification toast en haut à droite (4 secondes)
8. Le compteur de notifications non lues dans le header s'incrémente
9. Le son de notification est joué (si activé dans les préférences)

**Canal Email (si configuré) :**
10. Une tâche Celery asynchrone d'envoi d'email est créée
11. Le système génère l'email depuis le template approprié
12. L'email est envoyé via le serveur SMTP configuré
13. Le statut d'envoi est mis à jour (sent, failed, bounced)

**Canal Desktop Push (Tauri) :**
14. Si l'utilisateur utilise l'app desktop et que la fenêtre n'est pas au premier plan
15. Le système envoie une notification système native via Tauri
16. Une notification apparaît dans le centre de notifications de l'OS

**Règles métier :**
- Si l'utilisateur a désactivé ce type de notification → aucun canal n'est utilisé mais la notification est quand même enregistrée dans l'historique
- Les notifications critiques ignorent les préférences utilisateur (toujours envoyées)
- Les notifications en double (même type + même ressource + délai <5 minutes) sont automatiquement groupées

---

### UC-NOTIF-09 — Configurer ses préférences de notification

**Acteur :** Tout utilisateur  
**Préconditions :** L'utilisateur est authentifié  
**Postconditions :** Les préférences sont sauvegardées et appliquées immédiatement

**Scénario principal :**
1. L'utilisateur accède à ses paramètres de compte → section "Notifications"
2. Le système affiche les préférences organisées par catégorie :

**Catégorie "Documents" :**
- ☑ Document assigné → Canal : In-app + Email
- ☑ Nouvelle version disponible → Canal : In-app uniquement
- ☑ Commentaire ajouté → Canal : In-app uniquement
- ☐ Document consulté → Désactivé
- ☑ Document partagé avec moi → Canal : In-app + Email

**Catégorie "Workflows" :**
- ☑ Validation requise → Canal : In-app + Email immédiat
- ☑ Workflow rejeté → Canal : In-app + Email immédiat
- ☑ Workflow approuvé → Canal : In-app uniquement
- ☑ Délai workflow proche → Canal : In-app + Email

**Catégorie "Sécurité" :**
- ☑ Connexion nouveau appareil → Canal : Email immédiat (non désactivable)
- ☑ Anomalie détectée → Canal : In-app + Email (non désactivable)

**Catégorie "Système" :**
- ☑ Maintenance planifiée → Canal : In-app + Email
- ☑ Mise à jour disponible → Canal : In-app uniquement

3. L'utilisateur peut pour chaque type :
   - Activer/désactiver la notification
   - Choisir les canaux (in-app, email, SMS si disponible)
   - Choisir la fréquence pour les emails (immédiat, résumé quotidien, résumé hebdomadaire)

4. L'utilisateur configure les options globales :
   - **Mode silencieux** : Désactiver les notifications entre 22h et 7h
   - **Groupement intelligent** : Grouper les notifications similaires (activé par défaut)
   - **Son de notification** : Activer/désactiver
   - **Résumé quotidien** : Email récapitulatif envoyé à 9h chaque jour (activé/désactivé)
   - **Résumé hebdomadaire** : Email récapitulatif envoyé le lundi 9h (activé/désactivé)

5. L'utilisateur sauvegarde ses préférences
6. Le système applique immédiatement les nouvelles préférences

---

### UC-NOTIF-16 — Grouper les notifications similaires

**Acteur :** Système (automatique)  
**Préconditions :** Plusieurs notifications similaires sont créées dans un court délai  
**Postconditions :** Les notifications sont groupées en une seule notification agrégée

**Scénario principal :**
1. Un utilisateur effectue une action répétitive (ex: valide 10 étapes de workflow successivement)
2. Chaque action génère une notification pour le créateur des documents
3. Le système détecte que les notifications sont similaires :
   - Même catégorie (`WORKFLOW`)
   - Même type (`workflow_approved`)
   - Même utilisateur destinataire
   - Délai < 5 minutes entre chaque notification
4. Le système crée une notification groupée :

**Au lieu de :**
> ✅ Le workflow du document "Contrat A" a été approuvé  
> ✅ Le workflow du document "Contrat B" a été approuvé  
> ✅ Le workflow du document "Contrat C" a été approuvé  
> ... (10 notifications)

**Le système crée :**
> ✅ 10 workflows ont été approuvés par Jean Dupont  
> → Contrat A, Contrat B, Contrat C... [Voir tout]

5. Les notifications individuelles sont marquées comme "groupées" et liées à la notification agrégée
6. L'utilisateur peut déplier la notification groupée pour voir les détails de chaque élément

---

### UC-NOTIF-17 — Envoyer un résumé quotidien/hebdomadaire

**Acteur :** Système (tâche Celery planifiée)  
**Préconditions :** Des utilisateurs ont activé le résumé quotidien ou hebdomadaire  
**Postconditions :** Un email récapitulatif est envoyé

**Scénario principal :**
1. La tâche Celery `notifications.send_daily_digest` s'exécute chaque jour à 9h
2. Le système identifie tous les utilisateurs ayant activé le résumé quotidien
3. Pour chaque utilisateur, le système collecte toutes les notifications non lues des dernières 24h
4. Le système génère un email structuré :

**Structure de l'email de résumé :**
```
Objet : Résumé quotidien de vos notifications - 18 février 2026

Bonjour Jean,

Voici un récapitulatif de votre activité dans le système d'archivage :

🔴 Urgent (3)
─────────────
• Validation requise : Document "Budget 2026" (échéance : aujourd'hui)
• Validation requise : Document "Contrat Fournisseur X" (échéance : demain)
• Délai workflow dépassé : Document "Rapport Q1"

📄 Documents (5)
─────────────
• 3 nouveaux documents vous ont été assignés
• 2 documents que vous suivez ont été modifiés

💬 Mentions (1)
─────────────
• Marie Martin vous a mentionné dans un commentaire sur "Projet Alpha"

📊 Statistiques
─────────────
• Vous avez 12 notifications non lues au total
• 5 documents en attente de votre validation

[Accéder à mes notifications] [Gérer mes préférences]
```

5. L'email est envoyé via SMTP
6. Les notifications incluses dans le résumé sont marquées avec `included_in_digest_at`
7. Le système enregistre l'envoi du résumé dans `notification_digests`

---

### UC-NOTIF-22 — Envoyer une notification broadcast à tous les utilisateurs

**Acteur :** Admin  
**Préconditions :** L'acteur a la permission `notifications.send_broadcast`  
**Postconditions :** Tous les utilisateurs actifs reçoivent la notification

**Scénario principal :**
1. L'admin accède à l'interface d'administration → "Envoyer une annonce"
2. L'admin remplit le formulaire :
   - **Titre** : "Maintenance planifiée - 20 février 2026"
   - **Message** : "Le système sera indisponible de 2h à 4h pour maintenance..."
   - **Priorité** : Haute
   - **Catégorie** : Système
   - **Destinataires** : Tous les utilisateurs actifs / Rôles spécifiques / Départements spécifiques
   - **Canaux** : In-app + Email
   - **Date d'envoi** : Immédiat ou programmé
3. L'admin prévisualise la notification
4. L'admin confirme l'envoi
5. Le système crée une tâche Celery de broadcast
6. Le système crée une notification pour chaque utilisateur cible (insertion en masse)
7. Les notifications sont envoyées via tous les canaux configurés
8. Un rapport d'envoi est généré : X notifications créées, Y emails envoyés, Z échecs

---

## 5. Modèles de données

### 5.1 Modèle `Notification` (Notification principale)

**Table :** `notifications`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `recipient_id` | UUID | FK → users.id, NOT NULL, INDEX | Destinataire |
| `category` | VARCHAR(30) | NOT NULL, INDEX | Catégorie : `SYSTEM`, `AUTH`, `DOCUMENT`, `WORKFLOW`, etc. |
| `notification_type` | VARCHAR(50) | NOT NULL | Type précis : `document_assigned`, `workflow_validation_required`, etc. |
| `priority` | VARCHAR(20) | NOT NULL, INDEX | Priorité : `critical`, `high`, `normal`, `low` |
| `title` | VARCHAR(200) | NOT NULL | Titre de la notification |
| `message` | TEXT | NOT NULL | Message complet |
| `action_url` | VARCHAR(500) | NULL | URL vers la ressource concernée |
| `action_label` | VARCHAR(100) | NULL | Label du bouton d'action (ex: "Voir le document") |
| `resource_type` | VARCHAR(50) | NULL | Type de ressource : `document`, `workflow`, `user`, etc. |
| `resource_id` | UUID | NULL, INDEX | ID de la ressource |
| `resource_name` | VARCHAR(300) | NULL | Nom de la ressource (dénormalisé) |
| `sender_id` | UUID | FK → users.id, NULL | Qui a généré la notification (si applicable) |
| `is_read` | BOOLEAN | NOT NULL, DEFAULT FALSE, INDEX | Lue ou non |
| `read_at` | TIMESTAMP | NULL | Date de lecture |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | Supprimée par l'utilisateur |
| `deleted_at` | TIMESTAMP | NULL | Date de suppression |
| `is_grouped` | BOOLEAN | NOT NULL, DEFAULT FALSE | Fait partie d'un groupe |
| `group_id` | UUID | FK → notification_groups.id, NULL | Groupe parent |
| `included_in_digest_at` | TIMESTAMP | NULL | Incluse dans résumé email du... |
| `metadata` | JSONB | NULL | Métadonnées supplémentaires |
| `created_at` | TIMESTAMP | NOT NULL, AUTO, INDEX | Date de création |
| `expires_at` | TIMESTAMP | NULL | Date d'expiration (auto-suppression) |

**Index :**
- `idx_notifications_recipient_id_created_at` sur `(recipient_id, created_at DESC)`
- `idx_notifications_recipient_id_is_read` sur `(recipient_id, is_read)`
- `idx_notifications_resource_type_id` sur `(resource_type, resource_id)`
- `idx_notifications_priority` sur `priority`

---

### 5.2 Modèle `NotificationGroup` (Groupe de notifications)

**Table :** `notification_groups`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `recipient_id` | UUID | FK → users.id, NOT NULL | Destinataire |
| `category` | VARCHAR(30) | NOT NULL | Catégorie commune |
| `notification_type` | VARCHAR(50) | NOT NULL | Type commun |
| `title` | VARCHAR(200) | NOT NULL | Titre du groupe |
| `message` | TEXT | NOT NULL | Message agrégé |
| `item_count` | INTEGER | NOT NULL | Nombre de notifications groupées |
| `is_read` | BOOLEAN | NOT NULL, DEFAULT FALSE | Tout le groupe lu |
| `read_at` | TIMESTAMP | NULL | Date de lecture du groupe |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |

---

### 5.3 Modèle `NotificationPreference` (Préférences utilisateur)

**Table :** `notification_preferences`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `user_id` | UUID | FK → users.id, UNIQUE, NOT NULL | Utilisateur |
| `preferences` | JSONB | NOT NULL | Préférences structurées par type |
| `quiet_hours_enabled` | BOOLEAN | NOT NULL, DEFAULT FALSE | Mode silencieux activé |
| `quiet_hours_start` | TIME | NULL | Début du mode silencieux (ex: 22:00) |
| `quiet_hours_end` | TIME | NULL | Fin du mode silencieux (ex: 07:00) |
| `enable_sound` | BOOLEAN | NOT NULL, DEFAULT TRUE | Son de notification |
| `enable_grouping` | BOOLEAN | NOT NULL, DEFAULT TRUE | Groupement intelligent |
| `daily_digest_enabled` | BOOLEAN | NOT NULL, DEFAULT FALSE | Résumé quotidien activé |
| `daily_digest_time` | TIME | NOT NULL, DEFAULT '09:00' | Heure d'envoi du résumé |
| `weekly_digest_enabled` | BOOLEAN | NOT NULL, DEFAULT FALSE | Résumé hebdomadaire activé |
| `weekly_digest_day` | SMALLINT | NOT NULL, DEFAULT 1 | Jour (1=lundi, 7=dimanche) |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `updated_at` | TIMESTAMP | NOT NULL, AUTO | Date de mise à jour |

**Exemple de `preferences` JSONB :**
```json
{
  "document_assigned": {
    "enabled": true,
    "channels": ["in_app", "email"],
    "frequency": "immediate"
  },
  "workflow_validation_required": {
    "enabled": true,
    "channels": ["in_app", "email", "sms"],
    "frequency": "immediate"
  },
  "document_viewed": {
    "enabled": false,
    "channels": [],
    "frequency": null
  },
  "new_version_available": {
    "enabled": true,
    "channels": ["in_app"],
    "frequency": "daily_digest"
  }
}
```

---

### 5.4 Modèle `NotificationChannel` (Canal d'envoi)

**Table :** `notification_channels`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `notification_id` | UUID | FK → notifications.id, NOT NULL, INDEX | Notification |
| `channel_type` | VARCHAR(20) | NOT NULL | Type : `in_app`, `email`, `sms`, `push`, `webhook` |
| `status` | VARCHAR(20) | NOT NULL | Statut : `pending`, `sent`, `failed`, `bounced` |
| `sent_at` | TIMESTAMP | NULL | Date d'envoi |
| `delivered_at` | TIMESTAMP | NULL | Date de livraison (si tracking disponible) |
| `error_message` | TEXT | NULL | Message d'erreur (si échec) |
| `retry_count` | SMALLINT | NOT NULL, DEFAULT 0 | Nombre de tentatives |
| `external_id` | VARCHAR(200) | NULL | ID externe (ex: message ID email) |
| `metadata` | JSONB | NULL | Métadonnées du canal |

**Index :**
- `idx_notification_channels_notification_id` sur `notification_id`
- `idx_notification_channels_status` sur `status`

---

### 5.5 Modèle `NotificationTemplate` (Template de notification)

**Table :** `notification_templates`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `code` | VARCHAR(50) | UNIQUE, NOT NULL | Code technique |
| `name` | VARCHAR(200) | NOT NULL | Nom du template |
| `category` | VARCHAR(30) | NOT NULL | Catégorie |
| `notification_type` | VARCHAR(50) | NOT NULL | Type de notification |
| `title_template` | TEXT | NOT NULL | Template du titre (avec variables) |
| `message_template` | TEXT | NOT NULL | Template du message |
| `email_subject_template` | TEXT | NULL | Sujet de l'email |
| `email_body_template` | TEXT | NULL | Corps de l'email (HTML) |
| `default_priority` | VARCHAR(20) | NOT NULL | Priorité par défaut |
| `default_channels` | ARRAY VARCHAR | NOT NULL | Canaux par défaut |
| `variables` | JSONB | NULL | Variables disponibles |
| `is_system` | BOOLEAN | NOT NULL, DEFAULT FALSE | Template système (non modifiable) |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Template actif |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `updated_at` | TIMESTAMP | NOT NULL, AUTO | Date de mise à jour |

**Exemple de variables :**
```json
{
  "document_title": "string",
  "document_id": "uuid",
  "assigned_by": "string",
  "deadline": "date",
  "workflow_name": "string"
}
```

---

### 5.6 Modèle `NotificationDigest` (Résumé envoyé)

**Table :** `notification_digests`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `user_id` | UUID | FK → users.id, NOT NULL, INDEX | Utilisateur |
| `digest_type` | VARCHAR(20) | NOT NULL | Type : `daily`, `weekly` |
| `notification_count` | INTEGER | NOT NULL | Nombre de notifications incluses |
| `period_start` | TIMESTAMP | NOT NULL | Début de la période |
| `period_end` | TIMESTAMP | NOT NULL | Fin de la période |
| `sent_at` | TIMESTAMP | NOT NULL, AUTO | Date d'envoi |
| `email_status` | VARCHAR(20) | NOT NULL | Statut : `sent`, `failed` |

---

### 5.7 Modèle `NotificationSubscription` (Abonnement à une ressource)

**Table :** `notification_subscriptions`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `user_id` | UUID | FK → users.id, NOT NULL, INDEX | Utilisateur abonné |
| `resource_type` | VARCHAR(50) | NOT NULL | Type : `document`, `workflow`, `category` |
| `resource_id` | UUID | NOT NULL, INDEX | ID de la ressource |
| `subscription_type` | VARCHAR(30) | NOT NULL | Type : `all_events`, `important_only`, `mentions_only` |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date d'abonnement |

**Contrainte :** `UNIQUE (user_id, resource_type, resource_id)`

---

## 6. Matrice des permissions du module

| Permission code | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `notifications.view_own` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `notifications.manage_own_preferences` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `notifications.send_broadcast` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `notifications.manage_templates` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `notifications.view_statistics` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `notifications.view_failed` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |

---

## 7. Endpoints API du module

### 7.1 Notifications

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/notifications/` | Liste des notifications | Oui |
| GET | `/api/v1/notifications/unread-count/` | Nombre de non lues | Oui |
| GET | `/api/v1/notifications/{id}/` | Détail d'une notification | Oui |
| PATCH | `/api/v1/notifications/{id}/mark-read/` | Marquer comme lue | Oui |
| POST | `/api/v1/notifications/mark-all-read/` | Tout marquer comme lu | Oui |
| DELETE | `/api/v1/notifications/{id}/` | Supprimer une notification | Oui |
| POST | `/api/v1/notifications/clear-all/` | Tout supprimer | Oui |

### 7.2 Préférences

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/notifications/preferences/` | Mes préférences | Oui |
| PATCH | `/api/v1/notifications/preferences/` | Modifier préférences | Oui |
| POST | `/api/v1/notifications/preferences/reset/` | Réinitialiser (défaut) | Oui |

### 7.3 Abonnements

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/notifications/subscriptions/` | Mes abonnements | Oui |
| POST | `/api/v1/notifications/subscriptions/` | S'abonner à une ressource | Oui |
| DELETE | `/api/v1/notifications/subscriptions/{id}/` | Se désabonner | Oui |

### 7.4 Administration

| Méthode | URL | Description | Auth |
|---|---|---|---|
| POST | `/api/v1/notifications/broadcast/` | Envoyer annonce globale | Oui + Permission |
| GET | `/api/v1/notifications/templates/` | Liste des templates | Oui + Permission |
| PATCH | `/api/v1/notifications/templates/{id}/` | Modifier template | Oui + Permission |
| GET | `/api/v1/notifications/statistics/` | Statistiques d'envoi | Oui + Permission |

### 7.5 WebSocket (temps réel)

| Event | Description |
|---|---|
| `notification:new` | Nouvelle notification reçue |
| `notification:read` | Notification marquée comme lue |
| `notification:deleted` | Notification supprimée |

---

## 8. Tâches Celery planifiées

| Tâche | Fréquence | Description |
|---|---|---|
| `notifications.send_daily_digests` | Quotidien (9h) | Envoie résumés quotidiens |
| `notifications.send_weekly_digests` | Hebdomadaire (lundi 9h) | Envoie résumés hebdomadaires |
| `notifications.retry_failed_sends` | Horaire | Retente envoi notifications échouées |
| `notifications.cleanup_old_notifications` | Quotidien (3h) | Supprime notifications lues >90 jours |
| `notifications.cleanup_expired_notifications` | Horaire | Supprime notifications expirées |
| `notifications.send_pending_notifications` | Temps réel (queue) | Envoie notifications en attente |

---

## 9. Templates de notification prédéfinis

### 9.1 Documents

| Code | Titre | Canaux par défaut |
|---|---|---|
| `document_assigned` | "Un document vous a été assigné" | in_app + email |
| `document_shared` | "Un document a été partagé avec vous" | in_app + email |
| `new_version_available` | "Nouvelle version disponible" | in_app |
| `document_commented` | "Nouveau commentaire sur un document" | in_app |
| `document_expiring` | "Document arrivant à échéance" | in_app + email |

### 9.2 Workflows

| Code | Titre | Canaux par défaut |
|---|---|---|
| `workflow_validation_required` | "Validation requise" | in_app + email |
| `workflow_approved` | "Workflow approuvé" | in_app |
| `workflow_rejected` | "Workflow rejeté" | in_app + email |
| `workflow_deadline_near` | "Échéance workflow proche" | in_app + email |

### 9.3 Sécurité

| Code | Titre | Canaux par défaut |
|---|---|---|
| `auth_new_device` | "Connexion depuis un nouvel appareil" | email (obligatoire) |
| `security_anomaly` | "Anomalie de sécurité détectée" | in_app + email |
| `account_locked` | "Compte bloqué" | email (obligatoire) |

---

## 10. Événements journalisés (MODULE 11)

| Événement | Niveau | Détails |
|---|---|---|
| Notification envoyée | INFO | recipient_id, notification_type, channels |
| Notification lue | INFO | notification_id, read_by, timestamp |
| Préférences modifiées | INFO | user_id, changes |
| Broadcast envoyé | INFO | sender_id, recipient_count |
| Envoi échoué | WARNING | notification_id, channel, error |

---

*Fin du MODULE 12 — Notifications & Alertes Intelligentes*  
*Prochain module : MODULE 13 — Demandes d'accès & Partage contrôlé*

**🎯 Statut : 61% du cahier des charges (11/18 modules)**
