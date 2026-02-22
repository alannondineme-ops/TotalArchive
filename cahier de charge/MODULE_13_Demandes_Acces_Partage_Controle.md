# MODULE 13 — Demandes d'accès & Partage contrôlé

**Système d'Archivage Numérique — Administration Publique**  
**Version :** 1.0  
**Statut :** Spécification technique complète  
**Dépendances :** MODULE 02 (Auth), MODULE 05 (Confidentialité), MODULE 06 (Documents), MODULE 11 (Audit), MODULE 12 (Notifications)

---

## 1. PRÉSENTATION DU MODULE

### 1.1 Contexte et justification

Dans un système d'archivage pour administration publique, les documents ne sont pas tous accessibles à tous les agents. La gestion des accès et du partage constitue un enjeu critique à deux niveaux :

1. **Niveau interne :** Un agent peut avoir besoin d'accéder à un document classifié au-delà de ses permissions habituelles (document Confidentiel ou Secret). Il doit alors passer par un circuit de demande formelle, traçable et auditable.

2. **Niveau externe :** Des partenaires institutionnels, des citoyens, des prestataires ou des autorités de contrôle peuvent légitimement avoir besoin de consulter ou de recevoir des documents spécifiques, sans pour autant disposer d'un compte dans le système.

Ce module adresse ces deux besoins avec une approche sécurité-first : **aucun accès n'est accordé par défaut**, tout partage génère une trace d'audit complète, et tout document partagé peut être révoqué instantanément.

### 1.2 Périmètre fonctionnel

Ce module couvre :
- Le circuit complet de demande et d'approbation d'accès
- La génération et la gestion de liens de partage sécurisés
- Le partage avec utilisateurs externes (sans compte système)
- La signature numérique optionnelle des documents partagés (PyHanko)
- Le watermarking automatique des documents transmis
- La traçabilité exhaustive de chaque consultation et téléchargement
- La révocation immédiate de tout accès ou partage

### 1.3 Principes de sécurité fondamentaux

- **Principe du moindre privilège :** Chaque partage accorde uniquement les droits strictement nécessaires
- **Durée limitée obligatoire :** Tout accès temporaire a une date d'expiration (max 90 jours)
- **Traçabilité absolue :** Chaque action sur un partage génère un événement d'audit immuable (MODULE 11)
- **Révocabilité immédiate :** Tout partage peut être annulé sans préavis
- **Vérification confidentialité :** Un document Secret ne peut jamais être partagé avec un utilisateur externe
- **Watermarking systématique :** Tout document téléchargé via partage porte une marque d'identification

---

## 2. CAS D'UTILISATION — VUE D'ENSEMBLE

| Code | Cas d'utilisation | Acteur principal | Priorité |
|------|-------------------|-----------------|----------|
| UC-013-01 | Soumettre une demande d'accès à un document | Agent, Responsable | HAUTE |
| UC-013-02 | Consulter ses demandes d'accès en cours | Agent, Responsable | HAUTE |
| UC-013-03 | Approuver une demande d'accès | Responsable, Admin | HAUTE |
| UC-013-04 | Rejeter une demande d'accès avec motif | Responsable, Admin | HAUTE |
| UC-013-05 | Révoquer un accès accordé | Responsable, Admin | HAUTE |
| UC-013-06 | Créer un lien de partage temporaire | Archiviste, Responsable | HAUTE |
| UC-013-07 | Protéger un lien par mot de passe | Archiviste, Responsable | MOYENNE |
| UC-013-08 | Définir les permissions d'un lien (lecture, téléchargement, commentaire) | Archiviste, Responsable | HAUTE |
| UC-013-09 | Fixer la date d'expiration d'un partage | Archiviste, Responsable | HAUTE |
| UC-013-10 | Fixer le nombre max d'utilisations d'un lien | Archiviste, Responsable | MOYENNE |
| UC-013-11 | Partager un document avec un utilisateur externe | Archiviste, Responsable | HAUTE |
| UC-013-12 | Consulter un document via lien de partage (utilisateur externe) | Utilisateur externe | HAUTE |
| UC-013-13 | Télécharger un document via lien de partage | Utilisateur externe, Agent | HAUTE |
| UC-013-14 | Révoquer un lien de partage actif | Archiviste, Responsable, Admin | HAUTE |
| UC-013-15 | Partager un dossier complet | Responsable, Admin | MOYENNE |
| UC-013-16 | Consulter qui a accès à un document | Archiviste, Responsable, Admin | HAUTE |
| UC-013-17 | Consulter l'activité d'un lien de partage | Archiviste, Responsable | HAUTE |
| UC-013-18 | Signer numériquement un document PDF (PyHanko) | Responsable, Admin | MOYENNE |
| UC-013-19 | Vérifier une signature numérique | Tout utilisateur authentifié | MOYENNE |
| UC-013-20 | Configurer le watermark par défaut | Super Admin, Admin | HAUTE |
| UC-013-21 | Générer un document avec watermark personnalisé | Archiviste, Responsable | HAUTE |
| UC-013-22 | Envoyer un document par email depuis le système | Archiviste, Responsable | MOYENNE |
| UC-013-23 | Consulter le tableau de bord des partages actifs | Archiviste, Responsable, Admin | HAUTE |
| UC-013-24 | Exporter la liste des accès accordés pour audit | Admin, Auditeur | HAUTE |
| UC-013-25 | Configurer les règles d'approbation par niveau de confidentialité | Super Admin, Admin | HAUTE |
| UC-013-26 | Prolonger la durée d'un accès ou partage | Responsable, Admin | MOYENNE |
| UC-013-27 | Notifier le demandeur de la décision | Système (automatique) | HAUTE |
| UC-013-28 | Archiver automatiquement les partages expirés | Système (Celery) | HAUTE |

---

## 3. CAS D'UTILISATION DÉTAILLÉS

### UC-013-01 — Soumettre une demande d'accès à un document

**Acteur principal :** Agent, Responsable
**Pré-conditions :**
- L'utilisateur est authentifié
- Le document existe et est dans un état accessible (non supprimé, non archivé définitivement)
- L'utilisateur n'a pas déjà une demande en cours (`PENDING`) pour ce document
- Le niveau de confidentialité du document dépasse les droits habituels de l'utilisateur

**Flux principal :**
1. L'utilisateur accède à la fiche d'un document dont il ne peut voir que le titre/résumé
2. Il clique sur "Demander l'accès"
3. Le système affiche le formulaire de demande
4. L'utilisateur renseigne : motif de la demande (obligatoire, 50-500 caractères), durée souhaitée (1-30 jours), type d'accès demandé (lecture seule / lecture + téléchargement)
5. L'utilisateur soumet la demande
6. Le système crée un enregistrement `AccessRequest` avec statut `PENDING` et une référence lisible (`AR-2026-00001`)
7. Le système identifie les approbateurs selon les règles configurées (UC-013-25)
8. Les approbateurs reçoivent une notification (MODULE 12)
9. Un événement d'audit est généré (MODULE 11)
10. L'utilisateur reçoit une confirmation avec le numéro de demande

**Flux alternatifs :**
- **4a.** Le motif est trop court (< 50 caractères) → message d'erreur de validation
- **7a.** Aucun approbateur configuré → Super Admin est notifié en fallback
- **7b.** Le document est `SECRET` → seul l'Admin ou Super Admin peut approuver

**Post-conditions :**
- Demande enregistrée avec statut `PENDING`
- Approbateurs notifiés
- Événement `ACCESS_REQUEST_SUBMITTED` journalisé

---

### UC-013-03 — Approuver une demande d'accès

**Acteur principal :** Responsable, Admin
**Pré-conditions :**
- L'approbateur est authentifié avec les droits appropriés
- La demande est dans l'état `PENDING`
- La demande n'est pas expirée (délai max d'approbation configurable, défaut 5 jours ouvrés)

**Flux principal :**
1. L'approbateur reçoit la notification ou consulte la liste des demandes en attente
2. Il consulte le détail : qui demande, quel document, quel motif, quelle durée
3. Il peut consulter le document complet pour évaluer la pertinence
4. Il choisit d'approuver avec les paramètres définitifs :
   - Durée d'accès accordée (peut différer de la durée demandée, max 90 jours)
   - Permissions exactes (lecture seule ou lecture + téléchargement)
   - Conditions particulières (note interne optionnelle)
5. Il confirme l'approbation
6. Le système crée un enregistrement `DocumentAccess` lié à l'utilisateur et au document
7. La demande passe au statut `APPROVED`
8. Le demandeur reçoit une notification avec la durée et les permissions accordées (MODULE 12)
9. Un événement d'audit est généré (MODULE 11)

