# MODULE 17 — Mode Desktop Tauri & Synchronisation offline

**Système d'Archivage Numérique — Administration Publique**  
**Version :** 1.0  
**Statut :** Spécification technique complète  
**Dépendances :** MODULE 02 (Auth), MODULE 06 (Documents), MODULE 07 (OCR & Scanner), MODULE 11 (Audit), MODULE 12 (Notifications), MODULE 15 (Administration)

---

## 1. PRÉSENTATION DU MODULE

### 1.1 Contexte et justification

Les agents de l'administration publique ne travaillent pas toujours depuis un poste connecté en permanence au réseau. Les agents de terrain, les inspecteurs en déplacement, les archivistes dans des sites éloignés doivent pouvoir consulter des documents, numériser des pièces avec un scanner local, et saisir des informations — puis synchroniser leur travail à la reconnexion.

L'application desktop native construite avec **Tauri** (Rust + React) répond à ce besoin en apportant :
1. Une **expérience native** sur Windows, Linux et macOS : notifications système, menu contextuel, association de types de fichiers, accès direct au scanner TWAIN/SANE
2. Un **mode offline-first** : l'application fonctionne pleinement déconnectée, les actions sont mises en file d'attente et synchronisées à la reconnexion
3. Des **performances supérieures** à Electron : binaire 10x plus petit, consommation RAM réduite, démarrage rapide

### 1.2 Philosophie offline-first

Le principe cardinal de ce module est que **la connexion réseau est une opportunité, pas une exigence**. L'application ne doit jamais afficher "Impossible de se connecter" comme état bloquant. Quand le réseau est absent :
- Les documents mis en cache restent consultables
- Les numérisations peuvent être réalisées et stockées localement
- Les métadonnées peuvent être saisies
- Toutes ces actions sont enregistrées dans la file d'attente de synchronisation
- À la reconnexion, la synchronisation s'effectue automatiquement avec gestion des conflits

### 1.3 Choix Tauri vs Electron

| Critère | Tauri | Electron |
|---------|-------|---------|
| Taille binaire | ~10-20 Mo | ~100-200 Mo |
| RAM au démarrage | ~30-50 Mo | ~150-300 Mo |
| Langage natif | Rust | Node.js |
| Accès scanner | Via Rust (TWAIN/SANE) | Via Node addons |
| Sécurité | Sandbox strict | Moins restrictif |
| WebView | OS natif (WebKit/Edge) | Chromium embarqué |

### 1.4 Périmètre fonctionnel

- Application native installable (Windows MSI/NSIS, Linux AppImage/deb, macOS dmg)
- Authentification JWT persistante avec renouvellement silencieux
- Cache local SQLite des documents marqués pour accès offline
- File d'attente des actions offline avec persistance
- Synchronisation bidirectionnelle avec résolution de conflits
- Accès direct aux scanners via Rust (TWAIN sur Windows, SANE sur Linux)
- Notifications système natives (OS tray)
- Auto-update de l'application
- Association des types de fichiers documentaires
- Protocole URL personnalisé `archivage://`

---

## 2. CAS D'UTILISATION — VUE D'ENSEMBLE

