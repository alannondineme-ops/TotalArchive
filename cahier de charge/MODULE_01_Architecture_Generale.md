# MODULE 01 — Architecture Générale & Conventions du Projet

## Projet : Système d'Archivage Numérique des Documents
**Version :** 1.0  
**Statut :** Référence fondatrice — tous les modules suivants s'appuient sur ce document  
**Dépendances :** Aucune

---

## 1. Présentation du projet

Le projet consiste en la conception et le développement d'un système d'archivage numérique des documents destiné à une administration publique. La solution est une application web convertie en application desktop via Tauri, permettant une gestion documentaire complète, sécurisée, traçable et intelligente.

L'application doit fonctionner aussi bien en environnement connecté (web) qu'en environnement local (desktop), avec synchronisation différée en cas de mode hors-ligne partiel.

---

## 2. Stack technique

### 2.1 Backend
| Composant | Technologie | Rôle |
|---|---|---|
| Langage | Python | Langage principal du backend |
| Framework web | Django | Framework principal |
| API REST | Django REST Framework (DRF) | Exposition des endpoints API |
| Authentification | JWT (SimpleJWT) | Gestion des tokens d'accès et de rafraîchissement |
| Tâches asynchrones | Celery | Traitement OCR, notifications, sauvegardes planifiées |
| Broker de messages | Redis | Broker pour Celery |
| Base de données dev | SQLite | Développement local uniquement |
| Base de données prod | PostgreSQL | Environnement de production |
| Recherche full-text | PostgreSQL FTS + django-watson | Indexation et recherche documentaire |
| OCR | Tesseract + pytesseract | Extraction du contenu textuel des documents scannés |
| Traitement image | Pillow | Prétraitement des images avant OCR |
| Sécurité | django-axes | Verrouillage de compte après tentatives échouées |
| TOTP | django-otp + qrcode | Authentification à deux facteurs TOTP avec QR Code |
| Variables d'environnement | python-decouple | Gestion des configurations par environnement |
| Documentation API | drf-spectacular | Génération automatique OpenAPI/Swagger |
| Signature numérique | PyHanko | Signature numérique de documents PDF (option) |
| Chiffrement | cryptography (Fernet) | Chiffrement des données sensibles au repos |
| NLP / IA | spaCy + scikit-learn | Extraction d'entités, classification automatique |

### 2.2 Frontend
| Composant | Technologie | Rôle |
|---|---|---|
| Langage | JavaScript / TypeScript | Langage principal du frontend |
| Framework UI | React | Bibliothèque d'interface utilisateur |
| Style | Tailwind CSS | Framework CSS utilitaire |
| Gestionnaire de paquets | pnpm | Gestionnaire de dépendances (obligatoire) |
| État global | Zustand | Gestion de l'état applicatif |
| Requêtes API | TanStack Query (React Query) | Gestion du cache et des requêtes serveur |
| Client HTTP | Axios | Communication avec l'API Django |
| Formulaires | React Hook Form + Zod | Gestion et validation des formulaires |
| Graphiques | Recharts | Visualisation des données dans les dashboards |
| Notifications UI | Sonner | Toasts et notifications in-app |
| QR Code | qrcode.react | Affichage du QR Code pour le TOTP |
| Tables | TanStack Table | Tableaux de données avancés |
| Éditeur de métadonnées | composants custom React | Interface de saisie des métadonnées documentaires |
| Routeur | React Router v6 | Navigation entre les pages |

### 2.3 Desktop
| Composant | Technologie | Rôle |
|---|---|---|
| Wrapper desktop | Tauri (Rust) | Conversion de l'application web en application desktop |
| Accès système | API Tauri | Accès natif au scanner, système de fichiers, notifications OS |
| Stockage local | SQLite via Tauri | Cache local en mode offline |
| Mise à jour auto | Tauri Updater | Mise à jour automatique de l'application desktop |

---

## 3. Structure des dossiers du projet