**Flux alternatifs :**
- **4a.** La durée demandée dépasse 90 jours → le système la plafonne automatiquement à 90 jours et avertit l'approbateur
- **4b.** Le document est `SECRET` et l'approbateur est Responsable → le système bloque et oriente vers Admin/Super Admin

**Post-conditions :**
- `AccessRequest.status = 'APPROVED'`
- `DocumentAccess` créé avec `expires_at` défini
- Demandeur notifié
- Événement `ACCESS_REQUEST_APPROVED` journalisé

---

### UC-013-06 — Créer un lien de partage temporaire

**Acteur principal :** Archiviste, Responsable
**Pré-conditions :**
- L'utilisateur est authentifié avec les droits de partage
- Le document est dans un état publiable (statut `ACTIVE` ou `VALIDATED`)
- Le niveau de confidentialité du document est compatible avec le partage

**Flux principal :**
1. L'utilisateur ouvre la fiche du document et clique sur "Créer un lien de partage"
2. Le système affiche le formulaire de configuration :
   - Destination : interne ou externe (email)
   - Permissions : lecture seule / lecture + téléchargement / lecture + commentaire
   - Date d'expiration (obligatoire, max 90 jours)
   - Nombre max d'utilisations (optionnel, 0 = illimité)
   - Protection par mot de passe (optionnelle)
   - Message personnalisé pour le destinataire (optionnel)
   - Watermark personnalisé (ou utiliser le défaut)
3. L'utilisateur valide
4. Le système génère un token unique : `UUID v4 (sans tirets)` + `SHA-256(uuid + document_id + created_at + SECRET_KEY)[:32]`
5. Le système crée un enregistrement `ShareLink` avec tous les paramètres
6. Le lien complet est formé : `https://[domaine]/share/[token]`
7. Le lien est copié dans le presse-papier et affiché à l'écran
8. Un événement d'audit est généré (MODULE 11)

**Flux alternatifs :**
- **2a.** Le document est `CONFIDENTIEL` → un avertissement s'affiche, confirmation explicite requise
- **2b.** Le document est `SECRET` → le partage externe est bloqué, seul le partage interne est autorisé (Admin uniquement)
- **4a.** Collision de token (probabilité infime) → régénération automatique jusqu'à unicité

**Post-conditions :**
- `ShareLink` créé avec statut `ACTIVE`
- Lien disponible pour distribution
- Événement `SHARE_LINK_CREATED` journalisé

---

### UC-013-12 — Consulter un document via lien de partage (utilisateur externe)

**Acteur principal :** Utilisateur externe (non authentifié dans le système)
**Pré-conditions :**
- Le lien est valide (token correct)
- Le lien n'est pas expiré
- Le nombre max d'utilisations n'est pas atteint

**Flux principal :**
1. L'utilisateur externe clique sur le lien reçu par email
2. Le système vérifie la validité du token (format, existence en base)
3. Le système vérifie les conditions d'accès (expiration, quota utilisations)
4. Si le lien est protégé par mot de passe : affichage d'une page de saisie
5. L'utilisateur saisit le mot de passe (vérifié avec comparaison Argon2 côté serveur)
6. Le système affiche le document en visionneuse sécurisée (pas de sélection texte si lecture seule)
7. Le système crée un enregistrement `ShareActivity` (IP anonymisée, user-agent hashé, timestamp)
8. Le compteur d'utilisations du lien est incrémenté
9. Un événement d'audit est généré (MODULE 11)

**Flux alternatifs :**
- **2a.** Token invalide → page d'erreur générique "Lien invalide ou expiré" (ne pas révéler si le document existe)
- **3a.** Lien expiré → page d'erreur "Ce lien a expiré. Contactez l'émetteur."
- **3b.** Quota atteint → page d'erreur "Ce lien a atteint son nombre maximum d'utilisations."
- **5a.** Mot de passe incorrect → 3 tentatives max, puis blocage temporaire 15 minutes
- **6a.** Permissions sans téléchargement → bouton de téléchargement absent

**Post-conditions :**
- `ShareActivity` créé
- Compteur d'utilisations incrémenté
- Événement `SHARE_LINK_ACCESSED` journalisé

---

### UC-013-18 — Signer numériquement un document (PyHanko)

**Acteur principal :** Responsable, Admin
**Pré-conditions :**
- L'utilisateur dispose d'un certificat numérique configuré dans le système
- Le document est au format PDF
- Le document est dans un état finalisé (statut `VALIDATED`)

**Flux principal :**
1. L'utilisateur ouvre la fiche du document et clique sur "Signer numériquement"
2. Le système affiche les options de signature :
   - Position dans le document (page, coordonnées ou pied de page automatique)
   - Raison de la signature (champ libre, ex: "Approbation officielle")
   - Localisation (optionnel)
   - Certificat à utiliser (si plusieurs disponibles)
3. L'utilisateur confirme avec son mot de passe ou code TOTP
4. Le système récupère le fichier PDF depuis le stockage
5. PyHanko applique la signature numérique PAdES-T avec le certificat
6. Le fichier PDF signé est enregistré comme nouvelle version mineure (MODULE 06 versionning)
7. Un enregistrement `DigitalSignature` est créé avec le hash SHA-256 du fichier signé
8. L'utilisateur est notifié de la réussite
9. Un événement d'audit est généré

**Flux alternatifs :**
- **2a.** Aucun certificat configuré → message "Contactez l'administrateur pour configurer votre certificat"
- **4a.** Le document n'est pas un PDF → erreur "La signature numérique est disponible uniquement pour les fichiers PDF"
- **5a.** Erreur PyHanko → message d'erreur, tentative loguée, rollback (version précédente inchangée)

**Post-conditions :**
- Nouvelle version du PDF avec signature intégrée créée
- `DigitalSignature` enregistré avec hash SHA-256 du fichier signé
- Événement `DOCUMENT_SIGNED` journalisé

---

## 4. MODÈLES DE DONNÉES

### 4.1 AccessRequest — Demandes d'accès

```python
class AccessRequest(BaseModel):
    """
    Demande formelle d'accès à un document confidentiel.
    Circuit : PENDING → APPROVED | REJECTED | CANCELLED | EXPIRED
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    reference = models.CharField(
        max_length=20, unique=True,
        help_text="Référence lisible : AR-2026-00001"
    )

    # Relations
    document = models.ForeignKey(
        'documents.Document', on_delete=models.PROTECT,
        related_name='access_requests'
    )
    requester = models.ForeignKey(
        'users.User', on_delete=models.PROTECT,
        related_name='submitted_access_requests'
    )
    approved_by = models.ForeignKey(
        'users.User', on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='approved_access_requests'
    )

    # Détails de la demande
    justification = models.TextField(
        help_text="Motif obligatoire, 50-500 caractères"
    )
    requested_permission = models.CharField(
        max_length=30,
        choices=[
            ('READ_ONLY', 'Lecture seule'),
            ('READ_DOWNLOAD', 'Lecture et téléchargement'),
        ],
        default='READ_ONLY'
    )
    requested_duration_days = models.PositiveSmallIntegerField(
        help_text="Durée souhaitée en jours (1-30)"
    )

    # Statut et décision
    status = models.CharField(
        max_length=20,
        choices=[
            ('PENDING', 'En attente'),
            ('APPROVED', 'Approuvée'),
            ('REJECTED', 'Rejetée'),
            ('CANCELLED', 'Annulée'),
            ('EXPIRED', 'Expirée sans décision'),
        ],
        default='PENDING', db_index=True
    )
    decision_note = models.TextField(blank=True)
    decided_at = models.DateTimeField(null=True, blank=True)

    # Durée et permissions accordées (peuvent différer de ce qui a été demandé)
    granted_duration_days = models.PositiveSmallIntegerField(
        null=True, blank=True,
        help_text="Durée réellement accordée (max 90 jours)"
    )
    granted_permission = models.CharField(
        max_length=30,
        choices=[
            ('READ_ONLY', 'Lecture seule'),
            ('READ_DOWNLOAD', 'Lecture et téléchargement'),
        ],
        null=True, blank=True
    )

    # Deadline d'approbation
    approval_deadline = models.DateTimeField(
        help_text="Date limite pour décider (défaut +5 jours ouvrés)"
    )

    # Horodatage standard
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True, on_delete=models.SET_NULL,
        related_name='deleted_access_requests'
    )

    class Meta:
        db_table = 'access_requests'
        indexes = [
            models.Index(fields=['status', 'approval_deadline']),
            models.Index(fields=['requester', 'status']),
            models.Index(fields=['document', 'status']),
        ]
        constraints = [
            models.CheckConstraint(
                check=models.Q(requested_duration_days__gte=1) & models.Q(requested_duration_days__lte=30),
                name='access_request_duration_range'
            ),
            models.CheckConstraint(
                check=models.Q(granted_duration_days__lte=90),
                name='access_request_max_granted_duration'
            ),
            models.UniqueConstraint(
                fields=['document', 'requester'],
                condition=models.Q(status='PENDING'),
                name='unique_pending_request_per_doc_user'
            ),
        ]
```

