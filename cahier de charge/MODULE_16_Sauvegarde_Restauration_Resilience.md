# MODULE 16 — Sauvegarde, Restauration & Résilience

**Système d'Archivage Numérique — Administration Publique**  
**Version :** 1.0  
**Statut :** Spécification technique complète  
**Dépendances :** MODULE 02 (Auth), MODULE 11 (Audit), MODULE 12 (Notifications), MODULE 15 (Administration)

---

## 1. PRÉSENTATION DU MODULE

### 1.1 Contexte et justification

Un système d'archivage pour administration publique est par définition un système de référence : les documents qu'il contient ont valeur légale, certains sont irremplaçables. La perte de données suite à une défaillance matérielle, une erreur humaine ou une attaque ransomware serait une catastrophe institutionnelle.

Ce module ne se contente pas d'automatiser des sauvegardes. Il implémente une **stratégie de résilience complète** qui répond à trois questions fondamentales :

1. **RTO (Recovery Time Objective) :** En combien de temps le système peut-il redevenir opérationnel après une panne ? Objectif : **4 heures maximum**.
2. **RPO (Recovery Point Objective) :** Quelle est la perte de données maximale acceptable ? Objectif : **1 heure maximum** (sauvegarde incrémentielle toutes les heures).
3. **Testabilité :** Les sauvegardes sont-elles réellement utilisables ? Un test de restauration automatique est exécuté **chaque mois**.

### 1.2 Stratégie 3-2-1 obligatoire

La règle **3-2-1** est non négociable et constitue le socle de ce module :

- **3 copies** des données : l'original + 2 sauvegardes
- **2 supports différents** : disque local ET NAS/stockage réseau
- **1 copie hors site** : géographiquement distante (autre bâtiment, autre ville)

Cette stratégie protège contre : la défaillance d'un disque, l'incendie du datacenter local, le ransomware chiffrant le réseau local.

### 1.3 Périmètre fonctionnel

- Sauvegarde complète (full) hebdomadaire de la base de données et des fichiers
- Sauvegarde incrémentielle horaire des fichiers modifiés
- Sauvegarde différentielle quotidienne de la base de données (pg_dump)
- Chiffrement GPG obligatoire de toutes les sauvegardes
- Compression zstd (performances supérieures à gzip pour les grands volumes)
- Rotation et rétention automatiques selon politique configurable
- Test de restauration automatique mensuel sur environnement isolé
- Restauration granulaire : document unique, schéma DB, base complète, point-in-time
- Dashboard état des sauvegardes (MODULE 14 intégration)
- Alertes immédiates en cas d'échec

---

## 2. CAS D'UTILISATION — VUE D'ENSEMBLE

| Code | Cas d'utilisation | Acteur principal | Priorité |
|------|-------------------|-----------------|----------|
| UC-016-01 | Planifier une sauvegarde automatique (full ou incrémentielle) | Super Admin | HAUTE |
| UC-016-02 | Déclencher une sauvegarde complète manuelle immédiate | Super Admin, Admin | HAUTE |
| UC-016-03 | Déclencher une sauvegarde différentielle manuelle | Super Admin, Admin | HAUTE |
| UC-016-04 | Consulter la liste des snapshots de sauvegarde disponibles | Super Admin, Admin | HAUTE |
| UC-016-05 | Consulter le détail d'un snapshot (taille, durée, composants) | Super Admin, Admin | HAUTE |
| UC-016-06 | Vérifier l'intégrité d'un snapshot (hash + déchiffrement test) | Super Admin, Admin | HAUTE |
| UC-016-07 | Restaurer la base de données complète depuis un snapshot | Super Admin | HAUTE |
| UC-016-08 | Restaurer un document spécifique depuis une sauvegarde | Super Admin, Admin | HAUTE |
| UC-016-09 | Restaurer la base de données à un point dans le temps (PITR) | Super Admin | HAUTE |
| UC-016-10 | Restaurer les fichiers médias depuis une sauvegarde | Super Admin | HAUTE |
| UC-016-11 | Effectuer un test de restauration automatique (dry-run) | Système (Celery) | HAUTE |
| UC-016-12 | Déclencher manuellement un test de restauration | Super Admin | MOYENNE |
| UC-016-13 | Configurer les destinations de sauvegarde (local, NAS, hors-site) | Super Admin | HAUTE |
| UC-016-14 | Configurer la politique de rétention par type de sauvegarde | Super Admin | HAUTE |
| UC-016-15 | Consulter le dashboard de l'état des sauvegardes | Super Admin, Admin | HAUTE |
| UC-016-16 | Configurer les clés GPG de chiffrement | Super Admin | HAUTE |
| UC-016-17 | Exporter les données pour migration vers un autre système | Super Admin | MOYENNE |
| UC-016-18 | Supprimer manuellement un snapshot obsolète | Super Admin | BASSE |
| UC-016-19 | Configurer les alertes en cas d'échec de sauvegarde | Super Admin, Admin | HAUTE |
| UC-016-20 | Consulter l'historique des opérations de restauration | Super Admin, Admin | HAUTE |

---

## 3. CAS D'UTILISATION DÉTAILLÉS

### UC-016-02 — Déclencher une sauvegarde complète manuelle

**Acteur principal :** Super Admin, Admin
**Pré-conditions :** Utilisateur authentifié avec droits Admin minimum, espace disque suffisant (vérification préalable), pas de sauvegarde déjà en cours

**Flux principal :**
1. L'utilisateur accède à "Administration > Sauvegardes > Nouvelle sauvegarde"
2. Il configure :
   - Type : FULL (base de données + fichiers médias + configuration)
   - Destinations : sélection parmi les destinations configurées (locale, NAS, hors-site)
   - Commentaire facultatif (ex: "Avant migration v2.1")
3. Il clique sur "Démarrer la sauvegarde"
4. Le système vérifie les prérequis :
   - Espace disque disponible > 150% du volume estimé
   - Clé GPG configurée et valide
   - Destinations accessibles (test de connectivité)
   - Aucune sauvegarde déjà en cours
5. Un `BackupJob` est créé avec statut `RUNNING`
6. Une tâche Celery `run_full_backup` est enqueued avec haute priorité
7. L'utilisateur reçoit un message "Sauvegarde démarrée (ID : BKP-2026-00042)"
8. En arrière-plan, Celery exécute :
   a. `pg_dump` de la base PostgreSQL → fichier `.sql`
   b. Archivage `tar` des fichiers médias (incrémentiel par rapport au dernier full)
   c. Export de la configuration système (MODULE 15)
   d. Compression zstd de chaque composant
   e. Chiffrement GPG de chaque archive compressée
   f. Calcul du hash SHA-256 de chaque archive chiffrée
   g. Transfert vers chaque destination configurée
   h. Vérification de l'intégrité après transfert (re-hash côté destination)
   i. Création du `BackupSnapshot` avec tous les métadonnées
9. L'utilisateur est notifié de la réussite ou de l'échec (MODULE 12)
10. Événement d'audit généré (MODULE 11)

**Flux alternatifs :**
- **4a.** Espace disque insuffisant → erreur bloquante "Espace insuffisant : X Go disponible, Y Go estimé nécessaire"
- **4b.** Clé GPG expirée → erreur bloquante avec lien vers configuration GPG
- **4c.** Destination NAS inaccessible → avertissement non bloquant, sauvegarde locale uniquement avec alerte
- **8g.** Erreur de transfert vers une destination → le snapshot est créé avec statut partiel, alerte Admin

**Post-conditions :**
- `BackupSnapshot` créé avec statut `COMPLETED` ou `PARTIAL` ou `FAILED`
- Fichiers disponibles sur toutes les destinations accessibles
- Événement `BACKUP_COMPLETED` ou `BACKUP_FAILED` journalisé

---

### UC-016-07 — Restaurer la base de données complète

**Acteur principal :** Super Admin
**Pré-conditions :** Système en mode maintenance activé (MODULE 15), Super Admin authentifié avec TOTP, snapshot valide disponible