```
/ (racine du projet)
│
├── backend/                        # Application Django
│   ├── venv/                       # Environnement virtuel Python (nom obligatoire : venv)
│   ├── config/                     # Configuration principale Django
│   │   ├── settings/
│   │   │   ├── base.py             # Paramètres communs
│   │   │   ├── development.py      # Paramètres de développement (SQLite)
│   │   │   └── production.py       # Paramètres de production (PostgreSQL)
│   │   ├── urls.py                 # Routage principal
│   │   ├── wsgi.py
│   │   └── asgi.py
│   ├── apps/                       # Applications Django (une par module métier)
│   │   ├── authentication/         # MODULE 02
│   │   ├── taxonomy/               # MODULE 03
│   │   ├── lifecycle/              # MODULE 04
│   │   ├── confidentiality/        # MODULE 05
│   │   ├── documents/              # MODULE 06
│   │   ├── ocr/                    # MODULE 07
│   │   ├── workflow/               # MODULE 08
│   │   ├── search/                 # MODULE 09
│   │   ├── intelligence/           # MODULE 10
│   │   ├── audit/                  # MODULE 11
│   │   ├── notifications/          # MODULE 12
│   │   ├── access_requests/        # MODULE 13
│   │   ├── dashboard/              # MODULE 14
│   │   ├── administration/         # MODULE 15
│   │   ├── backup/                 # MODULE 16
│   │   └── integrations/           # MODULE 18
│   ├── core/                       # Utilitaires partagés entre toutes les apps
│   │   ├── models.py               # Modèle de base abstrait (timestamps, uuid)
│   │   ├── permissions.py          # Permissions personnalisées réutilisables
│   │   ├── pagination.py           # Pagination standard de l'API
│   │   ├── exceptions.py           # Exceptions personnalisées
│   │   └── utils.py                # Fonctions utilitaires globales
│   ├── media/                      # Fichiers uploadés (documents scannés, etc.)
│   ├── static/                     # Fichiers statiques Django
│   ├── logs/                       # Journaux d'application
│   ├── backups/                    # Sauvegardes locales
│   ├── manage.py
│   └── .env                        # Variables d'environnement (jamais versionné)
│
├── frontend/                       # Application React
│   ├── src/
│   │   ├── assets/                 # Images, icônes, polices
│   │   ├── components/             # Composants réutilisables
│   │   │   ├── ui/                 # Composants de base (Button, Input, Modal...)
│   │   │   └── shared/             # Composants métier partagés
│   │   ├── pages/                  # Pages de l'application (une par fonctionnalité majeure)
│   │   ├── features/               # Logique métier par fonctionnalité
│   │   │   ├── auth/
│   │   │   ├── documents/
│   │   │   ├── workflow/
│   │   │   └── ...
│   │   ├── hooks/                  # Hooks React personnalisés
│   │   ├── stores/                 # Stores Zustand
│   │   ├── services/               # Couche d'appel API (Axios)
│   │   ├── routes/                 # Configuration du routeur
│   │   ├── utils/                  # Fonctions utilitaires frontend
│   │   ├── types/                  # Types TypeScript globaux
│   │   ├── constants/              # Constantes de l'application
│   │   └── App.tsx
│   ├── public/
│   ├── package.json
│   ├── pnpm-lock.yaml
│   ├── tailwind.config.js
│   ├── tsconfig.json
│   └── vite.config.ts
│
├── src-tauri/                      # Configuration Tauri (desktop)
│   ├── src/
│   │   └── main.rs                 # Point d'entrée Rust de Tauri
│   ├── icons/                      # Icônes de l'application desktop
│   ├── tauri.conf.json             # Configuration Tauri
│   └── Cargo.toml                  # Dépendances Rust
│
├── docs/                           # Documentation du projet (UNIQUEMENT les fichiers *.md ici)
│   ├── MODULE_01_Architecture_Generale.md
│   ├── MODULE_02_Authentification.md
│   ├── MODULE_03_Taxonomie.md
│   ├── MODULE_04_Cycle_de_vie.md
│   ├── MODULE_05_Confidentialite.md
│   ├── MODULE_06_Documents.md
│   ├── MODULE_07_OCR_Numerisation.md
│   ├── MODULE_08_Workflow.md
│   ├── MODULE_09_Recherche.md
│   ├── MODULE_10_Intelligence_Artificielle.md
│   ├── MODULE_11_Audit_Tracabilite.md
│   ├── MODULE_12_Notifications.md
│   ├── MODULE_13_Acces_Partage.md
│   ├── MODULE_14_Dashboard_Reporting.md
│   ├── MODULE_15_Administration.md
│   ├── MODULE_16_Sauvegarde_Resilience.md
│   ├── MODULE_17_Desktop_Tauri.md
│   └── MODULE_18_API_Interoperabilite.md
│
└── requirements.txt                # Dépendances Python globales (racine du projet)
```