---

### 4.2 DocumentAccess — Accès accordés

```python
class DocumentAccess(BaseModel):
    """
    Accès temporaire accordé à un utilisateur sur un document spécifique.
    Créé lors de l'approbation d'une AccessRequest ou directement par un admin.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    document = models.ForeignKey(
        'documents.Document', on_delete=models.CASCADE,
        related_name='granted_accesses'
    )
    user = models.ForeignKey(
        'users.User', on_delete=models.CASCADE,
        related_name='document_accesses'
    )
    access_request = models.OneToOneField(
        AccessRequest, on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='resulting_access',
        help_text="La demande qui a généré cet accès (null si accordé directement)"
    )
    granted_by = models.ForeignKey(
        'users.User', on_delete=models.PROTECT,
        related_name='accesses_granted'
    )

    # Permissions
    can_read = models.BooleanField(default=True)
    can_download = models.BooleanField(default=False)
    can_comment = models.BooleanField(default=False)

    # Validité
    granted_at = models.DateTimeField(auto_now_add=True)
    expires_at = models.DateTimeField(
        help_text="Date d'expiration obligatoire (max 90 jours)"
    )
    is_active = models.BooleanField(default=True, db_index=True)
    revoked_at = models.DateTimeField(null=True, blank=True)
    revoked_by = models.ForeignKey(
        'users.User', on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='accesses_revoked'
    )
    revocation_reason = models.TextField(blank=True)

    # Statistiques d'utilisation
    access_count = models.PositiveIntegerField(default=0)
    last_accessed_at = models.DateTimeField(null=True, blank=True)

    # Horodatage
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True, on_delete=models.SET_NULL,
        related_name='deleted_document_accesses'
    )

    class Meta:
        db_table = 'document_accesses'
        indexes = [
            models.Index(fields=['user', 'is_active', 'expires_at']),
            models.Index(fields=['document', 'is_active']),
            models.Index(fields=['expires_at', 'is_active']),
        ]
        constraints = [
            models.UniqueConstraint(
                fields=['document', 'user'],
                condition=models.Q(is_active=True),
                name='unique_active_access_per_doc_user'
            ),
        ]
```

---

### 4.3 ShareLink — Liens de partage

```python
class ShareLink(BaseModel):
    """
    Lien de partage sécurisé, utilisable par des utilisateurs internes ou externes.
    Token = UUID v4 (sans tirets, 32 chars) + SHA-256[:32] = 64 chars URL-safe.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    token = models.CharField(
        max_length=128, unique=True, db_index=True,
        help_text="Token URL-safe 64 chars : uuid4_clean + sha256(uuid+doc+created+SECRET)[:32]"
    )

    # Document ou dossier partagé (l'un ou l'autre, pas les deux)
    document = models.ForeignKey(
        'documents.Document', on_delete=models.CASCADE,
        null=True, blank=True, related_name='share_links'
    )
    folder = models.ForeignKey(
        'documents.Folder', on_delete=models.CASCADE,
        null=True, blank=True, related_name='share_links',
        help_text="Si renseigné, tous les documents du dossier sont accessibles"
    )

    created_by = models.ForeignKey(
        'users.User', on_delete=models.PROTECT,
        related_name='created_share_links'
    )

    # Type de partage
    share_type = models.CharField(
        max_length=20,
        choices=[
            ('INTERNAL', 'Utilisateurs internes uniquement'),
            ('EXTERNAL', 'Utilisateurs externes (email)'),
            ('PUBLIC', 'Accessible sans authentification'),
        ],
        default='INTERNAL'
    )

    # Destinataires (pour type EXTERNAL)
    recipient_email = models.EmailField(blank=True)
    recipient_name = models.CharField(max_length=200, blank=True)
    recipient_organization = models.CharField(max_length=200, blank=True)

    # Permissions
    can_read = models.BooleanField(default=True)
    can_download = models.BooleanField(default=False)
    can_comment = models.BooleanField(default=False)

    # Protection par mot de passe
    password_hash = models.CharField(
        max_length=200, blank=True,
        help_text="Hash Argon2 du mot de passe (vide = pas de protection)"
    )
    password_attempts = models.PositiveSmallIntegerField(default=0)
    password_locked_until = models.DateTimeField(null=True, blank=True)

    # Validité et quotas
    expires_at = models.DateTimeField(
        help_text="Expiration obligatoire (max 90 jours)"
    )
    max_uses = models.PositiveIntegerField(
        default=0, help_text="0 = illimité"
    )
    use_count = models.PositiveIntegerField(default=0)

    # Statut
    status = models.CharField(
        max_length=20,
        choices=[
            ('ACTIVE', 'Actif'),
            ('REVOKED', 'Révoqué'),
            ('EXPIRED', 'Expiré'),
            ('EXHAUSTED', 'Quota atteint'),
        ],
        default='ACTIVE', db_index=True
    )
    revoked_at = models.DateTimeField(null=True, blank=True)
    revoked_by = models.ForeignKey(
        'users.User', on_delete=models.SET_NULL,
        null=True, blank=True, related_name='revoked_share_links'
    )
    revocation_reason = models.TextField(blank=True)

    # Watermark
    watermark_config = models.JSONField(
        default=dict,
        help_text="Config watermark : texte, position, opacité, couleur"
    )
    apply_watermark = models.BooleanField(
        default=True,
        help_text="Si False, pas de watermark (réservé Super Admin uniquement)"
    )

    # Message personnalisé affiché à l'ouverture
    custom_message = models.TextField(blank=True)

    # Horodatage
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True, on_delete=models.SET_NULL,
        related_name='deleted_share_links'
    )

    class Meta:
        db_table = 'share_links'
        indexes = [
            models.Index(fields=['token']),
            models.Index(fields=['status', 'expires_at']),
            models.Index(fields=['created_by', 'status']),
            models.Index(fields=['recipient_email']),
        ]
        constraints = [
            models.CheckConstraint(
                check=(
                    models.Q(document__isnull=False, folder__isnull=True) |
                    models.Q(document__isnull=True, folder__isnull=False)
                ),
                name='share_link_document_xor_folder'
            ),
        ]
```

---

### 4.4 ShareActivity — Traçabilité des accès via liens

```python
class ShareActivity(BaseModel):
    """
    Enregistre chaque consultation ou téléchargement via un lien de partage.
    Table append-only (jamais de UPDATE, sauf soft delete pour raisons légales RGPD).
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    share_link = models.ForeignKey(
        ShareLink, on_delete=models.PROTECT,
        related_name='activities'
    )
    document = models.ForeignKey(
        'documents.Document', on_delete=models.PROTECT,
        related_name='share_activities',
        help_text="Utile si partage de dossier (plusieurs documents)"
    )

    # Utilisateur système (null si utilisateur externe non authentifié)
    user = models.ForeignKey(
        'users.User', on_delete=models.SET_NULL,
        null=True, blank=True, related_name='share_activities'
    )

    action = models.CharField(
        max_length=30,
        choices=[
            ('VIEWED', 'Document consulté'),
            ('DOWNLOADED', 'Document téléchargé'),
            ('WATERMARKED_DOWNLOAD', 'Téléchargement avec watermark'),
            ('COMMENTED', 'Commentaire ajouté'),
            ('PASSWORD_FAILED', 'Tentative mot de passe échouée'),
        ]
    )

    # Informations réseau anonymisées (conformité RGPD - recommandations CNIL)
    ip_address_anonymized = models.GenericIPAddressField(
        null=True,
        help_text="IP tronquée : dernier octet IPv4 (ou 64 derniers bits IPv6) remplacés par 0"
    )
    user_agent_hash = models.CharField(
        max_length=64, blank=True,
        help_text="Hash SHA-256 du user-agent (corrélation sans stockage brut)"
    )
    country_code = models.CharField(
        max_length=2, blank=True,
        help_text="Code pays ISO 3166-1 déduit de l'IP (optionnel)"
    )

    # Horodatage immuable (pas de updated_at sur cette table)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)

    # Soft delete uniquement pour raisons légales (demande RGPD)
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True, on_delete=models.SET_NULL,
        related_name='deleted_share_activities'
    )

    class Meta:
        db_table = 'share_activities'
        indexes = [
            models.Index(fields=['share_link', 'created_at']),
            models.Index(fields=['document', 'action']),
            models.Index(fields=['created_at']),
        ]
```

---

### 4.5 DigitalSignature — Signatures numériques

