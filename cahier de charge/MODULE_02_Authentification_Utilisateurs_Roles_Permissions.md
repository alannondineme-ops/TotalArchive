# MODULE 02 — Authentification, Utilisateurs, Rôles & Permissions

## Projet : Système d'Archivage Numérique des Documents
**Version :** 1.0  
**Dépendances :** MODULE 01 — Architecture Générale & Conventions du Projet  
**Statut :** Fondation critique — tous les modules suivants dépendent de ce module

---

## 1. Présentation du module

Ce module constitue le pilier de sécurité de toute l'application. Il gouverne l'identité de chaque utilisateur, la manière dont il s'authentifie, les droits qu'il possède et les actions qu'il peut effectuer sur chaque ressource du système.

Aucun autre module ne peut fonctionner sans que celui-ci soit pleinement opérationnel. Toute requête à l'API, toute action sur un document, toute consultation d'un tableau de bord passe obligatoirement par les mécanismes définis ici.

---

## 2. Cas d'utilisation — Vue d'ensemble

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-AUTH-01 | Connexion à l'application | Tout utilisateur |
| UC-AUTH-02 | Déconnexion de l'application | Tout utilisateur connecté |
| UC-AUTH-03 | Rafraîchissement du token d'accès | Système (automatique) |
| UC-AUTH-04 | Activation du TOTP | Utilisateur connecté |
| UC-AUTH-05 | Désactivation du TOTP | Utilisateur connecté |
| UC-AUTH-06 | Vérification du code TOTP lors de la connexion | Utilisateur avec TOTP activé |
| UC-AUTH-07 | Réinitialisation du mot de passe (oubli) | Tout utilisateur |
| UC-AUTH-08 | Changement du mot de passe (connecté) | Tout utilisateur connecté |
| UC-AUTH-09 | Déverrouillage d'un compte bloqué | Super Admin / Admin |
| UC-AUTH-10 | Consultation de ses propres sessions actives | Tout utilisateur connecté |
| UC-AUTH-11 | Révocation d'une session active | Tout utilisateur connecté |
| UC-USER-01 | Créer un utilisateur | Super Admin / Admin |
| UC-USER-02 | Consulter la liste des utilisateurs | Super Admin / Admin |
| UC-USER-03 | Consulter le profil d'un utilisateur | Super Admin / Admin / Soi-même |
| UC-USER-04 | Modifier les informations d'un utilisateur | Super Admin / Admin |
| UC-USER-05 | Désactiver un compte utilisateur | Super Admin / Admin |
| UC-USER-06 | Réactiver un compte utilisateur | Super Admin / Admin |
| UC-USER-07 | Supprimer (logiquement) un compte utilisateur | Super Admin uniquement |
| UC-USER-08 | Modifier son propre profil | Tout utilisateur connecté |
| UC-USER-09 | Consulter son propre profil | Tout utilisateur connecté |
| UC-ROLE-01 | Créer un rôle personnalisé | Super Admin uniquement |
| UC-ROLE-02 | Modifier les permissions d'un rôle | Super Admin uniquement |
| UC-ROLE-03 | Assigner un rôle à un utilisateur | Super Admin / Admin |
| UC-ROLE-04 | Révoquer un rôle d'un utilisateur | Super Admin / Admin |
| UC-ROLE-05 | Consulter la matrice des permissions | Super Admin / Admin |
| UC-PERM-01 | Vérifier les permissions d'un utilisateur | Système (automatique) |
| UC-PERM-02 | Accès refusé — notification et journalisation | Système (automatique) |

---

## 3. Description détaillée des cas d'utilisation

### UC-AUTH-01 — Connexion à l'application

**Acteur :** Tout utilisateur enregistré  
**Préconditions :** L'utilisateur possède un compte actif  
**Postconditions :** L'utilisateur est authentifié et reçoit un token d'accès JWT et un token de rafraîchissement

**Scénario principal :**
1. L'utilisateur saisit son adresse email et son mot de passe
2. Le système vérifie que le compte existe et est actif
3. Le système vérifie que le compte n'est pas bloqué
4. Le système compare le mot de passe saisi avec le hash stocké (Argon2)
5. Si le TOTP est désactivé sur ce compte : le système génère et retourne un token d'accès JWT et un token de rafraîchissement
6. Si le TOTP est activé sur ce compte : le système retourne un token temporaire de vérification TOTP (valide 5 minutes) et redirige vers l'étape de vérification TOTP (UC-AUTH-06)
7. Le système enregistre la tentative de connexion réussie dans le journal d'audit
8. Le système met à jour la date de dernière connexion de l'utilisateur

**Scénarios alternatifs :**
- **3a.** Le compte est bloqué → retourner un message d'erreur générique sans préciser la raison du blocage → enregistrer la tentative dans le journal d'audit
- **4a.** Le mot de passe est incorrect → incrémenter le compteur de tentatives échouées → si le compteur atteint le seuil configuré (défaut : 5 tentatives), bloquer le compte → retourner un message d'erreur générique
- **4b.** Le compte n'existe pas → retourner exactement le même message d'erreur générique qu'en cas de mauvais mot de passe (protection contre l'énumération de comptes)