---

## 4. Architecture logicielle — Principes fondamentaux

### 4.1 Pattern architectural général
L'application suit le pattern **API-First** avec une séparation stricte des responsabilités :

- Le **backend Django** expose une API REST complète et ne sert aucune vue HTML
- Le **frontend React** consomme exclusivement l'API, sans logique métier propre
- **Tauri** enveloppe le frontend React et enrichit l'application de capacités natives (scanner, fichiers système, notifications OS)
- Cette séparation garantit que la version web et la version desktop partagent exactement le même backend et la même logique métier

### 4.2 Pattern des applications Django
Chaque application Django suit une architecture en couches internes :

```
apps/nom_application/
├── models.py          # Modèles de données (couche données)
├── serializers.py     # Sérialisation / désérialisation (couche transformation)
├── views.py           # ViewSets API (couche présentation API)
├── urls.py            # Routage de l'application
├── permissions.py     # Permissions spécifiques à l'application
├── services.py        # Logique métier (couche service — jamais dans les vues)
├── tasks.py           # Tâches Celery asynchrones
├── signals.py         # Signaux Django (événements inter-applications)
├── admin.py           # Interface d'administration Django
├── apps.py            # Configuration de l'application
└── tests/
    ├── test_models.py
    ├── test_views.py
    └── test_services.py
```

**Règle absolue :** toute logique métier réside dans `services.py`, jamais directement dans les vues ou les modèles.

### 4.3 Pattern des features React
Chaque feature frontend suit une organisation identique :

```
features/nom_feature/
├── components/        # Composants spécifiques à cette feature
├── hooks/             # Hooks spécifiques (appels API via React Query)
├── store.ts           # Store Zustand si état local nécessaire
├── types.ts           # Types TypeScript de la feature
└── index.ts           # Point d'exportation public de la feature
```

---

## 5. Conventions de nommage

### 5.1 Backend Python / Django
| Élément | Convention | Exemple |
|---|---|---|
| Fichiers | snake_case | `document_service.py` |
| Classes | PascalCase | `DocumentService` |
| Fonctions et méthodes | snake_case | `get_document_by_id()` |
| Variables | snake_case | `document_count` |
| Constantes | UPPER_SNAKE_CASE | `MAX_FILE_SIZE` |
| Modèles Django | PascalCase singulier | `Document`, `UserProfile` |
| Noms de tables DB | snake_case pluriel (auto Django) | `documents`, `user_profiles` |
| URLs API | kebab-case | `/api/v1/documents/`, `/api/v1/access-requests/` |
| Apps Django | snake_case | `access_requests`, `ocr` |

### 5.2 Frontend TypeScript / React
| Élément | Convention | Exemple |
|---|---|---|
| Fichiers composants | PascalCase | `DocumentCard.tsx` |
| Fichiers utilitaires | camelCase | `formatDate.ts` |
| Composants React | PascalCase | `DocumentCard` |
| Hooks personnalisés | camelCase préfixé `use` | `useDocumentList` |
| Variables et fonctions | camelCase | `documentCount` |
| Constantes | UPPER_SNAKE_CASE | `MAX_FILE_SIZE` |
| Types et interfaces | PascalCase préfixé `T` ou `I` | `TDocument`, `IUserRole` |
| Stores Zustand | camelCase suffixé `Store` | `documentStore` |
| CSS classes Tailwind | Utilisation directe des utilitaires Tailwind |  |

