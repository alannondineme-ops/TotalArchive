# MODULE 18 — API publique, Interopérabilité & Intégrations externes

**Système d'Archivage Numérique — Administration Publique**  
**Version :** 1.0  
**Statut :** Spécification technique complète — MODULE FINAL  
**Dépendances :** MODULE 02 (Auth), MODULE 06 (Documents), MODULE 11 (Audit), MODULE 12 (Notifications), MODULE 15 (Administration)

---

## 1. PRÉSENTATION DU MODULE

### 1.1 Contexte et justification

Un système d'archivage ne vit pas en silo. Dans une administration publique, il doit s'intégrer avec l'écosystème existant : la GED du service RH, le logiciel métier des finances, l'annuaire LDAP/Active Directory, la plateforme de dématérialisation des délibérations, et demain avec des systèmes partenaires (autres collectivités, services de l'État). Sans API publique documentée et sans webhooks, chaque intégration devient un projet sur-mesure coûteux.

Ce module final expose les capacités du système vers l'extérieur de manière sécurisée, standardisée et traçable. Il constitue la **surface d'exposition contrôlée** du système — tout ce qui passe par cette API est authentifié, autorisé, rate-limité et journalisé.

### 1.2 Périmètre fonctionnel

- **API REST publique** complète avec documentation OpenAPI 3.0 interactive (Swagger UI)
- **Authentification OAuth2** (Authorization Code Flow) pour les applications tierces
- **Clés API** (API Keys) pour les intégrations machine-à-machine
- **Rate limiting** granulaire par clé API et par endpoint
- **Webhooks** : notifications sortantes vers des systèmes tiers en temps réel
- **Connecteurs prêts à l'emploi** : Google Drive, OneDrive, Nextcloud, SharePoint, FTP/SFTP, WebDAV, Slack, Teams
- **Import en masse** (bulk import) depuis sources externes
- **Export en masse** vers formats standardisés (ZIP, EAD XML, Dublin Core)

### 1.3 Principes de conception

Trois principes guident ce module :

1. **Compatibilité maximale :** L'API REST suit le standard OpenAPI 3.0. Tout système capable d'effectuer des requêtes HTTP peut s'intégrer, sans SDK propriétaire obligatoire.
2. **Sécurité par défaut :** Toute clé API est limitée en débit, en périmètre d'accès (scopes), et révocable instantanément. Chaque appel est journalisé.
3. **Traçabilité complète :** Un appel API externe est indiscernable d'une action utilisateur dans le journal d'audit — on sait toujours quelle application a fait quoi sur quel document.

---

## 2. CAS D'UTILISATION — VUE D'ENSEMBLE

| Code | Cas d'utilisation | Acteur principal | Priorité |
|------|-------------------|-----------------|----------|
| UC-018-01 | Créer une clé API pour une application tierce | Admin | HAUTE |
| UC-018-02 | Configurer les scopes d'une clé API | Admin | HAUTE |
| UC-018-03 | Définir les rate limits d'une clé API | Admin | HAUTE |
| UC-018-04 | Révoquer une clé API | Admin | HAUTE |
| UC-018-05 | Consulter les statistiques d'usage d'une clé API | Admin | HAUTE |
| UC-018-06 | Configurer un client OAuth2 (Authorization Code) | Admin, Super Admin | HAUTE |
| UC-018-07 | Autoriser une application OAuth2 (consentement) | Utilisateur | HAUTE |
| UC-018-08 | Créer un webhook vers un système externe | Admin | HAUTE |
| UC-018-09 | Tester un webhook (envoi de payload de test) | Admin | HAUTE |
| UC-018-10 | Consulter l'historique des livraisons d'un webhook | Admin | HAUTE |
| UC-018-11 | Configurer le connecteur Google Drive | Admin | HAUTE |
| UC-018-12 | Importer des documents depuis Google Drive | Archiviste, Agent | HAUTE |
| UC-018-13 | Configurer le connecteur OneDrive / SharePoint | Admin | HAUTE |
| UC-018-14 | Importer des documents depuis OneDrive | Archiviste, Agent | HAUTE |
| UC-018-15 | Configurer le connecteur Nextcloud / WebDAV | Admin | HAUTE |
| UC-018-16 | Importer des documents depuis Nextcloud | Archiviste, Agent | HAUTE |
| UC-018-17 | Configurer le connecteur FTP/SFTP | Admin | HAUTE |
| UC-018-18 | Importer des documents depuis FTP/SFTP | Archiviste, Agent | HAUTE |
| UC-018-19 | Configurer le connecteur Slack | Admin | BASSE |
| UC-018-20 | Configurer le connecteur Microsoft Teams | Admin | BASSE |
| UC-018-21 | Effectuer un import en masse (bulk import) | Archiviste, Admin | HAUTE |
| UC-018-22 | Exporter des documents en masse (ZIP + manifeste) | Archiviste, Admin | HAUTE |
| UC-018-23 | Exporter des documents au format EAD XML | Archiviste, Admin | HAUTE |
| UC-018-24 | Consulter la documentation OpenAPI interactive | Tout développeur | HAUTE |
| UC-018-25 | Consulter les logs d'appels API externes | Admin, Auditeur | HAUTE |

---

## 3. CAS D'UTILISATION DÉTAILLÉS

### UC-018-01 — Créer une clé API pour une application tierce

**Acteur principal :** Admin
**Pré-conditions :** Utilisateur authentifié Admin minimum, application tierce identifiée

**Flux principal :**
1. L'Admin accède à "Administration > API > Clés API > Nouvelle clé"
2. Il renseigne :
   - Nom de l'application (ex: "GED-RH-v2")
   - Description et responsable technique
   - Scopes autorisés (sélection multiple parmi les scopes disponibles)
   - Rate limits (requêtes/minute, requêtes/jour)
   - IP autorisées (optionnel, whitelist)
   - Date d'expiration (optionnel, recommandé)
3. Il valide
4. Le système génère une clé API au format `arch_live_` + 64 caractères aléatoires (secrets.token_urlsafe)
5. La clé est affichée **une seule fois** dans son intégralité (message d'avertissement explicite)
6. Seul le hash SHA-256 de la clé est stocké en base (jamais la clé en clair)
7. L'Admin copie la clé et la transmet de manière sécurisée au développeur tiers
8. Événement `API_KEY_CREATED` journalisé (MODULE 11)

**Règle d'or :** La clé n'est affichée qu'une seule fois. Si elle est perdue, il faut en créer une nouvelle.

---

### UC-018-08 — Créer un webhook vers un système externe

**Acteur principal :** Admin
**Pré-conditions :** URL cible accessible depuis le serveur, Admin authentifié

**Flux principal :**
1. L'Admin accède à "Administration > API > Webhooks > Nouveau webhook"
2. Il configure :
   - URL cible (HTTPS obligatoire, sauf exception localhost pour les tests)
   - Événements déclencheurs (sélection multiple parmi les événements disponibles)
   - Secret de signature (généré automatiquement ou saisi manuellement)
   - Headers personnalisés optionnels
   - Nombre de tentatives en cas d'échec (1-5)
   - Délai entre les tentatives
3. Il teste la connectivité (UC-018-09)
4. Il active le webhook
5. Événement `WEBHOOK_CREATED` journalisé

**Format de livraison :** Chaque appel webhook est une requête HTTP POST avec :
- Header `X-Archivage-Event` : type d'événement
- Header `X-Archivage-Delivery` : UUID de la livraison (idempotence)
- Header `X-Archivage-Signature-256` : HMAC-SHA256 du corps avec le secret
- Body : JSON contenant l'événement et les données associées

---

### UC-018-21 — Import en masse (bulk import)

**Acteur principal :** Archiviste, Admin
**Pré-conditions :** Fichier CSV/JSON de mapping disponible, fichiers source accessibles

**Flux principal :**
1. L'utilisateur accède à "Import > Import en masse"
2. Il sélectionne la source :
   - Fichiers locaux (ZIP avec manifeste)
   - Connecteur externe configuré (Google Drive, FTP, etc.)
   - Fichier CSV de métadonnées + dossier de fichiers
3. Il télécharge le manifeste CSV/JSON décrivant les documents à importer
4. Le système valide le manifeste (colonnes obligatoires, cohérence, doublons)
5. Il affiche un aperçu : "1247 documents à importer — 3 erreurs de validation"
6. L'utilisateur corrige ou accepte en ignorant les erreurs
7. Il déclenche l'import
8. Un `BulkImportJob` est créé, traitement asynchrone via Celery
9. L'utilisateur reçoit une notification à la fin avec le rapport d'import

**Format CSV de manifeste d'import :**
```
titre;type_document;categorie;date_creation;auteur;confidentalite;fichier
Délibération 2024-001;DELIBERATION;Délibérations;2024-01-15;Jean Martin;PUBLIC;delib_001.pdf
Courrier préfecture;COURRIER;Courrier entrant;2024-01-16;Marie Dupont;INTERNAL;courrier_pref.pdf
```

---

### UC-018-23 — Export au format EAD XML

**Acteur principal :** Archiviste, Admin
**Pré-conditions :** Sélection de documents ou périmètre défini, droits suffisants

**Contexte :** EAD (Encoded Archival Description) est le standard international XML pour la description des instruments de recherche archivistiques. Il est exigé par de nombreuses autorités d'archivage (Archives Nationales, archives départementales) pour les versements électroniques.

**Flux principal :**
1. L'Archiviste sélectionne les documents ou définit un filtre (service, catégorie, période)
2. Il accède à "Exporter > Format EAD XML"
3. Il configure :
   - Niveau de description (fonds, sous-fonds, série, sous-série, dossier, pièce)
   - Métadonnées EAD de l'instrument (titre du fonds, dates extrêmes, producteur, langue)
   - Inclure les fichiers numériques : oui/non
   - Version EAD : 2002 ou EAD3
4. L'export est généré en arrière-plan (Celery)
5. Un fichier ZIP est produit contenant :
   - `instrument_de_recherche.xml` (EAD complet)
   - `documents/` (fichiers numériques, si demandé)
   - `README.txt` (guide de lecture)
6. Le fichier est disponible en téléchargement (rétention 7 jours)

---

## 4. MODÈLES DE DONNÉES

### 4.1 ApiKey — Clés API

```python
class ApiKey(BaseModel):
    """
    Clé API pour l'accès machine-à-machine ou par application tierce.
    La clé en clair n'est jamais stockée — uniquement son hash SHA-256.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    # Identification de l'application
    name = models.CharField(
        max_length=200,
        help_text="Nom de l'application tierce (ex: 'GED-RH-v2', 'Connecteur-Parapheur')"
    )
    description = models.TextField(blank=True)
    technical_contact = models.EmailField(
        blank=True,
        help_text="Email du responsable technique de l'application"
    )

    # Clé hashée (jamais la clé en clair)
    key_hash = models.CharField(
        max_length=64,
        unique=True,
        help_text="SHA-256 de la clé en clair. La clé elle-même n'est pas stockée."
    )
    key_prefix = models.CharField(
        max_length=12,
        help_text="Les 8 premiers caractères de la clé pour l'identification visuelle (ex: arch_live)"
    )

    # Scopes autorisés
    scopes = models.JSONField(
        default=list,
        help_text="""Scopes autorisés, ex:
        ['documents:read', 'documents:write', 'categories:read',
         'audit:read', 'workflows:read', 'users:read']"""
    )

    # Rate limiting
    rate_limit_per_minute = models.PositiveIntegerField(
        default=60,
        help_text="Requêtes maximales par minute (0 = illimité, déconseillé)"
    )
    rate_limit_per_day = models.PositiveIntegerField(
        default=10000,
        help_text="Requêtes maximales par jour"
    )

    # Restrictions IP (liste blanche optionnelle)
    allowed_ips = models.JSONField(
        default=list,
        help_text="IPs ou CIDR autorisés (liste vide = toutes les IPs autorisées)"
    )

    # Statut et expiration
    is_active = models.BooleanField(default=True)
    expires_at = models.DateTimeField(
        null=True, blank=True,
        help_text="Date d'expiration (null = pas d'expiration)"
    )
    last_used_at = models.DateTimeField(null=True, blank=True)
    last_used_ip = models.GenericIPAddressField(null=True, blank=True)

    # Statistiques d'usage
    total_requests = models.BigIntegerField(default=0)
    requests_today = models.PositiveIntegerField(default=0)
    requests_today_reset_at = models.DateField(null=True, blank=True)

    # Révocation
    revoked_at = models.DateTimeField(null=True, blank=True)
    revoked_by = models.ForeignKey(
        'users.User', on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='revoked_api_keys'
    )
    revocation_reason = models.TextField(blank=True)

    # Propriétaire
    created_by = models.ForeignKey(
        'users.User', on_delete=models.PROTECT,
        related_name='created_api_keys'
    )
    owned_by_user = models.ForeignKey(
        'users.User', on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='owned_api_keys',
        help_text="Utilisateur 'technique' associé à cette clé pour les droits d'accès"
    )

    # Horodatage
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='deleted_api_keys'
    )

    class Meta:
        db_table = 'api_keys'
        indexes = [
            models.Index(fields=['key_hash']),
            models.Index(fields=['is_active', 'expires_at']),
            models.Index(fields=['created_by']),
        ]

    @classmethod
    def verify_key(cls, raw_key: str) -> 'ApiKey | None':
        """
        Vérifie une clé API brute.
        Retourne l'instance ApiKey si valide, None sinon.
        """
        import hashlib
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
        try:
            key = cls.objects.get(
                key_hash=key_hash,
                is_active=True,
                is_deleted=False
            )
            if key.expires_at and key.expires_at < timezone.now():
                return None
            return key
        except cls.DoesNotExist:
            return None

    @classmethod
    def generate(cls) -> tuple[str, str]:
        """
        Génère une nouvelle clé API.
        Retourne (clé_en_clair, hash_sha256).
        La clé en clair doit être affichée une seule fois et jamais stockée.
        """
        import secrets, hashlib
        prefix = 'arch_live_'
        raw_key = prefix + secrets.token_urlsafe(48)
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
        return raw_key, key_hash
```

---

### 4.2 OAuthClient — Clients OAuth2

```python
class OAuthClient(BaseModel):
    """
    Client OAuth2 enregistré pour le flux Authorization Code.
    Permet à des applications tierces d'agir au nom d'un utilisateur humain
    après son consentement explicite.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    client_id = models.CharField(
        max_length=100, unique=True,
        help_text="Identifiant public du client OAuth2 (non secret)"
    )
    client_secret_hash = models.CharField(
        max_length=64,
        help_text="SHA-256 du secret client (jamais en clair)"
    )

    name = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    logo_url = models.URLField(blank=True)

    # URLs autorisées
    redirect_uris = models.JSONField(
        default=list,
        help_text="Liste des URI de redirection autorisées après autorisation"
    )
    allowed_origins = models.JSONField(
        default=list,
        help_text="Origines CORS autorisées pour les requêtes AJAX"
    )

    # Scopes disponibles pour ce client
    allowed_scopes = models.JSONField(default=list)

    # Type de client
    client_type = models.CharField(
        max_length=20,
        choices=[
            ('CONFIDENTIAL', 'Confidentiel (serveur, peut garder un secret)'),
            ('PUBLIC', 'Public (SPA, mobile, ne peut pas garder un secret)'),
        ],
        default='CONFIDENTIAL'
    )

    # Durées de vie des tokens
    access_token_lifetime_seconds = models.PositiveIntegerField(default=3600)
    refresh_token_lifetime_seconds = models.PositiveIntegerField(default=2592000)

    is_active = models.BooleanField(default=True)
    is_trusted = models.BooleanField(
        default=False,
        help_text="Si True, le consentement utilisateur est ignoré (applications internes)"
    )

    created_by = models.ForeignKey(
        'users.User', on_delete=models.PROTECT,
        related_name='created_oauth_clients'
    )

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='deleted_oauth_clients'
    )

    class Meta:
        db_table = 'oauth_clients'
        indexes = [
            models.Index(fields=['client_id']),
            models.Index(fields=['is_active']),
        ]
```

---

### 4.3 Webhook — Webhooks sortants

```python
class Webhook(BaseModel):
    """
    Webhook sortant : le système envoie des notifications HTTP POST
    vers des systèmes tiers lors d'événements spécifiques.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    name = models.CharField(max_length=200)
    description = models.TextField(blank=True)

    # URL cible
    url = models.URLField(
        max_length=2000,
        help_text="URL HTTPS vers laquelle les événements sont envoyés"
    )

    # Événements déclencheurs
    events = models.JSONField(
        default=list,
        help_text="""Événements déclencheurs, ex:
        ['document.created', 'document.validated', 'document.deleted',
         'workflow.completed', 'access_request.approved', 'user.created']"""
    )

    # Authentification
    secret = models.CharField(
        max_length=100,
        help_text="Secret HMAC-SHA256 pour signer les payloads (stocké chiffré)"
    )
    custom_headers = models.JSONField(
        default=dict,
        help_text="Headers HTTP supplémentaires (ex: Authorization, X-Custom-Header)"
    )

    # Paramètres de livraison
    is_active = models.BooleanField(default=True)
    timeout_seconds = models.PositiveSmallIntegerField(
        default=10,
        help_text="Timeout de la requête HTTP sortante"
    )
    max_retries = models.PositiveSmallIntegerField(
        default=3,
        help_text="Tentatives en cas d'échec (délai exponentiel : 1min, 5min, 30min)"
    )

    # Filtres avancés (optionnel)
    filter_conditions = models.JSONField(
        default=dict,
        help_text="""Filtres sur les données de l'événement, ex:
        {'confidentiality_level': 'PUBLIC', 'category_id': 'uuid-...'}"""
    )

    # Statistiques
    total_deliveries = models.BigIntegerField(default=0)
    successful_deliveries = models.BigIntegerField(default=0)
    failed_deliveries = models.BigIntegerField(default=0)
    last_delivery_at = models.DateTimeField(null=True, blank=True)
    last_delivery_status = models.CharField(max_length=20, blank=True)

    # Créateur
    created_by = models.ForeignKey(
        'users.User', on_delete=models.PROTECT,
        related_name='created_webhooks'
    )

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='deleted_webhooks'
    )

    class Meta:
        db_table = 'webhooks'
        indexes = [
            models.Index(fields=['is_active']),
            models.Index(fields=['created_by']),
        ]
```

---

### 4.4 WebhookDelivery — Historique des livraisons

```python
class WebhookDelivery(BaseModel):
    """
    Enregistrement de chaque tentative de livraison d'un webhook.
    Append-only : jamais de modification après création.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    webhook = models.ForeignKey(
        Webhook, on_delete=models.CASCADE,
        related_name='deliveries'
    )

    # Événement déclencheur
    event_type = models.CharField(max_length=100)
    event_id = models.UUIDField(
        help_text="ID unique de l'événement (pour idempotence côté récepteur)"
    )

    # Payload envoyé
    payload = models.JSONField(help_text="Corps JSON envoyé au webhook")
    headers_sent = models.JSONField(
        default=dict,
        help_text="Headers envoyés (sans les secrets)"
    )

    # Résultat
    status = models.CharField(
        max_length=20,
        choices=[
            ('PENDING', 'En attente'),
            ('SUCCESS', 'Livré'),
            ('FAILED', 'Échec'),
            ('RETRYING', 'Nouvelle tentative en cours'),
        ],
        default='PENDING', db_index=True
    )
    attempt_number = models.PositiveSmallIntegerField(default=1)
    http_status_code = models.PositiveSmallIntegerField(null=True, blank=True)
    response_body = models.TextField(
        blank=True,
        help_text="Corps de la réponse (premiers 1000 caractères)"
    )
    response_time_ms = models.PositiveIntegerField(null=True, blank=True)
    error_message = models.TextField(blank=True)

    # Prochaine tentative
    next_retry_at = models.DateTimeField(null=True, blank=True)

    # Horodatage
    delivered_at = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='deleted_webhook_deliveries'
    )

    class Meta:
        db_table = 'webhook_deliveries'
        indexes = [
            models.Index(fields=['webhook', 'created_at']),
            models.Index(fields=['status', 'next_retry_at']),
            models.Index(fields=['event_type', 'status']),
        ]
```

---

### 4.5 ExternalConnector — Connecteurs externes

```python
class ExternalConnector(BaseModel):
    """
    Configuration d'un connecteur vers un service de stockage externe
    (Google Drive, OneDrive, Nextcloud, FTP, WebDAV, etc.)
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    name = models.CharField(max_length=200)
    description = models.TextField(blank=True)

    connector_type = models.CharField(
        max_length=30,
        choices=[
            ('GOOGLE_DRIVE', 'Google Drive'),
            ('ONEDRIVE', 'Microsoft OneDrive'),
            ('SHAREPOINT', 'Microsoft SharePoint'),
            ('NEXTCLOUD', 'Nextcloud'),
            ('WEBDAV', 'WebDAV générique'),
            ('FTP', 'FTP'),
            ('SFTP', 'SFTP'),
            ('SLACK', 'Slack'),
            ('TEAMS', 'Microsoft Teams'),
            ('S3_COMPATIBLE', 'S3 compatible (MinIO, etc.)'),
        ],
        db_index=True
    )

    # Configuration (champs variables selon le type)
    config = models.JSONField(
        default=dict,
        help_text="""Configuration selon le type :
        Google Drive : {client_id, client_secret, refresh_token, folder_id}
        OneDrive     : {tenant_id, client_id, client_secret, drive_id, folder_path}
        Nextcloud    : {base_url, username, password_encrypted, folder_path}
        FTP          : {host, port, username, password_encrypted, passive_mode, base_path}
        SFTP         : {host, port, username, private_key_path, base_path}
        WebDAV       : {base_url, username, password_encrypted, verify_ssl}
        Slack        : {bot_token_encrypted, default_channel}
        Teams        : {webhook_url_encrypted, team_id, channel_id}
        S3           : {endpoint_url, access_key, secret_key_encrypted, bucket_name, prefix}"""
    )

    # Statut
    is_active = models.BooleanField(default=True)
    last_tested_at = models.DateTimeField(null=True, blank=True)
    last_test_status = models.CharField(
        max_length=20,
        choices=[('OK', 'OK'), ('FAILED', 'Échec')],
        null=True, blank=True
    )
    last_test_error = models.TextField(blank=True)

    # Mapping de catégories : dossier externe → catégorie interne
    category_mapping = models.JSONField(
        default=dict,
        help_text="Mapping dossier externe → ID catégorie interne pour l'import automatique"
    )

    # Synchronisation automatique
    auto_import_enabled = models.BooleanField(
        default=False,
        help_text="Si True, importe automatiquement les nouveaux fichiers (Celery Beat)"
    )
    auto_import_interval_minutes = models.PositiveIntegerField(
        default=60,
        help_text="Fréquence de vérification pour l'import automatique"
    )
    auto_import_last_run = models.DateTimeField(null=True, blank=True)
    auto_import_cursor = models.TextField(
        blank=True,
        help_text="Curseur/token de pagination pour reprendre là où on s'est arrêté"
    )

    created_by = models.ForeignKey(
        'users.User', on_delete=models.PROTECT,
        related_name='created_connectors'
    )

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='deleted_connectors'
    )

    class Meta:
        db_table = 'external_connectors'
        indexes = [
            models.Index(fields=['connector_type', 'is_active']),
            models.Index(fields=['auto_import_enabled']),
        ]
```

---

### 4.6 BulkImportJob — Jobs d'import en masse

```python
class BulkImportJob(BaseModel):
    """
    Job d'import en masse depuis une source externe.
    La génération est asynchrone (Celery).
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    reference = models.CharField(
        max_length=30, unique=True,
        help_text="Référence lisible : IMP-2026-00001"
    )

    source_type = models.CharField(
        max_length=30,
        choices=[
            ('ZIP_MANIFEST', 'Archive ZIP avec manifeste'),
            ('CSV_MANIFEST', 'Fichier CSV de métadonnées + dossier de fichiers'),
            ('CONNECTOR', 'Connecteur externe configuré'),
            ('API_PUSH', 'Push via l\'API publique'),
        ]
    )

    connector = models.ForeignKey(
        ExternalConnector, on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='import_jobs'
    )

    # Configuration
    import_config = models.JSONField(
        default=dict,
        help_text="Paramètres d'import : mapping colonnes, catégorie cible, etc."
    )

    status = models.CharField(
        max_length=20,
        choices=[
            ('PENDING', 'En attente'),
            ('VALIDATING', 'Validation du manifeste'),
            ('PROCESSING', 'Import en cours'),
            ('COMPLETED', 'Terminé avec succès'),
            ('PARTIAL', 'Terminé avec erreurs'),
            ('FAILED', 'Échec'),
            ('CANCELLED', 'Annulé'),
        ],
        default='PENDING', db_index=True
    )

    # Métriques
    total_items = models.PositiveIntegerField(default=0)
    processed_items = models.PositiveIntegerField(default=0)
    success_items = models.PositiveIntegerField(default=0)
    error_items = models.PositiveIntegerField(default=0)
    skipped_items = models.PositiveIntegerField(default=0)

    # Rapport d'import
    error_report = models.JSONField(
        default=list,
        help_text="Liste des erreurs : [{row, field, error_code, error_message}]"
    )

    celery_task_id = models.CharField(max_length=255, blank=True)
    initiated_by = models.ForeignKey(
        'users.User', on_delete=models.PROTECT,
        related_name='initiated_import_jobs'
    )

    started_at = models.DateTimeField(null=True, blank=True)
    completed_at = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='deleted_import_jobs'
    )

    class Meta:
        db_table = 'bulk_import_jobs'
        indexes = [
            models.Index(fields=['status', 'created_at']),
            models.Index(fields=['initiated_by']),
        ]
```

---

## 5. SCOPES DE L'API PUBLIQUE

| Scope | Description | Endpoints concernés |
|-------|-------------|---------------------|
| `documents:read` | Lecture des documents | GET /documents/*, GET /documents/{id}/ |
| `documents:write` | Création et modification | POST, PATCH /documents/ |
| `documents:delete` | Suppression | DELETE /documents/{id}/ |
| `categories:read` | Lecture du référentiel | GET /categories/, GET /document-types/ |
| `workflows:read` | Lecture des workflows | GET /workflow-instances/ |
| `workflows:write` | Actions sur les workflows | POST /workflow-instances/{id}/action/ |
| `access:read` | Lecture des demandes d'accès | GET /access-requests/ |
| `access:write` | Gestion des accès | POST /access-requests/, PATCH ... |
| `audit:read` | Lecture des journaux d'audit | GET /audit-logs/ |
| `users:read` | Lecture des utilisateurs | GET /users/ |
| `reports:read` | Accès aux rapports | GET /reports/ |
| `search:read` | Recherche full-text | GET /search/ |
| `admin:read` | Lecture de la configuration admin | GET /admin/settings/ |
| `admin:write` | Modification de la configuration | PATCH /admin/settings/ |
| `bulk:import` | Import en masse | POST /bulk-import/ |
| `bulk:export` | Export en masse | POST /bulk-export/ |
| `webhooks:manage` | Gestion des webhooks | /webhooks/ |

---

## 6. MATRICE DES PERMISSIONS

| Action | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|--------|:-----------:|:-----:|:----------:|:-----------:|:-----:|:--------:|
| Créer une clé API | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Révoquer une clé API | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Voir statistiques clés API | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Configurer OAuth2 client | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Créer un webhook | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Voir historique livraisons webhook | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Configurer connecteur externe | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Importer depuis connecteur externe | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Import en masse (bulk) | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Export en masse (ZIP/EAD) | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| Consulter logs appels API | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Voir documentation OpenAPI | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

---

## 7. SÉQUENCES DÉTAILLÉES

### Séquence 1 — Appel API avec clé API (machine-à-machine)

```
Application tierce           Serveur (DRF)              PostgreSQL / Redis
      |                           |                            |
      |-- GET /api/v1/documents/ ->|                           |
      |   Header: X-Api-Key: arch_live_AbCd...                |
      |                           |                            |
      |                           |-- ApiKeyAuthentication ---|
      |                           |   SHA-256(raw_key) ------->|
      |                           |   SELECT * FROM api_keys  |
      |                           |   WHERE key_hash = '...'  |
      |                           |<-- ApiKey instance --------|
      |                           |                            |
      |                           |-- Vérifie is_active ✅    |
      |                           |-- Vérifie expiration ✅   |
      |                           |-- Vérifie IP whitelist ✅ |
      |                           |-- Vérifie scope 'documents:read' ✅
      |                           |                            |
      |                           |-- Rate limiting (Redis) ---|
      |                           |   INCR api:rate:key_id:min|
      |                           |   (TTL 60s)               |
      |                           |   count = 23 < 60 → OK    |
      |                           |                            |
      |                           |-- Journalise appel ------->|
      |                           |   (API_KEY_USED, endpoint)|
      |                           |                            |
      |                           |-- Exécute la vue ---------->|
      |                           |<-- Documents (filtrés) ----|
      |                           |                            |
      |<-- 200 JSON documents -----|                           |
```

---

### Séquence 2 — Livraison de webhook avec retry

```
Événement interne        Serveur (Celery)             Système tiers
       |                       |                           |
       |-- document.validated   |                           |
       |   (signal Django) ---->|                           |
       |                       |                           |
       |                       |-- Cherche webhooks ------->|
       |                       |   actifs pour cet event   |
       |                       |<-- [Webhook A, Webhook B] -|
       |                       |                           |
       |                       |-- Crée WebhookDelivery    |
       |                       |   (attempt=1, PENDING)    |
       |                       |                           |
       |                       |-- Task: deliver_webhook() ->|
       |                       |                           |
       |                       |-- Signe le payload ------->|
       |                       |   HMAC-SHA256(body, secret)|
       |                       |                           |
       |                       |-- POST {url_cible} ------->|
       |                       |   X-Archivage-Event: document.validated
       |                       |   X-Archivage-Delivery: {uuid}
       |                       |   X-Archivage-Signature-256: sha256=...
       |                       |   Body: {event, data}     |
       |                       |                           |
       |                       |<-- 200 OK (78ms) ----------|
       |                       |                           |
       |                       |-- Update WebhookDelivery  |
       |                       |   (status=SUCCESS)        |
       |                       |                           |
       |   [Si 5xx ou timeout]  |                           |
       |                       |-- Retry après 1 min ------>|
       |                       |<-- 503 (toujours en erreur)|
       |                       |-- Retry après 5 min ------>|
       |                       |<-- 200 OK -----------------|
       |                       |-- Update SUCCESS (attempt=3)|
```

---

### Séquence 3 — Import automatique depuis Google Drive

```
Celery Beat              Connector Service            Google Drive API
     |                        |                            |
     |-- auto_import_gdrive -->|                            |
     |                        |                            |
     |                        |-- Charge ExternalConnector |
     |                        |   (type=GOOGLE_DRIVE)      |
     |                        |                            |
     |                        |-- Refresh OAuth token ---->|
     |                        |<-- access_token -----------|
     |                        |                            |
     |                        |-- GET /drive/v3/files/ --->|
     |                        |   ?q=modifiedTime > cursor |
     |                        |   &orderBy=modifiedTime    |
     |                        |<-- {files: [{id, name, ...}]}
     |                        |                            |
     |                        |-- Pour chaque fichier :    |
     |                        |-- GET /drive/v3/files/{id} -->|
     |                        |   ?alt=media (download)    |
     |                        |<-- Binaire du fichier -----|
     |                        |                            |
     |                        |-- Applique category_mapping|
     |                        |   (dossier Drive → catég.) |
     |                        |                            |
     |                        |-- Crée Document en base    |
     |                        |   (via MODULE 06)          |
     |                        |-- Déclenche OCR (MOD 07)  |
     |                        |                            |
     |                        |-- Met à jour cursor ------>|
     |                        |   (dernier modifiedTime)   |
     |                        |                            |
     |                        |-- Journalise IMPORT_COMPLETED
```

---

### Séquence 4 — OAuth2 Authorization Code Flow

```
Utilisateur          Application tierce           Serveur OAuth2
     |                      |                           |
     |-- Clique "Connecter" ->|                          |
     |                      |-- Redirect vers ----------->|
     |                      |   /oauth/authorize/         |
     |                      |   ?client_id=APP_001       |
     |                      |   &scope=documents:read    |
     |                      |   &redirect_uri=https://....|
     |                      |   &state=random123         |
     |                      |                            |
     |<----- Page consentement ----------------------------|
     |   "L'application APP_001 demande l'accès à :       |
     |    - Lecture de vos documents                       |
     |    Autoriser ou Refuser ?"                          |
     |                      |                            |
     |-- Clique "Autoriser" ->|                           |
     |                      |                            |
     |<---- Redirect vers redirect_uri -------------------|
     |      ?code=AUTH_CODE_ABC                           |
     |      &state=random123                              |
     |                      |                            |
     |                      |-- POST /oauth/token/ ------>|
     |                      |   {code, client_secret,    |
     |                      |    redirect_uri}            |
     |                      |<-- {access_token,          |
     |                      |     refresh_token,         |
     |                      |     expires_in}            |
     |                      |                            |
     |                      |-- GET /api/v1/documents/-->|
     |                      |   Authorization: Bearer .. |
     |                      |<-- Documents de l'utilisateur
```

---

## 8. ENDPOINTS API

### 8.1 Gestion des clés API

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/admin/api-keys/` | Lister les clés API | JWT + Admin |
| `POST` | `/api/v1/admin/api-keys/` | Créer une nouvelle clé API | JWT + Admin |
| `GET` | `/api/v1/admin/api-keys/{id}/` | Détail d'une clé API | JWT + Admin |
| `PATCH` | `/api/v1/admin/api-keys/{id}/` | Modifier scopes, rate limits | JWT + Admin |
| `POST` | `/api/v1/admin/api-keys/{id}/revoke/` | Révoquer la clé | JWT + Admin |
| `GET` | `/api/v1/admin/api-keys/{id}/stats/` | Statistiques d'usage | JWT + Admin |
| `GET` | `/api/v1/admin/api-keys/{id}/logs/` | Logs d'appels de cette clé | JWT + Admin |

### 8.2 OAuth2

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/oauth/authorize/` | Page de consentement utilisateur | JWT |
| `POST` | `/oauth/authorize/` | Soumettre le consentement | JWT |
| `POST` | `/oauth/token/` | Obtenir access + refresh token | Client credentials |
| `POST` | `/oauth/token/revoke/` | Révoquer un token | Client credentials |
| `GET` | `/oauth/userinfo/` | Infos utilisateur (OIDC) | Bearer |
| `GET` | `/api/v1/admin/oauth-clients/` | Lister les clients OAuth2 | JWT + Admin |
| `POST` | `/api/v1/admin/oauth-clients/` | Créer un client OAuth2 | JWT + Admin |
| `PATCH` | `/api/v1/admin/oauth-clients/{id}/` | Modifier un client | JWT + Admin |
| `DELETE` | `/api/v1/admin/oauth-clients/{id}/` | Soft delete | JWT + Admin |

### 8.3 Webhooks

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/admin/webhooks/` | Lister les webhooks | JWT + Admin |
| `POST` | `/api/v1/admin/webhooks/` | Créer un webhook | JWT + Admin |
| `GET` | `/api/v1/admin/webhooks/{id}/` | Détail d'un webhook | JWT + Admin |
| `PATCH` | `/api/v1/admin/webhooks/{id}/` | Modifier | JWT + Admin |
| `POST` | `/api/v1/admin/webhooks/{id}/test/` | Envoyer un payload de test | JWT + Admin |
| `POST` | `/api/v1/admin/webhooks/{id}/activate/` | Activer | JWT + Admin |
| `POST` | `/api/v1/admin/webhooks/{id}/deactivate/` | Désactiver | JWT + Admin |
| `DELETE` | `/api/v1/admin/webhooks/{id}/` | Soft delete | JWT + Admin |
| `GET` | `/api/v1/admin/webhooks/{id}/deliveries/` | Historique des livraisons | JWT + Admin |
| `POST` | `/api/v1/admin/webhooks/deliveries/{id}/retry/` | Relancer une livraison échouée | JWT + Admin |
| `GET` | `/api/v1/admin/webhooks/events/` | Liste des événements disponibles | JWT + Admin |

### 8.4 Connecteurs externes

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/admin/connectors/` | Lister les connecteurs | JWT + Admin |
| `POST` | `/api/v1/admin/connectors/` | Créer un connecteur | JWT + Admin |
| `GET` | `/api/v1/admin/connectors/{id}/` | Détail | JWT + Admin |
| `PATCH` | `/api/v1/admin/connectors/{id}/` | Modifier | JWT + Admin |
| `POST` | `/api/v1/admin/connectors/{id}/test/` | Tester la connectivité | JWT + Admin |
| `POST` | `/api/v1/admin/connectors/{id}/import/` | Déclencher un import | JWT + Archiviste |
| `DELETE` | `/api/v1/admin/connectors/{id}/` | Soft delete | JWT + Admin |

### 8.5 Import et export en masse

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/bulk-import/` | Lister les jobs d'import | JWT + Perm |
| `POST` | `/api/v1/bulk-import/` | Démarrer un import en masse | JWT + Perm |
| `GET` | `/api/v1/bulk-import/{id}/` | Statut et détail d'un job | JWT + Perm |
| `POST` | `/api/v1/bulk-import/{id}/cancel/` | Annuler un import en cours | JWT + Perm |
| `GET` | `/api/v1/bulk-import/{id}/report/` | Rapport CSV des erreurs | JWT + Perm |
| `GET` | `/api/v1/bulk-import/template/` | Télécharger le CSV modèle | JWT |
| `GET` | `/api/v1/bulk-export/` | Lister les exports | JWT + Perm |
| `POST` | `/api/v1/bulk-export/` | Démarrer un export (ZIP, EAD, CSV) | JWT + Perm |
| `GET` | `/api/v1/bulk-export/{id}/` | Statut d'un export | JWT + Perm |
| `GET` | `/api/v1/bulk-export/{id}/download/` | Télécharger le fichier exporté | JWT + Perm |

### 8.6 Documentation et monitoring API

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/docs/` | Documentation Swagger UI interactive | JWT |
| `GET` | `/api/v1/schema/` | Schéma OpenAPI 3.0 (JSON) | JWT |
| `GET` | `/api/v1/schema/yaml/` | Schéma OpenAPI 3.0 (YAML) | JWT |
| `GET` | `/api/v1/admin/api-logs/` | Logs de tous les appels API | JWT + Admin |
| `GET` | `/api/v1/admin/api-stats/` | Statistiques d'usage globales | JWT + Admin |

---

## 9. FORMAT DES WEBHOOKS

### Payload standard

```json
{
  "id": "evt_a1b2c3d4-...",
  "event": "document.validated",
  "created_at": "2026-02-18T10:30:00Z",
  "system": {
    "name": "Archivage Numérique",
    "organization": "Préfecture de Paris",
    "version": "1.0.0"
  },
  "data": {
    "document": {
      "id": "doc_uuid_here",
      "reference": "DOC-2026-00843",
      "title": "Délibération du conseil municipal - Séance du 15 janvier 2026",
      "document_type": "DELIBERATION",
      "category": {
        "id": "cat_uuid",
        "name": "Délibérations"
      },
      "confidentiality_level": "PUBLIC",
      "lifecycle_status": "ACTIVE",
      "processing_status": "VALIDATED",
      "file_size_bytes": 284920,
      "created_at": "2026-02-18T09:00:00Z",
      "validated_at": "2026-02-18T10:30:00Z",
      "validated_by": {
        "id": "user_uuid",
        "full_name": "Marie Archiviste",
        "role": "ARCHIVISTE"
      }
    }
  }
}
```

### Headers HTTP envoyés

```
POST https://webhook.exemple.fr/archivage-events
Content-Type: application/json
User-Agent: Archivage-Numerique/1.0 WebhookDelivery/1.0
X-Archivage-Event: document.validated
X-Archivage-Delivery: evt_a1b2c3d4-...
X-Archivage-Timestamp: 1708255800
X-Archivage-Signature-256: sha256=a1b2c3d4e5f6...
```

### Liste des événements webhook disponibles

| Événement | Déclencheur |
|-----------|-------------|
| `document.created` | Nouveau document créé |
| `document.updated` | Document modifié |
| `document.validated` | Document approuvé par un archiviste |
| `document.rejected` | Document rejeté |
| `document.deleted` | Document supprimé (soft delete) |
| `document.archived` | Document passé au statut archivé |
| `workflow.started` | Nouveau workflow démarré |
| `workflow.step_completed` | Étape de workflow validée |
| `workflow.completed` | Workflow clôturé |
| `workflow.escalated` | Workflow escaladé |
| `access_request.submitted` | Demande d'accès soumise |
| `access_request.approved` | Demande d'accès approuvée |
| `access_request.rejected` | Demande d'accès rejetée |
| `share_link.created` | Lien de partage créé |
| `share_link.accessed` | Lien de partage consulté |
| `user.created` | Nouvel utilisateur créé |
| `user.deactivated` | Utilisateur désactivé |
| `backup.completed` | Sauvegarde terminée |
| `backup.failed` | Sauvegarde échouée |

---

## 10. DOCUMENTATION OPENAPI — CONFIGURATION

```python
# settings.py — Configuration drf-spectacular

SPECTACULAR_SETTINGS = {
    'TITLE': 'API Archivage Numérique',
    'DESCRIPTION': """
## API REST du Système d'Archivage Numérique

Cette API permet l'intégration du système d'archivage avec des applications tierces.

### Authentification

Deux modes d'authentification sont supportés :

**1. JWT (Bearer Token)** — Pour les utilisateurs humains :
```
Authorization: Bearer <access_token>
```

**2. Clé API (X-Api-Key)** — Pour les intégrations machine-à-machine :
```
X-Api-Key: arch_live_<votre_cle>
```

### Rate Limiting

Chaque clé API dispose de limites configurées par l'administrateur.
Les headers de réponse indiquent les limites et le compteur actuel :
- `X-RateLimit-Limit-Minute` : Limite par minute
- `X-RateLimit-Remaining-Minute` : Requêtes restantes cette minute
- `X-RateLimit-Reset` : Timestamp de réinitialisation

### Pagination

Toutes les listes sont paginées avec les paramètres `?page=N&page_size=M` (max 100).

### Gestion des erreurs

Les erreurs retournent un JSON structuré :
```json
{"success": false, "error": {"code": "NOT_FOUND", "message": "Document introuvable"}}
```
    """,
    'VERSION': '1.0.0',
    'SERVE_INCLUDE_SCHEMA': False,
    'COMPONENT_SPLIT_REQUEST': True,
    'TAGS': [
        {'name': 'Documents', 'description': 'Gestion des documents archivés'},
        {'name': 'Catégories', 'description': 'Référentiel taxonomique'},
        {'name': 'Workflows', 'description': 'Circuits de validation'},
        {'name': 'Accès & Partage', 'description': 'Gestion des accès et partages'},
        {'name': 'Recherche', 'description': 'Recherche full-text'},
        {'name': 'Audit', 'description': 'Journal d\'audit'},
        {'name': 'Webhooks', 'description': 'Notifications sortantes'},
        {'name': 'Import/Export', 'description': 'Import et export en masse'},
        {'name': 'Administration', 'description': 'Configuration et administration'},
    ],
    'SWAGGER_UI_SETTINGS': {
        'persistAuthorization': True,
        'displayRequestDuration': True,
        'filter': True,
    },
    'AUTHENTICATION_WHITELIST': [
        'rest_framework.authentication.SessionAuthentication',
        'apps.api.authentication.ApiKeyAuthentication',
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}
```

---

## 11. MIDDLEWARE DE RATE LIMITING

```python
# apps/api/middleware/rate_limiting.py

import hashlib
from django.core.cache import cache
from django.http import JsonResponse


class ApiKeyRateLimitMiddleware:
    """
    Middleware de rate limiting pour les appels avec clé API.
    Utilise Redis pour les compteurs avec TTL automatique.
    """

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        api_key = request.META.get('HTTP_X_API_KEY')

        if not api_key:
            return self.get_response(request)

        # Récupérer la clé API (via cache Redis d'abord)
        from apps.api.models import ApiKey
        key_instance = ApiKey.verify_key(api_key)

        if not key_instance:
            return JsonResponse(
                {'success': False, 'error': {'code': 'INVALID_API_KEY', 'message': 'Clé API invalide ou révoquée'}},
                status=401
            )

        # Vérification rate limit par minute
        minute_key = f"api:rate:{key_instance.id}:minute"
        minute_count = cache.get(minute_key, 0)

        if minute_count >= key_instance.rate_limit_per_minute:
            return JsonResponse(
                {'success': False, 'error': {'code': 'RATE_LIMIT_EXCEEDED', 'message': 'Limite de requêtes par minute dépassée'}},
                status=429,
                headers={
                    'X-RateLimit-Limit-Minute': str(key_instance.rate_limit_per_minute),
                    'X-RateLimit-Remaining-Minute': '0',
                    'Retry-After': '60',
                }
            )

        # Vérification rate limit par jour
        day_key = f"api:rate:{key_instance.id}:day"
        day_count = cache.get(day_key, 0)

        if day_count >= key_instance.rate_limit_per_day:
            return JsonResponse(
                {'success': False, 'error': {'code': 'DAILY_RATE_LIMIT_EXCEEDED', 'message': 'Limite de requêtes journalière dépassée'}},
                status=429
            )

        # Incrémenter les compteurs
        pipe = cache.client.pipeline()
        pipe.incr(minute_key)
        pipe.expire(minute_key, 60)
        pipe.incr(day_key)
        pipe.expire(day_key, 86400)
        pipe.execute()

        # Attacher la clé à la request pour les vues
        request.api_key = key_instance

        response = self.get_response(request)

        # Ajouter les headers informatifs
        response['X-RateLimit-Limit-Minute'] = str(key_instance.rate_limit_per_minute)
        response['X-RateLimit-Remaining-Minute'] = str(
            max(0, key_instance.rate_limit_per_minute - minute_count - 1)
        )

        return response
```

---

## 12. SERVICE D'EXPORT EAD XML

```python
# apps/api/services/ead_export_service.py

from lxml import etree
from datetime import datetime
from django.utils import timezone


class EADExportService:
    """
    Génère un instrument de recherche au format EAD 2002 ou EAD3.
    Standard international pour la description archivistique.
    """

    def generate_ead_xml(
        self,
        documents: list,
        config: dict,
        ead_version: str = '2002'
    ) -> bytes:
        """
        Génère le fichier EAD XML complet.
        """
        if ead_version == '3':
            return self._generate_ead3(documents, config)
        return self._generate_ead2002(documents, config)

    def _generate_ead2002(self, documents: list, config: dict) -> bytes:
        """Génère un fichier EAD 2002."""
        nsmap = {
            None: 'urn:isbn:1-931666-22-9',
            'xlink': 'http://www.w3.org/1999/xlink',
            'xsi': 'http://www.w3.org/2001/XMLSchema-instance',
        }

        ead = etree.Element('ead', nsmap=nsmap)
        ead.set(
            '{http://www.w3.org/2001/XMLSchema-instance}schemaLocation',
            'urn:isbn:1-931666-22-9 http://www.loc.gov/ead/ead.xsd'
        )

        # En-tête EAD
        eadheader = etree.SubElement(ead, 'eadheader')
        eadheader.set('langencoding', 'iso639-2b')
        eadheader.set('scriptencoding', 'iso15924')
        eadheader.set('dateencoding', 'iso8601')
        eadheader.set('countryencoding', 'iso3166-1')
        eadheader.set('repositoryencoding', 'iso15511')

        eadid = etree.SubElement(eadheader, 'eadid')
        eadid.set('countrycode', 'FR')
        eadid.text = config.get('eadid', f"FR-{datetime.now().strftime('%Y%m%d%H%M%S')}")

        filedesc = etree.SubElement(eadheader, 'filedesc')
        titlestmt = etree.SubElement(filedesc, 'titlestmt')
        titleproper = etree.SubElement(titlestmt, 'titleproper')
        titleproper.text = config.get('title', 'Instrument de recherche')

        publicationstmt = etree.SubElement(filedesc, 'publicationstmt')
        publisher = etree.SubElement(publicationstmt, 'publisher')
        publisher.text = config.get('organization', 'Administration')
        date_pub = etree.SubElement(publicationstmt, 'date')
        date_pub.text = timezone.now().strftime('%Y-%m-%d')

        # Corps de l'instrument (archdesc)
        archdesc = etree.SubElement(ead, 'archdesc')
        archdesc.set('level', config.get('level', 'fonds'))
        archdesc.set('type', 'inventory')

        did = etree.SubElement(archdesc, 'did')
        unitid = etree.SubElement(did, 'unitid')
        unitid.text = config.get('unitid', 'ARC')
        unittitle = etree.SubElement(did, 'unittitle')
        unittitle.text = config.get('title', 'Fonds archivé')

        # Dates extrêmes calculées depuis les documents
        if documents:
            min_date = min(d.get('created_at', '')[:10] for d in documents)
            max_date = max(d.get('created_at', '')[:10] for d in documents)
            unitdate = etree.SubElement(did, 'unitdate')
            unitdate.set('normal', f"{min_date}/{max_date}")
            unitdate.text = f"{min_date} / {max_date}"

        # Dossier (dsc) contenant les descriptions de pièces
        dsc = etree.SubElement(archdesc, 'dsc')
        dsc.set('type', 'combined')

        for doc in documents:
            c = etree.SubElement(dsc, 'c')
            c.set('level', 'item')
            c.set('id', f"doc_{doc['id'][:8]}")

            doc_did = etree.SubElement(c, 'did')

            doc_unitid = etree.SubElement(doc_did, 'unitid')
            doc_unitid.text = doc.get('reference', '')

            doc_title = etree.SubElement(doc_did, 'unittitle')
            doc_title.text = doc.get('title', '')

            if doc.get('created_at'):
                doc_date = etree.SubElement(doc_did, 'unitdate')
                doc_date.text = doc['created_at'][:10]

            if doc.get('file_size_bytes'):
                physdesc = etree.SubElement(doc_did, 'physdesc')
                extent = etree.SubElement(physdesc, 'extent')
                size_kb = round(doc['file_size_bytes'] / 1024, 2)
                extent.text = f"{size_kb} Ko"

            # Lien vers le fichier numérique si demandé
            if config.get('include_dao') and doc.get('file_path'):
                dao = etree.SubElement(c, 'dao')
                dao.set('{http://www.w3.org/1999/xlink}href', doc['file_path'])
                dao.set('{http://www.w3.org/1999/xlink}role', 'electronic-record-master')

        return etree.tostring(
            ead,
            xml_declaration=True,
            encoding='UTF-8',
            pretty_print=True
        )
```

---

## 13. RÈGLES MÉTIER CRITIQUES

### RB-018-01 — Clé API jamais stockée en clair

La clé API n'est affichée qu'une seule fois, immédiatement après sa création. Seul son hash SHA-256 est persisté. Si la clé est perdue, elle doit être régénérée. Cette règle est non contournable — même un Super Admin ne peut pas afficher une clé existante en clair.

---

### RB-018-02 — Scopes minimum requis (principe du moindre privilège)

Une clé API ne peut pas avoir plus de droits que l'utilisateur "propriétaire" auquel elle est associée. Si cet utilisateur est désactivé, toutes ses clés API sont automatiquement révoquées. Les scopes disponibles pour une clé sont limités aux droits de l'utilisateur propriétaire.

---

### RB-018-03 — Webhooks uniquement vers HTTPS

Sauf configuration explicite de l'administrateur pour les environnements de test, les webhooks n'acceptent que des URLs HTTPS. Les URLs HTTP sont rejetées avec un message explicatif. Cette règle protège contre l'interception des payloads en transit.

---

### RB-018-04 — Signature HMAC obligatoire sur tous les webhooks

Chaque payload webhook est signé avec HMAC-SHA256. Le système récepteur doit valider la signature avant de traiter le payload. Le secret de signature est unique par webhook et est renouvelable sans supprimer le webhook.

---

### RB-018-05 — Rate limiting incontournable par clé API

Chaque clé API a des limites par minute et par jour. Ces limites ne peuvent pas être mises à zéro (désactivées) sans accord d'un Super Admin. Le taux minimum est de 10 requêtes/minute. Les limites sont vérifiées via Redis avant tout accès à la base de données.

---

### RB-018-06 — Traçabilité complète de tous les appels API externes

Chaque appel API authentifié par clé API ou token OAuth2 génère un événement `API_CALL` dans le journal d'audit (MODULE 11). Cet événement contient : l'ID de la clé/token, l'endpoint appelé, la méthode HTTP, le code de réponse, le temps de traitement et l'IP source. Ces logs sont conservés 1 an.

---

### RB-018-07 — Documents SECRET exclus de l'API publique

Les documents avec niveau de confidentialité `SECRET` ne sont jamais retournés par l'API publique, même si la clé API possède le scope `documents:read`. Pour accéder à un document SECRET via API, il faut une clé avec le scope spécial `documents:secret:read`, réservé aux clés créées par un Super Admin uniquement.

---

### RB-018-08 — Import en masse : validation avant traitement

Tout import en masse passe obligatoirement par une phase de validation complète du manifeste avant de traiter un seul document. Si le manifeste contient plus de 20% d'erreurs de validation, l'import est automatiquement bloqué et l'utilisateur est invité à corriger le manifeste avant de relancer.

---

## 14. TÂCHES CELERY PLANIFIÉES

### TASK-018-01 : Livraison des webhooks avec retry exponentiel
```python
@shared_task(name='api.deliver_webhook', bind=True, max_retries=3)
# Déclenchement : Via signal Django (post événement)
# Retry delays   : 1 minute, 5 minutes, 30 minutes (délai exponentiel)
# Timeout        : 30 secondes par tentative
# Post-failure   : WebhookDelivery.status = FAILED, notification admin si > 10 échecs consécutifs
```

### TASK-018-02 : Import automatique depuis connecteurs (auto_import_enabled)
```python
@shared_task(name='api.auto_import_connectors')
# Fréquence      : Toutes les 15 minutes (vérifie chaque connecteur auto-import)
# Description    : Pour chaque ExternalConnector actif avec auto_import_enabled=True,
#                  vérifie si next_run est dépassé, déclenche l'import
# Cursor         : Utilise auto_import_cursor pour reprendre sans doublons
```

### TASK-018-03 : Génération des exports en masse
```python
@shared_task(name='api.generate_bulk_export', bind=True, max_retries=1)
# Déclenchement : À la demande via POST /bulk-export/
# Timeout        : 2 heures
# Formats        : ZIP (fichiers + manifeste CSV), EAD XML, CSV métadonnées seules
```

### TASK-018-04 : Traitement des imports en masse
```python
@shared_task(name='api.process_bulk_import', bind=True, max_retries=0)
# Déclenchement : À la demande via POST /bulk-import/
# Timeout        : 4 heures
# Traitement     : Par lot de 50 documents pour éviter les timeouts
# Rapport        : BulkImportJob.error_report mis à jour en temps réel
```

### TASK-018-05 : Nettoyage des tokens OAuth2 expirés
```python
@shared_task(name='api.cleanup_expired_tokens')
# Fréquence      : Quotidien à 4h00
# Description    : Supprime les access_token et refresh_token expirés
```

### TASK-018-06 : Rotation des statistiques d'usage API (par jour)
```python
@shared_task(name='api.rotate_api_stats')
# Fréquence      : Quotidien à 0h01
# Description    : Snapshot les statistiques d'usage de la veille pour historique
#                  Remet à zéro requests_today sur chaque ApiKey
```

### TASK-018-07 : Vérification de la validité des connecteurs
```python
@shared_task(name='api.verify_connectors_health')
# Fréquence      : Quotidien à 5h00
# Description    : Teste la connectivité de chaque connecteur actif
#                  Met à jour last_test_status
# Alerte         : Si connecteur auto_import devient inaccessible → notification Admin
```

---

## 15. ÉVÉNEMENTS JOURNALISÉS (MODULE 11)

| Code événement | Description | Sévérité | Données clés |
|----------------|-------------|----------|--------------|
| `API_KEY_CREATED` | Clé API créée | INFO | key_id, name, scopes, created_by_id |
| `API_KEY_REVOKED` | Clé API révoquée | WARNING | key_id, revoked_by_id, reason |
| `API_KEY_USED` | Appel API authentifié par clé | INFO | key_id, endpoint, method, response_code, duration_ms |
| `API_KEY_RATE_LIMITED` | Limite de débit atteinte | WARNING | key_id, endpoint, limit_type |
| `API_UNAUTHORIZED_SCOPE` | Scope insuffisant pour l'action | WARNING | key_id, required_scope, endpoint |
| `OAUTH_CLIENT_CREATED` | Client OAuth2 créé | INFO | client_id, name, created_by_id |
| `OAUTH_AUTHORIZED` | Utilisateur a autorisé un client OAuth2 | INFO | client_id, user_id, scopes |
| `OAUTH_TOKEN_REVOKED` | Token OAuth2 révoqué | INFO | client_id, user_id |
| `WEBHOOK_CREATED` | Webhook créé | INFO | webhook_id, url, events |
| `WEBHOOK_DELIVERY_SUCCESS` | Livraison webhook réussie | INFO | webhook_id, event_type, delivery_id, attempt |
| `WEBHOOK_DELIVERY_FAILED` | Livraison webhook échouée (après retries) | WARNING | webhook_id, event_type, error, attempts |
| `CONNECTOR_CREATED` | Connecteur externe créé | INFO | connector_id, type, created_by_id |
| `CONNECTOR_IMPORT_STARTED` | Import depuis connecteur démarré | INFO | connector_id, job_id |
| `CONNECTOR_IMPORT_COMPLETED` | Import depuis connecteur terminé | INFO | job_id, success_count, error_count |
| `BULK_IMPORT_STARTED` | Import en masse démarré | INFO | job_id, source_type, total_items |
| `BULK_IMPORT_COMPLETED` | Import en masse terminé | INFO | job_id, success_count, error_count |
| `BULK_EXPORT_GENERATED` | Export en masse généré | INFO | job_id, format, document_count |
| `SECRET_DOCUMENT_API_BLOCKED` | Tentative d'accès API à document SECRET | WARNING | document_id, key_id, endpoint |

---

## 16. NOTIFICATIONS GÉNÉRÉES (MODULE 12)

| Déclencheur | Destinataire(s) | Canal | Priorité | Template |
|-------------|----------------|-------|----------|----------|
| Clé API expirée (J-7) | Propriétaire + Admin | In-app + Email | HAUTE | `api_key_expiring` |
| Clé API révoquée | Propriétaire | Email | HAUTE | `api_key_revoked` |
| Webhook en échec répété (> 10 échecs) | Admin | In-app + Email | HAUTE | `webhook_repeated_failures` |
| Import en masse terminé | Déclencheur | In-app + Email | NORMALE | `bulk_import_completed` |
| Import en masse en échec partiel | Déclencheur | In-app + Email | HAUTE | `bulk_import_partial` |
| Export en masse disponible | Déclencheur | In-app | NORMALE | `bulk_export_ready` |
| Connecteur externe inaccessible | Admin | In-app + Email | HAUTE | `connector_unavailable` |

---

## 17. DÉPENDANCES PYTHON SUPPLÉMENTAIRES

```
# requirements.txt — ajouts spécifiques à ce module

# Documentation OpenAPI
drf-spectacular==0.27.*

# OAuth2 / OpenID Connect
django-oauth-toolkit==2.4.*

# Génération EAD XML
lxml==5.*

# HTTP client pour les connecteurs et webhooks
httpx==0.27.*

# Google Drive API
google-api-python-client==2.*
google-auth==2.*

# Microsoft Graph (OneDrive, SharePoint, Teams)
msal==1.*

# WebDAV (Nextcloud)
webdavclient3==3.*

# FTP / SFTP
paramiko==3.*     # SFTP

# Déjà présents dans la stack
# celery, redis, requests, django-rest-framework
```

---

## 18. STRUCTURE DES FICHIERS

```
apps/
└── api/
    ├── __init__.py
    ├── admin.py
    ├── apps.py
    ├── models/
    │   ├── __init__.py
    │   ├── api_key.py
    │   ├── oauth_client.py
    │   ├── webhook.py
    │   ├── webhook_delivery.py
    │   ├── external_connector.py
    │   └── bulk_import_job.py
    ├── serializers/
    │   ├── __init__.py
    │   ├── api_key_serializers.py
    │   ├── oauth_serializers.py
    │   ├── webhook_serializers.py
    │   └── connector_serializers.py
    ├── views/
    │   ├── __init__.py
    │   ├── api_key_views.py
    │   ├── oauth_views.py
    │   ├── webhook_views.py
    │   ├── connector_views.py
    │   ├── bulk_import_views.py
    │   └── bulk_export_views.py
    ├── authentication.py              # ApiKeyAuthentication, OAuthAuthentication
    ├── permissions.py                 # HasApiScope, ApiKeyIsActive
    ├── middleware/
    │   ├── __init__.py
    │   └── rate_limiting.py
    ├── urls.py
    ├── tasks.py
    ├── signals.py                     # Émission des événements webhook
    ├── services/
    │   ├── __init__.py
    │   ├── webhook_service.py
    │   ├── ead_export_service.py
    │   ├── bulk_import_service.py
    │   └── connectors/
    │       ├── __init__.py
    │       ├── base_connector.py
    │       ├── google_drive_connector.py
    │       ├── onedrive_connector.py
    │       ├── nextcloud_connector.py
    │       ├── ftp_connector.py
    │       ├── sftp_connector.py
    │       ├── webdav_connector.py
    │       ├── slack_connector.py
    │       └── teams_connector.py
    └── tests/
        ├── __init__.py
        ├── test_api_key_auth.py
        ├── test_rate_limiting.py
        ├── test_webhooks.py
        ├── test_connectors.py
        ├── test_bulk_import.py
        └── test_ead_export.py

docs/
└── api/
    └── MODULE_18_API_Interoperabilite_Integrations.md
```

---

## ✅ RÉCAPITULATIF MODULE 18

| Critère | Valeur |
|---------|--------|
| Cas d'utilisation | 25 UC |
| Modèles de données | 6 tables |
| Endpoints API | 40 endpoints |
| Événements webhook disponibles | 19 événements |
| Connecteurs externes | 9 connecteurs (GDrive, OneDrive, SharePoint, Nextcloud, WebDAV, FTP, SFTP, Slack, Teams) |
| Tâches Celery planifiées | 7 tâches |
| Événements journalisés (MODULE 11) | 18 événements |
| Notifications générées (MODULE 12) | 7 types |
| Scopes API disponibles | 17 scopes |
| Format d'export XML | EAD 2002 + EAD3 |

---

## 🏁 RÉCAPITULATIF GÉNÉRAL DU PROJET — 18 MODULES COMPLÉTÉS

| Module | Titre | UC | Tables | Endpoints | Statut |
|--------|-------|:--:|:------:|:---------:|--------|
| 01 | Architecture & Conventions | — | — | — | ✅ |
| 02 | Authentification & Gestion des utilisateurs | 24 | 10 | 30+ | ✅ |
| 03 | Classification & Taxonomie | 24 | 7 | 28+ | ✅ |
| 04 | Cycle de vie documentaire | 24 | 8 | 26+ | ✅ |
| 05 | Confidentialité & RGPD | 20 | 6 | 22+ | ✅ |
| 06 | Gestion des documents & Versioning | 53 | 8 | 45+ | ✅ |
| 07 | OCR & Numérisation | 18 | 4 | 20+ | ✅ |
| 08 | Workflows & Validation | 22 | 5 | 28+ | ✅ |
| 09 | Recherche & Indexation | 29 | 4 | 18+ | ✅ |
| 10 | Intelligence Artificielle | 20 | 6 | 22+ | ✅ |
| 11 | Journal d'audit | 20 | 4 | 15+ | ✅ |
| 12 | Notifications multi-canal | 25 | 7 | 20+ | ✅ |
| 13 | Accès & Partage contrôlé | 28 | 7 | 34 | ✅ |
| 14 | Tableaux de bord & Reporting | 26 | 4 | 40 | ✅ |
| 15 | Administration & Configuration | 25 | 5 | 38 | ✅ |
| 16 | Sauvegarde & Résilience | 20 | 4 | 25 | ✅ |
| 17 | Desktop Tauri & Synchronisation offline | 22 | 2+7SQLite | 16 | ✅ |
| 18 | API publique & Intégrations | 25 | 6 | 40 | ✅ |
| **TOTAL** | | **~425 UC** | **~110 tables** | **~500 endpoints** | **✅** |

---

*Système d'Archivage Numérique pour Administration Publique — Spécification technique complète*  
*18 modules — Architecture Django REST Framework + Tauri + PostgreSQL + Redis + Celery*  
*Conforme : normes archivistiques (SEDA, EAD), OHADA, stratégie 3-2-1*