**Règles métier :**
- Le message d'erreur en cas d'échec est toujours identique, quel que soit le motif (compte inexistant, mauvais mot de passe, compte bloqué), pour empêcher l'énumération
- La tentative de connexion (réussie ou échouée) est toujours journalisée avec l'adresse IP, le navigateur/client et l'horodatage

---

### UC-AUTH-02 — Déconnexion de l'application

**Acteur :** Tout utilisateur connecté  
**Préconditions :** L'utilisateur est authentifié  
**Postconditions :** Les tokens de l'utilisateur sont révoqués et sa session est terminée

**Scénario principal :**
1. L'utilisateur déclenche la déconnexion
2. Le système ajoute le token de rafraîchissement actuel à la liste noire (blacklist Redis)
3. Le système invalide le token d'accès actuel (blacklist Redis avec TTL correspondant à la durée de vie restante du token)
4. Le système supprime les données de session côté client (localStorage vidé, cookies supprimés)
5. Le système redirige vers la page de connexion
6. L'événement de déconnexion est journalisé

**Règles métier :**
- La déconnexion révoque uniquement la session courante, pas toutes les sessions (sauf si l'utilisateur choisit "Déconnecter toutes les sessions" via UC-AUTH-11)
- En mode desktop Tauri, la déconnexion supprime également le cache local chiffré

---

### UC-AUTH-03 — Rafraîchissement du token d'accès

**Acteur :** Système (déclenché automatiquement par le frontend)  
**Préconditions :** L'utilisateur possède un token de rafraîchissement valide et non révoqué  
**Postconditions :** Un nouveau token d'accès est émis

**Scénario principal :**
1. Le frontend détecte que le token d'accès est expiré ou sur le point d'expirer (seuil : 2 minutes avant expiration)
2. Le frontend envoie le token de rafraîchissement à l'endpoint dédié
3. Le système vérifie que le token de rafraîchissement est valide, non expiré et non présent dans la blacklist
4. Le système émet un nouveau token d'accès (rotation simple)
5. Le frontend remplace le token d'accès en mémoire

**Scénarios alternatifs :**
- **3a.** Le token de rafraîchissement est invalide ou expiré → le système retourne une erreur 401 → le frontend redirige vers la page de connexion → la session est considérée terminée

**Paramètres de durée de vie (configurables) :**
| Token | Durée de vie par défaut |
|---|---|
| Token d'accès (JWT) | 15 minutes |
| Token de rafraîchissement | 7 jours |
| Token temporaire TOTP | 5 minutes |

---

### UC-AUTH-04 — Activation du TOTP

**Acteur :** Tout utilisateur connecté  
**Préconditions :** L'utilisateur est authentifié. Le TOTP est désactivé sur son compte  
**Postconditions :** Le TOTP est activé. L'application TOTP de l'utilisateur est configurée

**Scénario principal :**
1. L'utilisateur accède à la section "Sécurité" de ses paramètres de compte
2. L'utilisateur voit le bouton bascule TOTP en position "Désactivé"
3. L'utilisateur clique sur le bouton bascule pour activer
4. Le système demande confirmation du mot de passe courant de l'utilisateur (vérification d'identité)
5. Le système génère un secret TOTP unique pour cet utilisateur (algorithme TOTP standard RFC 6238)
6. Le système génère un QR Code encodant l'URI TOTP (otpauth://) contenant : l'émetteur (nom de l'application), l'email de l'utilisateur, le secret
7. Le système affiche le QR Code à l'utilisateur accompagné du secret en clair (pour saisie manuelle en cas d'impossibilité de scanner)
8. L'utilisateur scanne le QR Code avec son application d'authentification (Google Authenticator, Authy, etc.)
9. L'utilisateur saisit le code à 6 chiffres généré par son application pour confirmer la bonne configuration
10. Le système vérifie le code saisi
11. Le système active le TOTP sur le compte et stocke le secret chiffré
12. Le système génère et affiche les codes de récupération d'urgence (10 codes à usage unique)
13. Le système invite fortement l'utilisateur à sauvegarder ces codes de récupération
14. Le bouton bascule passe en position "Activé"

**Scénarios alternatifs :**
- **10a.** Le code est incorrect → afficher un message d'erreur → l'utilisateur peut réessayer → le secret n'est pas encore sauvegardé
- **4a.** Le mot de passe saisi est incorrect → l'activation est interrompue

**Règles métier :**
- Le secret TOTP est stocké chiffré en base de données (Fernet)
- Les codes de récupération sont hachés (Argon2) avant stockage — ils ne peuvent pas être récupérés une deuxième fois
- L'activation du TOTP est journalisée dans l'audit

---

### UC-AUTH-05 — Désactivation du TOTP

**Acteur :** Tout utilisateur connecté  
**Préconditions :** L'utilisateur est authentifié. Le TOTP est activé sur son compte  
**Postconditions :** Le TOTP est désactivé. Le secret TOTP est supprimé