```python
class DigitalSignature(BaseModel):
    """
    Enregistre les signatures numériques apposées sur des documents PDF via PyHanko.
    Permet la vérification ultérieure de validité (certificat, intégrité du document).
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    document = models.ForeignKey(
        'documents.Document', on_delete=models.PROTECT,
        related_name='digital_signatures'
    )
    document_version = models.ForeignKey(
        'documents.DocumentVersion', on_delete=models.PROTECT,
        related_name='digital_signatures',
        help_text="La version exacte du document qui a été signée"
    )
    signer = models.ForeignKey(
        'users.User', on_delete=models.PROTECT,
        related_name='digital_signatures'
    )

    # Informations du certificat utilisé
    certificate_subject = models.CharField(max_length=500)
    certificate_issuer = models.CharField(max_length=500)
    certificate_serial = models.CharField(max_length=100)
    certificate_valid_from = models.DateTimeField()
    certificate_valid_to = models.DateTimeField()

    # Type et métadonnées de la signature
    signature_type = models.CharField(
        max_length=20,
        choices=[
            ('PADES_B', 'PAdES-B (de base)'),
            ('PADES_T', 'PAdES-T (avec timestamp)'),
            ('PADES_LT', 'PAdES-LT (avec révocation)'),
            ('PADES_LTA', 'PAdES-LTA (archivage long terme)'),
        ],
        default='PADES_T'
    )
    signature_reason = models.CharField(max_length=300, blank=True)
    signature_location = models.CharField(max_length=200, blank=True)

    # Intégrité
    document_hash_at_signing = models.CharField(
        max_length=64,
        help_text="Hash SHA-256 du fichier PDF au moment exact de la signature"
    )
    signature_valid = models.BooleanField(default=True)
    last_verified_at = models.DateTimeField(null=True, blank=True)
    verification_details = models.JSONField(
        default=dict,
        help_text="Détails PyHanko de la dernière vérification (status, chain, revocation)"
    )

    # Position visible dans le PDF
    page_number = models.PositiveSmallIntegerField(
        null=True, blank=True,
        help_text="Page où la signature est visible (null = signature invisible/embedded)"
    )
    signature_box = models.JSONField(
        null=True, blank=True,
        help_text="Coordonnées {x1, y1, x2, y2} de la zone signature visible"
    )

    # Horodatage
    signed_at = models.DateTimeField(auto_now_add=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True, on_delete=models.SET_NULL,
        related_name='deleted_signatures'
    )

    class Meta:
        db_table = 'digital_signatures'
        indexes = [
            models.Index(fields=['document', 'signed_at']),
            models.Index(fields=['signer']),
            models.Index(fields=['certificate_serial']),
            models.Index(fields=['signature_valid']),
        ]
```

---

### 4.6 ExternalUser — Utilisateurs externes invités

```python
class ExternalUser(BaseModel):
    """
    Représente un utilisateur externe invité via partage.
    PAS un compte système complet : email + token d'identification uniquement.
    Collecte minimale de données conformément au RGPD.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    email = models.EmailField(db_index=True)
    full_name = models.CharField(max_length=200, blank=True)
    organization = models.CharField(max_length=200, blank=True)

    # Token d'accès sécurisé (identifie l'utilisateur sans mot de passe)
    access_token = models.CharField(
        max_length=128, unique=True,
        help_text="Token secrets.token_urlsafe(64) — expire après 1 an d'inactivité"
    )
    token_expires_at = models.DateTimeField()

    invited_by = models.ForeignKey(
        'users.User', on_delete=models.PROTECT,
        related_name='invited_external_users'
    )

    # Statistiques
    total_accesses = models.PositiveIntegerField(default=0)
    last_access_at = models.DateTimeField(null=True, blank=True)

    # Consentement RGPD (obligatoire pour utilisateurs externes)
    gdpr_consent_given = models.BooleanField(default=False)
    gdpr_consent_at = models.DateTimeField(null=True, blank=True)

    # Horodatage
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True, on_delete=models.SET_NULL,
        related_name='deleted_external_users'
    )

    class Meta:
        db_table = 'external_users'
        indexes = [
            models.Index(fields=['email']),
            models.Index(fields=['access_token']),
        ]
```

---

### 4.7 WatermarkConfig — Configuration des watermarks

```python
class WatermarkConfig(BaseModel):
    """
    Configuration des watermarks appliqués aux documents partagés.
    Un watermark système par défaut + configurations personnalisées possibles.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    name = models.CharField(
        max_length=100,
        help_text="Nom de la configuration (ex: 'Défaut système', 'Externe confidentiel')"
    )
    is_default = models.BooleanField(
        default=False,
        help_text="Configuration utilisée par défaut pour tous les partages"
    )

    # Texte du watermark avec variables
    text_template = models.CharField(
        max_length=500,
        default="COPIE NON CERTIFIÉE - {recipient_name} - {date} - {document_reference}",
        help_text="Variables disponibles : {recipient_name}, {recipient_email}, {date}, {document_reference}, {share_link_id}"
    )

    # Apparence
    font_size = models.PositiveSmallIntegerField(default=36)
    font_color = models.CharField(
        max_length=7, default='#CC0000',
        help_text="Couleur hexadécimale (ex: #CC0000 pour rouge)"
    )
    opacity = models.FloatField(
        default=0.3,
        help_text="Opacité 0.0 (invisible) à 1.0 (opaque plein)"
    )
    rotation_angle = models.SmallIntegerField(
        default=45,
        help_text="Angle de rotation en degrés (-180 à 180)"
    )

    # Position et couverture
    position = models.CharField(
        max_length=20,
        choices=[
            ('CENTER', 'Centre'),
            ('DIAGONAL', 'Diagonal'),
            ('FOOTER', 'Pied de page'),
            ('HEADER', 'En-tête'),
            ('CORNER', 'Coin inférieur droit'),
        ],
        default='DIAGONAL'
    )
    apply_to_all_pages = models.BooleanField(
        default=True,
        help_text="Si False, appliqué uniquement à la première page"
    )

    # Horodatage
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    created_by = models.ForeignKey(
        'users.User', on_delete=models.PROTECT,
        related_name='created_watermark_configs'
    )

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True, on_delete=models.SET_NULL,
        related_name='deleted_watermark_configs'
    )

    class Meta:
        db_table = 'watermark_configs'
        constraints = [
            models.UniqueConstraint(
                fields=['is_default'],
                condition=models.Q(is_default=True),
                name='unique_default_watermark_config'
            ),
        ]
```

---

## 5. MATRICE DES PERMISSIONS

| Action | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|--------|:-----------:|:-----:|:----------:|:-----------:|:-----:|:--------:|
| Soumettre demande d'accès | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Voir ses propres demandes | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Approuver/Rejeter une demande (doc Interne/Confidentiel) | ✅ | ✅ | ❌ | ✅ (son service) | ❌ | ❌ |
| Approuver demande sur doc Secret | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Voir toutes les demandes | ✅ | ✅ | ❌ | ✅ (son service) | ❌ | ✅ (lecture) |
| Créer lien partage (doc Public/Interne) | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| Créer lien partage (doc Confidentiel) | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| Créer lien partage (doc Secret) | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Partage externe (email) | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| Révoquer ses propres liens | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| Révoquer n'importe quel lien | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Voir tous les partages actifs | ✅ | ✅ | ✅ (ses liens) | ✅ (son service) | ❌ | ✅ (lecture) |
| Signer numériquement | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| Vérifier une signature | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Configurer watermarks | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Désactiver watermark sur un lien | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Configurer règles d'approbation | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Exporter rapport accès/partages | ✅ | ✅ | ❌ | ✅ (son service) | ❌ | ✅ |
| Accéder via lien de partage | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

---

## 6. SÉQUENCES DÉTAILLÉES

### Séquence 1 — Circuit complet demande d'accès et approbation

```
Agent                    Système                  Responsable
  |                         |                         |
  |-- Clique "Demander" --> |                         |
  |<-- Formulaire demande --|                         |
  |                         |                         |
  |-- Soumet (motif, durée, type) -----------------> |
  |                         |                         |
  |                         |-- Vérifie no doublon    |
  |                         |-- Vérifie confidentialité
  |                         |                         |
  |                         |-- Crée AccessRequest    |
  |                         |   (status=PENDING,      |
  |                         |    ref=AR-2026-00042)   |
  |                         |                         |
  |                         |-- Identifie approbateurs|
  |                         |   (selon niveau confid.)|
  |                         |                         |
  |                         |-- Notifie approbateurs  |
  |                         |   (MODULE 12)          |
  |                         |                         |
  |<-- Confirmation AR-2026-00042 ------------------|
  |                         |                         |
  |                         |             [Responsable connecté]
  |                         |<-- Consulte demande ----|
  |                         |-- Affiche détails ----->|
  |                         |                         |
  |                         |<-- Approuve (15j, READ_ONLY)
  |                         |                         |
  |                         |-- Crée DocumentAccess   |
  |                         |   (expires_at=now+15j)  |
  |                         |-- Update AccessRequest  |
  |                         |   (status=APPROVED)     |
  |                         |-- Journalise (MOD 11)   |
  |                         |                         |
  |<-- Notif "Accès accordé 15 jours, lecture seule" |
```