### 5.3 Base de données
| Élément | Convention |
|---|---|
| Noms de tables | snake_case pluriel |
| Noms de colonnes | snake_case |
| Clés primaires | `id` (UUID v4 — jamais d'auto-incrément entier) |
| Clés étrangères | `nom_modele_id` |
| Tables de jonction | `modele_a_modele_b` (alphabétique) |
| Index | `idx_nom_table_nom_colonne` |

**Règle importante :** toutes les clés primaires de l'application utilisent des **UUID v4**, jamais des entiers auto-incrémentés. Cette règle garantit la sécurité (impossibilité de deviner un ID), la portabilité et la cohérence en mode offline/sync Tauri.

---

## 6. Conventions de l'API REST

### 6.1 Versioning
Toutes les URLs de l'API sont préfixées par la version :
```
/api/v1/
```
Le versioning permet d'évoluer l'API sans casser les clients existants.

### 6.2 Structure des URLs
```
GET    /api/v1/documents/              → liste des documents
POST   /api/v1/documents/              → créer un document
GET    /api/v1/documents/{id}/         → détail d'un document
PUT    /api/v1/documents/{id}/         → mise à jour complète
PATCH  /api/v1/documents/{id}/         → mise à jour partielle
DELETE /api/v1/documents/{id}/         → suppression
GET    /api/v1/documents/{id}/versions/→ sous-ressource (versions du document)
```

### 6.3 Format de réponse standard
Toutes les réponses de l'API suivent une enveloppe JSON unifiée :

**Succès :**
```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "page": 1,
    "total": 150,
    "per_page": 20
  }
}
```

**Erreur :**
```json
{
  "success": false,
  "error": {
    "code": "DOCUMENT_NOT_FOUND",
    "message": "Le document demandé n'existe pas.",
    "details": {}
  }
}
```

### 6.4 Codes d'erreur métier
Chaque erreur possède un code applicatif en plus du code HTTP, permettant au frontend de gérer les cas d'erreur précisément sans interpréter les messages texte.

### 6.5 Pagination
Toutes les listes sont paginées. La pagination est basée sur les **curseurs** (cursor-based pagination) plutôt que sur les offsets, pour de meilleures performances sur les grands volumes de données.

---

## 7. Gestion des environnements

### 7.1 Variables d'environnement
Toutes les configurations sensibles ou dépendantes de l'environnement sont gérées via un fichier `.env` à la racine du dossier `backend/`. Ce fichier n'est **jamais versionné** dans le contrôle de source.

Un fichier `.env.example` est versionné et documente toutes les variables attendues sans leurs valeurs réelles.

### 7.2 Variables attendues (liste non exhaustive)
| Variable | Description |
|---|---|
| `SECRET_KEY` | Clé secrète Django |
| `DEBUG` | Mode debug (`True` en dev, `False` en prod) |
| `ALLOWED_HOSTS` | Hôtes autorisés |
| `DATABASE_URL` | URL de connexion à la base de données |
| `REDIS_URL` | URL de connexion Redis |
| `MEDIA_ROOT` | Chemin de stockage des fichiers |
| `JWT_SECRET_KEY` | Clé de signature des JWT |
| `JWT_ACCESS_TOKEN_LIFETIME` | Durée de vie du token d'accès (minutes) |
| `JWT_REFRESH_TOKEN_LIFETIME` | Durée de vie du token de rafraîchissement (jours) |
| `EMAIL_HOST` | Serveur SMTP |
| `EMAIL_PORT` | Port SMTP |
| `EMAIL_HOST_USER` | Utilisateur SMTP |
| `EMAIL_HOST_PASSWORD` | Mot de passe SMTP |
| `CELERY_BROKER_URL` | URL du broker Celery (Redis) |
| `ENCRYPTION_KEY` | Clé Fernet pour le chiffrement des données sensibles |
| `TOTP_ISSUER_NAME` | Nom affiché dans l'application d'authentification TOTP |
| `MAX_FILE_SIZE_MB` | Taille maximale d'un fichier uploadé (en Mo) |
| `TESSERACT_CMD` | Chemin vers l'exécutable Tesseract |

---

## 8. Gestion des migrations de base de données

### 8.1 Règles de migration
- Chaque modification de modèle génère une migration Django dédiée
- Les migrations sont versionnées dans le contrôle de source
- En production, les migrations sont appliquées manuellement et validées avant déploiement
- Toute migration irréversible (suppression de colonne ou de table) doit être documentée

### 8.2 Modèle de base abstrait
Tous les modèles de l'application héritent d'un modèle de base abstrait défini dans `core/models.py` qui fournit automatiquement :

| Champ | Type | Description |
|---|---|---|
| `id` | UUIDField (v4) | Clé primaire unique |
| `created_at` | DateTimeField | Date et heure de création (auto) |
| `updated_at` | DateTimeField | Date et heure de dernière modification (auto) |
| `is_deleted` | BooleanField | Suppression logique (soft delete) |
| `deleted_at` | DateTimeField | Date de suppression logique |

**Règle de suppression :** aucun enregistrement n'est jamais physiquement supprimé de la base de données. La suppression est toujours logique (`is_deleted = True`). La suppression physique est réservée aux procédures d'élimination documentaire officielles du MODULE 04.

---

## 9. Sécurité — Principes transversaux

Ces principes s'appliquent à l'ensemble de l'application et doivent être respectés dans chaque module.

### 9.1 Authentification et autorisation
- Tous les endpoints API sont protégés par JWT, sauf les endpoints publics explicitement désignés (`/api/v1/auth/login/`, `/api/v1/auth/token/refresh/`)
- Chaque action est soumise à une vérification de permission granulaire (rôle + ressource + action)
- Le principe du **moindre privilège** s'applique systématiquement

### 9.2 Sécurité des données
- Toutes les communications transitent en **HTTPS** en production
- Les données sensibles au repos sont chiffrées avec **Fernet (AES-128)**
- Chaque document possède un **hash SHA-256** calculé à l'upload, permettant de détecter toute altération ultérieure
- Les mots de passe sont hachés avec **Argon2** (algorithme de hachage recommandé)

### 9.3 Protection des API
- **Rate limiting** sur tous les endpoints publics (notamment l'authentification)
- **CORS** configuré strictement (origines autorisées explicitement listées)
- **CSRF** activé pour les formulaires web
- Validation stricte de toutes les entrées utilisateur côté backend (jamais de confiance côté client)
- Prévention des injections SQL via l'ORM Django exclusivement (requêtes brutes interdites sauf exception documentée)

### 9.4 Journalisation de sécurité
Tout événement de sécurité est enregistré dans le journal d'audit (MODULE 11) :
- Tentatives de connexion échouées
- Connexions réussies
- Accès à des documents confidentiels
- Modifications de permissions
- Exports de données

---

## 10. Gestion des fichiers et du stockage

### 10.1 Fichiers uploadés
- Les fichiers sont stockés dans le dossier `media/` du backend, organisés par année/mois/type
- Le nom de fichier original n'est jamais conservé sur le disque (renommage automatique en UUID à l'upload)
- Le nom original est stocké dans les métadonnées du document en base de données
- La taille maximale d'un fichier uploadé est configurable via la variable d'environnement `MAX_FILE_SIZE_MB`

### 10.2 Formats de fichiers acceptés
| Catégorie | Formats |
|---|---|
| Documents | PDF, PDF/A, DOCX, ODT, TXT |
| Images scannées | JPEG, PNG, TIFF, BMP |
| Tableurs | XLSX, ODS, CSV |
| Archives | ZIP (contenu analysé et extrait) |

### 10.3 Organisation du stockage
```
media/
├── documents/
│   └── {année}/
│       └── {mois}/
│           └── {uuid_document}.{extension}
├── thumbnails/
│   └── {uuid_document}_thumb.jpg
└── temp/
    └── (fichiers temporaires OCR — nettoyés automatiquement)
```

---

## 11. Gestion des tâches asynchrones (Celery)

Les opérations longues ou différées sont traitées de manière asynchrone via Celery. Les tâches sont organisées par application :

| Tâche | Application | Déclencheur |
|---|---|---|
| Traitement OCR | `ocr` | Upload d'un document |
| Classification IA | `intelligence` | Fin du traitement OCR |
| Envoi de notifications | `notifications` | Événements applicatifs |
| Sauvegarde planifiée | `backup` | Planification (cron) |
| Nettoyage des fichiers temp | `ocr` | Planification (cron) |
| Expiration des liens de partage | `access_requests` | Planification (cron) |
| Alertes d'expiration documentaire | `notifications` | Planification (cron) |

---

## 12. Versioning du code

### 12.1 Stratégie de branches Git
| Branche | Rôle |
|---|---|
| `main` | Code de production stable uniquement |
| `develop` | Branche d'intégration principale |
| `feature/nom-feature` | Développement d'une fonctionnalité |
| `fix/nom-bug` | Correction de bug |
| `release/vX.Y.Z` | Préparation d'une version |

### 12.2 Convention de commits
Les messages de commit suivent le standard **Conventional Commits** :
```
type(scope): description courte

feat(documents): ajouter le versioning des documents
fix(auth): corriger l'expiration du token de rafraîchissement
docs(api): mettre à jour la documentation des endpoints
refactor(ocr): extraire la logique de prétraitement dans un service dédié
```

---

## 13. Documentation

### 13.1 Règle absolue
**Tout fichier de documentation au format `.md` est obligatoirement placé dans le dossier `/docs/`.** Aucun fichier `.md` ne doit exister à l'extérieur de ce dossier dans le projet.

### 13.2 Documentation de l'API
La documentation de l'API REST est générée automatiquement via `drf-spectacular` et accessible aux URLs suivantes en développement :
- `/api/schema/` — Schéma OpenAPI brut (JSON/YAML)
- `/api/docs/` — Interface Swagger UI interactive
- `/api/redoc/` — Interface ReDoc alternative

### 13.3 Documentation des modules
Chaque module du projet possède son fichier de documentation dédié dans `/docs/`, suivant la nomenclature : `MODULE_XX_Nom_Module.md`.

---

## 14. Règles de développement non négociables

Ces règles s'appliquent à toutes les phases d'implémentation du projet :

1. **Environnement virtuel** : le dossier de l'environnement virtuel Python se nomme obligatoirement `venv`
2. **Docker** : son utilisation est interdite dans ce projet
3. **Gestionnaire de paquets frontend** : `pnpm` est le seul gestionnaire autorisé (npm et yarn sont interdits)
4. **Commandes terminal** : les commandes ne sont jamais combinées sur la même ligne
5. **Opérateur `&&` PowerShell** : son utilisation est interdite
6. **Fichiers `.md`** : ils ne peuvent exister qu'à l'intérieur du dossier `/docs/`
7. **Clés primaires** : exclusivement des UUID v4, jamais des entiers auto-incrémentés
8. **Suppression** : toujours logique (soft delete), jamais physique sauf procédure officielle
9. **Logique métier** : toujours dans `services.py`, jamais dans les vues ou modèles
10. **Requêtes SQL brutes** : interdites sauf exception documentée et justifiée
11. **Variables d'environnement** : le fichier `.env` n'est jamais versionné
12. **Code sans exemples dans les modules** : les artefacts du cahier des charges sont 100% théoriques

---

*Fin du MODULE 01 — Architecture Générale & Conventions du Projet*  
*Prochain module : MODULE 02 — Authentification, Utilisateurs, Rôles & Permissions*