**Scénario principal :**
1. L'utilisateur accède à la section "Sécurité" de ses paramètres de compte
2. L'utilisateur clique sur le bouton bascule TOTP (actuellement en position "Activé")
3. Le système affiche une boîte de dialogue de confirmation soulignant la réduction du niveau de sécurité
4. Le système demande le mot de passe courant ET un code TOTP valide pour confirmer
5. Le système vérifie le mot de passe et le code TOTP
6. Le système supprime le secret TOTP et tous les codes de récupération restants
7. Le bouton bascule passe en position "Désactivé"
8. La désactivation est journalisée

**Règles métier :**
- Un Super Admin peut désactiver le TOTP d'un utilisateur sans son code TOTP (uniquement avec son propre mot de passe super admin) — cas de perte d'accès à l'application d'authentification

---

### UC-AUTH-06 — Vérification TOTP lors de la connexion

**Acteur :** Utilisateur ayant réussi l'étape mot de passe avec TOTP activé  
**Préconditions :** L'utilisateur a reçu un token temporaire TOTP (étape 6 de UC-AUTH-01)  
**Postconditions :** L'utilisateur est pleinement authentifié et reçoit ses tokens définitifs

**Scénario principal :**
1. L'utilisateur est redirigé vers l'écran de vérification TOTP
2. L'utilisateur saisit le code à 6 chiffres affiché sur son application d'authentification
3. Le système vérifie le token temporaire TOTP (validité 5 minutes)
4. Le système vérifie le code TOTP saisi (avec tolérance d'un intervalle de 30 secondes de chaque côté pour compenser les décalages d'horloge)
5. Le système s'assure que ce code n'a pas déjà été utilisé (protection anti-rejeu)
6. Le système émet les tokens définitifs (accès + rafraîchissement)
7. Le token temporaire TOTP est invalidé

**Scénarios alternatifs :**
- **3a.** Le token temporaire est expiré → rediriger vers la page de connexion
- **4a.** Le code TOTP est incorrect → incrémenter le compteur d'échecs TOTP → après 3 échecs consécutifs, invalider le token temporaire et renvoyer vers la connexion
- **Utilisation d'un code de récupération :** l'utilisateur peut saisir un de ses codes de récupération à usage unique à la place du code TOTP → le code utilisé est immédiatement marqué comme consommé

---

### UC-AUTH-07 — Réinitialisation du mot de passe (oubli)

**Acteur :** Tout utilisateur  
**Préconditions :** L'utilisateur possède un compte avec une adresse email valide  
**Postconditions :** Le mot de passe de l'utilisateur est réinitialisé

**Scénario principal :**
1. L'utilisateur clique sur "Mot de passe oublié" sur la page de connexion
2. L'utilisateur saisit son adresse email
3. Le système vérifie si l'email correspond à un compte (sans révéler si le compte existe ou non dans le message de confirmation)
4. Si le compte existe et est actif : le système génère un token de réinitialisation unique (UUID v4), le stocke haché en base avec une expiration de 1 heure, et envoie un email contenant le lien de réinitialisation
5. Le système affiche toujours le même message de confirmation neutre ("Si un compte existe avec cet email, vous recevrez les instructions")
6. L'utilisateur clique sur le lien dans l'email
7. Le système valide le token (existence, non-utilisation, non-expiration)
8. L'utilisateur saisit un nouveau mot de passe et le confirme
9. Le système valide la conformité du nouveau mot de passe aux règles de sécurité
10. Le système hache le nouveau mot de passe et le sauvegarde
11. Le système révoque tous les tokens de rafraîchissement actifs du compte (déconnexion de toutes les sessions)
12. Le token de réinitialisation est marqué comme utilisé
13. L'utilisateur est redirigé vers la page de connexion

**Règles de sécurité du mot de passe :**
| Règle | Valeur |
|---|---|
| Longueur minimale | 12 caractères |
| Doit contenir | Au moins 1 majuscule, 1 minuscule, 1 chiffre, 1 caractère spécial |
| Interdit | Les 5 derniers mots de passe utilisés |
| Interdit | Les mots de passe courants (liste de mots de passe compromis) |

---

### UC-AUTH-09 — Déverrouillage d'un compte bloqué

**Acteur :** Super Admin ou Admin  
**Préconditions :** Un compte utilisateur est dans l'état "Bloqué"  
**Postconditions :** Le compte est débloqué et le compteur de tentatives remis à zéro

**Scénario principal :**
1. L'administrateur consulte la liste des utilisateurs filtrée sur l'état "Bloqué"
2. L'administrateur sélectionne le compte à débloquer
3. Le système affiche le détail du blocage (date, nombre de tentatives, adresses IP des tentatives)
4. L'administrateur confirme le déverrouillage
5. Le système remet le compteur de tentatives à zéro et passe l'état du compte à "Actif"
6. L'action est journalisée avec l'identité de l'administrateur

---

### UC-USER-01 — Créer un utilisateur