**Flux principal :**
1. Le Super Admin active le mode maintenance (MODULE 15, UC-015-16)
2. Il accède à "Administration > Sauvegardes > Restaurer"
3. Il sélectionne le snapshot source (avec date, taille, résultat de vérification d'intégrité)
4. Il choisit le périmètre : Base de données complète
5. Le système affiche un avertissement explicite :
   "⚠️ ATTENTION : Cette opération remplacera TOUTES les données actuelles par celles du snapshot du [date]. Cette action est IRRÉVERSIBLE. L'état actuel n'est pas automatiquement sauvegardé."
6. Le système propose de créer une sauvegarde de sécurité avant restauration (recommandé)
7. Le Super Admin confirme avec TOTP + saisie du texte "CONFIRMER RESTAURATION"
8. La tâche Celery `restore_full_database` est enqueued
9. En arrière-plan :
   a. Vérification de l'intégrité du snapshot (hash SHA-256)
   b. Déchiffrement GPG du fichier pg_dump
   c. Décompression zstd
   d. Arrêt des connexions PostgreSQL actives (SELECT pg_terminate_backend)
   e. `psql` : restauration depuis le fichier SQL
   f. Vérification post-restauration (compte de lignes des tables principales)
   g. Redémarrage des services applicatifs
10. Le Super Admin est notifié de la réussite
11. La maintenance est désactivée manuellement par le Super Admin

**Flux alternatifs :**
- **9a.** Hash invalide → STOP immédiat, erreur "Snapshot corrompu, restauration annulée"
- **9e.** Erreur SQL → STOP, rollback si possible, alerte CRITICAL

**Post-conditions :**
- Base de données restaurée à l'état du snapshot
- `RestoreOperation` enregistré avec tous les détails
- Événement `DATABASE_RESTORED` journalisé (sévérité ALERT)

---

### UC-016-08 — Restaurer un document spécifique

**Acteur principal :** Super Admin, Admin
**Pré-conditions :** Document identifié (UUID), snapshot contenant ce document disponible

**Flux principal :**
1. L'utilisateur accède à la fiche du document dans l'interface principale
2. Il clique sur "Restaurer depuis sauvegarde"
3. Le système liste les snapshots disponibles contenant ce document (avec date)
4. L'utilisateur sélectionne le snapshot souhaité
5. Le système propose deux options :
   - Remplacer la version actuelle (si le document existe encore)
   - Créer comme nouveau document (avec référence à l'original)
6. L'utilisateur confirme
7. En arrière-plan, Celery :
   a. Localise le fichier dans le snapshot (index des fichiers)
   b. Déchiffre et décompresse le fichier concerné uniquement
   c. Restaure le fichier dans le stockage principal
   d. Crée une nouvelle version du document (MODULE 06 versionning)
8. L'utilisateur est notifié

**Post-conditions :**
- Fichier restauré disponible dans le système
- Nouvelle version créée avec commentaire "Restauré depuis sauvegarde du [date]"
- Événement `DOCUMENT_RESTORED` journalisé

---

### UC-016-11 — Test de restauration automatique (dry-run mensuel)

**Acteur principal :** Système (Celery Beat)
**Pré-conditions :** Au moins un snapshot complet disponible, environnement de test isolé configuré

**Flux principal :**
1. Celery Beat déclenche `test_restore_dry_run` le 1er dimanche du mois à 3h00
2. Le système sélectionne le snapshot complet le plus récent
3. Dans un répertoire temporaire isolé (`/tmp/restore_test_YYYYMMDD/`) :
   a. Vérification de l'intégrité du snapshot (hash SHA-256 de toutes les archives)
   b. Déchiffrement GPG test (déchiffre sans écrire sur disque pour les fichiers volumineux)
   c. Décompression partielle (premiers 10 Mo de chaque composant)
   d. Tentative de restauration PostgreSQL dans une base temporaire (`archivage_test_YYYYMMDD`)
   e. Vérification des comptes de lignes (documents, utilisateurs, audit logs)
   f. Comparaison des comptes avec le snapshot actuel
   g. Suppression de la base temporaire et nettoyage du répertoire
4. Un `BackupVerification` est créé avec le rapport complet
5. Super Admin et Admin sont notifiés du résultat
6. Si échec : alerte CRITICAL, escalade immédiate

**Post-conditions :**
- `BackupVerification` enregistré avec statut `PASSED` ou `FAILED`
- Événement `BACKUP_VERIFICATION_COMPLETED` journalisé
- Rapport disponible dans MODULE 14 (historique des tests)

---

## 4. MODÈLES DE DONNÉES

### 4.1 BackupJob — Tâches de sauvegarde planifiées

```python
class BackupJob(BaseModel):
    """
    Configuration d'une tâche de sauvegarde (planifiée ou à la demande).
    Chaque exécution génère un BackupSnapshot.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    name = models.CharField(
        max_length=200,
        help_text="Nom de la tâche (ex: 'Sauvegarde complète hebdomadaire')"
    )

    # Type de sauvegarde
    backup_type = models.CharField(
        max_length=20,
        choices=[
            ('FULL', 'Sauvegarde complète (DB + médias + config)'),
            ('DIFFERENTIAL', 'Différentielle DB (depuis dernier FULL)'),
            ('INCREMENTAL', 'Incrémentielle médias (depuis dernière sauvegarde)'),
            ('DB_ONLY', 'Base de données uniquement'),
            ('FILES_ONLY', 'Fichiers médias uniquement'),
            ('CONFIG_ONLY', 'Configuration système uniquement'),
        ]
    )

    # Composants inclus
    include_database = models.BooleanField(default=True)
    include_media_files = models.BooleanField(default=True)
    include_system_config = models.BooleanField(default=True)
    include_audit_logs = models.BooleanField(
        default=False,
        help_text="Les logs d'audit sont volumineux et sauvegardés séparément"
    )

    # Destinations de sauvegarde (au moins 2 pour la stratégie 3-2-1)
    destination_local = models.BooleanField(
        default=True,
        help_text="Copie locale (même serveur, disque différent)"
    )
    destination_nas = models.BooleanField(
        default=True,
        help_text="Copie NAS (réseau local)"
    )
    destination_offsite = models.BooleanField(
        default=True,
        help_text="Copie hors site (géographiquement distante)"
    )
    destination_local_path = models.CharField(max_length=500, blank=True)
    destination_nas_path = models.CharField(max_length=500, blank=True)
    destination_offsite_config = models.JSONField(
        default=dict,
        help_text="Config hors site : {type: 'rsync'|'sftp', host, user, path, port}"
    )

    # Compression et chiffrement
    compression = models.CharField(
        max_length=10,
        choices=[('zstd', 'zstd (recommandé)'), ('gzip', 'gzip'), ('none', 'Aucune')],
        default='zstd'
    )
    compression_level = models.PositiveSmallIntegerField(
        default=3,
        help_text="Niveau de compression zstd (1=rapide, 19=max, défaut=3)"
    )
    gpg_key_id = models.CharField(
        max_length=100,
        help_text="ID de la clé GPG publique pour le chiffrement des archives"
    )

    # Planification (null si déclenchement manuel uniquement)
    is_scheduled = models.BooleanField(default=True)
    cron_expression = models.CharField(
        max_length=100,
        blank=True,
        help_text="Expression cron pour Celery Beat (ex: '0 2 * * 0' = dim 2h)"
    )

    # Politique de rétention
    retention_local_days = models.PositiveIntegerField(
        default=7,
        help_text="Durée de rétention locale en jours"
    )
    retention_nas_days = models.PositiveIntegerField(
        default=30,
        help_text="Durée de rétention NAS en jours"
    )
    retention_offsite_days = models.PositiveIntegerField(
        default=365,
        help_text="Durée de rétention hors site en jours (conformité légale)"
    )

    # Statut
    is_active = models.BooleanField(default=True)
    last_run_at = models.DateTimeField(null=True, blank=True)
    last_run_status = models.CharField(
        max_length=20,
        choices=[('SUCCESS', 'Succès'), ('PARTIAL', 'Partiel'), ('FAILED', 'Échec')],
        null=True, blank=True
    )
    next_run_at = models.DateTimeField(null=True, blank=True)

    # Auteur
    created_by = models.ForeignKey(
        'users.User', on_delete=models.PROTECT,
        related_name='created_backup_jobs'
    )

    # Horodatage
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True, on_delete=models.SET_NULL,
        related_name='deleted_backup_jobs'
    )

    class Meta:
        db_table = 'backup_jobs'
        indexes = [
            models.Index(fields=['is_active', 'next_run_at']),
            models.Index(fields=['backup_type', 'is_active']),
        ]
        constraints = [
            models.CheckConstraint(
                check=models.Q(compression_level__gte=1) & models.Q(compression_level__lte=19),
                name='backup_job_valid_compression_level'
            ),
        ]
```

---

### 4.2 BackupSnapshot — Snapshots de sauvegarde

```python
class BackupSnapshot(BaseModel):
    """
    Instance d'une sauvegarde exécutée.
    Contient les métadonnées et les références aux fichiers générés sur chaque destination.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    # Référence lisible
    reference = models.CharField(
        max_length=30, unique=True,
        help_text="Référence lisible : BKP-2026-00001"
    )

    # Job source
    backup_job = models.ForeignKey(
        BackupJob, on_delete=models.PROTECT,
        related_name='snapshots',
        null=True, blank=True,
        help_text="Null si sauvegarde déclenchée manuellement sans job planifié"
    )
    backup_type = models.CharField(
        max_length=20,
        choices=[
            ('FULL', 'Complète'),
            ('DIFFERENTIAL', 'Différentielle'),
            ('INCREMENTAL', 'Incrémentielle'),
            ('DB_ONLY', 'Base de données uniquement'),
            ('FILES_ONLY', 'Fichiers médias uniquement'),
            ('CONFIG_ONLY', 'Configuration uniquement'),
        ]
    )

    # Référence au snapshot précédent (pour incrémentiel/différentiel)
    parent_snapshot = models.ForeignKey(
        'self', on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='child_snapshots'
    )

    # Statut global
    status = models.CharField(
        max_length=20,
        choices=[
            ('RUNNING', 'En cours'),
            ('COMPLETED', 'Terminée avec succès'),
            ('PARTIAL', 'Partielle (certaines destinations en échec)'),
            ('FAILED', 'Échec'),
            ('EXPIRED', 'Expirée (fichiers supprimés)'),
            ('DELETED', 'Supprimée manuellement'),
        ],
        default='RUNNING', db_index=True
    )

    # Composants sauvegardés
    components = models.JSONField(
        default=dict,
        help_text="""Détail par composant :
        {
          'database': {
            'status': 'SUCCESS', 'size_bytes': 1234567890,
            'duration_seconds': 45, 'pg_dump_format': 'custom',
            'tables_count': 42, 'rows_count': 1500000
          },
          'media_files': {
            'status': 'SUCCESS', 'size_bytes': 50000000000,
            'files_count': 14382, 'duration_seconds': 120
          },
          'system_config': {
            'status': 'SUCCESS', 'size_bytes': 50000
          }
        }"""
    )

    # Destinations et fichiers
    destinations = models.JSONField(
        default=dict,
        help_text="""Statut et chemins par destination :
        {
          'local': {
            'status': 'SUCCESS',
            'path': '/var/archivage/backups/2026/02/BKP-2026-00001/',
            'files': {
              'database': {'filename': 'db_20260218_020000.sql.zst.gpg', 'sha256': '...'},
              'media': {'filename': 'media_20260218_020000.tar.zst.gpg', 'sha256': '...'},
              'config': {'filename': 'config_20260218_020000.yaml.zst.gpg', 'sha256': '...'}
            }
          },
          'nas': {'status': 'SUCCESS', ...},
          'offsite': {'status': 'FAILED', 'error': 'Connexion refusée'}
        }"""
    )

    # Métriques globales
    total_size_bytes = models.BigIntegerField(
        null=True, blank=True,
        help_text="Taille totale des archives générées (chiffrées + compressées)"
    )
    original_size_bytes = models.BigIntegerField(
        null=True, blank=True,
        help_text="Taille des données avant compression"
    )
    compression_ratio = models.FloatField(
        null=True, blank=True,
        help_text="Ratio de compression (ex: 0.3 = données réduites à 30%)"
    )
    duration_seconds = models.FloatField(null=True, blank=True)

    # Intégrité
    integrity_verified = models.BooleanField(
        default=False,
        help_text="True si l'intégrité a été vérifiée après transfert"
    )
    last_verified_at = models.DateTimeField(null=True, blank=True)

    # Déclencheur
    triggered_by = models.ForeignKey(
        'users.User', on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='triggered_backups',
        help_text="Null si déclenchement automatique"
    )
    trigger_comment = models.TextField(
        blank=True,
        help_text="Commentaire saisi lors du déclenchement manuel"
    )

    # Expiration
    expires_local_at = models.DateTimeField(null=True, blank=True)
    expires_nas_at = models.DateTimeField(null=True, blank=True)
    expires_offsite_at = models.DateTimeField(null=True, blank=True)

    # Horodatage
    started_at = models.DateTimeField(auto_now_add=True)
    completed_at = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True, on_delete=models.SET_NULL,
        related_name='deleted_snapshots'
    )

    class Meta:
        db_table = 'backup_snapshots'
        indexes = [
            models.Index(fields=['status', 'started_at']),
            models.Index(fields=['backup_type', 'status']),
            models.Index(fields=['expires_local_at']),
            models.Index(fields=['expires_nas_at']),
        ]
```

---

### 4.3 RestoreOperation — Opérations de restauration

```python
class RestoreOperation(BaseModel):
    """
    Enregistre chaque opération de restauration effectuée.
    Traçabilité complète : qui, quand, depuis quel snapshot, quel périmètre, résultat.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    reference = models.CharField(
        max_length=30, unique=True,
        help_text="Référence lisible : RST-2026-00001"
    )

    # Snapshot source
    source_snapshot = models.ForeignKey(
        BackupSnapshot, on_delete=models.PROTECT,
        related_name='restore_operations'
    )

    # Type de restauration
    restore_type = models.CharField(
        max_length=30,
        choices=[
            ('FULL_DATABASE', 'Base de données complète'),
            ('FULL_SYSTEM', 'Système complet (DB + médias + config)'),
            ('DATABASE_PITR', 'Base de données point-in-time (PITR)'),
            ('SINGLE_DOCUMENT', 'Document unique'),
            ('MEDIA_FILES', 'Fichiers médias'),
            ('CONFIG_ONLY', 'Configuration système'),
            ('DRY_RUN', 'Test de restauration (dry-run)'),
        ]
    )

    # Paramètres spécifiques
    restore_parameters = models.JSONField(
        default=dict,
        help_text="""Paramètres selon le type :
        SINGLE_DOCUMENT : {document_id, version}
        DATABASE_PITR   : {target_timestamp}
        DRY_RUN         : {test_db_name, temp_dir}"""
    )

    # Statut
    status = models.CharField(
        max_length=20,
        choices=[
            ('PENDING', 'En attente'),
            ('RUNNING', 'En cours'),
            ('COMPLETED', 'Terminée avec succès'),
            ('FAILED', 'Échec'),
            ('CANCELLED', 'Annulée'),
        ],
        default='PENDING', db_index=True
    )

    # Résultat
    result_details = models.JSONField(
        default=dict,
        help_text="Détails du résultat : tables restaurées, lignes, erreurs éventuelles"
    )
    error_message = models.TextField(blank=True)

    # Métriques
    duration_seconds = models.FloatField(null=True, blank=True)
    data_restored_bytes = models.BigIntegerField(null=True, blank=True)

    # Initiateur
    initiated_by = models.ForeignKey(
        'users.User', on_delete=models.PROTECT,
        related_name='initiated_restores',
        null=True, blank=True,
        help_text="Null si restauration automatique (dry-run mensuel)"
    )

    # Sauvegarde pré-restauration (si créée)
    pre_restore_snapshot = models.ForeignKey(
        BackupSnapshot, on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='pre_restore_for',
        help_text="Snapshot de sécurité créé avant la restauration"
    )

    # Celery
    celery_task_id = models.CharField(max_length=255, blank=True)

    # Horodatage
    started_at = models.DateTimeField(null=True, blank=True)
    completed_at = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True, on_delete=models.SET_NULL,
        related_name='deleted_restore_operations'
    )

    class Meta:
        db_table = 'restore_operations'
        indexes = [
            models.Index(fields=['status', 'created_at']),
            models.Index(fields=['restore_type', 'status']),
            models.Index(fields=['initiated_by']),
        ]
```

---

### 4.4 BackupVerification — Vérifications d'intégrité

```python
class BackupVerification(BaseModel):
    """
    Résultat d'une vérification d'intégrité d'un snapshot.
    Peut être automatique (mensuel) ou déclenchée manuellement.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    snapshot = models.ForeignKey(
        BackupSnapshot, on_delete=models.PROTECT,
        related_name='verifications'
    )

    verification_type = models.CharField(
        max_length=20,
        choices=[
            ('HASH_CHECK', 'Vérification hash SHA-256 uniquement'),
            ('DECRYPT_TEST', 'Test de déchiffrement GPG'),
            ('FULL_RESTORE_TEST', 'Test de restauration complète (dry-run)'),
        ]
    )

    # Résultat global
    status = models.CharField(
        max_length=20,
        choices=[
            ('PASSED', 'Vérification réussie'),
            ('FAILED', 'Vérification échouée'),
            ('PARTIAL', 'Partielle (certains composants échoués)'),
        ],
        db_index=True
    )

    # Résultats détaillés par composant et par destination
    results = models.JSONField(
        default=dict,
        help_text="""Résultats par composant :
        {
          'local': {
            'database': {
              'hash_expected': 'abc123...', 'hash_actual': 'abc123...',
              'hash_match': true, 'decrypt_success': true,
              'restore_rows': 1500000, 'restore_tables': 42
            }
          },
          'nas': {...},
          'offsite': {...}
        }"""
    )

    duration_seconds = models.FloatField(null=True, blank=True)
    error_details = models.TextField(blank=True)

    # Initiateur (null si automatique)
    verified_by = models.ForeignKey(
        'users.User', on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='performed_verifications'
    )

    # Horodatage
    verified_at = models.DateTimeField(auto_now_add=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True, on_delete=models.SET_NULL,
        related_name='deleted_verifications'
    )

    class Meta:
        db_table = 'backup_verifications'
        indexes = [
            models.Index(fields=['snapshot', 'verified_at']),
            models.Index(fields=['status', 'verified_at']),
            models.Index(fields=['verification_type']),
        ]
```

---

## 5. MATRICE DES PERMISSIONS

| Action | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|--------|:-----------:|:-----:|:----------:|:-----------:|:-----:|:--------:|
| Consulter liste des snapshots | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ (lecture) |
| Consulter détail d'un snapshot | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ (lecture) |
| Déclencher sauvegarde manuelle | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Configurer planifications backup | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Configurer destinations backup | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Configurer politique rétention | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Configurer clés GPG | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Vérifier intégrité snapshot | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Restaurer document unique | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Restaurer base de données complète | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Restaurer système complet | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Restauration PITR | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Déclencher test restauration | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Consulter historique restaurations | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ (lecture) |
| Consulter résultats vérifications | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ (lecture) |
| Supprimer snapshot manuellement | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Consulter dashboard sauvegardes | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ (lecture) |
| Exporter données pour migration | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

---

## 6. SÉQUENCES DÉTAILLÉES

### Séquence 1 — Sauvegarde complète automatique (pipeline complet)

```
Celery Beat              Celery Worker              Destinations
     |                        |                          |
     |-- Déclenche task ------>|                          |
     |   run_full_backup(      |                          |
     |     job_id=UUID)        |                          |
     |                        |                          |
     |                        |-- Crée BackupSnapshot    |
     |                        |   (status=RUNNING)       |
     |                        |                          |
     |                        |-- pg_dump PostgreSQL ---->|
     |                        |   (format custom, -Fc)   |
     |                        |<-- db.dump (2.1 Go) -----|
     |                        |                          |
     |                        |-- zstd compress -------->|
     |                        |<-- db.dump.zst (430 Mo) -|
     |                        |                          |
     |                        |-- gpg --encrypt -------->|
     |                        |   (clé publique GPG)     |
     |                        |<-- db.dump.zst.gpg ------|
     |                        |                          |
     |                        |-- SHA-256 hash calc ----->|
     |                        |<-- hash: abc123... ------|
     |                        |                          |
     |                        |-- [Répète pour médias + config]
     |                        |                          |
     |                        |-- Copie vers local ------->|
     |                        |   /var/backups/BKP-001/  |
     |                        |<-- OK + re-hash vérifié --|
     |                        |                          |
     |                        |-- Copie vers NAS --------->|
     |                        |   //nas/backups/BKP-001/ |
     |                        |<-- OK + re-hash vérifié --|
     |                        |                          |
     |                        |-- Copie hors site (rsync) ->|
     |                        |   rsync + SSH vers serveur distant
     |                        |<-- OK + re-hash vérifié --|
     |                        |                          |
     |                        |-- Update BackupSnapshot  |
     |                        |   (status=COMPLETED,     |
     |                        |    all destinations OK)  |
     |                        |-- Journalise (MOD 11)    |
     |                        |-- Notifie Admin (MOD 12) |
```

---

### Séquence 2 — Test de restauration automatique mensuel (dry-run)

```
Celery Beat              Celery Worker              PostgreSQL temp
     |                        |                          |
     |-- test_restore_dry_run->|                          |
     |   (1er dim. du mois)    |                          |
     |                        |                          |
     |                        |-- Sélectionne dernier    |
     |                        |   snapshot FULL          |
     |                        |                          |
     |                        |-- Vérifie hash SHA-256   |
     |                        |   (toutes destinations)  |
     |                        |                          |
     |                        |-- gpg --decrypt -------->|
     |                        |   (clé privée GPG)       |
     |                        |<-- db.dump.zst ----------|
     |                        |                          |
     |                        |-- zstd decompress ------->|
     |                        |<-- db.dump --------------|
     |                        |                          |
     |                        |-- CREATE DATABASE ------->|
     |                        |   archivage_test_20260201|
     |                        |                          |
     |                        |-- pg_restore -------------->|
     |                        |   (dans la DB test)       |
     |                        |<-- Restauration terminée --|
     |                        |                          |
     |                        |-- Vérifications ---------->|
     |                        |   COUNT documents ≈ N    |
     |                        |   COUNT users ≈ M        |
     |                        |   COUNT audit_logs ≈ P   |
     |                        |<-- Counts OK (±5%) -------|
     |                        |                          |
     |                        |-- DROP DATABASE --------->|
     |                        |   archivage_test_*        |
     |                        |<-- Nettoyé ---------------|
     |                        |                          |
     |                        |-- Crée BackupVerification|
     |                        |   (status=PASSED)        |
     |                        |-- Journalise (MOD 11)    |
     |                        |-- Notifie Admins (MOD 12)|
```

---

### Séquence 3 — Restauration base complète avec précautions

```
Super Admin              Système                     PostgreSQL
     |                      |                             |
     |-- Active maintenance ->|                            |
     |   (MODULE 15)         |                             |
     |                       |                             |
     |-- Sélectionne snapshot->|                           |
     |-- Choisit FULL_DATABASE|                            |
     |                       |                             |
     |<-- Avertissement ====  |                             |
     |   "IRRÉVERSIBLE -      |                             |
     |    Sauvegarder avant?" |                             |
     |                       |                             |
     |-- Accepte sauvegarde  ->|                            |
     |   de sécurité          |                             |
     |                       |-- [Sauvegarde rapide DB] -->|
     |                       |-- Crée pre_restore_snapshot|
     |                       |                             |
     |<-- TOTP challenge -----|                             |
     |-- Code TOTP + "CONFIRMER RESTAURATION"              |
     |                       |                             |
     |                       |-- Vérifie intégrité snapshot|
     |                       |-- gpg --decrypt ----------->|
     |                       |-- zstd decompress          |
     |                       |                             |
     |                       |-- Termine connexions ----->|
     |                       |   SELECT pg_terminate_backend
     |                       |                             |
     |                       |-- pg_restore --------------->|
     |                       |   (DROP + RECREATE schema) |
     |                       |<-- Restauration OK ----------|
     |                       |                             |
     |                       |-- Vérifications post-restore|
     |                       |<-- Tables OK ---------------:|
     |                       |                             |
     |                       |-- Update RestoreOperation   |
     |                       |   (status=COMPLETED)        |
     |                       |-- Journalise ALERT (MOD 11)|
     |                       |                             |
     |<-- Notification succès-|                             |
     |-- Désactive maintenance>|                            |
```

---

## 7. ENDPOINTS API

### 7.1 Jobs de sauvegarde

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/backup-jobs/` | Lister les jobs de sauvegarde | JWT + Admin |
| `POST` | `/api/v1/backup-jobs/` | Créer un nouveau job | JWT + Super Admin |
| `GET` | `/api/v1/backup-jobs/{id}/` | Détail d'un job | JWT + Admin |
| `PATCH` | `/api/v1/backup-jobs/{id}/` | Modifier un job (rétention, destinations) | JWT + Super Admin |
| `POST` | `/api/v1/backup-jobs/{id}/activate/` | Activer un job planifié | JWT + Super Admin |
| `POST` | `/api/v1/backup-jobs/{id}/deactivate/` | Désactiver un job planifié | JWT + Super Admin |
| `POST` | `/api/v1/backup-jobs/{id}/run-now/` | Déclencher immédiatement | JWT + Admin |
| `DELETE` | `/api/v1/backup-jobs/{id}/` | Soft delete | JWT + Super Admin |

### 7.2 Snapshots

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/backup-snapshots/` | Lister les snapshots (filtrables) | JWT + Admin |
| `GET` | `/api/v1/backup-snapshots/{id}/` | Détail complet d'un snapshot | JWT + Admin |
| `GET` | `/api/v1/backup-snapshots/{id}/status/` | Statut en temps réel (polling) | JWT + Admin |
| `POST` | `/api/v1/backup-snapshots/{id}/verify/` | Déclencher vérification d'intégrité | JWT + Admin |
| `DELETE` | `/api/v1/backup-snapshots/{id}/` | Suppression manuelle (soft delete) | JWT + Super Admin |
| `GET` | `/api/v1/backup-snapshots/latest/` | Dernier snapshot réussi par type | JWT + Admin |
| `GET` | `/api/v1/backup-snapshots/dashboard/` | Données pour le dashboard sauvegardes | JWT + Admin |

### 7.3 Restaurations

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/restore-operations/` | Historique des restaurations | JWT + Admin |
| `POST` | `/api/v1/restore-operations/` | Initier une restauration | JWT + Super Admin |
| `GET` | `/api/v1/restore-operations/{id}/` | Statut et détail d'une restauration | JWT + Admin |
| `POST` | `/api/v1/restore-operations/{id}/cancel/` | Annuler (si status=PENDING) | JWT + Super Admin |
| `POST` | `/api/v1/restore-operations/dry-run/` | Déclencher un test immédiat | JWT + Super Admin |

### 7.4 Vérifications d'intégrité

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/backup-verifications/` | Historique des vérifications | JWT + Admin |
| `GET` | `/api/v1/backup-verifications/{id}/` | Détail d'une vérification | JWT + Admin |
| `GET` | `/api/v1/backup-verifications/latest/` | Dernière vérification par snapshot | JWT + Admin |

### 7.5 Configuration et export

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/backup-config/` | Configuration globale des sauvegardes | JWT + Super Admin |
| `PATCH` | `/api/v1/backup-config/gpg/` | Configurer la clé GPG | JWT + Super Admin |
| `POST` | `/api/v1/backup-config/test-destination/` | Tester l'accessibilité d'une destination | JWT + Admin |
| `POST` | `/api/v1/backup-config/export-for-migration/` | Initier export pour migration | JWT + Super Admin |

---

## 8. SERVICE DE SAUVEGARDE — IMPLÉMENTATION

### 8.1 Service principal

```python
# apps/backup/services/backup_service.py

import os
import subprocess
import hashlib
import shutil
from datetime import datetime
from pathlib import Path
from django.conf import settings
from apps.administration.models import SystemSetting


class BackupService:
    """
    Service principal d'exécution des sauvegardes.
    Toujours appelé depuis une tâche Celery, jamais depuis une vue HTTP.
    """

    def run_full_backup(self, snapshot: 'BackupSnapshot', job: 'BackupJob') -> bool:
        """
        Exécute une sauvegarde complète.
        Retourne True si tous les composants sont sauvegardés sur au moins 2 destinations.
        """
        work_dir = Path(f"/tmp/backup_work_{snapshot.id}")
        work_dir.mkdir(parents=True, exist_ok=True)

        results = {}

        try:
            # 1. Sauvegarde base de données
            if job.include_database:
                db_result = self._backup_database(work_dir, job)
                results['database'] = db_result

            # 2. Sauvegarde fichiers médias
            if job.include_media_files:
                media_result = self._backup_media_files(work_dir, job, snapshot.parent_snapshot)
                results['media_files'] = media_result

            # 3. Sauvegarde configuration
            if job.include_system_config:
                config_result = self._backup_system_config(work_dir, job)
                results['system_config'] = config_result

            # 4. Distribution vers les destinations
            destination_results = self._distribute_to_destinations(work_dir, snapshot, job)

            # 5. Mise à jour du snapshot
            self._update_snapshot_completion(snapshot, results, destination_results)

            success_destinations = sum(
                1 for d in destination_results.values() if d.get('status') == 'SUCCESS'
            )
            return success_destinations >= 2  # Minimum 2 destinations pour stratégie 3-2-1

        finally:
            # Nettoyage du répertoire de travail temporaire
            shutil.rmtree(work_dir, ignore_errors=True)

    def _backup_database(self, work_dir: Path, job: 'BackupJob') -> dict:
        """Sauvegarde PostgreSQL via pg_dump au format custom."""
        output_file = work_dir / "database.dump"

        db_settings = settings.DATABASES['default']
        env = os.environ.copy()
        env['PGPASSWORD'] = db_settings.get('PASSWORD', '')

        cmd = [
            'pg_dump',
            '-h', db_settings.get('HOST', 'localhost'),
            '-p', str(db_settings.get('PORT', 5432)),
            '-U', db_settings.get('USER', 'postgres'),
            '-d', db_settings.get('NAME', 'archivage'),
            '--format=custom',          # Format compressé natif PostgreSQL
            '--no-owner',               # Pas de chown dans le dump
            '--no-acl',                 # Pas de GRANT dans le dump
            '--file', str(output_file),
        ]

        result = subprocess.run(
            cmd, env=env, capture_output=True, text=True, timeout=3600
        )

        if result.returncode != 0:
            raise RuntimeError(f"pg_dump échoué : {result.stderr}")

        original_size = output_file.stat().st_size

        # Compression + chiffrement
        compressed_file = self._compress_file(output_file, job.compression, job.compression_level)
        encrypted_file = self._encrypt_file(compressed_file, job.gpg_key_id)
        output_file.unlink()
        compressed_file.unlink()

        sha256 = self._compute_sha256(encrypted_file)

        return {
            'status': 'SUCCESS',
            'filename': encrypted_file.name,
            'original_size_bytes': original_size,
            'compressed_encrypted_size_bytes': encrypted_file.stat().st_size,
            'sha256': sha256,
        }

    def _backup_media_files(
        self,
        work_dir: Path,
        job: 'BackupJob',
        parent_snapshot: 'BackupSnapshot'
    ) -> dict:
        """
        Sauvegarde incrémentielle des fichiers médias avec tar.
        Si parent_snapshot fourni : incrémentielle depuis ce snapshot.
        Sinon : archive complète.
        """
        media_path = SystemSetting.get('storage.main_path', '/var/archivage/media')
        output_file = work_dir / "media.tar"

        cmd = ['tar', '--create', '--file', str(output_file)]

        if parent_snapshot:
            # Incrémentielle : only files newer than parent snapshot date
            parent_date = parent_snapshot.started_at.strftime('%Y-%m-%d %H:%M:%S')
            cmd.extend(['--newer-mtime', parent_date])

        cmd.append(media_path)

        result = subprocess.run(cmd, capture_output=True, timeout=7200)
        if result.returncode not in [0, 1]:  # 1 = fichiers modifiés pendant archivage (normal)
            raise RuntimeError(f"tar échoué (code {result.returncode}) : {result.stderr.decode()}")

        original_size = output_file.stat().st_size
        compressed_file = self._compress_file(output_file, job.compression, job.compression_level)
        encrypted_file = self._encrypt_file(compressed_file, job.gpg_key_id)
        output_file.unlink()
        compressed_file.unlink()

        sha256 = self._compute_sha256(encrypted_file)

        return {
            'status': 'SUCCESS',
            'filename': encrypted_file.name,
            'original_size_bytes': original_size,
            'compressed_encrypted_size_bytes': encrypted_file.stat().st_size,
            'sha256': sha256,
            'incremental': parent_snapshot is not None,
        }

    def _compress_file(self, input_file: Path, compression: str, level: int) -> Path:
        """Compresse un fichier avec zstd ou gzip."""
        if compression == 'zstd':
            output_file = input_file.with_suffix(input_file.suffix + '.zst')
            subprocess.run(
                ['zstd', f'-{level}', '--rm', '-o', str(output_file), str(input_file)],
                check=True, capture_output=True
            )
        elif compression == 'gzip':
            output_file = input_file.with_suffix(input_file.suffix + '.gz')
            subprocess.run(
                ['gzip', f'-{level}', '-c', str(input_file)],
                check=True, capture_output=True,
                stdout=output_file.open('wb')
            )
        else:
            return input_file
        return output_file

    def _encrypt_file(self, input_file: Path, gpg_key_id: str) -> Path:
        """Chiffre un fichier avec GPG (chiffrement asymétrique, clé publique)."""
        output_file = input_file.with_suffix(input_file.suffix + '.gpg')
        result = subprocess.run(
            [
                'gpg', '--batch', '--yes',
                '--recipient', gpg_key_id,
                '--encrypt',
                '--output', str(output_file),
                str(input_file)
            ],
            capture_output=True, timeout=600
        )
        if result.returncode != 0:
            raise RuntimeError(f"GPG chiffrement échoué : {result.stderr.decode()}")
        return output_file

    def _compute_sha256(self, file_path: Path) -> str:
        """Calcule le hash SHA-256 d'un fichier."""
        sha256_hash = hashlib.sha256()
        with open(file_path, 'rb') as f:
            for chunk in iter(lambda: f.read(8192), b''):
                sha256_hash.update(chunk)
        return sha256_hash.hexdigest()

    def _distribute_to_destinations(
        self,
        work_dir: Path,
        snapshot: 'BackupSnapshot',
        job: 'BackupJob'
    ) -> dict:
        """Copie les archives vers toutes les destinations configurées."""
        results = {}
        snapshot_dir = f"BKP_{snapshot.reference}_{datetime.now().strftime('%Y%m%d_%H%M%S')}"

        if job.destination_local:
            results['local'] = self._copy_to_local(
                work_dir, job.destination_local_path, snapshot_dir
            )

        if job.destination_nas:
            results['nas'] = self._copy_to_nas(
                work_dir, job.destination_nas_path, snapshot_dir
            )

        if job.destination_offsite:
            results['offsite'] = self._copy_to_offsite(
                work_dir, job.destination_offsite_config, snapshot_dir
            )

        return results

    def _copy_to_offsite(self, work_dir: Path, config: dict, snapshot_dir: str) -> dict:
        """Copie hors site via rsync + SSH."""
        offsite_type = config.get('type', 'rsync')
        if offsite_type == 'rsync':
            dest = f"{config['user']}@{config['host']}:{config['path']}/{snapshot_dir}/"
            cmd = [
                'rsync', '-avz', '--checksum',
                '-e', f"ssh -p {config.get('port', 22)} -i {config.get('key_path', '')}",
                f"{work_dir}/", dest
            ]
            result = subprocess.run(cmd, capture_output=True, timeout=3600)
            if result.returncode == 0:
                return {'status': 'SUCCESS', 'path': dest}
            else:
                return {'status': 'FAILED', 'error': result.stderr.decode()[:500]}
        return {'status': 'FAILED', 'error': f"Type hors-site non supporté : {offsite_type}"}
```

---

### 8.2 Service de vérification d'intégrité

```python
# apps/backup/services/verification_service.py

import subprocess
import hashlib
from pathlib import Path
from django.conf import settings


class BackupVerificationService:

    def verify_snapshot(
        self,
        snapshot: 'BackupSnapshot',
        verification_type: str = 'FULL_RESTORE_TEST'
    ) -> dict:
        """
        Vérifie l'intégrité d'un snapshot selon le niveau demandé.
        Retourne un rapport détaillé par composant et par destination.
        """
        results = {}

        for dest_name, dest_data in snapshot.destinations.items():
            if dest_data.get('status') != 'SUCCESS':
                results[dest_name] = {'skipped': True, 'reason': 'Destination en échec'}
                continue

            dest_results = {}
            for component_name, file_info in dest_data.get('files', {}).items():
                component_result = {}

                file_path = Path(dest_data['path']) / file_info['filename']

                # Niveau 1 : Vérification hash
                if file_path.exists():
                    actual_hash = self._compute_sha256(file_path)
                    component_result['hash_expected'] = file_info['sha256']
                    component_result['hash_actual'] = actual_hash
                    component_result['hash_match'] = actual_hash == file_info['sha256']
                else:
                    component_result['hash_match'] = False
                    component_result['error'] = 'Fichier introuvable'
                    dest_results[component_name] = component_result
                    continue

                if not component_result['hash_match']:
                    dest_results[component_name] = component_result
                    continue

                # Niveau 2 : Test de déchiffrement
                if verification_type in ('DECRYPT_TEST', 'FULL_RESTORE_TEST'):
                    decrypt_result = self._test_decrypt(file_path)
                    component_result['decrypt_success'] = decrypt_result['success']
                    if not decrypt_result['success']:
                        component_result['decrypt_error'] = decrypt_result.get('error')

                dest_results[component_name] = component_result

            results[dest_name] = dest_results

        # Niveau 3 : Test restauration complète (dry-run)
        if verification_type == 'FULL_RESTORE_TEST':
            db_result = self._test_restore_database(snapshot)
            results['_dry_run'] = db_result

        overall_status = self._compute_overall_status(results)
        return {'status': overall_status, 'results': results}

    def _test_decrypt(self, encrypted_file: Path) -> dict:
        """Teste le déchiffrement GPG sans écrire sur disque."""
        result = subprocess.run(
            ['gpg', '--batch', '--decrypt', '--output', '/dev/null', str(encrypted_file)],
            capture_output=True, timeout=120
        )
        return {
            'success': result.returncode == 0,
            'error': result.stderr.decode()[:200] if result.returncode != 0 else None
        }

    def _test_restore_database(self, snapshot: 'BackupSnapshot') -> dict:
        """
        Test de restauration dans une base temporaire PostgreSQL.
        La base temporaire est créée, peuplée, vérifiée, puis supprimée.
        """
        import tempfile
        from datetime import datetime
        from django.db import connection

        test_db_name = f"archivage_test_{datetime.now().strftime('%Y%m%d%H%M%S')}"
        temp_dir = tempfile.mkdtemp(prefix='backup_restore_test_')

        try:
            # Récupérer le fichier DB depuis la première destination disponible
            db_file_info = self._get_db_file(snapshot)
            if not db_file_info:
                return {'success': False, 'error': 'Fichier DB introuvable dans le snapshot'}

            # Déchiffrer
            decrypted_file = Path(temp_dir) / 'database.dump.zst'
            subprocess.run(
                ['gpg', '--batch', '--decrypt', '--output', str(decrypted_file),
                 str(db_file_info['path'])],
                check=True, capture_output=True, timeout=300
            )

            # Décompresser
            decompressed_file = Path(temp_dir) / 'database.dump'
            subprocess.run(
                ['zstd', '--decompress', '-o', str(decompressed_file), str(decrypted_file)],
                check=True, capture_output=True, timeout=300
            )

            db_settings = settings.DATABASES['default']
            env = os.environ.copy()
            env['PGPASSWORD'] = db_settings.get('PASSWORD', '')
            base_cmd = [
                '-h', db_settings.get('HOST', 'localhost'),
                '-p', str(db_settings.get('PORT', 5432)),
                '-U', db_settings.get('USER', 'postgres'),
            ]

            # Créer la base de test
            subprocess.run(
                ['createdb'] + base_cmd + [test_db_name],
                env=env, check=True, capture_output=True, timeout=30
            )

            # Restaurer
            subprocess.run(
                ['pg_restore'] + base_cmd + ['-d', test_db_name, str(decompressed_file)],
                env=env, check=True, capture_output=True, timeout=1800
            )

            # Vérifier les comptes de lignes
            with connection.cursor() as cursor:
                cursor.execute(f"SELECT COUNT(*) FROM {test_db_name}.documents")
                doc_count = cursor.fetchone()[0]

            # Comparer avec les comptes actuels
            with connection.cursor() as cursor:
                cursor.execute("SELECT COUNT(*) FROM documents WHERE is_deleted=false")
                current_count = cursor.fetchone()[0]

            ratio = doc_count / current_count if current_count > 0 else 1
            counts_ok = 0.95 <= ratio <= 1.05  # Tolérance 5%

            return {
                'success': counts_ok,
                'test_db': test_db_name,
                'document_count_restored': doc_count,
                'document_count_current': current_count,
                'ratio': round(ratio, 3),
                'counts_acceptable': counts_ok,
            }

        except Exception as e:
            return {'success': False, 'error': str(e)[:500]}

        finally:
            # Nettoyage toujours exécuté
            import shutil
            shutil.rmtree(temp_dir, ignore_errors=True)
            try:
                env = os.environ.copy()
                env['PGPASSWORD'] = settings.DATABASES['default'].get('PASSWORD', '')
                subprocess.run(
                    ['dropdb'] + [
                        '-h', settings.DATABASES['default'].get('HOST', 'localhost'),
                        '-U', settings.DATABASES['default'].get('USER', 'postgres'),
                        '--if-exists', test_db_name
                    ],
                    env=env, capture_output=True, timeout=30
                )
            except Exception:
                pass
```

---

## 9. RÈGLES MÉTIER CRITIQUES

### RB-016-01 — Stratégie 3-2-1 non contournable

Toute sauvegarde complète doit être distribuée sur au minimum 2 destinations distinctes. Si seulement 1 destination est disponible lors d'une sauvegarde planifiée, celle-ci échoue avec statut `PARTIAL`, une alerte CRITICAL est générée, et une nouvelle tentative est programmée dans les 30 minutes. Un Super Admin ne peut pas désactiver cette règle.

---

### RB-016-02 — Chiffrement GPG obligatoire

Aucune archive n'est stockée sans chiffrement GPG, quelle que soit la destination. Les archives en clair ne doivent jamais apparaître sur les destinations finales. Le répertoire de travail temporaire (`/tmp/backup_work_*/`) est nettoyé immédiatement après chaque transfert, qu'il ait réussi ou non (bloc `finally`).

---

### RB-016-03 — Vérification d'intégrité après chaque transfert

Après chaque copie vers une destination, le système recalcule le SHA-256 du fichier transféré et le compare au hash calculé avant transfert. Une différence indique une corruption lors du transfert ; le snapshot est marqué `PARTIAL` pour cette destination, et l'alerte est déclenchée.

---

### RB-016-04 — Test de restauration mensuel obligatoire et alerté

Le test de restauration automatique (dry-run) est non désactivable. S'il échoue pendant 2 mois consécutifs, une notification CRITICAL est envoyée au Super Admin et à l'Admin, et un événement `BACKUP_SYSTEM_COMPROMISED` est journalisé avec sévérité CRITICAL, déclenchant une revue manuelle obligatoire.

---

### RB-016-05 — Restauration complète DB uniquement en mode maintenance

Une restauration de type `FULL_DATABASE` ou `FULL_SYSTEM` ne peut être déclenchée que si le mode maintenance est actif (MODULE 15). Le système vérifie cette condition avant d'accepter la demande. Cette règle prévient une restauration accidentelle sur un système en production avec des utilisateurs connectés.

---

### RB-016-06 — Sauvegarde de sécurité avant toute restauration critique

Avant toute restauration `FULL_DATABASE` ou `FULL_SYSTEM`, le système propose (fortement recommandé) et peut automatiquement déclencher une sauvegarde rapide de sécurité (`DB_ONLY`). Si l'utilisateur refuse, un avertissement explicite est affiché et journalisé.

---

### RB-016-07 — Rétention minimale légale hors-site

La destination hors-site doit conserver au minimum les sauvegardes de la dernière année, quelle que soit la politique de rétention configurée. Une valeur `retention_offsite_days` inférieure à 365 est refusée par le système avec un message explicatif. Cette règle garantit la conformité aux obligations légales de conservation des archives publiques.

---

### RB-016-08 — RTO et RPO comme contraintes de planification

Les planifications créées sont validées par le système pour vérifier qu'elles respectent les objectifs RTO=4h et RPO=1h. Si la planification des sauvegardes incrémentielle est configurée à une fréquence supérieure à 1h, le système affiche un avertissement "Cette configuration ne respecte pas l'objectif RPO=1h de votre politique de résilience."

---

## 10. TÂCHES CELERY PLANIFIÉES

### TASK-016-01 : Sauvegarde complète hebdomadaire
```python
@shared_task(name='backup.run_full_backup', bind=True, max_retries=1)
# Cron          : '0 2 * * 0' (dimanche 2h00)
# Description   : Sauvegarde complète DB + médias + config vers toutes destinations
# Timeout       : 6 heures (grands volumes)
# Retry         : 1 tentative si échec, après 30 minutes
# Alerte        : Super Admin + Admin si FAILED ou PARTIAL
```

### TASK-016-02 : Sauvegarde différentielle quotidienne (DB)
```python
@shared_task(name='backup.run_differential_backup', bind=True, max_retries=2)
# Cron          : '0 3 * * 1-6' (lun-sam 3h00)
# Description   : pg_dump différentiel depuis le dernier FULL
# Timeout       : 2 heures
# Retry         : 2 tentatives avec délai 15 minutes
```

### TASK-016-03 : Sauvegarde incrémentielle horaire (fichiers médias)
```python
@shared_task(name='backup.run_incremental_backup', bind=True, max_retries=2)
# Cron          : '30 * * * *' (toutes les heures à H:30)
# Description   : tar incrémentiel des fichiers médias modifiés depuis la dernière sauvegarde
# Timeout       : 45 minutes
# Note          : Garantit le RPO = 1h
```

### TASK-016-04 : Test de restauration mensuel (dry-run)
```python
@shared_task(name='backup.monthly_restore_test', bind=True)
# Cron          : '0 3 * * 0#1' (1er dimanche du mois à 3h00)
# Description   : Test complet : hash + déchiffrement + restauration DB test
# Timeout       : 3 heures
# Alerte CRITICAL si échec 2 mois consécutifs
```

### TASK-016-05 : Vérification hebdomadaire d'intégrité des snapshots récents
```python
@shared_task(name='backup.weekly_integrity_check')
# Cron          : '0 4 * * 3' (mercredi 4h00)
# Description   : Vérifie les hash SHA-256 de tous les snapshots des 7 derniers jours
#                 sur toutes les destinations (détecte corruption silencieuse)
# Alerte        : Si un hash ne correspond pas → CRITICAL immédiat
```

### TASK-016-06 : Rotation et purge selon politique de rétention
```python
@shared_task(name='backup.rotate_and_purge')
# Cron          : '0 5 * * *' (quotidien 5h00)
# Description   : Supprime physiquement les fichiers dont expires_*_at < now()
#                 Met à jour BackupSnapshot.status = 'EXPIRED'
# Sécurité      : Ne supprime jamais le dernier FULL disponible sur chaque destination
# Rapport       : Volume supprimé journalisé (INFO)
```

### TASK-016-07 : Vérification de l'accessibilité des destinations
```python
@shared_task(name='backup.check_destinations_availability')
# Cron          : '*/30 * * * *' (toutes les 30 minutes)
# Description   : Ping + test d'écriture sur chaque destination configurée
# Alerte        : Si une destination devient inaccessible → WARNING
#                 Si 2 destinations inaccessibles → CRITICAL
```

### TASK-016-08 : Alertes sauvegarde manquante
```python
@shared_task(name='backup.check_missing_backups')
# Cron          : '0 * * * *' (toutes les heures)
# Description   : Vérifie que la dernière sauvegarde réussie est récente
#                 Si > 2h sans incrémentielle → WARNING
#                 Si > 26h sans différentielle → WARNING
#                 Si > 8 jours sans FULL → CRITICAL
```

---

## 11. ÉVÉNEMENTS JOURNALISÉS (MODULE 11)

| Code événement | Description | Sévérité | Données clés |
|----------------|-------------|----------|--------------|
| `BACKUP_STARTED` | Sauvegarde démarrée | INFO | snapshot_id, backup_type, triggered_by_id |
| `BACKUP_COMPLETED` | Sauvegarde terminée avec succès | INFO | snapshot_id, size_bytes, duration_seconds, destinations |
| `BACKUP_PARTIAL` | Sauvegarde partielle (certaines destinations KO) | WARNING | snapshot_id, failed_destinations |
| `BACKUP_FAILED` | Sauvegarde échouée | CRITICAL | snapshot_id, error_message |
| `BACKUP_VERIFICATION_COMPLETED` | Vérification d'intégrité réussie | INFO | verification_id, snapshot_id, type |
| `BACKUP_VERIFICATION_FAILED` | Vérification d'intégrité échouée | CRITICAL | verification_id, snapshot_id, failed_components |
| `RESTORE_STARTED` | Restauration démarrée | ALERT | restore_id, restore_type, snapshot_id, initiated_by_id |
| `RESTORE_COMPLETED` | Restauration terminée avec succès | ALERT | restore_id, duration_seconds, data_restored_bytes |
| `RESTORE_FAILED` | Restauration échouée | CRITICAL | restore_id, error_message |
| `DRY_RUN_PASSED` | Test de restauration mensuel réussi | INFO | verification_id, snapshot_id, counts_ratio |
| `DRY_RUN_FAILED` | Test de restauration mensuel échoué | CRITICAL | verification_id, error_details |
| `BACKUP_SYSTEM_COMPROMISED` | 2 mois consécutifs d'échec dry-run | CRITICAL | consecutive_failures, last_passed_at |
| `DESTINATION_UNAVAILABLE` | Destination de sauvegarde inaccessible | WARNING | destination_name, error |
| `BACKUP_ROTATION_COMPLETED` | Rotation des sauvegardes exécutée | INFO | snapshots_deleted, bytes_freed |
| `BACKUP_HASH_MISMATCH` | Corruption détectée (hash incohérent) | CRITICAL | snapshot_id, destination, component, expected_hash, actual_hash |
| `GPG_KEY_EXPIRING` | Clé GPG expire dans moins de 30 jours | WARNING | key_id, expires_at |

---

## 12. NOTIFICATIONS GÉNÉRÉES (MODULE 12)

| Déclencheur | Destinataire(s) | Canal | Priorité | Template |
|-------------|----------------|-------|----------|----------|
| Sauvegarde terminée avec succès | Super Admin | In-app | BASSE | `backup_success` |
| Sauvegarde partielle (dest. manquante) | Super Admin + Admin | In-app + Email | HAUTE | `backup_partial` |
| Sauvegarde échouée | Super Admin + Admin | In-app + Email | CRITIQUE | `backup_failed` |
| Test restauration réussi (dry-run) | Super Admin + Admin | In-app | BASSE | `dry_run_passed` |
| Test restauration échoué | Super Admin + Admin | In-app + Email | CRITIQUE | `dry_run_failed` |
| 2 dry-runs consécutifs en échec | Super Admin + Admin | In-app + Email | CRITIQUE | `backup_system_compromised` |
| Destination inaccessible | Super Admin + Admin | In-app + Email | HAUTE | `backup_destination_down` |
| Corruption détectée (hash mismatch) | Super Admin + Admin | In-app + Email | CRITIQUE | `backup_corruption_detected` |
| Sauvegarde manquante (> seuil RPO) | Super Admin + Admin | In-app + Email | HAUTE | `backup_rpo_breach` |
| Clé GPG expirant dans < 30 jours | Super Admin | In-app + Email | HAUTE | `gpg_key_expiring` |

---

## 13. POLITIQUE DE RÉTENTION RECOMMANDÉE

| Niveau | Local | NAS | Hors site |
|--------|-------|-----|-----------|
| Incrémentielle (horaire) | 2 jours | 7 jours | Non conservée |
| Différentielle DB (quotidienne) | 7 jours | 30 jours | 30 jours |
| Complète (hebdomadaire) | 14 jours | 90 jours | 365 jours |
| Mensuelle (1er de chaque mois) | 30 jours | 1 an | 5 ans (légal) |

**Note :** La sauvegarde mensuelle est une sauvegarde complète du premier dimanche du mois, conservée plus longtemps que les autres. Elle est identifiée par le flag `is_monthly_reference = True` dans le `BackupJob`.

---

## 14. VARIABLES D'ENVIRONNEMENT SPÉCIFIQUES

```env
# .env — ne jamais versionner

# Destinations
BACKUP_LOCAL_PATH=/var/archivage/backups
BACKUP_NAS_PATH=//nas-01.interne/archivage/backups
BACKUP_OFFSITE_HOST=backup-distant.admin.fr
BACKUP_OFFSITE_USER=backup_agent
BACKUP_OFFSITE_PATH=/backups/archivage
BACKUP_OFFSITE_SSH_KEY=/etc/archivage/keys/backup_rsa
BACKUP_OFFSITE_PORT=22

# GPG
BACKUP_GPG_KEY_ID=0xABCDEF1234567890
BACKUP_GPG_HOMEDIR=/etc/archivage/gnupg

# PostgreSQL (pour pg_dump / pg_restore)
BACKUP_DB_HOST=localhost
BACKUP_DB_PORT=5432
BACKUP_DB_USER=archivage_backup
BACKUP_DB_NAME=archivage

# Alertes timing
BACKUP_RPO_WARNING_HOURS=2
BACKUP_FULL_MISSING_WARNING_DAYS=8
```

---

## 15. STRUCTURE DES FICHIERS

```
apps/
└── backup/
    ├── __init__.py
    ├── admin.py
    ├── apps.py
    ├── models/
    │   ├── __init__.py
    │   ├── backup_job.py
    │   ├── backup_snapshot.py
    │   ├── restore_operation.py
    │   └── backup_verification.py
    ├── serializers/
    │   ├── __init__.py
    │   ├── backup_job_serializers.py
    │   ├── snapshot_serializers.py
    │   └── restore_serializers.py
    ├── views/
    │   ├── __init__.py
    │   ├── backup_job_views.py
    │   ├── snapshot_views.py
    │   ├── restore_views.py
    │   └── verification_views.py
    ├── urls.py
    ├── permissions.py
    ├── tasks.py
    ├── services/
    │   ├── __init__.py
    │   ├── backup_service.py
    │   ├── restore_service.py
    │   └── verification_service.py
    └── tests/
        ├── __init__.py
        ├── test_backup_service.py
        ├── test_restore_service.py
        ├── test_verification_service.py
        └── test_tasks.py

docs/
└── backup/
    └── MODULE_16_Sauvegarde_Restauration_Resilience.md
```

---

## ✅ RÉCAPITULATIF MODULE 16

| Critère | Valeur |
|---------|--------|
| Cas d'utilisation | 20 UC |
| Modèles de données | 4 tables |
| Endpoints API | 25 endpoints |
| Tâches Celery planifiées | 8 tâches |
| Événements journalisés (MODULE 11) | 16 événements |
| Notifications générées (MODULE 12) | 10 types |
| Services techniques | 3 services |
| RTO cible | 4 heures |
| RPO cible | 1 heure |
| Stratégie | 3-2-1 (3 copies, 2 supports, 1 hors site) |

---

**Prochain module :** MODULE 17 — Mode Desktop Tauri & Synchronisation offline
Ce module spécifie l'application desktop native (Windows/Linux/macOS) construite avec Tauri, le mode offline-first avec cache SQLite local, la synchronisation bidirectionnelle, la résolution de conflits et l'intégration native du scanner.