---

### Séquence 2 — Création et utilisation d'un lien de partage externe

```
Archiviste               Système                 Utilisateur Externe
     |                      |                           |
     |-- Crée lien partage ->|                          |
     |   (doc, email dest., |                           |
     |    7j, dl autorisé)  |                           |
     |                      |                           |
     |                      |-- Vérifie confidentialité |
     |                      |-- Génère token (64 chars) |
     |                      |-- Crée ShareLink          |
     |                      |   (status=ACTIVE)         |
     |                      |-- Envoie email ----------->|
     |<-- Lien affiché ------|                           |
     |                      |                           |
     |                      |       [Utilisateur clique le lien]
     |                      |<-- GET /share/{token} ----|
     |                      |                           |
     |                      |-- Valide token            |
     |                      |-- Vérifie expiration      |
     |                      |-- Vérifie quota           |
     |                      |-- Génère PDF watermarqué ->|
     |                      |   "COPIE NON CERTIFIÉE    |
     |                      |   Jean Dupont - 18/02/26" |
     |                      |-- Affiche visionneuse --->|
     |                      |-- Crée ShareActivity      |
     |                      |-- Incrémente use_count    |
     |                      |-- Journalise (MOD 11)     |
     |                      |                           |
     |                      |    [Utilisateur télécharge]
     |                      |<-- GET /share/{token}/download
     |                      |                           |
     |                      |-- Vérifie can_download    |
     |                      |-- Génère PDF watermarqué ->|
     |                      |-- Stream fichier -------->|
     |                      |-- ShareActivity (DOWNLOADED)
     |                      |-- Journalise (MOD 11)     |
```

---

### Séquence 3 — Signature numérique PyHanko

```
Responsable              Système                     PyHanko
     |                      |                            |
     |-- Demande signature ->|                           |
     |   (doc PDF, raison)   |                           |
     |                       |                           |
     |                       |-- Vérifie doc = PDF       |
     |                       |-- Vérifie status=VALIDATED|
     |                       |-- Vérifie certificat dispo|
     |                       |                           |
     |<-- TOTP challenge -----|                           |
     |-- Code TOTP ---------->|                           |
     |                       |-- Valide TOTP             |
     |                       |                           |
     |                       |-- Récupère PDF (stockage) |
     |                       |                           |
     |                       |-- Appelle PyHanko ------->|
     |                       |   sign_pdf(pdf_bytes,     |
     |                       |     cert, key, reason,    |
     |                       |     sig_box)              |
     |                       |                           |
     |                       |<-- PDF signé (bytes) -----|
     |                       |                           |
     |                       |-- Hash SHA-256 du signé   |
     |                       |-- Sauvegarde v.mineure    |
     |                       |-- Crée DigitalSignature   |
     |                       |-- Journalise (MOD 11)     |
     |                       |                           |
     |<-- Confirmation "Document signé (v1.1)" ---------|
```

---

## 7. ENDPOINTS API

### 7.1 Demandes d'accès

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/access-requests/` | Lister les demandes (filtrables par statut, document) | JWT |
| `POST` | `/api/v1/access-requests/` | Soumettre une nouvelle demande | JWT |
| `GET` | `/api/v1/access-requests/{id}/` | Détail d'une demande | JWT |
| `POST` | `/api/v1/access-requests/{id}/approve/` | Approuver avec paramètres (durée, permissions) | JWT + Perm |
| `POST` | `/api/v1/access-requests/{id}/reject/` | Rejeter avec motif obligatoire | JWT + Perm |
| `POST` | `/api/v1/access-requests/{id}/cancel/` | Annuler sa propre demande | JWT |
| `GET` | `/api/v1/access-requests/pending/` | File d'attente des approbateurs | JWT + Perm |
| `GET` | `/api/v1/documents/{doc_id}/access-requests/` | Demandes pour un document | JWT + Perm |

### 7.2 Accès accordés

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/document-accesses/` | Lister les accès actifs | JWT + Perm |
| `GET` | `/api/v1/document-accesses/{id}/` | Détail d'un accès | JWT + Perm |
| `POST` | `/api/v1/document-accesses/{id}/revoke/` | Révoquer un accès (motif obligatoire) | JWT + Perm |
| `POST` | `/api/v1/document-accesses/{id}/extend/` | Prolonger (dans la limite 90 jours total) | JWT + Perm |
| `GET` | `/api/v1/documents/{doc_id}/accesses/` | Tous les accès d'un document | JWT + Perm |
| `GET` | `/api/v1/users/{user_id}/accesses/` | Tous les accès d'un utilisateur | JWT + Perm |

### 7.3 Liens de partage (authentifiés)

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/share-links/` | Lister ses liens | JWT |
| `POST` | `/api/v1/share-links/` | Créer un lien | JWT + Perm |
| `GET` | `/api/v1/share-links/{id}/` | Détail d'un lien | JWT + Perm |
| `PATCH` | `/api/v1/share-links/{id}/` | Modifier expiration ou quota | JWT + Perm |
| `POST` | `/api/v1/share-links/{id}/revoke/` | Révoquer | JWT + Perm |
| `POST` | `/api/v1/share-links/{id}/extend/` | Prolonger l'expiration | JWT + Perm |
| `GET` | `/api/v1/share-links/{id}/activity/` | Historique d'utilisation | JWT + Perm |
| `POST` | `/api/v1/share-links/{id}/resend-email/` | Renvoyer l'email au destinataire | JWT + Perm |
| `GET` | `/api/v1/documents/{doc_id}/share-links/` | Liens actifs d'un document | JWT + Perm |

### 7.4 Accès public via token (sans JWT)

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/share/{token}/` | Valider le token et obtenir les métadonnées | Token URL |
| `POST` | `/api/v1/share/{token}/verify-password/` | Soumettre le mot de passe d'un lien protégé | Token URL |
| `GET` | `/api/v1/share/{token}/document/` | Obtenir le document pour visionneuse (stream) | Token URL |
| `GET` | `/api/v1/share/{token}/download/` | Télécharger le document watermarqué | Token URL |
| `POST` | `/api/v1/share/{token}/comment/` | Ajouter un commentaire (si permission) | Token URL |

### 7.5 Signatures numériques

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/digital-signatures/` | Lister les signatures | JWT |
| `POST` | `/api/v1/documents/{doc_id}/sign/` | Signer numériquement (TOTP requis) | JWT + TOTP |
| `GET` | `/api/v1/digital-signatures/{id}/` | Détail d'une signature | JWT |
| `POST` | `/api/v1/digital-signatures/{id}/verify/` | Relancer la vérification PyHanko | JWT |
| `GET` | `/api/v1/documents/{doc_id}/signatures/` | Toutes les signatures d'un document | JWT |

### 7.6 Watermarks

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/watermark-configs/` | Lister les configurations | JWT + Perm |
| `POST` | `/api/v1/watermark-configs/` | Créer une configuration | JWT + Admin |
| `GET` | `/api/v1/watermark-configs/{id}/` | Détail | JWT + Perm |
| `PATCH` | `/api/v1/watermark-configs/{id}/` | Modifier | JWT + Admin |
| `DELETE` | `/api/v1/watermark-configs/{id}/` | Soft delete | JWT + Admin |
| `POST` | `/api/v1/watermark-configs/{id}/set-default/` | Définir comme config par défaut | JWT + Admin |
| `POST` | `/api/v1/documents/{doc_id}/preview-watermark/` | Prévisualiser le watermark | JWT + Perm |