**Acteur :** Super Admin ou Admin  
**Préconditions :** L'acteur est authentifié et possède la permission de création d'utilisateurs  
**Postconditions :** Un nouveau compte utilisateur est créé en état "Actif"

**Scénario principal :**
1. L'acteur accède au module de gestion des utilisateurs
2. L'acteur remplit le formulaire de création : nom, prénom, email, rôle, service/département
3. Le système vérifie que l'email n'est pas déjà utilisé
4. Le système génère un mot de passe temporaire aléatoire conforme aux règles de sécurité
5. Le système crée le compte avec l'état "Actif" et le flag "Changement de mot de passe requis à la première connexion"
6. Le système envoie un email de bienvenue à l'utilisateur contenant son mot de passe temporaire
7. La création est journalisée

**Règles métier :**
- Un Admin ne peut pas créer un compte avec le rôle Super Admin
- Un Admin ne peut pas créer un compte avec un rôle supérieur au sien
- Le mot de passe temporaire expire après 48 heures — si non utilisé, l'administrateur doit en générer un nouveau

---

### UC-ROLE-03 — Assigner un rôle à un utilisateur

**Acteur :** Super Admin ou Admin  
**Préconditions :** L'acteur est authentifié. L'utilisateur cible et le rôle existent  
**Postconditions :** Le rôle est assigné à l'utilisateur