| Code | Cas d'utilisation | Acteur principal | Priorité |
|------|-------------------|-----------------|----------|
| UC-017-01 | Installer l'application desktop | Agent, Archiviste, Responsable | HAUTE |
| UC-017-02 | Se connecter depuis l'application desktop | Tout utilisateur | HAUTE |
| UC-017-03 | Configurer la synchronisation (fréquence, limite cache) | Utilisateur authentifié | HAUTE |
| UC-017-04 | Marquer des documents pour accès offline | Tout utilisateur authentifié | HAUTE |
| UC-017-05 | Marquer un dossier complet pour accès offline | Archiviste, Responsable | HAUTE |
| UC-017-06 | Consulter un document en mode offline | Tout utilisateur authentifié | HAUTE |
| UC-017-07 | Numériser un document en mode offline | Archiviste | HAUTE |
| UC-017-08 | Saisir des métadonnées en mode offline | Archiviste, Agent | HAUTE |
| UC-017-09 | Annoter un document en mode offline | Tout utilisateur authentifié | MOYENNE |
| UC-017-10 | Consulter la file d'attente de synchronisation | Tout utilisateur authentifié | HAUTE |
| UC-017-11 | Synchroniser manuellement à la reconnexion | Tout utilisateur authentifié | HAUTE |
| UC-017-12 | Résoudre un conflit de synchronisation | Tout utilisateur authentifié | HAUTE |
| UC-017-13 | Consulter l'état de la synchronisation en temps réel | Tout utilisateur authentifié | HAUTE |
| UC-017-14 | Vider le cache local (libérer de l'espace) | Tout utilisateur authentifié | MOYENNE |
| UC-017-15 | Configurer le scanner depuis l'application | Archiviste | HAUTE |
| UC-017-16 | Numériser et indexer via scanner local (connecté) | Archiviste | HAUTE |
| UC-017-17 | Consulter les notifications système natives | Tout utilisateur authentifié | HAUTE |
| UC-017-18 | Ouvrir un document depuis l'explorateur de fichiers (association) | Tout utilisateur | MOYENNE |
| UC-017-19 | Mettre à jour l'application automatiquement | Système | HAUTE |
| UC-017-20 | Configurer la limite de volumétrie du cache | Tout utilisateur authentifié | HAUTE |

---

## 3. CAS D'UTILISATION DÉTAILLÉS

### UC-017-04 — Marquer des documents pour accès offline

**Acteur principal :** Tout utilisateur authentifié
**Pré-conditions :** Application connectée au serveur, document accessible selon les droits de l'utilisateur

**Flux principal :**
1. L'utilisateur consulte la liste des documents ou la fiche d'un document
2. Il clique sur l'icône "Disponible hors ligne" (nuage avec flèche) ou active le bouton toggle
3. L'application vérifie l'espace disponible dans le cache local
4. Si suffisant : téléchargement du document (fichier + métadonnées + historique de versions) vers le cache SQLite local
5. L'indicateur passe au vert "Disponible hors ligne"
6. Un enregistrement `SyncStatus` est créé ou mis à jour avec `offline_available = True`
7. La taille du cache local est mise à jour dans les préférences

**Flux alternatifs :**
- **3a.** Cache plein (> limite configurée) → dialogue "Votre cache est plein (X Go / Y Go). Retirer des documents pour libérer de l'espace ?"
- **3b.** Document > 100 Mo → avertissement "Ce document est volumineux (X Mo). Confirmer le téléchargement ?"
- **4a.** Erreur réseau pendant le téléchargement → retry automatique 3 fois, puis notification d'échec

**Post-conditions :**
- Document disponible dans le cache local chiffré
- `SyncStatus.offline_available = True`
- Indicateur visuel vert sur le document

---

### UC-017-07 — Numériser un document en mode offline

**Acteur principal :** Archiviste
**Pré-conditions :** Scanner physiquement connecté et reconnu par le système (TWAIN/SANE), application en mode offline

**Flux principal :**
1. L'archiviste accède à "Numérisation" depuis l'application desktop
2. L'application liste les scanners disponibles (via commande Rust TWAIN/SANE)
3. L'archiviste sélectionne le scanner et configure les paramètres (DPI, couleur, format)
4. Il place le document dans le scanner et clique "Numériser"
5. L'application reçoit les pages numérisées via l'interface Rust
6. Un aperçu est affiché page par page
7. L'archiviste saisit les métadonnées minimales : titre, catégorie approximative (obligatoire)
8. Il confirme
9. L'application crée une entrée `OfflineQueueItem` avec type `DOCUMENT_SCAN` contenant :
   - Les images numérisées (stockées dans le cache local chiffré)
   - Les métadonnées saisies
   - L'horodatage de numérisation
   - L'identifiant de l'appareil de numérisation
10. Un numéro temporaire local est attribué `LOCAL-2026-00042`
11. L'archiviste voit le document dans sa liste avec indicateur "En attente de synchronisation"

**À la synchronisation :**
- Le serveur reçoit les images et les métadonnées
- L'OCR est déclenché (MODULE 07)
- Un vrai numéro d'enregistrement est attribué
- Le numéro temporaire local est mis à jour

**Post-conditions :**
- Document numérisé stocké localement
- Action enregistrée dans la file d'attente offline
- À la reconnexion : document envoyé et indexé côté serveur

---

### UC-017-12 — Résoudre un conflit de synchronisation

**Acteur principal :** Tout utilisateur authentifié
**Pré-conditions :** Conflit détecté entre une modification offline et une modification serveur sur le même document

**Description du conflit :**
Un conflit survient quand un document a été modifié côté serveur (par un autre utilisateur) ET côté client offline (par l'utilisateur local) pendant la période de déconnexion.

**Flux principal :**
1. La synchronisation automatique détecte un conflit (timestamps divergents, hash différents)
2. Un `SyncConflict` est créé avec statut `PENDING`
3. L'utilisateur est notifié par une notification système native "1 conflit de synchronisation à résoudre"
4. Il accède au gestionnaire de conflits
5. L'interface affiche en regard :
   - **Version serveur** : qui a modifié, quand, quelles métadonnées
   - **Version locale** : ce que l'utilisateur a modifié offline
   - **Différences surlignées** champ par champ
6. L'utilisateur choisit l'une des options :
   - **Garder la version serveur** (annule les modifications locales)
   - **Garder la version locale** (écrase la version serveur avec confirmation)
   - **Fusionner manuellement** (éditer un résultat hybride champ par champ)
7. La décision est appliquée immédiatement
8. Le `SyncConflict` passe au statut `RESOLVED`
9. Un événement d'audit est généré (MODULE 11)

**Stratégie par défaut (si l'utilisateur ne résout pas dans 48h) :**
La stratégie "last write wins" s'applique automatiquement : la version la plus récente (horodatage serveur) est conservée. L'utilisateur est notifié de la résolution automatique.

---

### UC-017-19 — Mise à jour automatique de l'application

**Acteur principal :** Système (Tauri auto-updater)
**Pré-conditions :** Application connectée, serveur de mise à jour accessible

**Flux principal :**
1. Au démarrage ou toutes les 24h, l'application vérifie le serveur de mise à jour
2. Si une nouvelle version est disponible :
   - Une notification système native s'affiche "Mise à jour disponible (v2.1.0)"
   - Le changelog est affiché
3. L'utilisateur choisit "Mettre à jour maintenant" ou "Plus tard"
4. Si "Maintenant" : téléchargement en arrière-plan, vérification du hash SHA-256 du binaire
5. Installation silencieuse avec relance automatique de l'application
6. L'ancienne version est conservée 48h (rollback possible si problème)

**Règle de sécurité :** Les mises à jour sont signées avec la clé privée de l'éditeur. L'application vérifie la signature avant installation. Une mise à jour dont la signature ne correspond pas est rejetée et journalisée.

---

## 4. MODÈLES DE DONNÉES

### 4.1 SyncStatus — État de synchronisation par document

```python
class SyncStatus(BaseModel):
    """
    État de synchronisation d'un document sur un appareil spécifique.
    Un document peut avoir plusieurs SyncStatus (un par appareil de l'utilisateur).
    Stocké côté SERVEUR pour centraliser le suivi.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    # Document et utilisateur concernés
    document = models.ForeignKey(
        'documents.Document',
        on_delete=models.CASCADE,
        related_name='sync_statuses'
    )
    user = models.ForeignKey(
        'users.User',
        on_delete=models.CASCADE,
        related_name='sync_statuses'
    )

    # Identifiant unique de l'appareil (généré à l'installation de l'app desktop)
    device_id = models.CharField(
        max_length=64,
        db_index=True,
        help_text="UUID v4 généré à l'installation et stocké dans le keychain OS"
    )
    device_name = models.CharField(
        max_length=200,
        blank=True,
        help_text="Nom lisible de l'appareil (ex: 'Laptop-Archiviste-01')"
    )
    device_platform = models.CharField(
        max_length=20,
        choices=[
            ('WINDOWS', 'Windows'),
            ('LINUX', 'Linux'),
            ('MACOS', 'macOS'),
        ],
        blank=True
    )

    # État du cache local
    offline_available = models.BooleanField(
        default=False,
        help_text="Document marqué pour accès offline et présent dans le cache"
    )
    cached_at = models.DateTimeField(
        null=True, blank=True,
        help_text="Date de mise en cache sur cet appareil"
    )
    cached_version_id = models.UUIDField(
        null=True, blank=True,
        help_text="Version du document actuellement en cache"
    )
    cached_file_hash = models.CharField(
        max_length=64, blank=True,
        help_text="Hash SHA-256 du fichier en cache (vérification d'intégrité)"
    )

    # Synchronisation
    last_synced_at = models.DateTimeField(
        null=True, blank=True,
        help_text="Date de la dernière synchronisation réussie"
    )
    sync_status = models.CharField(
        max_length=20,
        choices=[
            ('SYNCED', 'Synchronisé'),
            ('PENDING_UPLOAD', 'En attente d\'envoi vers le serveur'),
            ('PENDING_DOWNLOAD', 'En attente de téléchargement'),
            ('CONFLICT', 'Conflit à résoudre'),
            ('ERROR', 'Erreur de synchronisation'),
        ],
        default='SYNCED',
        db_index=True
    )
    sync_error_message = models.TextField(blank=True)

    # Statistiques d'accès offline
    offline_access_count = models.PositiveIntegerField(default=0)
    last_offline_access_at = models.DateTimeField(null=True, blank=True)

    # Horodatage
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='deleted_sync_statuses'
    )

    class Meta:
        db_table = 'sync_statuses'
        indexes = [
            models.Index(fields=['user', 'device_id', 'sync_status']),
            models.Index(fields=['document', 'device_id']),
            models.Index(fields=['offline_available', 'user']),
        ]
        constraints = [
            models.UniqueConstraint(
                fields=['document', 'user', 'device_id'],
                condition=models.Q(is_deleted=False),
                name='unique_sync_status_per_doc_user_device'
            ),
        ]
```

---

### 4.2 SyncConflict — Conflits de synchronisation

```python
class SyncConflict(BaseModel):
    """
    Conflit détecté entre une version locale offline et la version serveur.
    Nécessite une résolution manuelle ou automatique (last write wins après 48h).
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    # Contexte du conflit
    document = models.ForeignKey(
        'documents.Document',
        on_delete=models.CASCADE,
        related_name='sync_conflicts'
    )
    user = models.ForeignKey(
        'users.User',
        on_delete=models.CASCADE,
        related_name='sync_conflicts'
    )
    device_id = models.CharField(max_length=64)

    # Les deux versions en conflit
    server_version_id = models.UUIDField(
        help_text="Version du document sur le serveur au moment de la détection"
    )
    local_version_snapshot = models.JSONField(
        help_text="Snapshot JSON des données locales au moment du conflit"
    )
    server_version_snapshot = models.JSONField(
        help_text="Snapshot JSON des données serveur au moment du conflit"
    )

    # Différences calculées
    conflicting_fields = models.JSONField(
        default=list,
        help_text="Liste des champs en conflit : ['title', 'metadata', 'file_content']"
    )

    # Statut de résolution
    status = models.CharField(
        max_length=20,
        choices=[
            ('PENDING', 'En attente de résolution'),
            ('RESOLVED_SERVER', 'Résolu : version serveur conservée'),
            ('RESOLVED_LOCAL', 'Résolu : version locale appliquée'),
            ('RESOLVED_MERGE', 'Résolu : fusion manuelle'),
            ('RESOLVED_AUTO', 'Résolu automatiquement (last write wins)'),
        ],
        default='PENDING',
        db_index=True
    )
    resolved_at = models.DateTimeField(null=True, blank=True)
    resolved_by = models.ForeignKey(
        'users.User',
        on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='resolved_conflicts'
    )
    resolution_notes = models.TextField(blank=True)

    # Expiration auto-résolution
    auto_resolve_at = models.DateTimeField(
        help_text="Date à laquelle la résolution automatique s'applique (created_at + 48h)"
    )

    # Horodatage
    detected_at = models.DateTimeField(auto_now_add=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='deleted_sync_conflicts'
    )

    class Meta:
        db_table = 'sync_conflicts'
        indexes = [
            models.Index(fields=['user', 'status']),
            models.Index(fields=['document', 'status']),
            models.Index(fields=['status', 'auto_resolve_at']),
        ]
```

---

### 4.3 OfflineQueueItem — File d'attente des actions offline

```python
class OfflineQueueItem(BaseModel):
    """
    Action réalisée en mode offline, en attente de synchronisation vers le serveur.
    Stocké côté SERVEUR une fois transmis, ou dans SQLite local tant que déconnecté.
    Ce modèle représente la version serveur (reçue lors de la sync).
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    # Identifiant local temporaire (attribué par l'app desktop avant sync)
    local_id = models.CharField(
        max_length=64,
        help_text="ID temporaire généré côté client (ex: LOCAL-2026-00042)"
    )

    user = models.ForeignKey(
        'users.User',
        on_delete=models.CASCADE,
        related_name='offline_queue_items'
    )
    device_id = models.CharField(max_length=64)

    # Type d'action
    action_type = models.CharField(
        max_length=30,
        choices=[
            ('DOCUMENT_SCAN', 'Numérisation d\'un nouveau document'),
            ('DOCUMENT_METADATA_UPDATE', 'Mise à jour métadonnées'),
            ('DOCUMENT_ANNOTATION', 'Ajout d\'une annotation'),
            ('DOCUMENT_COMMENT', 'Ajout d\'un commentaire'),
            ('DOCUMENT_UPLOAD', 'Upload d\'un fichier existant'),
            ('FOLDER_CREATE', 'Création d\'un dossier'),
        ],
        db_index=True
    )

    # Données de l'action
    payload = models.JSONField(
        help_text="Données de l'action sérialisées (métadonnées, commentaire, etc.)"
    )
    file_references = models.JSONField(
        default=list,
        help_text="Références aux fichiers attachés (stockés dans le cache local)"
    )

    # Horodatage côté client (important pour la résolution de conflits)
    performed_at_client = models.DateTimeField(
        help_text="Horodatage de l'action côté client (peut différer de created_at serveur)"
    )

    # Statut de traitement côté serveur
    status = models.CharField(
        max_length=20,
        choices=[
            ('RECEIVED', 'Reçu, en attente de traitement'),
            ('PROCESSING', 'En cours de traitement'),
            ('COMPLETED', 'Traité avec succès'),
            ('FAILED', 'Échec du traitement'),
            ('CONFLICT', 'Conflit détecté'),
        ],
        default='RECEIVED',
        db_index=True
    )

    # Résultat après traitement
    server_document_id = models.UUIDField(
        null=True, blank=True,
        help_text="ID serveur du document créé/modifié (pour mise à jour du client)"
    )
    processing_error = models.TextField(blank=True)

    # Horodatage
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    processed_at = models.DateTimeField(null=True, blank=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='deleted_queue_items'
    )

    class Meta:
        db_table = 'offline_queue_items'
        indexes = [
            models.Index(fields=['user', 'status', 'created_at']),
            models.Index(fields=['device_id', 'status']),
            models.Index(fields=['action_type', 'status']),
        ]
```

---

### 4.4 SyncLog — Historique des synchronisations

```python
class SyncLog(BaseModel):
    """
    Historique des sessions de synchronisation.
    Une session = une connexion de l'app desktop avec synchronisation.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    user = models.ForeignKey(
        'users.User',
        on_delete=models.CASCADE,
        related_name='sync_logs'
    )
    device_id = models.CharField(max_length=64, db_index=True)
    device_name = models.CharField(max_length=200, blank=True)

    # Métriques de la session de sync
    sync_type = models.CharField(
        max_length=20,
        choices=[
            ('AUTO', 'Automatique (reconnexion)'),
            ('MANUAL', 'Déclenchée manuellement'),
            ('SCHEDULED', 'Planifiée'),
        ]
    )
    status = models.CharField(
        max_length=20,
        choices=[
            ('COMPLETED', 'Terminée'),
            ('PARTIAL', 'Partielle (erreurs non bloquantes)'),
            ('FAILED', 'Échouée'),
        ],
        db_index=True
    )

    # Statistiques
    items_uploaded = models.PositiveIntegerField(default=0)
    items_downloaded = models.PositiveIntegerField(default=0)
    conflicts_detected = models.PositiveIntegerField(default=0)
    conflicts_auto_resolved = models.PositiveIntegerField(default=0)
    bytes_uploaded = models.BigIntegerField(default=0)
    bytes_downloaded = models.BigIntegerField(default=0)
    duration_seconds = models.FloatField(null=True, blank=True)

    error_details = models.TextField(blank=True)

    # Horodatage
    started_at = models.DateTimeField()
    completed_at = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='deleted_sync_logs'
    )

    class Meta:
        db_table = 'sync_logs'
        indexes = [
            models.Index(fields=['user', 'started_at']),
            models.Index(fields=['device_id', 'started_at']),
            models.Index(fields=['status']),
        ]
```

---

## 5. ARCHITECTURE TAURI

### 5.1 Structure du projet Tauri

```
desktop/
├── src-tauri/                    # Code Rust (backend natif Tauri)
│   ├── Cargo.toml
│   ├── tauri.conf.json           # Configuration Tauri (permissions, titre, icône)
│   ├── build.rs
│   └── src/
│       ├── main.rs               # Point d'entrée Rust
│       ├── commands/             # Commandes Tauri exposées au frontend
│       │   ├── mod.rs
│       │   ├── auth.rs           # Gestion tokens JWT (keychain OS)
│       │   ├── scanner.rs        # Intégration TWAIN/SANE
│       │   ├── sync.rs           # Synchronisation bidirectionnelle
│       │   ├── cache.rs          # Gestion cache SQLite local
│       │   ├── files.rs          # Accès système de fichiers
│       │   └── notifications.rs  # Notifications système natives
│       ├── db/                   # Base SQLite locale
│       │   ├── mod.rs
│       │   ├── migrations.rs     # Migrations SQLite (rusqlite)
│       │   └── models.rs         # Modèles SQLite locaux
│       ├── scanner/              # Intégration scanners
│       │   ├── mod.rs
│       │   ├── twain.rs          # Interface TWAIN (Windows)
│       │   └── sane.rs           # Interface SANE (Linux)
│       └── crypto.rs             # Chiffrement cache local (AES-256)
│
├── src/                          # Frontend React/TypeScript (partagé avec web)
│   ├── main.tsx
│   ├── App.tsx
│   ├── components/
│   │   ├── desktop/              # Composants spécifiques desktop
│   │   │   ├── SyncStatusBar.tsx
│   │   │   ├── OfflineIndicator.tsx
│   │   │   ├── ConflictResolver.tsx
│   │   │   ├── ScannerPanel.tsx
│   │   │   └── SyncQueue.tsx
│   │   └── shared/               # Composants partagés avec l'app web
│   ├── hooks/
│   │   ├── useSync.ts            # Hook de synchronisation
│   │   ├── useOfflineQueue.ts    # Gestion file d'attente offline
│   │   ├── useScanner.ts         # Interface scanner via Tauri commands
│   │   └── useNetworkStatus.ts   # Détection connexion réseau
│   ├── stores/
│   │   ├── syncStore.ts          # État global sync (Zustand)
│   │   └── offlineStore.ts       # Documents disponibles offline
│   └── services/
│       ├── tauriApi.ts           # Appels aux commandes Tauri Rust
│       └── syncService.ts        # Logique de synchronisation
│
├── package.json
└── pnpm-lock.yaml
```

---

### 5.2 Schéma SQLite local (côté client desktop)

La base SQLite locale (`~/.local/share/archivage/local.db` sur Linux, `%APPDATA%\Archivage\local.db` sur Windows) est chiffrée avec SQLCipher (AES-256) en utilisant la clé dérivée des credentials de l'utilisateur.

```sql
-- Documents mis en cache
CREATE TABLE IF NOT EXISTS cached_documents (
    id TEXT PRIMARY KEY,              -- UUID serveur
    local_id TEXT,                    -- ID temporaire si créé offline
    title TEXT NOT NULL,
    reference TEXT,
    category_id TEXT,
    category_name TEXT,
    document_type TEXT,
    confidentiality_level TEXT,
    lifecycle_status TEXT,
    processing_status TEXT,
    metadata_json TEXT,               -- JSON des métadonnées
    file_path TEXT,                   -- Chemin local du fichier chiffré
    file_hash TEXT,                   -- SHA-256 du fichier en cache
    file_size_bytes INTEGER,
    thumbnail_path TEXT,              -- Miniature pour affichage liste
    version_id TEXT,
    version_number TEXT,
    cached_at TEXT NOT NULL,          -- ISO 8601
    last_synced_at TEXT,
    is_modified_offline INTEGER DEFAULT 0,  -- 1 si modifié localement
    offline_modifications_json TEXT,  -- JSON des modifications locales
    sync_status TEXT DEFAULT 'SYNCED'
);

-- File d'attente des actions offline
CREATE TABLE IF NOT EXISTS offline_queue (
    id TEXT PRIMARY KEY,              -- UUID local
    action_type TEXT NOT NULL,
    document_id TEXT,                 -- Null si nouveau document
    payload_json TEXT NOT NULL,
    file_paths_json TEXT,             -- Chemins des fichiers à uploader
    performed_at TEXT NOT NULL,       -- ISO 8601
    retry_count INTEGER DEFAULT 0,
    status TEXT DEFAULT 'PENDING',    -- PENDING, UPLOADING, COMPLETED, FAILED
    error_message TEXT,
    server_id TEXT,                   -- Rempli après sync réussie
    created_at TEXT NOT NULL
);

-- Conflits en attente de résolution
CREATE TABLE IF NOT EXISTS sync_conflicts (
    id TEXT PRIMARY KEY,
    document_id TEXT NOT NULL,
    conflicting_fields_json TEXT,
    local_snapshot_json TEXT,
    server_snapshot_json TEXT,
    detected_at TEXT NOT NULL,
    auto_resolve_at TEXT NOT NULL,    -- detected_at + 48h
    status TEXT DEFAULT 'PENDING'
);

-- Dossiers marqués pour offline
CREATE TABLE IF NOT EXISTS offline_folders (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    parent_id TEXT,
    cached_at TEXT NOT NULL,
    document_count INTEGER DEFAULT 0
);

-- Métadonnées de synchronisation
CREATE TABLE IF NOT EXISTS sync_metadata (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    updated_at TEXT NOT NULL
);
-- Exemples de clés : last_sync_at, server_version, device_id, cache_size_bytes

-- Index pour performances
CREATE INDEX IF NOT EXISTS idx_cached_docs_status ON cached_documents(sync_status);
CREATE INDEX IF NOT EXISTS idx_offline_queue_status ON offline_queue(status);
CREATE INDEX IF NOT EXISTS idx_conflicts_status ON sync_conflicts(status);
```

---

### 5.3 Commandes Tauri Rust (interface JS ↔ Rust)

```rust
// src-tauri/src/commands/scanner.rs

use tauri::command;
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
pub struct ScannerDevice {
    pub id: String,
    pub name: String,
    pub vendor: String,
    pub model: String,
    pub connection_type: String,  // USB, Network
    pub available: bool,
}

#[derive(Serialize, Deserialize)]
pub struct ScanParameters {
    pub dpi: u32,           // 150, 300, 600
    pub color_mode: String, // Color, Grayscale, BlackWhite
    pub paper_size: String, // A4, A3, Letter, Auto
    pub duplex: bool,
    pub page_count: Option<u32>,  // None = toutes les pages
}

#[derive(Serialize, Deserialize)]
pub struct ScanResult {
    pub pages: Vec<ScannedPage>,
    pub total_pages: u32,
    pub scan_duration_ms: u64,
}

#[derive(Serialize, Deserialize)]
pub struct ScannedPage {
    pub page_number: u32,
    pub file_path: String,   // Chemin temporaire du fichier image
    pub width_px: u32,
    pub height_px: u32,
    pub dpi: u32,
    pub file_size_bytes: u64,
}

/// Liste tous les scanners disponibles sur le système
#[command]
pub async fn list_scanners() -> Result<Vec<ScannerDevice>, String> {
    #[cfg(target_os = "windows")]
    {
        crate::scanner::twain::list_devices()
            .map_err(|e| format!("Erreur TWAIN : {}", e))
    }
    #[cfg(target_os = "linux")]
    {
        crate::scanner::sane::list_devices()
            .map_err(|e| format!("Erreur SANE : {}", e))
    }
    #[cfg(target_os = "macos")]
    {
        crate::scanner::twain::list_devices()
            .map_err(|e| format!("Erreur TWAIN : {}", e))
    }
}

/// Déclenche une numérisation avec les paramètres fournis
#[command]
pub async fn scan_document(
    device_id: String,
    params: ScanParameters,
    output_dir: String,
) -> Result<ScanResult, String> {
    let output_path = std::path::PathBuf::from(&output_dir);

    #[cfg(target_os = "windows")]
    let result = crate::scanner::twain::scan(&device_id, &params, &output_path);
    #[cfg(target_os = "linux")]
    let result = crate::scanner::sane::scan(&device_id, &params, &output_path);

    result.map_err(|e| format!("Erreur numérisation : {}", e))
}

/// Teste la connexion à un scanner
#[command]
pub async fn test_scanner(device_id: String) -> Result<bool, String> {
    #[cfg(target_os = "windows")]
    return crate::scanner::twain::test_device(&device_id)
        .map_err(|e| format!("Erreur : {}", e));
    #[cfg(target_os = "linux")]
    return crate::scanner::sane::test_device(&device_id)
        .map_err(|e| format!("Erreur : {}", e));
}
```

```rust
// src-tauri/src/commands/auth.rs

use keyring::Entry;  // Accès au keychain OS (Windows Credential Manager, macOS Keychain, Linux Secret Service)
use tauri::command;

/// Stocke le token JWT dans le keychain sécurisé de l'OS
#[command]
pub fn store_tokens(
    access_token: String,
    refresh_token: String,
    user_id: String,
) -> Result<(), String> {
    let access_entry = Entry::new("archivage_numerique", &format!("access_{}", user_id))
        .map_err(|e| e.to_string())?;
    access_entry.set_password(&access_token)
        .map_err(|e| format!("Erreur stockage access token : {}", e))?;

    let refresh_entry = Entry::new("archivage_numerique", &format!("refresh_{}", user_id))
        .map_err(|e| e.to_string())?;
    refresh_entry.set_password(&refresh_token)
        .map_err(|e| format!("Erreur stockage refresh token : {}", e))?;

    Ok(())
}

/// Récupère les tokens depuis le keychain
#[command]
pub fn get_tokens(user_id: String) -> Result<(String, String), String> {
    let access_entry = Entry::new("archivage_numerique", &format!("access_{}", user_id))
        .map_err(|e| e.to_string())?;
    let access_token = access_entry.get_password()
        .map_err(|_| "Token non trouvé".to_string())?;

    let refresh_entry = Entry::new("archivage_numerique", &format!("refresh_{}", user_id))
        .map_err(|e| e.to_string())?;
    let refresh_token = refresh_entry.get_password()
        .map_err(|_| "Refresh token non trouvé".to_string())?;

    Ok((access_token, refresh_token))
}

/// Supprime les tokens du keychain (déconnexion)
#[command]
pub fn clear_tokens(user_id: String) -> Result<(), String> {
    let _ = Entry::new("archivage_numerique", &format!("access_{}", user_id))
        .and_then(|e| e.delete_password());
    let _ = Entry::new("archivage_numerique", &format!("refresh_{}", user_id))
        .and_then(|e| e.delete_password());
    Ok(())
}
```

```rust
// src-tauri/src/commands/sync.rs

use tauri::{command, AppHandle};
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
pub struct SyncProgress {
    pub phase: String,          // "upload", "download", "conflict_check", "complete"
    pub items_total: u32,
    pub items_processed: u32,
    pub bytes_transferred: u64,
    pub conflicts_found: u32,
    pub errors: Vec<String>,
}

/// Déclenche une synchronisation complète et émet des événements de progression
#[command]
pub async fn start_sync(
    app_handle: AppHandle,
    server_url: String,
    access_token: String,
) -> Result<SyncProgress, String> {
    let client = reqwest::Client::new();

    // Phase 1 : Upload de la file d'attente offline
    app_handle.emit_all("sync_progress", &SyncProgress {
        phase: "upload".to_string(),
        items_total: 0,
        items_processed: 0,
        bytes_transferred: 0,
        conflicts_found: 0,
        errors: vec![],
    }).ok();

    // Récupérer les items de la file d'attente depuis SQLite local
    let queue_items = crate::db::get_pending_queue_items()?;
    let total = queue_items.len() as u32;

    for (i, item) in queue_items.iter().enumerate() {
        match upload_queue_item(&client, &server_url, &access_token, item).await {
            Ok(server_id) => {
                crate::db::mark_queue_item_completed(&item.id, &server_id)?;
            }
            Err(e) => {
                crate::db::mark_queue_item_failed(&item.id, &e.to_string())?;
            }
        }

        app_handle.emit_all("sync_progress", &SyncProgress {
            phase: "upload".to_string(),
            items_total: total,
            items_processed: (i + 1) as u32,
            bytes_transferred: 0,
            conflicts_found: 0,
            errors: vec![],
        }).ok();
    }

    // Phase 2 : Téléchargement des mises à jour serveur
    // ... (logique de téléchargement des documents modifiés côté serveur)

    // Phase 3 : Détection et notification des conflits
    // ... (comparaison des versions)

    Ok(SyncProgress {
        phase: "complete".to_string(),
        items_total: total,
        items_processed: total,
        bytes_transferred: 0,
        conflicts_found: 0,
        errors: vec![],
    })
}
```

---

### 5.4 Hook React de synchronisation

```typescript
// src/hooks/useSync.ts

import { invoke } from '@tauri-apps/api/tauri';
import { listen } from '@tauri-apps/api/event';
import { useState, useEffect, useCallback } from 'react';
import { useSyncStore } from '../stores/syncStore';

interface SyncProgress {
  phase: 'upload' | 'download' | 'conflict_check' | 'complete';
  items_total: number;
  items_processed: number;
  bytes_transferred: number;
  conflicts_found: number;
  errors: string[];
}

export function useSync() {
  const { setProgress, setSyncing, setLastSyncAt, addConflict } = useSyncStore();
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  // Écouter les événements de progression Tauri
  useEffect(() => {
    const unlistenProgress = listen<SyncProgress>('sync_progress', (event) => {
      setProgress(event.payload);
      if (event.payload.phase === 'complete') {
        setSyncing(false);
        setLastSyncAt(new Date());
      }
    });

    // Écouter les changements de connexion réseau
    const handleOnline = () => {
      setIsOnline(true);
      // Sync automatique à la reconnexion
      triggerSync('AUTO');
    };
    const handleOffline = () => setIsOnline(false);

    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);

    return () => {
      unlistenProgress.then(fn => fn());
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);

  const triggerSync = useCallback(async (syncType: 'AUTO' | 'MANUAL') => {
    if (!isOnline) return;

    setSyncing(true);

    try {
      const serverUrl = localStorage.getItem('server_url') || '';
      const userId = localStorage.getItem('user_id') || '';

      // Récupérer le token depuis le keychain Rust
      const [accessToken] = await invoke<[string, string]>('get_tokens', { userId });

      await invoke('start_sync', { serverUrl, accessToken });
    } catch (error) {
      setSyncing(false);
      console.error('Erreur synchronisation:', error);
    }
  }, [isOnline]);

  return { isOnline, triggerSync };
}
```

---

### 5.5 Store Zustand pour l'état offline

```typescript
// src/stores/offlineStore.ts

import { create } from 'zustand';
import { invoke } from '@tauri-apps/api/tauri';

interface CachedDocument {
  id: string;
  localId?: string;
  title: string;
  reference: string;
  filePath: string;
  thumbnailPath?: string;
  cachedAt: string;
  syncStatus: 'SYNCED' | 'PENDING_UPLOAD' | 'CONFLICT' | 'ERROR';
  isModifiedOffline: boolean;
  offlineSizeBytes: number;
}

interface OfflineQueueItem {
  id: string;
  actionType: string;
  documentId?: string;
  status: 'PENDING' | 'UPLOADING' | 'COMPLETED' | 'FAILED';
  performedAt: string;
  retryCount: number;
}

interface OfflineStore {
  cachedDocuments: CachedDocument[];
  queueItems: OfflineQueueItem[];
  totalCacheSizeBytes: number;
  maxCacheSizeBytes: number;

  // Actions
  loadCachedDocuments: () => Promise<void>;
  markDocumentForOffline: (documentId: string) => Promise<void>;
  removeDocumentFromCache: (documentId: string) => Promise<void>;
  addToQueue: (item: Omit<OfflineQueueItem, 'id' | 'status'>) => Promise<void>;
  clearCache: () => Promise<void>;
  updateCacheLimit: (limitBytes: number) => void;
}

export const useOfflineStore = create<OfflineStore>((set, get) => ({
  cachedDocuments: [],
  queueItems: [],
  totalCacheSizeBytes: 0,
  maxCacheSizeBytes: 5 * 1024 * 1024 * 1024, // 5 Go par défaut

  loadCachedDocuments: async () => {
    const docs = await invoke<CachedDocument[]>('get_cached_documents');
    const queue = await invoke<OfflineQueueItem[]>('get_queue_items');
    const stats = await invoke<{ total_bytes: number }>('get_cache_stats');

    set({
      cachedDocuments: docs,
      queueItems: queue,
      totalCacheSizeBytes: stats.total_bytes,
    });
  },

  markDocumentForOffline: async (documentId: string) => {
    const { maxCacheSizeBytes, totalCacheSizeBytes } = get();

    // Vérifier l'espace avant le téléchargement
    const docInfo = await invoke<{ size_bytes: number }>('get_document_size', { documentId });

    if (totalCacheSizeBytes + docInfo.size_bytes > maxCacheSizeBytes) {
      throw new Error('Cache plein : libérez de l\'espace avant d\'ajouter des documents');
    }

    await invoke('cache_document', { documentId });
    await get().loadCachedDocuments();
  },

  removeDocumentFromCache: async (documentId: string) => {
    await invoke('remove_from_cache', { documentId });
    await get().loadCachedDocuments();
  },

  addToQueue: async (item) => {
    const id = crypto.randomUUID();
    await invoke('add_to_offline_queue', { ...item, id });
    await get().loadCachedDocuments();
  },

  clearCache: async () => {
    await invoke('clear_all_cache');
    set({ cachedDocuments: [], totalCacheSizeBytes: 0 });
  },

  updateCacheLimit: (limitBytes: number) => {
    set({ maxCacheSizeBytes: limitBytes });
    invoke('save_cache_limit', { limitBytes });
  },
}));
```

---

## 6. MATRICE DES PERMISSIONS

| Action | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|--------|:-----------:|:-----:|:----------:|:-----------:|:-----:|:--------:|
| Installer l'application desktop | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Se connecter via l'app desktop | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Marquer documents pour offline | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Marquer dossier complet pour offline | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| Consulter documents en mode offline | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Numériser en mode offline | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Modifier métadonnées en mode offline | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| Annoter un document offline | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Déclencher synchronisation manuelle | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Résoudre un conflit de synchronisation | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Vider le cache local | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Configurer limite cache | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Configurer scanner | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Accéder aux logs de synchronisation | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |

---

## 7. SÉQUENCES DÉTAILLÉES

### Séquence 1 — Cycle complet offline : déconnexion → travail → reconnexion → sync

```
Archiviste (Desktop)       App Tauri                   Serveur Django
        |                      |                              |
        |  [Réseau disponible] |                              |
        |-- Lance app -------->|                              |
        |                      |-- Vérifie token JWT -------->|
        |                      |<-- Token valide ------------|
        |                      |-- Sync initiale (delta) ---->|
        |                      |<-- Docs mis à jour ---------|
        |<-- App prête --------|                              |
        |                      |                              |
        |  [Réseau coupé]      |                              |
        |                      |-- Détecte offline           |
        |                      |-- Bascule mode offline       |
        |<-- Indicateur offline (barre rouge)                 |
        |                      |                              |
        |-- Consulte doc A --->|                              |
        |                      |-- Lit depuis cache SQLite    |
        |<-- Document affiché--|                              |
        |                      |                              |
        |-- Numérisation ------>|                              |
        |   (nouveau doc)       |-- Scanner TWAIN/SANE ------>|
        |                      |<-- Pages numérisées ---------|
        |                      |-- Stocke dans cache local    |
        |                      |-- Ajoute à offline_queue     |
        |<-- "En attente sync" |                              |
        |                      |                              |
        |  [Réseau rétabli]    |                              |
        |                      |-- Détecte reconnexion       |
        |<-- Notification "Réseau rétabli — Sync en cours"   |
        |                      |                              |
        |                      |-- Upload offline_queue ------>|
        |                      |   (doc numérisé + métadata) |
        |                      |<-- Accusé réception + IDs --|
        |                      |                              |
        |                      |-- Télécharge delta serveur ->|
        |                      |<-- Mises à jour depuis sync -|
        |                      |                              |
        |                      |-- Mise à jour SQLite local   |
        |                      |-- SyncLog créé (serveur)     |
        |                      |                              |
        |<-- Notification "Sync terminée : 3 items envoyés"  |
```

---

### Séquence 2 — Résolution d'un conflit de synchronisation

```
Archiviste               App Tauri                    Serveur
     |                      |                             |
     |  [Pendant offline]   |                             |
     |-- Modifie titre doc ->|                             |
     |   "Rapport v2"        |-- Stocke en SQLite local   |
     |                      |-- Marque MODIFIED_OFFLINE   |
     |                      |                             |
     |  [Autre utilisateur côté serveur]                  |
     |                      |                   [Admin modifie le même doc]
     |                      |                   titre → "Rapport final 2026"
     |                      |                             |
     |  [Reconnexion]       |                             |
     |                      |-- Tente upload ------------>|
     |                      |<-- 409 Conflict ------------|
     |                      |   {                         |
     |                      |     server_version: {...},  |
     |                      |     conflicting_fields:     |
     |                      |       ['title']             |
     |                      |   }                         |
     |                      |                             |
     |                      |-- Crée SyncConflict (local) |
     |<-- Notification native "1 conflit à résoudre"      |
     |                      |                             |
     |-- Ouvre gestionnaire->|                             |
     |<-- Affiche comparaison                              |
     |   LOCAL  : "Rapport v2"                             |
     |   SERVEUR: "Rapport final 2026"                     |
     |   Modifié par: Admin, le 18/02 à 14h32             |
     |                      |                             |
     |-- Choisit SERVEUR --->|                             |
     |   (annule sa modif)   |                             |
     |                      |-- Met à jour SQLite local   |
     |                      |-- POST /api/v1/sync/conflicts/{id}/resolve/
     |                      |<-- OK ----------------------|
     |                      |                             |
     |<-- "Conflit résolu"  |                             |
```

---

## 8. ENDPOINTS API (côté serveur Django)

Ces endpoints sont consommés exclusivement par l'application Tauri.

### 8.1 Synchronisation

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `POST` | `/api/v1/sync/init/` | Initialisation : enregistrer un appareil | JWT |
| `GET` | `/api/v1/sync/delta/` | Obtenir le delta depuis le dernier sync | JWT + Device-ID header |
| `POST` | `/api/v1/sync/upload/` | Uploader la file d'attente offline | JWT |
| `POST` | `/api/v1/sync/upload/files/` | Uploader les fichiers binaires (multipart) | JWT |
| `GET` | `/api/v1/sync/status/` | État de la synchronisation de l'appareil | JWT + Device-ID header |
| `POST` | `/api/v1/sync/heartbeat/` | Signaler que l'appareil est connecté | JWT |

### 8.2 Documents offline

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/offline/documents/` | Documents marqués pour offline | JWT |
| `POST` | `/api/v1/offline/documents/{id}/mark/` | Marquer un document pour offline | JWT |
| `DELETE` | `/api/v1/offline/documents/{id}/unmark/` | Retirer du cache | JWT |
| `GET` | `/api/v1/offline/documents/{id}/package/` | Télécharger le package offline (fichier + métadonnées) | JWT |

### 8.3 Conflits

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/sync/conflicts/` | Lister les conflits en attente | JWT |
| `GET` | `/api/v1/sync/conflicts/{id}/` | Détail d'un conflit | JWT |
| `POST` | `/api/v1/sync/conflicts/{id}/resolve/` | Résoudre un conflit | JWT |

### 8.4 Logs de synchronisation

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/sync/logs/` | Historique des sessions de synchronisation | JWT |
| `POST` | `/api/v1/sync/logs/` | Créer un log de session (envoyé par le client) | JWT |

### 8.5 Appareils enregistrés

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/sync/devices/` | Appareils de l'utilisateur | JWT |
| `DELETE` | `/api/v1/sync/devices/{device_id}/` | Révoquer un appareil | JWT |
| `GET` | `/api/v1/sync/devices/{device_id}/status/` | État d'un appareil spécifique | JWT |

---

## 9. RÈGLES MÉTIER CRITIQUES

### RB-017-01 — Limite de cache obligatoire et configurable

Le cache local est limité en volumétrie. La limite par défaut est **5 Go**, configurable par l'utilisateur entre 1 Go et 50 Go. Toute tentative de mise en cache dépassant la limite génère un dialogue de confirmation non contournable. Le système ne permet jamais de dépasser automatiquement la limite.

---

### RB-017-02 — Chiffrement obligatoire du cache local

La base SQLite locale (`local.db`) est chiffrée avec SQLCipher (AES-256). La clé de chiffrement est dérivée des credentials de l'utilisateur et stockée dans le keychain sécurisé de l'OS (Windows Credential Manager, macOS Keychain, Linux Secret Service via libsecret). Les fichiers documents mis en cache sont stockés chiffrés avec une clé spécifique par appareil.

---

### RB-017-03 — Résolution automatique des conflits après 48h

Tout conflit non résolu manuellement dans les 48 heures est résolu automatiquement par la stratégie "last write wins" : la version ayant l'horodatage serveur le plus récent est conservée. L'utilisateur est notifié de la résolution automatique avec un résumé de ce qui a été écrasé.

---

### RB-017-04 — Actions offline limitées selon le rôle

En mode offline, certaines actions sont restreintes même si l'utilisateur a les droits en ligne. Un Agent ne peut pas créer de documents en mode offline (numérisations réservées aux Archivistes). Les modifications de configuration (MODULE 15) sont bloquées en mode offline. Les actions de workflow (approuver, rejeter) sont bloquées en mode offline.

---

### RB-017-05 — Intégrité des fichiers en cache

Lors de chaque ouverture d'un document en cache, l'application vérifie le hash SHA-256 du fichier local contre le hash enregistré dans SQLite. Une divergence indique une corruption du cache ; le fichier est supprimé et une notification est affichée "Ce document a été supprimé du cache (corruption détectée). Reconnectez-vous pour le retélécharger."

---

### RB-017-06 — Authentification offline avec expiration

Le JWT access token est stocké dans le keychain OS. En mode offline, l'utilisateur peut rester authentifié pendant la durée du refresh token (configurable, défaut 7 jours). Au-delà, une reconnexion au serveur est requise pour renouveler les tokens. L'application ne conserve jamais le mot de passe de l'utilisateur localement.

---

### RB-017-07 — Mise à jour signée obligatoirement

Les binaires de mise à jour sont signés avec la clé privée Ed25519 de l'éditeur. L'application vérifie la signature avant toute installation. Une signature invalide bloque l'installation et journalise une alerte de sécurité. Ce mécanisme est non désactivable et non contournable.

---

## 10. TÂCHES CELERY PLANIFIÉES (côté serveur)

### TASK-017-01 : Résolution automatique des conflits expirés
```python
@shared_task(name='sync.auto_resolve_expired_conflicts')
# Fréquence     : Toutes les heures
# Description   : Résout automatiquement les SyncConflict dont auto_resolve_at < now()
#                 Stratégie : conserve la version la plus récente (horodatage serveur)
# Notification  : Utilisateur notifié de la résolution automatique (MODULE 12)
```

### TASK-017-02 : Nettoyage des appareils inactifs
```python
@shared_task(name='sync.cleanup_inactive_devices')
# Fréquence     : Hebdomadaire (dimanche 5h00)
# Description   : Marque comme inactifs les appareils sans heartbeat > 90 jours
#                 Supprime les SyncStatus associés (soft delete)
# Notification  : Aucune (nettoyage silencieux)
```

### TASK-017-03 : Nettoyage des OfflineQueueItem traités anciens
```python
@shared_task(name='sync.cleanup_processed_queue_items')
# Fréquence     : Quotidienne à 4h00
# Description   : Soft delete des OfflineQueueItem (status=COMPLETED) > 30 jours
```

### TASK-017-04 : Nettoyage des SyncLog anciens
```python
@shared_task(name='sync.cleanup_old_sync_logs')
# Fréquence     : Mensuelle (1er du mois)
# Description   : Soft delete des SyncLog > 1 an
```

### TASK-017-05 : Vérification des documents en cache non mis à jour
```python
@shared_task(name='sync.notify_stale_cached_documents')
# Fréquence     : Quotidienne à 8h00
# Description   : Notifie les utilisateurs dont des documents en cache
#                 n'ont pas été synchronisés depuis > 7 jours (risque d'obsolescence)
```

---

## 11. ÉVÉNEMENTS JOURNALISÉS (MODULE 11)

| Code événement | Description | Sévérité | Données clés |
|----------------|-------------|----------|--------------|
| `DEVICE_REGISTERED` | Nouvel appareil desktop enregistré | INFO | device_id, device_name, platform, user_id |
| `DEVICE_REVOKED` | Appareil révoqué | WARNING | device_id, revoked_by_id |
| `SYNC_COMPLETED` | Session de synchronisation terminée | INFO | device_id, items_up, items_down, conflicts, duration_s |
| `SYNC_FAILED` | Session de synchronisation échouée | WARNING | device_id, error_message |
| `CONFLICT_DETECTED` | Conflit de synchronisation détecté | WARNING | conflict_id, document_id, conflicting_fields |
| `CONFLICT_RESOLVED_MANUAL` | Conflit résolu manuellement | INFO | conflict_id, resolution_type, resolved_by_id |
| `CONFLICT_RESOLVED_AUTO` | Conflit résolu automatiquement | WARNING | conflict_id, applied_strategy |
| `OFFLINE_SCAN_UPLOADED` | Numérisation offline synchronisée | INFO | queue_item_id, document_id, device_id |
| `CACHE_INTEGRITY_ERROR` | Corruption de cache détectée | ALERT | device_id, document_id, expected_hash, actual_hash |
| `UPDATE_INSTALLED` | Mise à jour application installée | INFO | old_version, new_version, device_id |
| `UPDATE_SIGNATURE_INVALID` | Tentative install update signature invalide | CRITICAL | device_id, attempted_version |
| `OFFLINE_ACTION_BLOCKED` | Action bloquée en mode offline (droits insuffisants) | INFO | action_type, user_id, reason |

---

## 12. NOTIFICATIONS GÉNÉRÉES (MODULE 12)

| Déclencheur | Destinataire | Canal | Priorité | Template |
|-------------|-------------|-------|----------|----------|
| Sync automatique terminée | Utilisateur | In-app (desktop) | BASSE | `sync_completed` |
| Items en attente depuis > 24h | Utilisateur | Notification OS native | NORMALE | `sync_items_pending` |
| Conflit détecté | Utilisateur | Notification OS native | HAUTE | `conflict_detected` |
| Conflit résolu automatiquement | Utilisateur | Notification OS native | NORMALE | `conflict_auto_resolved` |
| Cache presque plein (> 90%) | Utilisateur | Notification OS native | HAUTE | `cache_nearly_full` |
| Mise à jour disponible | Utilisateur | Notification OS native | NORMALE | `update_available` |
| Corruption de cache détectée | Utilisateur + Admin | Notification OS + In-app web | HAUTE | `cache_corruption` |
| Appareil inactif révoqué | Utilisateur | In-app web | NORMALE | `device_revoked` |
| Token expiré (offline trop long) | Utilisateur | Notification OS native | HAUTE | `token_expired_offline` |

---

## 13. CONFIGURATION TAURI (tauri.conf.json — extrait)

```json
{
  "package": {
    "productName": "Archivage Numérique",
    "version": "1.0.0"
  },
  "tauri": {
    "allowlist": {
      "all": false,
      "shell": {
        "all": false,
        "execute": false,
        "sidecar": false,
        "open": true
      },
      "fs": {
        "all": false,
        "readFile": true,
        "writeFile": true,
        "readDir": true,
        "createDir": true,
        "removeFile": false,
        "removeDir": false,
        "scope": [
          "$APPDATA/*",
          "$APPLOCAL/*",
          "$TEMP/archivage_scan_*"
        ]
      },
      "http": {
        "all": false,
        "request": true,
        "scope": ["https://*.admin.fr/*", "http://localhost:*/*"]
      },
      "notification": {
        "all": true
      },
      "os": {
        "all": false
      },
      "window": {
        "all": false,
        "create": true,
        "setTitle": true
      }
    },
    "bundle": {
      "active": true,
      "targets": ["msi", "nsis", "deb", "appimage", "dmg"],
      "identifier": "fr.administration.archivage",
      "icon": ["icons/32x32.png", "icons/128x128.png", "icons/icon.icns", "icons/icon.ico"],
      "category": "Productivity",
      "fileAssociations": [
        {
          "ext": ["archdoc"],
          "name": "Document d'archivage",
          "role": "Viewer"
        }
      ],
      "windows": {
        "certificateThumbprint": null,
        "digestAlgorithm": "sha256",
        "timestampUrl": ""
      }
    },
    "updater": {
      "active": true,
      "endpoints": ["https://updates.admin.fr/archivage/{{target}}/{{arch}}/{{current_version}}"],
      "dialog": true,
      "pubkey": "dW50cnVzdGVkIGNvbW1lbnQ6...base64_ed25519_pubkey..."
    },
    "security": {
      "csp": "default-src 'self'; img-src 'self' data: https:; script-src 'self'"
    },
    "systemTray": {
      "iconPath": "icons/tray.png",
      "iconAsTemplate": true,
      "menuOnLeftClick": false
    }
  }
}
```

---

## 14. VARIABLES D'ENVIRONNEMENT (côté serveur)

```env
# .env — ne jamais versionner

# Synchronisation
SYNC_DELTA_MAX_ITEMS=500        # Max items retournés par delta
SYNC_DEVICE_INACTIVE_DAYS=90    # Jours d'inactivité avant marquage inactif
SYNC_CONFLICT_AUTO_RESOLVE_HOURS=48  # Délai avant résolution automatique

# Package offline
OFFLINE_PACKAGE_MAX_SIZE_MB=500  # Taille max d'un package offline téléchargeable

# Mise à jour
TAURI_UPDATE_SERVER=https://updates.admin.fr
TAURI_SIGNING_KEY_PATH=/etc/archivage/keys/tauri_update.key
```

---

## 15. STRUCTURE DES FICHIERS (côté serveur Django)

```
apps/
└── sync/
    ├── __init__.py
    ├── admin.py
    ├── apps.py
    ├── models/
    │   ├── __init__.py
    │   ├── sync_status.py
    │   ├── sync_conflict.py
    │   ├── offline_queue_item.py
    │   └── sync_log.py
    ├── serializers/
    │   ├── __init__.py
    │   ├── sync_serializers.py
    │   └── conflict_serializers.py
    ├── views/
    │   ├── __init__.py
    │   ├── sync_views.py
    │   ├── conflict_views.py
    │   ├── device_views.py
    │   └── offline_views.py
    ├── urls.py
    ├── permissions.py
    ├── tasks.py
    └── tests/
        ├── __init__.py
        ├── test_sync_views.py
        ├── test_conflict_resolution.py
        └── test_tasks.py
```

---

## ✅ RÉCAPITULATIF MODULE 17

| Critère | Valeur |
|---------|--------|
| Cas d'utilisation | 20 UC |
| Modèles de données Django | 4 tables |
| Schéma SQLite local | 6 tables |
| Endpoints API Django | 20 endpoints |
| Commandes Tauri Rust | 8 commandes |
| Tâches Celery planifiées | 5 tâches |
| Événements journalisés (MODULE 11) | 12 événements |
| Notifications générées (MODULE 12) | 9 types |
| Plateformes supportées | Windows, Linux, macOS |
| Technologies Rust | TWAIN, SANE, SQLCipher, keyring, reqwest |

---

**Prochain module :** MODULE 18 — API, Interopérabilité & Intégrations externes
Ce module expose une API publique documentée (OpenAPI 3.0) avec clés API et rate limiting, configure les webhooks avec retry exponentiel, et gère les intégrations avec des systèmes tiers (Google Drive, OneDrive, Nextcloud, Slack, FTP/SFTP, WebDAV). C'est le module final du cahier des charges.