### 7.7 Tableau de bord et exports

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/sharing/dashboard/` | Vue d'ensemble des partages actifs | JWT + Perm |
| `GET` | `/api/v1/sharing/export/` | Export CSV/PDF des accès pour audit | JWT + Perm |
| `GET` | `/api/v1/sharing/statistics/` | Statistiques d'usage des partages | JWT + Admin |

---

## 8. FORMAT DES RÉPONSES API

### Enveloppe standard

```json
{
  "success": true,
  "data": { },
  "meta": {
    "cursor": "eyJpZCI6IjEyMyJ9",
    "has_next": true,
    "total_count": 47
  }
}
```

### Exemple : Création d'un lien de partage (POST `/api/v1/share-links/`)

**Requête :**
```json
{
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "share_type": "EXTERNAL",
  "recipient_email": "jean.dupont@partenaire.fr",
  "recipient_name": "Jean Dupont",
  "recipient_organization": "Ministère XYZ",
  "can_read": true,
  "can_download": true,
  "can_comment": false,
  "expires_at": "2026-03-20T23:59:59Z",
  "max_uses": 3,
  "custom_message": "Veuillez consulter le document ci-joint dans le cadre de notre collaboration.",
  "apply_watermark": true,
  "watermark_config_id": null
}
```

**Réponse (201 Created) :**
```json
{
  "success": true,
  "data": {
    "id": "7f3d8c2a-9b4e-4f1d-a8c3-2e5b6d9f0a1c",
    "token": "a1b2c3d4e5f6789012345678901234567f8e9d0c1b2a3456",
    "share_url": "https://archive.prefectures.fr/share/a1b2c3d4e5f6789012345678901234567f8e9d0c1b2a3456",
    "document": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "title": "Rapport annuel 2025",
      "reference": "DOC-2026-00042"
    },
    "share_type": "EXTERNAL",
    "recipient_email": "jean.dupont@partenaire.fr",
    "recipient_name": "Jean Dupont",
    "can_read": true,
    "can_download": true,
    "can_comment": false,
    "expires_at": "2026-03-20T23:59:59Z",
    "max_uses": 3,
    "use_count": 0,
    "status": "ACTIVE",
    "created_at": "2026-02-18T10:30:00Z",
    "created_by": {
      "id": "...",
      "full_name": "Marie Archiviste"
    }
  }
}
```

### Exemple : Accès via token (GET `/api/v1/share/{token}/`)

**Réponse (200 OK) :**
```json
{
  "success": true,
  "data": {
    "valid": true,
    "document": {
      "title": "Rapport annuel 2025",
      "document_type": "Rapport",
      "page_count": 42,
      "file_size_bytes": 2048576
    },
    "permissions": {
      "can_read": true,
      "can_download": true,
      "can_comment": false
    },
    "requires_password": false,
    "expires_at": "2026-03-20T23:59:59Z",
    "custom_message": "Veuillez consulter le document ci-joint..."
  }
}
```

**Réponse token invalide (404 avec message générique) :**
```json
{
  "success": false,
  "error": {
    "code": "SHARE_LINK_INVALID",
    "message": "Lien invalide ou expiré.",
    "detail": null
  }
}
```

---

## 9. RÈGLES MÉTIER CRITIQUES

### RB-013-01 — Vérification confidentialité avant tout partage

Avant de créer tout lien ou d'accorder tout accès, le système vérifie le niveau de confidentialité du document (MODULE 05) et applique les règles suivantes :

```
PUBLIC       → Partage interne et externe autorisé sans restriction particulière
INTERNE      → Partage interne libre, partage externe avec confirmation explicite
CONFIDENTIEL → Partage interne libre, partage externe avec avertissement + double confirmation
SECRET       → Partage interne uniquement (Admin/Super Admin), partage externe toujours bloqué
```

Toute tentative de partage externe d'un document `SECRET` est bloquée avec le code `403 FORBIDDEN` et journalisée comme événement `SECRET_SHARE_ATTEMPT` avec sévérité `ALERT` (MODULE 11).

---

### RB-013-02 — Durée maximale absolue de 90 jours

Aucun accès temporaire (demande approuvée ou lien de partage) ne peut excéder **90 jours calendaires** à compter de la date de création. Cette règle est non outrepassable, même par le Super Admin. Au-delà de 90 jours, un nouveau lien ou un renouvellement d'accès doit être créé.

---

### RB-013-03 — Watermark obligatoire sur tout téléchargement via partage

Tout document téléchargé via un lien de partage porte un watermark. Le fichier watermarqué est généré à la volée à chaque téléchargement (non stocké sur disque) et contient au minimum : le nom du destinataire, la date et heure de téléchargement, et la référence du document. La désactivation du watermark est une prérogative exclusive du Super Admin et génère un événement `WATERMARK_DISABLED` avec sévérité `ALERT`.

---

### RB-013-04 — Unicité des demandes en attente

Un utilisateur ne peut soumettre qu'une seule demande d'accès en statut `PENDING` par document. Une nouvelle demande n'est autorisée que si la précédente a été traitée (`APPROVED`, `REJECTED`, `CANCELLED`, `EXPIRED`). Cette contrainte est assurée en base par une `UniqueConstraint` partielle.

---

### RB-013-05 — Révocation immédiate et message d'erreur générique

La révocation d'un accès ou lien prend effet immédiatement. Un lien révoqué, expiré ou épuisé retourne toujours le même message d'erreur générique "Lien invalide ou expiré" afin de ne pas révéler d'informations sur l'état interne du système à des personnes non autorisées.

---

### RB-013-06 — Délai maximal d'approbation configurable

Toute demande d'accès non traitée dans le délai configuré (défaut : 5 jours ouvrés, configurable via MODULE 15 par niveau de confidentialité) passe automatiquement au statut `EXPIRED`. Le demandeur est notifié. Il peut soumettre une nouvelle demande.

---

### RB-013-07 — Blocage des liens après tentatives mot de passe répétées

Un lien protégé par mot de passe bloque l'accès après 3 tentatives incorrectes consécutives pendant 15 minutes. Après 10 tentatives totales sur une fenêtre de 24 heures, le lien est automatiquement révoqué et le créateur reçoit une alerte de sécurité (MODULE 12, priorité HAUTE).

---

### RB-013-08 — Immuabilité des documents signés numériquement

Un document ayant reçu une signature numérique ne peut plus être modifié directement sur la version signée. Toute modification ultérieure crée une nouvelle version sans signature (la signature n'est pas héritée). La version signée reste archivée et sa signature reste vérifiable indépendamment.

---

## 10. TÂCHES CELERY PLANIFIÉES

### TASK-013-01 : Expiration automatique des liens de partage
```python
@shared_task(name='sharing.expire_share_links')
# Fréquence     : Toutes les heures
# Description   : Passe les ShareLink dont expires_at < now() de ACTIVE à EXPIRED
# Volume        : Traitement par lot de 500 (évite les locks)
# Audit         : Événement ACCESS_REQUEST_EXPIRED journalisé par lien expiré
# Notification  : Silencieuse (expiration naturelle attendue)
```

### TASK-013-02 : Expiration automatique des accès accordés
```python
@shared_task(name='sharing.expire_document_accesses')
# Fréquence     : Toutes les heures
# Description   : Passe is_active=False sur les DocumentAccess dont expires_at < now()
# Notification  : Aucune à l'expiration (la notification préventive est faite par TASK-013-03)
```

### TASK-013-03 : Alerte de veille d'expiration (J-1 et J-2)
```python
@shared_task(name='sharing.notify_expiring_shares')
# Fréquence     : Quotidienne à 8h00
# Description   : Notifie les créateurs de liens et les approbateurs d'accès expirant dans 24h et 48h
# Destinataires : Créateur du ShareLink / approbateur du DocumentAccess
# Canal         : In-app + Email selon préférences MODULE 12
```

### TASK-013-04 : Expiration des demandes non traitées
```python
@shared_task(name='sharing.expire_pending_access_requests')
# Fréquence     : Quotidienne à 7h00 (jours ouvrés)
# Description   : Passe PENDING → EXPIRED pour les AccessRequest dépassant approval_deadline
# Notification  : Demandeur notifié, mention que la demande peut être resousmise
```

### TASK-013-05 : Rappel approbateurs (demandes proches deadline)
```python
@shared_task(name='sharing.remind_approvers')
# Fréquence     : Quotidienne à 9h00
# Description   : Notifie les approbateurs des demandes expirant dans les 24h suivantes
# Canal         : In-app + Email (priorité HAUTE)
```

### TASK-013-06 : Vérification périodique des signatures numériques
```python
@shared_task(name='sharing.verify_digital_signatures')
# Fréquence     : Mensuelle (1er du mois à 3h00)
# Description   : Re-vérifie la validité de toutes les DigitalSignature via PyHanko
# Alerte        : Si signature_valid passe à False → événement SIGNATURE_INVALID (CRITICAL)
# Notification  : Super Admin + Admin + signataire concerné
```

### TASK-013-07 : Nettoyage ShareActivity (RGPD)
```python
@shared_task(name='sharing.cleanup_share_activities')
# Fréquence     : Hebdomadaire (dimanche à 2h00)
# Description   : Soft delete des ShareActivity > durée légale (configurable, défaut 2 ans)
# Rapport       : Nombre d'enregistrements traités journalisé en audit (INFO)
```

### TASK-013-08 : Rapport hebdomadaire des partages actifs
```python
@shared_task(name='sharing.weekly_share_report')
# Fréquence     : Lundi à 7h30
# Description   : Récapitulatif des partages actifs pour les Admins
# Contenu       : Liens actifs, accès actifs, nouvelles demandes semaine écoulée, révocations
# Canal         : In-app ou Email selon préférences (MODULE 12)
```

---

## 11. ÉVÉNEMENTS JOURNALISÉS (MODULE 11)

| Code événement | Description | Sévérité | Données clés |
|----------------|-------------|----------|--------------|
| `ACCESS_REQUEST_SUBMITTED` | Demande d'accès soumise | INFO | document_id, requester_id, justification[:100] |
| `ACCESS_REQUEST_APPROVED` | Demande approuvée | INFO | request_id, approver_id, duration_days, permissions |
| `ACCESS_REQUEST_REJECTED` | Demande rejetée | INFO | request_id, approver_id, rejection_reason |
| `ACCESS_REQUEST_EXPIRED` | Demande expirée sans décision | WARNING | request_id, document_id, approval_deadline |
| `ACCESS_REQUEST_CANCELLED` | Demande annulée par le demandeur | INFO | request_id |
| `DOCUMENT_ACCESS_GRANTED` | Accès temporaire accordé | INFO | access_id, user_id, document_id, expires_at |
| `DOCUMENT_ACCESS_REVOKED` | Accès révoqué manuellement | WARNING | access_id, revoked_by_id, reason |
| `DOCUMENT_ACCESS_EXPIRED` | Accès expiré automatiquement | INFO | access_id, user_id, document_id |
| `SHARE_LINK_CREATED` | Lien de partage créé | INFO | link_id, document_id, share_type, expires_at |
| `SHARE_LINK_ACCESSED` | Lien accédé (consultation) | INFO | link_id, ip_anonymized, use_count |
| `SHARE_LINK_DOWNLOADED` | Document téléchargé via lien | INFO | link_id, document_id, watermark_applied |
| `SHARE_LINK_REVOKED` | Lien révoqué manuellement | WARNING | link_id, revoked_by_id, reason |
| `SHARE_LINK_EXPIRED` | Lien expiré automatiquement | INFO | link_id, use_count_final |
| `SHARE_LINK_EXHAUSTED` | Lien épuisé (quota atteint) | INFO | link_id, max_uses |
| `SHARE_PASSWORD_FAILED` | Tentative mot de passe incorrecte | WARNING | link_id, ip_anonymized, attempt_count |
| `SHARE_PASSWORD_LOCKED` | Lien verrouillé après abus | ALERT | link_id, locked_until |
| `SHARE_LINK_AUTO_REVOKED` | Lien révoqué automatiquement pour abus | ALERT | link_id, reason, total_failed_attempts |
| `DOCUMENT_SIGNED` | Document signé numériquement | INFO | signature_id, document_id, signer_id, type |
| `SIGNATURE_VERIFIED` | Signature vérifiée avec succès | INFO | signature_id, valid=true, verified_by_id |
| `SIGNATURE_INVALID` | Signature devenue invalide détectée | CRITICAL | signature_id, document_id, reason |
| `SECRET_SHARE_ATTEMPT` | Tentative partage externe d'un doc Secret | ALERT | document_id, attempted_by_id, share_type |
| `WATERMARK_DISABLED` | Désactivation watermark par Super Admin | ALERT | document_id, share_link_id, disabled_by_id |

---

## 12. NOTIFICATIONS GÉNÉRÉES (MODULE 12)

| Déclencheur | Destinataire(s) | Canal | Priorité | Template |
|-------------|----------------|-------|----------|----------|
| Nouvelle demande d'accès soumise | Approbateurs identifiés | In-app + Email | HAUTE | `access_request_pending` |
| Demande approuvée | Demandeur | In-app + Email | HAUTE | `access_request_approved` |
| Demande rejetée | Demandeur | In-app + Email | HAUTE | `access_request_rejected` |
| Demande expirée sans décision | Demandeur + Approbateurs | In-app | NORMALE | `access_request_expired` |
| Rappel demande (48h avant deadline) | Approbateurs | In-app + Email | HAUTE | `access_request_reminder` |
| Accès expirant dans 24h | Détenteur de l'accès | In-app | NORMALE | `access_expiring_soon` |
| Accès révoqué | Détenteur de l'accès | In-app + Email | HAUTE | `access_revoked` |
| 1ère consultation d'un lien créé | Créateur du lien | In-app | BASSE | `share_link_first_access` |
| Lien épuisé (quota atteint) | Créateur du lien | In-app | NORMALE | `share_link_exhausted` |
| Lien expirant dans 24h | Créateur du lien | In-app | NORMALE | `share_link_expiring_soon` |
| Lien révoqué par un admin tiers | Créateur du lien | In-app + Email | HAUTE | `share_link_revoked_by_admin` |
| Alerte tentatives mot de passe | Créateur + Admins | In-app + Email | HAUTE | `share_link_security_alert` |
| Document signé avec succès | Signataire + Resp. document | In-app | NORMALE | `document_signed_success` |
| Signature invalide détectée | Super Admin + Admin + Signataire | In-app + Email | CRITIQUE | `signature_invalid_alert` |
| Tentative partage doc Secret | Super Admin + Admin | In-app + Email | CRITIQUE | `secret_share_attempt_alert` |

---

## 13. SERVICES TECHNIQUES

### 13.1 Génération de tokens sécurisés

```python
# apps/sharing/utils/token_generator.py