**Scénario principal :**
1. L'acteur accède au profil de l'utilisateur cible
2. L'acteur sélectionne le nouveau rôle dans la liste des rôles disponibles
3. L'acteur peut optionnellement définir une portée (scope) : le rôle peut être limité à un service ou département spécifique
4. Le système vérifie que l'acteur a le droit d'assigner ce rôle
5. Le système assigne le rôle (remplacement de l'ancien rôle)
6. Si l'utilisateur est actuellement connecté, ses permissions sont mises à jour à la prochaine vérification de token
7. L'assignation est journalisée

---

## 4. Modèles de données

### 4.1 Modèle `User` (Utilisateur)

**Table :** `users`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `email` | VARCHAR(255) | UNIQUE, NOT NULL, INDEX | Adresse email (identifiant de connexion) |
| `password_hash` | VARCHAR(255) | NOT NULL | Mot de passe haché (Argon2) |
| `first_name` | VARCHAR(100) | NOT NULL | Prénom |
| `last_name` | VARCHAR(100) | NOT NULL | Nom de famille |
| `phone_number` | VARCHAR(20) | NULL | Numéro de téléphone |
| `avatar` | VARCHAR(500) | NULL | Chemin vers l'avatar (fichier) |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT 'active' | Statut : `active`, `inactive`, `locked`, `pending` |
| `role_id` | UUID | FK → roles.id, NOT NULL | Rôle assigné |
| `department_id` | UUID | FK → departments.id, NULL | Service/département d'appartenance |
| `must_change_password` | BOOLEAN | NOT NULL, DEFAULT FALSE | Forcer le changement de mot de passe |
| `password_changed_at` | TIMESTAMP | NULL | Date du dernier changement de mot de passe |
| `last_login_at` | TIMESTAMP | NULL | Date de dernière connexion réussie |
| `failed_login_count` | SMALLINT | NOT NULL, DEFAULT 0 | Compteur de tentatives échouées |
| `locked_at` | TIMESTAMP | NULL | Date de blocage du compte |
| `lock_reason` | VARCHAR(255) | NULL | Raison du blocage |
| `totp_enabled` | BOOLEAN | NOT NULL, DEFAULT FALSE | TOTP activé sur ce compte |
| `totp_secret_encrypted` | TEXT | NULL | Secret TOTP chiffré (Fernet) |
| `totp_activated_at` | TIMESTAMP | NULL | Date d'activation du TOTP |
| `email_verified` | BOOLEAN | NOT NULL, DEFAULT FALSE | Email vérifié |
| `email_verified_at` | TIMESTAMP | NULL | Date de vérification de l'email |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | Suppression logique |
| `deleted_at` | TIMESTAMP | NULL | Date de suppression logique |
| `deleted_by_id` | UUID | FK → users.id, NULL | Qui a supprimé ce compte |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `updated_at` | TIMESTAMP | NOT NULL, AUTO | Date de mise à jour |
| `created_by_id` | UUID | FK → users.id, NULL | Qui a créé ce compte |

**Contraintes supplémentaires :**
- `email` : format email valide, lowercase forcé avant stockage
- `status` : valeur dans l'ensemble `{active, inactive, locked, pending}`
- `failed_login_count` : entre 0 et 99
- Un utilisateur `is_deleted = True` ne peut pas se connecter

**Index :**
- `idx_users_email` sur `email`
- `idx_users_status` sur `status`
- `idx_users_role_id` sur `role_id`
- `idx_users_department_id` sur `department_id`

---

### 4.2 Modèle `Role` (Rôle)

**Table :** `roles`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `name` | VARCHAR(100) | UNIQUE, NOT NULL | Nom du rôle |
| `code` | VARCHAR(50) | UNIQUE, NOT NULL | Code technique du rôle (ex: `super_admin`) |
| `description` | TEXT | NULL | Description du rôle |
| `is_system` | BOOLEAN | NOT NULL, DEFAULT FALSE | Rôle système (non modifiable ni supprimable) |
| `level` | SMALLINT | NOT NULL | Niveau hiérarchique (plus grand = plus de droits) |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `updated_at` | TIMESTAMP | NOT NULL, AUTO | Date de mise à jour |
| `created_by_id` | UUID | FK → users.id, NULL | Créateur |

**Rôles système prédéfinis (is_system = True) :**

| Code | Nom | Niveau | Description |
|---|---|---|---|
| `super_admin` | Super Administrateur | 100 | Accès total, gestion des rôles, suppression définitive |
| `admin` | Administrateur | 80 | Gestion des utilisateurs, configuration système |
| `archiviste` | Archiviste | 60 | Gestion complète des documents et archives |
| `responsable` | Responsable | 50 | Validation documentaire, accès étendu à son service |
| `agent` | Agent | 30 | Consultation et dépôt de documents selon les permissions |
| `auditeur` | Auditeur | 20 | Consultation des journaux d'audit, accès lecture seule |

**Règles métier des rôles :**
- Les rôles système (`is_system = True`) ne peuvent pas être modifiés ni supprimés
- Un utilisateur ne peut assigner qu'un rôle de niveau strictement inférieur au sien
- Un rôle personnalisé créé par le Super Admin peut avoir un niveau entre 1 et 79

---

### 4.3 Modèle `Permission` (Permission)

**Table :** `permissions`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `code` | VARCHAR(100) | UNIQUE, NOT NULL | Code unique de la permission (ex: `documents.create`) |
| `name` | VARCHAR(150) | NOT NULL | Nom lisible de la permission |
| `description` | TEXT | NULL | Description détaillée |
| `resource` | VARCHAR(50) | NOT NULL, INDEX | Ressource concernée (ex: `documents`, `users`) |
| `action` | VARCHAR(50) | NOT NULL | Action (ex: `create`, `read`, `update`, `delete`, `export`) |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |

**Index :**
- `idx_permissions_resource` sur `resource`
- `idx_permissions_resource_action` sur `(resource, action)` — UNIQUE

---

### 4.4 Modèle `RolePermission` (Liaison Rôle ↔ Permission)

**Table :** `role_permissions`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `role_id` | UUID | FK → roles.id, NOT NULL | Rôle |
| `permission_id` | UUID | FK → permissions.id, NOT NULL | Permission |
| `granted_at` | TIMESTAMP | NOT NULL, AUTO | Date d'attribution |
| `granted_by_id` | UUID | FK → users.id, NULL | Qui a accordé cette permission |

**Contrainte :** `UNIQUE (role_id, permission_id)`

---

### 4.5 Modèle `RefreshToken` (Token de rafraîchissement)

**Table :** `refresh_tokens`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `user_id` | UUID | FK → users.id, NOT NULL, INDEX | Utilisateur propriétaire |
| `token_hash` | VARCHAR(255) | UNIQUE, NOT NULL | Hash SHA-256 du token |
| `device_info` | VARCHAR(500) | NULL | Info sur le client/appareil |
| `ip_address` | INET | NULL | Adresse IP d'origine |
| `user_agent` | TEXT | NULL | User-Agent HTTP |
| `is_revoked` | BOOLEAN | NOT NULL, DEFAULT FALSE | Token révoqué |
| `revoked_at` | TIMESTAMP | NULL | Date de révocation |
| `expires_at` | TIMESTAMP | NOT NULL | Date d'expiration |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |

**Index :**
- `idx_refresh_tokens_user_id` sur `user_id`
- `idx_refresh_tokens_token_hash` sur `token_hash`
- `idx_refresh_tokens_expires_at` sur `expires_at` (pour le nettoyage automatique)

**Règle :** Une tâche Celery planifiée supprime physiquement les tokens expirés depuis plus de 30 jours (seul cas de suppression physique autorisée dans ce module).

---

### 4.6 Modèle `TotpRecoveryCode` (Codes de récupération TOTP)

**Table :** `totp_recovery_codes`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `user_id` | UUID | FK → users.id, NOT NULL, INDEX | Utilisateur propriétaire |
| `code_hash` | VARCHAR(255) | NOT NULL | Hash Argon2 du code |
| `is_used` | BOOLEAN | NOT NULL, DEFAULT FALSE | Code déjà utilisé |
| `used_at` | TIMESTAMP | NULL | Date d'utilisation |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |

**Règle :** 10 codes générés à l'activation du TOTP. Chaque code est utilisable une seule fois. Quand tous les codes sont utilisés, l'utilisateur doit en générer de nouveaux (action disponible dans les paramètres de compte).

---

### 4.7 Modèle `PasswordResetToken` (Token de réinitialisation de mot de passe)

**Table :** `password_reset_tokens`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `user_id` | UUID | FK → users.id, NOT NULL, INDEX | Utilisateur concerné |
| `token_hash` | VARCHAR(255) | UNIQUE, NOT NULL | Hash SHA-256 du token |
| `is_used` | BOOLEAN | NOT NULL, DEFAULT FALSE | Token déjà utilisé |
| `used_at` | TIMESTAMP | NULL | Date d'utilisation |
| `expires_at` | TIMESTAMP | NOT NULL | Expiration (1 heure après création) |
| `ip_address` | INET | NULL | Adresse IP de la demande |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |

---

### 4.8 Modèle `PasswordHistory` (Historique des mots de passe)

**Table :** `password_history`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `user_id` | UUID | FK → users.id, NOT NULL, INDEX | Utilisateur |
| `password_hash` | VARCHAR(255) | NOT NULL | Hash du mot de passe historisé |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de l'enregistrement |

**Règle :** Les 5 derniers mots de passe de chaque utilisateur sont conservés. Un nouveau mot de passe ne peut pas être identique à l'un des 5 précédents.

---

### 4.9 Modèle `Department` (Service / Département)

**Table :** `departments`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `name` | VARCHAR(150) | UNIQUE, NOT NULL | Nom du service |
| `code` | VARCHAR(30) | UNIQUE, NOT NULL | Code court du service |
| `description` | TEXT | NULL | Description |
| `parent_id` | UUID | FK → departments.id, NULL | Service parent (hiérarchie) |
| `manager_id` | UUID | FK → users.id, NULL | Responsable du service |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Service actif |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | Suppression logique |
| `deleted_at` | TIMESTAMP | NULL | Date de suppression |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `updated_at` | TIMESTAMP | NOT NULL, AUTO | Date de mise à jour |

---

### 4.10 Modèle `UserSession` (Session utilisateur)

**Table :** `user_sessions`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `user_id` | UUID | FK → users.id, NOT NULL, INDEX | Utilisateur |
| `refresh_token_id` | UUID | FK → refresh_tokens.id, NOT NULL | Token associé |
| `device_type` | VARCHAR(20) | NULL | Type : `web`, `desktop`, `mobile` |
| `device_name` | VARCHAR(255) | NULL | Nom de l'appareil (ex: "Chrome sur Windows") |
| `ip_address` | INET | NULL | Adresse IP |
| `location` | VARCHAR(255) | NULL | Localisation approximative |
| `last_activity_at` | TIMESTAMP | NOT NULL | Dernière activité |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de début de session |

---

## 5. Matrice des permissions

### 5.1 Ressources et actions couvertes par ce module

| Permission code | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `users.create` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `users.read_list` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `users.read_detail` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `users.update` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `users.deactivate` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `users.delete` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `users.unlock` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `users.self_update` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `users.self_read` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `roles.create` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `roles.read` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `roles.update_permissions` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `roles.assign` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `totp.manage_self` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `totp.manage_others` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `sessions.read_self` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `sessions.revoke_self` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `sessions.read_all` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `sessions.revoke_any` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `departments.create` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `departments.update` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `departments.delete` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `departments.read` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

---

## 6. Séquences détaillées des flux principaux

### 6.1 Séquence — Connexion complète avec TOTP

```
Utilisateur          Frontend React         API Django            Base de données       Redis
    |                     |                     |                       |                 |
    |-- saisit email ----->|                     |                       |                 |
    |   et mot de passe    |                     |                       |                 |
    |                     |-- POST /auth/login ->|                       |                 |
    |                     |                     |-- SELECT user WHERE -->|                 |
    |                     |                     |   email = ?            |                 |
    |                     |                     |<-- user record --------|                 |
    |                     |                     |-- vérifier status      |                 |
    |                     |                     |-- vérifier Argon2      |                 |
    |                     |                     |-- totp_enabled = True  |                 |
    |                     |                     |-- générer temp_token -->|                |
    |                     |                     |-- STORE temp_token ---------------------------->|
    |                     |<-- 200 + temp_token--|   (TTL 5 min)          |                 |
    |<-- écran TOTP -------|                     |                       |                 |
    |                     |                     |                       |                 |
    |-- saisit code TOTP ->|                     |                       |                 |
    |                     |-- POST /auth/totp -->|                       |                 |
    |                     |   verify             |-- GET temp_token -------------------------------->|
    |                     |                     |<-- token valide -------------------------------------|
    |                     |                     |-- vérifier code TOTP   |                 |
    |                     |                     |-- anti-rejeu check      |                 |
    |                     |                     |-- générer access_token  |                 |
    |                     |                     |-- générer refresh_token |                 |
    |                     |                     |-- INSERT refresh_token->|                 |
    |                     |                     |-- INSERT user_session ->|                 |
    |                     |                     |-- DEL temp_token ---------------------------------->|
    |                     |                     |-- UPDATE last_login --->|                 |
    |                     |<-- 200 + tokens -----|                       |                 |
    |<-- accès dashboard --|                     |                       |                 |
```

### 6.2 Séquence — Rafraîchissement automatique du token

```
Frontend React         Intercepteur Axios      API Django            Redis (Blacklist)
    |                       |                     |                       |
    |-- requête quelconque ->|                     |                       |
    |                       |-- vérifier expiry   |                       |
    |                       |   token < 2 min     |                       |
    |                       |-- POST /auth/token/ >|                      |
    |                       |   refresh           |-- vérifier blacklist ------------->|
    |                       |                     |<-- not blacklisted ----------------|
    |                       |                     |-- vérifier signature  |            |
    |                       |                     |-- vérifier expiry     |            |
    |                       |                     |-- générer nouveau     |            |
    |                       |                     |   access_token        |            |
    |                       |<-- 200 new token ----|                      |            |
    |                       |-- mettre à jour     |                       |            |
    |                       |   token en mémoire  |                       |            |
    |                       |-- reprendre requête  >|                     |            |
    |<-- réponse originale --|                     |                       |            |
```

### 6.3 Séquence — Activation du TOTP

```
Utilisateur          Frontend React         API Django            Base de données
    |                     |                     |                       |
    |-- ouvre paramètres ->|                     |                       |
    |<-- toggle TOTP OFF --|                     |                       |
    |                     |                     |                       |
    |-- clique toggle ---->|                     |                       |
    |<-- modal confirm mot de passe             |                       |
    |-- saisit mot passe ->|                     |                       |
    |                     |-- POST /auth/totp/ ->|                       |
    |                     |   setup              |-- vérifier mdp ------->|
    |                     |                     |<-- OK ------------------|
    |                     |                     |-- générer secret TOTP   |
    |                     |                     |-- générer QR Code URI   |
    |                     |<-- 200 + QR code ----|                       |
    |<-- affiche QR Code --|                     |                       |
    |                     |                     |                       |
    |-- scanne QR Code     |                     |                       |
    |-- saisit code ------->|                    |                       |
    |                     |-- POST /auth/totp/ ->|                       |
    |                     |   confirm            |-- vérifier code TOTP   |
    |                     |                     |-- chiffrer secret       |
    |                     |                     |-- INSERT secret chiffré->|
    |                     |                     |-- générer 10 codes récup.|
    |                     |                     |-- INSERT codes hachés -->|
    |                     |                     |-- UPDATE totp_enabled -->|
    |                     |<-- 200 + codes récup |                       |
    |<-- affiche codes ----|                     |                       |
    |   à sauvegarder      |                     |                       |
```

---

## 7. Endpoints API du module

### 7.1 Authentification

| Méthode | URL | Description | Auth requise |
|---|---|---|---|
| POST | `/api/v1/auth/login/` | Connexion (email + mot de passe) | Non |
| POST | `/api/v1/auth/totp/verify/` | Vérification code TOTP | Token temporaire |
| POST | `/api/v1/auth/logout/` | Déconnexion | Oui |
| POST | `/api/v1/auth/token/refresh/` | Rafraîchissement du token | Non (refresh token) |
| POST | `/api/v1/auth/password/reset/request/` | Demande réinitialisation mdp | Non |
| POST | `/api/v1/auth/password/reset/confirm/` | Confirmation réinitialisation | Non (reset token) |
| POST | `/api/v1/auth/password/change/` | Changement mdp (connecté) | Oui |

### 7.2 TOTP

| Méthode | URL | Description | Auth requise |
|---|---|---|---|
| POST | `/api/v1/auth/totp/setup/` | Initier l'activation TOTP (retourne QR Code) | Oui |
| POST | `/api/v1/auth/totp/confirm/` | Confirmer l'activation TOTP | Oui |
| POST | `/api/v1/auth/totp/disable/` | Désactiver le TOTP | Oui |
| POST | `/api/v1/auth/totp/recovery-codes/regenerate/` | Régénérer les codes de récupération | Oui + TOTP |

### 7.3 Utilisateurs

| Méthode | URL | Description | Auth requise |
|---|---|---|---|
| GET | `/api/v1/users/` | Liste des utilisateurs | Oui + Permission |
| POST | `/api/v1/users/` | Créer un utilisateur | Oui + Permission |
| GET | `/api/v1/users/{id}/` | Détail d'un utilisateur | Oui + Permission |
| PATCH | `/api/v1/users/{id}/` | Modifier un utilisateur | Oui + Permission |
| DELETE | `/api/v1/users/{id}/` | Suppression logique | Oui + Super Admin |
| POST | `/api/v1/users/{id}/activate/` | Activer un compte | Oui + Permission |
| POST | `/api/v1/users/{id}/deactivate/` | Désactiver un compte | Oui + Permission |
| POST | `/api/v1/users/{id}/unlock/` | Débloquer un compte | Oui + Permission |
| GET | `/api/v1/users/me/` | Profil propre | Oui |
| PATCH | `/api/v1/users/me/` | Modifier son propre profil | Oui |

### 7.4 Sessions

| Méthode | URL | Description | Auth requise |
|---|---|---|---|
| GET | `/api/v1/auth/sessions/` | Mes sessions actives | Oui |
| DELETE | `/api/v1/auth/sessions/{id}/` | Révoquer une session | Oui |
| DELETE | `/api/v1/auth/sessions/` | Révoquer toutes mes sessions | Oui |

### 7.5 Rôles et permissions

| Méthode | URL | Description | Auth requise |
|---|---|---|---|
| GET | `/api/v1/roles/` | Liste des rôles | Oui + Permission |
| POST | `/api/v1/roles/` | Créer un rôle | Oui + Super Admin |
| GET | `/api/v1/roles/{id}/` | Détail d'un rôle | Oui + Permission |
| PATCH | `/api/v1/roles/{id}/permissions/` | Modifier les permissions | Oui + Super Admin |
| GET | `/api/v1/permissions/` | Liste de toutes les permissions | Oui + Permission |
| GET | `/api/v1/departments/` | Liste des départements | Oui |
| POST | `/api/v1/departments/` | Créer un département | Oui + Permission |
| PATCH | `/api/v1/departments/{id}/` | Modifier un département | Oui + Permission |

---

## 8. Règles de sécurité spécifiques au module

### 8.1 Protection contre les attaques

| Attaque | Contre-mesure |
|---|---|
| Brute force mot de passe | Blocage compte après 5 tentatives + rate limiting IP |
| Énumération de comptes | Message d'erreur identique quel que soit le motif |
| Rejeu de token TOTP | Chaque code TOTP ne peut être utilisé qu'une fois dans sa fenêtre |
| Vol de token JWT | Durée de vie courte (15 min) + blacklist sur déconnexion |
| CSRF | Token CSRF sur toutes les mutations (formulaires) |
| Injection dans les formulaires | Validation stricte côté backend via DRF serializers |
| Tokens de réinitialisation prédictibles | UUID v4 aléatoire, stocké haché |

### 8.2 Règles de gestion des tokens JWT

- Le token d'accès contient : `user_id`, `email`, `role_code`, `permissions_hash`, `exp`, `iat`, `jti`
- Le `permissions_hash` est un hash des permissions courantes — si les permissions changent, la vérification suivante détecte l'invalidation
- Le `jti` (JWT ID) est unique par token et permet la blacklist granulaire
- Les tokens ne contiennent **jamais** de données sensibles (mot de passe, secret TOTP, etc.)

### 8.3 Politique de session Tauri (Desktop)

- En mode desktop, les tokens sont stockés dans le store sécurisé de Tauri (chiffré par l'OS, pas dans le localStorage)
- La déconnexion en mode desktop efface le store sécurisé local
- Le mode desktop affiche une notification OS lors d'une connexion depuis un nouvel appareil

---

## 9. Événements journalisés (MODULE 11 — Audit)

Ce module génère les événements d'audit suivants, tous transmis au MODULE 11 :

| Événement | Niveau | Détails transmis |
|---|---|---|
| Connexion réussie | INFO | user_id, ip, device, timestamp |
| Connexion échouée | WARNING | email_tenté, ip, timestamp, reason |
| Compte bloqué | ALERT | user_id, ip, timestamp, attempts_count |
| Déconnexion | INFO | user_id, session_id, timestamp |
| TOTP activé | INFO | user_id, timestamp |
| TOTP désactivé | WARNING | user_id, timestamp, by_admin |
| Mot de passe changé | INFO | user_id, timestamp |
| Mot de passe réinitialisé | INFO | user_id, ip, timestamp |
| Utilisateur créé | INFO | user_id, created_by, role, timestamp |
| Utilisateur désactivé | WARNING | user_id, deactivated_by, timestamp |
| Utilisateur supprimé | ALERT | user_id, deleted_by, timestamp |
| Rôle modifié | WARNING | role_id, modified_by, changes, timestamp |
| Rôle assigné | INFO | user_id, role_id, assigned_by, timestamp |
| Accès refusé | WARNING | user_id, resource, action, timestamp |
| Session révoquée | INFO | user_id, session_id, revoked_by, timestamp |
| Compte débloqué | INFO | user_id, unlocked_by, timestamp |

---

## 10. Notifications générées (MODULE 12)

| Déclencheur | Destinataire | Canal | Message |
|---|---|---|---|
| Nouveau compte créé | Nouvel utilisateur | Email | Email de bienvenue avec mot de passe temporaire |
| Compte bloqué | Utilisateur + Admin | Email + In-app | Alerte de blocage de compte |
| Connexion depuis nouvel appareil | Utilisateur | Email + In-app | Alerte de sécurité |
| Mot de passe réinitialisé | Utilisateur | Email | Confirmation de réinitialisation |
| TOTP désactivé | Utilisateur | Email | Alerte de réduction du niveau de sécurité |
| Mot de passe temporaire expiré | Admin | In-app | Le mot de passe de [utilisateur] a expiré |
| Codes de récupération épuisés | Utilisateur | In-app | Alerte pour régénérer les codes |

---

*Fin du MODULE 02 — Authentification, Utilisateurs, Rôles & Permissions*  
*Prochain module : MODULE 03 — Taxonomie documentaire*