import hashlib
import secrets
import uuid
from django.conf import settings


def generate_share_token(document_id: str, created_at_iso: str) -> str:
    """
    Génère un token de partage sécurisé et URL-safe de 64 caractères.

    Structure :
    - 32 chars : UUID v4 sans tirets (entropie brute)
    - 32 chars : SHA-256(uuid + doc_id + created_at + SECRET_KEY)[:32]

    La combinaison rend le token imprévisible et vérifiable côté serveur.
    """
    unique_id = str(uuid.uuid4()).replace('-', '')
    payload = f"{unique_id}{document_id}{created_at_iso}{settings.SHARE_TOKEN_SECRET_KEY}"
    hash_suffix = hashlib.sha256(payload.encode('utf-8')).hexdigest()[:32]
    return f"{unique_id}{hash_suffix}"


def generate_external_user_token() -> str:
    """
    Token d'identification pour utilisateur externe.
    128 caractères URL-safe via secrets.token_urlsafe.
    """
    return secrets.token_urlsafe(96)
```

---

### 13.2 Service de watermarking (extrait)

```python
# apps/sharing/services/watermark_service.py

import io
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import A4
from pypdf import PdfReader, PdfWriter


class WatermarkService:

    def apply_watermark_to_pdf(
        self,
        pdf_bytes: bytes,
        config: 'WatermarkConfig',
        context: dict
    ) -> bytes:
        """
        Applique un watermark sur un PDF et retourne les bytes résultants.
        Le fichier watermarqué n'est JAMAIS stocké sur disque.

        context = {
            'recipient_name': 'Jean Dupont',
            'recipient_email': 'jean@example.com',
            'date': '18/02/2026 10:30',
            'document_reference': 'DOC-2026-00042',
            'share_link_id': 'abc123...'
        }
        """
        watermark_text = config.text_template.format(**context)
        watermark_pdf_bytes = self._create_watermark_overlay(
            text=watermark_text,
            config=config,
            page_size=A4
        )

        reader = PdfReader(io.BytesIO(pdf_bytes))
        watermark_reader = PdfReader(io.BytesIO(watermark_pdf_bytes))
        watermark_page = watermark_reader.pages[0]

        writer = PdfWriter()
        for i, page in enumerate(reader.pages):
            if config.apply_to_all_pages or i == 0:
                page.merge_page(watermark_page)
            writer.add_page(page)

        output = io.BytesIO()
        writer.write(output)
        return output.getvalue()

    def _create_watermark_overlay(
        self,
        text: str,
        config: 'WatermarkConfig',
        page_size: tuple
    ) -> bytes:
        """Crée un PDF overlay transparent contenant uniquement le texte watermark."""
        packet = io.BytesIO()
        c = canvas.Canvas(packet, pagesize=page_size)
        width, height = page_size

        hex_color = config.font_color.lstrip('#')
        r = int(hex_color[0:2], 16) / 255
        g = int(hex_color[2:4], 16) / 255
        b = int(hex_color[4:6], 16) / 255
        c.setFillColorRGB(r, g, b, alpha=config.opacity)
        c.setFont("Helvetica-Bold", config.font_size)

        if config.position == 'DIAGONAL':
            c.saveState()
            c.translate(width / 2, height / 2)
            c.rotate(config.rotation_angle)
            c.drawCentredString(0, 0, text)
            c.restoreState()
        elif config.position == 'FOOTER':
            c.drawCentredString(width / 2, 30, text)
        elif config.position == 'HEADER':
            c.drawCentredString(width / 2, height - 50, text)
        elif config.position == 'CENTER':
            c.drawCentredString(width / 2, height / 2, text)
        elif config.position == 'CORNER':
            c.setFont("Helvetica", config.font_size * 0.6)
            c.drawString(width - 200, 20, text)

        c.save()
        packet.seek(0)
        return packet.getvalue()
```

---

### 13.3 Service de signature numérique (extrait)

```python
# apps/sharing/services/signature_service.py

import io
from pyhanko.sign import signers
from pyhanko.pdf_utils.incremental_writer import IncrementalPdfFileWriter
from pyhanko.sign.fields import SigFieldSpec
from cryptography.hazmat.primitives.serialization import pkcs12


class SignatureService:

    def sign_document(
        self,
        pdf_bytes: bytes,
        certificate_p12_bytes: bytes,
        passphrase: bytes,
        reason: str,
        location: str = '',
        page: int = 0,
        sig_box: dict = None
    ) -> bytes:
        """
        Applique une signature PAdES-T sur un PDF.
        Retourne les bytes du PDF signé.

        Lève ValueError si le certificat est invalide ou expiré.
        Lève RuntimeError en cas d'erreur PyHanko (rollback côté appelant).
        """
        private_key, certificate, chain = pkcs12.load_key_and_certificates(
            certificate_p12_bytes, passphrase
        )

        signer = signers.SimpleSigner(
            signing_cert=certificate,
            signing_key=private_key,
            cert_registry=signers.SimpleCertificateStore.from_certs(chain or [])
        )

        box = (50, 50, 250, 100)
        if sig_box:
            box = (sig_box['x1'], sig_box['y1'], sig_box['x2'], sig_box['y2'])

        sig_field_spec = SigFieldSpec(
            sig_field_name='Signature1',
            on_page=page,
            box=box
        )

        writer = IncrementalPdfFileWriter(io.BytesIO(pdf_bytes))

        signature_meta = signers.PdfSignatureMetadata(
            field_name='Signature1',
            reason=reason,
            location=location,
        )

        output = io.BytesIO()
        signers.sign_pdf(
            writer,
            signature_meta=signature_meta,
            signer=signer,
            output=output,
            new_field_spec=sig_field_spec
        )
        return output.getvalue()

    def verify_signatures(self, pdf_bytes: bytes) -> dict:
        """
        Vérifie toutes les signatures présentes dans un PDF via PyHanko.
        Retourne un rapport structuré de la validité de chaque signature.
        """
        from pyhanko.sign import validation
        from pyhanko.pdf_utils.reader import PdfFileReader
        from pyhanko_certvalidator import ValidationContext

        reader = PdfFileReader(io.BytesIO(pdf_bytes))
        embedded_sigs = reader.embedded_signatures

        if not embedded_sigs:
            return {'has_signatures': False, 'signatures': []}

        vc = ValidationContext(trust_roots=None, allow_fetching=True)
        results = []

        for sig in embedded_sigs:
            try:
                status = validation.validate_pdf_signature(sig, vc)
                results.append({
                    'field_name': sig.field_name,
                    'signer': str(sig.signer_cert.subject),
                    'signing_time': str(sig.self_reported_timestamp),
                    'intact': status.intact,
                    'trusted': status.trusted,
                    'valid': status.bottom_line,
                })
            except Exception as e:
                results.append({
                    'field_name': sig.field_name,
                    'valid': False,
                    'error': str(e)
                })

        return {
            'has_signatures': True,
            'signatures': results,
            'all_valid': all(r.get('valid', False) for r in results)
        }
```

---

## 14. CONSIDÉRATIONS RGPD

### 14.1 Données personnelles traitées dans ce module

Ce module traite plusieurs catégories de données personnelles, chacune avec sa propre base légale :

**Utilisateurs internes** — Nom, email, historique des demandes d'accès, actions sur les partages : traités dans le cadre de la relation de travail (base légale : intérêt légitime de l'administration publique dans l'exercice de l'autorité publique).

**Utilisateurs externes** — Email, nom, organisation, IP anonymisée, user-agent hashé : collectés avec consentement explicite recueilli lors de la première utilisation d'un lien de partage (case `gdpr_consent_given` dans `ExternalUser`).

### 14.2 Mesures techniques d'anonymisation

Les `ShareActivity` ne stockent jamais l'adresse IP complète : le dernier octet pour IPv4 (ou les 64 derniers bits pour IPv6) est remplacé par des zéros, conformément aux recommandations CNIL en vigueur. Le `user_agent` n'est pas stocké brut mais hashé (SHA-256), ce qui permet la corrélation de sessions sans permettre l'identification.

### 14.3 Durées de conservation

| Donnée | Durée | Base légale |
|--------|-------|-------------|
| `AccessRequest` | 5 ans après clôture | Obligation légale audit public |
| `DocumentAccess` | 3 ans après expiration | Traçabilité administrative |
| `ShareLink` | 2 ans après expiration/révocation | Traçabilité des accès accordés |
| `ShareActivity` | 2 ans (paramétrable) | Conformité + droit à l'oubli |
| `ExternalUser` | 1 an après dernière activité | Proportionnalité RGPD |
| `DigitalSignature` | Durée de vie légale du document | Valeur probante juridique |

---

## 15. STRUCTURE DES FICHIERS

```
apps/
└── sharing/
    ├── __init__.py
    ├── admin.py
    ├── apps.py
    ├── models/
    │   ├── __init__.py
    │   ├── access_request.py
    │   ├── document_access.py
    │   ├── share_link.py
    │   ├── share_activity.py
    │   ├── digital_signature.py
    │   ├── external_user.py
    │   └── watermark_config.py
    ├── serializers/
    │   ├── __init__.py
    │   ├── access_request_serializers.py
    │   ├── share_link_serializers.py
    │   └── signature_serializers.py
    ├── views/
    │   ├── __init__.py
    │   ├── access_request_views.py
    │   ├── share_link_views.py
    │   ├── public_share_views.py      # Routes sans JWT (token URL uniquement)
    │   └── signature_views.py
    ├── urls.py
    ├── permissions.py
    ├── tasks.py
    ├── signals.py
    ├── services/
    │   ├── __init__.py
    │   ├── watermark_service.py
    │   ├── signature_service.py
    │   ├── access_request_service.py
    │   └── share_link_service.py
    ├── utils/
    │   ├── __init__.py
    │   └── token_generator.py
    └── tests/
        ├── __init__.py
        ├── test_access_requests.py
        ├── test_share_links.py
        ├── test_public_access.py
        ├── test_watermark.py
        └── test_signature.py
```

---

## 16. DÉPENDANCES PYTHON SUPPLÉMENTAIRES

```
# requirements.txt — ajouts spécifiques à ce module

# Signature numérique PDF
pyhanko==0.21.*
pyhanko-certvalidator==0.26.*

# Manipulation PDF pour watermarking
pypdf==4.*
reportlab==4.*

# Déjà présents dans la stack globale (aucun ajout requis)
# Pillow          → manipulation d'images
# argon2-cffi     → hashage des mots de passe de liens
# djangorestframework-simplejwt → JWT
# celery + redis  → tâches planifiées
```

---

## 17. VARIABLES D'ENVIRONNEMENT SPÉCIFIQUES

```env
# .env — ne jamais versionner ni committer

# Sécurité des tokens de partage
SHARE_TOKEN_SECRET_KEY=<clé-aléatoire-distincte-de-SECRET_KEY>

# Limites partage
SHARE_DEFAULT_EXPIRY_DAYS=30
SHARE_MAX_EXPIRY_DAYS=90
SHARE_MAX_PASSWORD_ATTEMPTS=3
SHARE_PASSWORD_LOCKOUT_MINUTES=15
SHARE_AUTO_REVOKE_AFTER_FAILED_ATTEMPTS=10

# Approbation des demandes
ACCESS_REQUEST_DEFAULT_APPROVAL_DAYS=5
ACCESS_REQUEST_MAX_DURATION_DAYS=30
ACCESS_GRANT_MAX_DURATION_DAYS=90

# Watermark
WATERMARK_APPLY_BY_DEFAULT=True

# Certificats de signature (chemin absolu, hors du projet)
SIGNATURE_CERTIFICATES_PATH=/etc/archivage/certificates/
SIGNATURE_DEFAULT_TYPE=PADES_T
```

---

## ✅ RÉCAPITULATIF MODULE 13

| Critère | Valeur |
|---------|--------|
| Cas d'utilisation | 28 UC |
| Modèles de données | 7 tables |
| Endpoints API | 34 endpoints |
| Tâches Celery planifiées | 8 tâches |
| Événements journalisés (MODULE 11) | 22 événements |
| Notifications générées (MODULE 12) | 15 types |
| Services techniques | 4 services |
| Tests à implémenter | 5 fichiers de tests |

---

**Prochain module :** MODULE 14 — Tableaux de bord & Reporting
Ce module agrège les données de l'ensemble des modules pour fournir des dashboards visuels et des rapports exportables (PDF, Excel, CSV) destinés au pilotage opérationnel, à la conformité réglementaire et aux audits.
