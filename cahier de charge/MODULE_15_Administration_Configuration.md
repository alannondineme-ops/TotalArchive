# MODULE 15 — Administration système & Configuration

**Système d'Archivage Numérique — Administration Publique**  
**Version :** 1.0  
**Statut :** Spécification technique complète  
**Dépendances :** MODULE 02 (Auth), MODULE 11 (Audit), MODULE 12 (Notifications), MODULE 14 (Reporting)

---

## 1. PRÉSENTATION DU MODULE

### 1.1 Contexte et justification

Tout système d'archivage déployé en production évolue dans le temps : l'organisation change de prestataire SMTP, les durées légales de conservation sont mises à jour par décret, un nouveau scanner est ajouté, les seuils d'alerte de stockage doivent être ajustés. Sans module d'administration dédié, chaque changement exige une intervention technique sur le code ou les fichiers de configuration — avec redéploiement, coupure de service et risque d'erreur.

Ce module garantit que **l'intégralité de la configuration du système est modifiable sans redéploiement**, depuis une interface sécurisée, avec traçabilité complète de chaque modification et capacité de rollback immédiat.

### 1.2 Périmètre fonctionnel

Ce module couvre :
- La gestion des paramètres globaux du système (key-value JSONB)
- La configuration du serveur SMTP et des templates d'emails
- La configuration du stockage (local, NAS, chemins)
- La gestion des packs OCR (langues Tesseract)
- La définition des jours fériés (impact sur les délais workflows)
- La gestion des fenêtres de maintenance planifiées
- Les opérations de maintenance base de données (VACUUM, REINDEX)
- Le monitoring des ressources système (CPU, RAM, disque, PostgreSQL)
- L'historique complet de toutes les modifications de configuration
- L'import/export de la configuration complète (YAML/JSON)
- La vérification de santé du système (health check multi-composants)

### 1.3 Principe cardinal

**Aucune modification de configuration ne s'applique sans validation préalable.** Chaque paramètre modifié passe par un cycle : modification → validation automatique de la cohérence → aperçu de l'impact → confirmation → application → journalisation. Un rollback vers la valeur précédente est possible à tout moment depuis l'historique.

---

## 2. CAS D'UTILISATION — VUE D'ENSEMBLE

| Code | Cas d'utilisation | Acteur principal | Priorité |
|------|-------------------|-----------------|----------|
| UC-015-01 | Consulter tous les paramètres système | Super Admin, Admin | HAUTE |
| UC-015-02 | Modifier un paramètre global (nom org., logo, fuseau horaire) | Super Admin, Admin | HAUTE |
| UC-015-03 | Configurer le serveur SMTP (hôte, port, auth, TLS) | Super Admin, Admin | HAUTE |
| UC-015-04 | Tester la configuration SMTP (envoi email de test) | Super Admin, Admin | HAUTE |
| UC-015-05 | Créer ou modifier un template d'email | Super Admin, Admin | HAUTE |
| UC-015-06 | Prévisualiser un template d'email avec données de test | Super Admin, Admin | MOYENNE |
| UC-015-07 | Configurer les chemins de stockage (local, NAS) | Super Admin | HAUTE |
| UC-015-08 | Tester la connectivité du stockage | Super Admin, Admin | HAUTE |
| UC-015-09 | Gérer les packs de langues OCR (Tesseract) | Super Admin, Admin | MOYENNE |
| UC-015-10 | Configurer les seuils d'alerte globaux | Super Admin, Admin | HAUTE |
| UC-015-11 | Définir les durées de rétention par défaut par type documentaire | Super Admin, Admin | HAUTE |
| UC-015-12 | Gérer le calendrier des jours fériés | Super Admin, Admin | HAUTE |
| UC-015-13 | Configurer les paramètres d'authentification (timeout session, TOTP) | Super Admin | HAUTE |
| UC-015-14 | Configurer les règles de mot de passe | Super Admin | HAUTE |
| UC-015-15 | Planifier une fenêtre de maintenance | Super Admin, Admin | MOYENNE |
| UC-015-16 | Activer le mode maintenance (accès restreint) | Super Admin | HAUTE |
| UC-015-17 | Consulter les logs système (distinct de l'audit) | Super Admin, Admin | HAUTE |
| UC-015-18 | Effectuer un VACUUM et REINDEX PostgreSQL | Super Admin | MOYENNE |
| UC-015-19 | Consulter le monitoring des ressources (CPU, RAM, disque, DB) | Super Admin, Admin | HAUTE |
| UC-015-20 | Effectuer un health check complet du système | Super Admin, Admin | HAUTE |
| UC-015-21 | Consulter l'historique des modifications de configuration | Super Admin, Admin | HAUTE |
| UC-015-22 | Effectuer un rollback vers une configuration précédente | Super Admin | HAUTE |
| UC-015-23 | Exporter la configuration complète (YAML/JSON) | Super Admin | MOYENNE |
| UC-015-24 | Importer une configuration depuis un fichier | Super Admin | MOYENNE |
| UC-015-25 | Gérer les clés API des intégrations externes | Super Admin, Admin | HAUTE |

---

## 3. CAS D'UTILISATION DÉTAILLÉS

### UC-015-02 — Modifier un paramètre global

**Acteur principal :** Super Admin, Admin
**Pré-conditions :** Utilisateur authentifié avec les droits requis, paramètre non verrouillé (certains paramètres critiques sont réservés Super Admin)

**Flux principal :**
1. L'utilisateur accède à "Administration > Paramètres système"
2. Le système affiche les paramètres organisés par catégorie (Général, Authentification, Stockage, OCR, Notifications, RGPD)
3. L'utilisateur localise le paramètre à modifier et clique sur "Modifier"
4. Un formulaire contextuel s'affiche avec la valeur actuelle, la description du paramètre, le type attendu (texte, nombre, booléen, JSON), et des exemples
5. L'utilisateur saisit la nouvelle valeur
6. Le système valide la valeur (type, plage, format) avant toute modification
7. Le système affiche un résumé "Valeur actuelle → Nouvelle valeur" et demande confirmation
8. L'utilisateur confirme (avec TOTP si paramètre critique)
9. Le système sauvegarde la nouvelle valeur dans `SystemSetting`
10. L'ancienne valeur est archivée dans `ConfigChangeLog`
11. Le paramètre est appliqué immédiatement (mise à jour du cache Redis de configuration)
12. Un événement d'audit est généré (MODULE 11)

**Flux alternatifs :**
- **6a.** Valeur invalide (type incorrect, hors plage) → message d'erreur précis, pas de sauvegarde
- **6b.** Incohérence détectée (ex: SMTP port 465 avec TLS=False) → avertissement avec recommandation
- **8a.** Confirmation refusée → aucune modification
- **9a.** Erreur d'application (Redis indisponible) → rollback automatique, alerte Admin

**Post-conditions :**
- Nouvelle valeur active immédiatement
- Ancienne valeur archivée avec horodatage
- Événement `CONFIG_PARAMETER_CHANGED` journalisé

---

### UC-015-04 — Tester la configuration SMTP

**Acteur principal :** Super Admin, Admin
**Pré-conditions :** Configuration SMTP renseignée (hôte, port, credentials)

**Flux principal :**
1. L'utilisateur accède à "Administration > Messagerie > Tester SMTP"
2. Il saisit une adresse email de destination pour le test
3. Il clique sur "Envoyer un email de test"
4. Le système tente une connexion SMTP avec les paramètres configurés
5. Si la connexion réussit, un email de test est envoyé avec le template `smtp_test`
6. Le résultat s'affiche : succès (avec délai de connexion) ou échec (avec message d'erreur détaillé)
7. Un événement d'audit est généré `SMTP_TEST_PERFORMED`

**Flux alternatifs :**
- **4a.** Hôte SMTP inaccessible → "Timeout de connexion après 10s. Vérifiez l'hôte et le port."
- **4b.** Authentification refusée → "Identifiants SMTP incorrects."
- **4c.** Certificat TLS invalide → "Certificat TLS non reconnu. Vérifiez la configuration TLS."

---

### UC-015-16 — Activer le mode maintenance

**Acteur principal :** Super Admin
**Pré-conditions :** Utilisateur authentifié Super Admin, TOTP validé

**Flux principal :**
1. Le Super Admin accède à "Administration > Maintenance > Mode maintenance"
2. Il configure :
   - Message affiché aux utilisateurs pendant la maintenance (obligatoire)
   - Durée estimée (informative, ex: "2 heures")
   - Exclure les IPs administrateurs (liste blanche)
3. Il valide avec TOTP
4. Le système active le mode maintenance :
   - Toutes les nouvelles connexions reçoivent une page "Maintenance en cours" avec le message configuré
   - Les sessions actives reçoivent une notification (MODULE 12, priorité CRITIQUE) avec un délai de grâce de 5 minutes
   - Après 5 minutes, les sessions sont fermées proprement (pas de kill brutal)
   - Les IPs en liste blanche restent accessibles
5. Un événement d'audit est généré (MODULE 11, sévérité ALERT)
6. Le Super Admin réalise les opérations de maintenance
7. Il désactive le mode maintenance depuis la même interface
8. Les utilisateurs reçoivent une notification "Système disponible"

**Règle de sécurité :** Le mode maintenance ne peut être activé que par un Super Admin avec TOTP validé. Un Admin seul ne peut pas activer ce mode.

---

### UC-015-20 — Health check complet du système

**Acteur principal :** Super Admin, Admin
**Pré-conditions :** Utilisateur authentifié Admin minimum

**Flux principal :**
1. L'utilisateur accède à "Administration > Santé du système" ou appelle `GET /api/v1/admin/health/`
2. Le système effectue les vérifications suivantes en parallèle :
   - **PostgreSQL :** Connexion active, temps de réponse, taille des tables, index invalides, sessions bloquées
   - **Redis :** Connexion active, mémoire utilisée, hit rate du cache
   - **Celery :** Workers actifs, file d'attente (taille et âge des tâches en attente)
   - **Celery Beat :** Dernière exécution des tâches planifiées, prochaine exécution
   - **Stockage :** Accessibilité du chemin principal, espace disponible, droits en écriture
   - **OCR (Tesseract) :** Version installée, packs de langues disponibles
   - **SMTP :** Connectivité (ping uniquement, pas d'envoi)
   - **Sauvegardes :** Date de la dernière sauvegarde réussie (depuis MODULE 16)
3. Chaque composant retourne un statut : `OK`, `WARNING`, ou `CRITICAL`
4. Un statut global est calculé : `OK` si tout est OK, `WARNING` si au moins un WARNING, `CRITICAL` si au moins un CRITICAL
5. Le résultat est affiché dans un tableau de bord dédié avec indicateurs visuels

**Post-conditions :** Résultat du health check mis en cache Redis (TTL 60 secondes)

---

### UC-015-22 — Rollback vers une configuration précédente

**Acteur principal :** Super Admin
**Pré-conditions :** Historique de modifications disponible (`ConfigChangeLog`)

**Flux principal :**
1. Le Super Admin accède à "Administration > Historique des configurations"
2. Il filtre par paramètre, catégorie ou période
3. Il localise la version à restaurer dans l'historique
4. Il clique sur "Restaurer cette valeur"
5. Le système affiche une confirmation "Vous allez restaurer [paramètre] à sa valeur du [date] : [valeur]"
6. Le Super Admin confirme avec TOTP
7. La valeur est restaurée (crée un nouveau `SystemSetting` et un nouveau `ConfigChangeLog` de type `ROLLBACK`)
8. Un événement d'audit est généré

---

## 4. MODÈLES DE DONNÉES

### 4.1 SystemSetting — Paramètres globaux

```python
class SystemSetting(BaseModel):
    """
    Paramètre de configuration global du système.
    Structure key-value avec JSONB pour les valeurs complexes.
    Accès via cache Redis (TTL 5 minutes) pour les lectures fréquentes.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    # Clé unique du paramètre
    key = models.CharField(
        max_length=200,
        unique=True,
        db_index=True,
        help_text="Clé dot-notation : ex: smtp.host, auth.session_timeout_minutes, storage.main_path"
    )

    # Catégorie d'organisation
    category = models.CharField(
        max_length=50,
        choices=[
            ('GENERAL', 'Général'),
            ('AUTH', 'Authentification'),
            ('SMTP', 'Messagerie SMTP'),
            ('STORAGE', 'Stockage'),
            ('OCR', 'OCR & Numérisation'),
            ('NOTIFICATIONS', 'Notifications'),
            ('RGPD', 'RGPD & Conformité'),
            ('SECURITY', 'Sécurité'),
            ('WORKFLOW', 'Workflows'),
            ('REPORTING', 'Reporting'),
            ('INTEGRATION', 'Intégrations'),
        ],
        db_index=True
    )

    # Valeur (JSONB pour supporter tous les types)
    value = models.JSONField(
        help_text="Valeur du paramètre (string, int, bool, list, dict selon value_type)"
    )

    # Type de valeur attendu (pour validation et affichage UI)
    value_type = models.CharField(
        max_length=20,
        choices=[
            ('STRING', 'Texte'),
            ('INTEGER', 'Entier'),
            ('FLOAT', 'Décimal'),
            ('BOOLEAN', 'Booléen'),
            ('JSON', 'JSON complexe'),
            ('EMAIL', 'Adresse email'),
            ('URL', 'URL'),
            ('PATH', 'Chemin fichier'),
            ('SECRET', 'Secret (masqué dans l\'interface)'),
            ('COLOR', 'Couleur hexadécimale'),
        ]
    )

    # Métadonnées du paramètre
    label = models.CharField(
        max_length=300,
        help_text="Libellé affiché dans l'interface"
    )
    description = models.TextField(
        blank=True,
        help_text="Explication détaillée et exemples de valeurs valides"
    )
    default_value = models.JSONField(
        null=True, blank=True,
        help_text="Valeur par défaut du système (pour pouvoir réinitialiser)"
    )

    # Validation
    validation_regex = models.CharField(
        max_length=500,
        blank=True,
        help_text="Expression régulière de validation (optionnelle)"
    )
    min_value = models.FloatField(
        null=True, blank=True,
        help_text="Valeur minimale pour les types numériques"
    )
    max_value = models.FloatField(
        null=True, blank=True,
        help_text="Valeur maximale pour les types numériques"
    )
    allowed_values = models.JSONField(
        null=True, blank=True,
        help_text="Liste des valeurs autorisées (enum) — null si valeur libre"
    )

    # Sécurité et restrictions
    is_sensitive = models.BooleanField(
        default=False,
        help_text="Si True, valeur masquée dans les logs et l'interface (mots de passe, tokens)"
    )
    requires_super_admin = models.BooleanField(
        default=False,
        help_text="Si True, seul un Super Admin peut modifier ce paramètre"
    )
    requires_totp = models.BooleanField(
        default=False,
        help_text="Si True, confirmation TOTP obligatoire avant application"
    )
    requires_restart = models.BooleanField(
        default=False,
        help_text="Si True, un redémarrage du serveur est requis (rare, documenté)"
    )
    is_locked = models.BooleanField(
        default=False,
        help_text="Si True, paramètre verrouillé et non modifiable via l'interface"
    )

    # Ordre d'affichage dans l'interface
    display_order = models.PositiveSmallIntegerField(default=0)

    # Horodatage
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    updated_by = models.ForeignKey(
        'users.User',
        on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='updated_settings'
    )

    # Soft delete (rare, uniquement pour paramètres dépréciés)
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='deleted_settings'
    )

    class Meta:
        db_table = 'system_settings'
        indexes = [
            models.Index(fields=['category', 'display_order']),
            models.Index(fields=['key']),
        ]
        ordering = ['category', 'display_order']

    @classmethod
    def get(cls, key: str, default=None):
        """
        Méthode d'accès rapide avec cache Redis.
        Usage : SystemSetting.get('smtp.host', default='localhost')
        """
        from django.core.cache import cache
        cache_key = f"system_setting:{key}"
        cached = cache.get(cache_key)
        if cached is not None:
            return cached
        try:
            setting = cls.objects.get(key=key, is_deleted=False)
            value = setting.value
            cache.set(cache_key, value, timeout=300)
            return value
        except cls.DoesNotExist:
            return default

    @classmethod
    def set(cls, key: str, value, user=None):
        """
        Méthode de mise à jour avec invalidation du cache et archivage.
        """
        from django.core.cache import cache
        setting = cls.objects.get(key=key, is_deleted=False)
        old_value = setting.value

        # Archiver l'ancienne valeur
        ConfigChangeLog.objects.create(
            setting=setting,
            old_value=old_value,
            new_value=value,
            changed_by=user,
            change_type='UPDATE'
        )

        # Appliquer la nouvelle valeur
        setting.value = value
        setting.updated_by = user
        setting.save(update_fields=['value', 'updated_by', 'updated_at'])

        # Invalider le cache
        cache.delete(f"system_setting:{key}")
```

---

### 4.2 ConfigChangeLog — Historique des modifications

```python
class ConfigChangeLog(BaseModel):
    """
    Historique immuable de toutes les modifications de configuration.
    Append-only : jamais de UPDATE ni de DELETE sur cette table.
    Permet le rollback et l'audit des changements de configuration.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    # Paramètre modifié
    setting = models.ForeignKey(
        SystemSetting,
        on_delete=models.PROTECT,
        related_name='change_history'
    )
    setting_key = models.CharField(
        max_length=200,
        help_text="Copie dénormalisée de la clé pour l'historique (même si le paramètre est supprimé)"
    )

    # Valeurs
    old_value = models.JSONField(
        null=True,
        help_text="Valeur avant modification (null pour la création initiale)"
    )
    new_value = models.JSONField(
        help_text="Valeur après modification"
    )

    # Type de changement
    change_type = models.CharField(
        max_length=20,
        choices=[
            ('CREATE', 'Création du paramètre'),
            ('UPDATE', 'Modification de la valeur'),
            ('ROLLBACK', 'Restauration d\'une valeur précédente'),
            ('RESET', 'Réinitialisation à la valeur par défaut'),
            ('IMPORT', 'Import depuis fichier de configuration'),
        ]
    )

    # Auteur de la modification
    changed_by = models.ForeignKey(
        'users.User',
        on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='config_changes',
        help_text="Null si modification automatique du système"
    )

    # Contexte
    change_reason = models.TextField(
        blank=True,
        help_text="Raison optionnelle saisie par l'administrateur"
    )
    source_ip = models.GenericIPAddressField(
        null=True, blank=True,
        help_text="IP de l'administrateur au moment du changement"
    )

    # Référence rollback (si ce changement annule un changement précédent)
    rollback_of = models.ForeignKey(
        'self',
        on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='rolled_back_by'
    )

    # Horodatage immuable
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)

    # Soft delete (uniquement pour raisons légales)
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='deleted_config_changes'
    )

    class Meta:
        db_table = 'config_change_logs'
        indexes = [
            models.Index(fields=['setting', 'created_at']),
            models.Index(fields=['changed_by', 'created_at']),
            models.Index(fields=['setting_key', 'created_at']),
            models.Index(fields=['change_type']),
        ]
        ordering = ['-created_at']
```

---

### 4.3 EmailTemplate — Templates d'emails

```python
class EmailTemplate(BaseModel):
    """
    Template d'email personnalisable pour toutes les notifications envoyées par le système.
    Utilise le moteur de template Django ({{ variable }}) pour l'interpolation.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    # Identifiant technique unique (utilisé dans le code pour référencer le template)
    slug = models.CharField(
        max_length=100,
        unique=True,
        help_text="Identifiant technique non modifiable (ex: access_request_approved, smtp_test)"
    )

    name = models.CharField(
        max_length=300,
        help_text="Nom lisible (ex: 'Approbation demande d\\'accès')"
    )
    description = models.TextField(
        blank=True,
        help_text="Description de quand ce template est utilisé"
    )

    # Contenu du template
    subject = models.CharField(
        max_length=500,
        help_text="Objet de l'email avec variables : ex: [{{ organization_name }}] Votre demande a été approuvée"
    )
    body_html = models.TextField(
        help_text="Corps HTML de l'email avec variables Jinja2"
    )
    body_text = models.TextField(
        blank=True,
        help_text="Version texte brut (fallback si HTML non supporté)"
    )

    # Variables disponibles dans ce template
    available_variables = models.JSONField(
        default=list,
        help_text="Liste des variables disponibles avec description : [{name, description, example}]"
    )

    # Paramètres d'envoi
    from_name = models.CharField(
        max_length=200,
        blank=True,
        help_text="Nom d'expéditeur spécifique (si vide, utilise le paramètre global)"
    )
    reply_to = models.EmailField(
        blank=True,
        help_text="Adresse reply-to spécifique (si vide, utilise le paramètre global)"
    )

    # Statut
    is_active = models.BooleanField(
        default=True,
        help_text="Si False, le template système par défaut est utilisé"
    )
    is_system = models.BooleanField(
        default=False,
        help_text="Si True, template prédéfini non supprimable (modifiable uniquement)"
    )

    # Horodatage
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    updated_by = models.ForeignKey(
        'users.User',
        on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='updated_email_templates'
    )

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='deleted_email_templates'
    )

    class Meta:
        db_table = 'email_templates'
        indexes = [
            models.Index(fields=['slug']),
            models.Index(fields=['is_active']),
        ]
```

---

### 4.4 Holiday — Calendrier des jours fériés

```python
class Holiday(BaseModel):
    """
    Jours fériés impactant le calcul des délais de workflow.
    Un jour férié exclut ce jour du calcul des "jours ouvrés".
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    name = models.CharField(
        max_length=200,
        help_text="Libellé du jour férié (ex: 'Fête Nationale', '1er Mai')"
    )
    date = models.DateField(
        db_index=True,
        help_text="Date exacte du jour férié"
    )

    # Type de récurrence
    recurrence = models.CharField(
        max_length=20,
        choices=[
            ('ONE_TIME', 'Date unique (ex: pont décidé cette année)'),
            ('ANNUAL_FIXED', 'Annuel date fixe (ex: 14 juillet)'),
            ('ANNUAL_VARIABLE', 'Annuel date variable (ex: lundi de Pâques)'),
        ],
        default='ANNUAL_FIXED'
    )

    # Pour les récurrences annuelles fixes (mois + jour)
    recurrence_month = models.PositiveSmallIntegerField(
        null=True, blank=True,
        help_text="Mois pour les récurrences annuelles (1-12)"
    )
    recurrence_day = models.PositiveSmallIntegerField(
        null=True, blank=True,
        help_text="Jour pour les récurrences annuelles fixes (1-31)"
    )

    # Périmètre géographique (utile si plusieurs sites/régions)
    scope = models.CharField(
        max_length=50,
        default='NATIONAL',
        help_text="Périmètre : NATIONAL, REGIONAL, LOCAL (ex: code région)"
    )

    is_active = models.BooleanField(default=True)

    # Horodatage
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    created_by = models.ForeignKey(
        'users.User',
        on_delete=models.PROTECT,
        related_name='created_holidays'
    )

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='deleted_holidays'
    )

    class Meta:
        db_table = 'holidays'
        indexes = [
            models.Index(fields=['date', 'is_active']),
            models.Index(fields=['recurrence', 'is_active']),
        ]
        constraints = [
            models.UniqueConstraint(
                fields=['date', 'scope'],
                condition=models.Q(is_deleted=False),
                name='unique_holiday_per_date_scope'
            ),
        ]
```

---

### 4.5 MaintenanceWindow — Fenêtres de maintenance

```python
class MaintenanceWindow(BaseModel):
    """
    Fenêtre de maintenance planifiée ou active.
    Pendant une maintenance active, les connexions sont bloquées
    sauf pour les IPs en liste blanche.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    name = models.CharField(
        max_length=200,
        help_text="Libellé interne (ex: 'Migration base de données v2.1')"
    )
    user_message = models.TextField(
        help_text="Message affiché aux utilisateurs sur la page de maintenance"
    )
    estimated_duration_minutes = models.PositiveIntegerField(
        help_text="Durée estimée en minutes (affichée à titre informatif)"
    )

    # Planification
    scheduled_start = models.DateTimeField(
        null=True, blank=True,
        help_text="Démarrage planifié (null si déclenchement manuel)"
    )
    scheduled_end = models.DateTimeField(
        null=True, blank=True,
        help_text="Fin planifiée"
    )

    # Statut réel
    status = models.CharField(
        max_length=20,
        choices=[
            ('SCHEDULED', 'Planifiée'),
            ('ACTIVE', 'En cours'),
            ('COMPLETED', 'Terminée'),
            ('CANCELLED', 'Annulée'),
        ],
        default='SCHEDULED',
        db_index=True
    )
    actual_start = models.DateTimeField(null=True, blank=True)
    actual_end = models.DateTimeField(null=True, blank=True)

    # Accès pendant la maintenance
    whitelist_ips = models.JSONField(
        default=list,
        help_text="IPs autorisées pendant la maintenance (admins) : ['192.168.1.10', '10.0.0.1']"
    )
    grace_period_minutes = models.PositiveSmallIntegerField(
        default=5,
        help_text="Délai avant déconnexion forcée des sessions actives"
    )

    # Auteur
    created_by = models.ForeignKey(
        'users.User',
        on_delete=models.PROTECT,
        related_name='created_maintenance_windows'
    )
    activated_by = models.ForeignKey(
        'users.User',
        on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='activated_maintenance_windows'
    )

    # Notes post-maintenance
    completion_notes = models.TextField(
        blank=True,
        help_text="Notes rédigées après la maintenance (ce qui a été fait, incidents)"
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
        related_name='deleted_maintenance_windows'
    )

    class Meta:
        db_table = 'maintenance_windows'
        indexes = [
            models.Index(fields=['status', 'scheduled_start']),
            models.Index(fields=['status']),
        ]
```

---

## 5. PARAMÈTRES SYSTÈME PRÉDÉFINIS

Voici l'inventaire complet des paramètres système initialisés lors du déploiement :

### 5.1 Catégorie GENERAL

| Clé | Type | Valeur par défaut | Description |
|-----|------|-------------------|-------------|
| `general.organization_name` | STRING | "Administration" | Nom affiché dans l'interface et les rapports |
| `general.organization_logo_url` | URL | "" | URL du logo (hébergé localement) |
| `general.timezone` | STRING | "Europe/Paris" | Fuseau horaire du système |
| `general.default_language` | STRING | "fr" | Langue par défaut de l'interface |
| `general.date_format` | STRING | "DD/MM/YYYY" | Format d'affichage des dates |
| `general.max_upload_size_mb` | INTEGER | 100 | Taille maximale d'upload en MB |
| `general.allowed_file_extensions` | JSON | [".pdf",".docx",...] | Extensions autorisées à l'upload |

### 5.2 Catégorie AUTH

| Clé | Type | Valeur par défaut | Description |
|-----|------|-------------------|-------------|
| `auth.session_timeout_minutes` | INTEGER | 30 | Durée de validité de la session inactive |
| `auth.jwt_access_token_lifetime_minutes` | INTEGER | 15 | Durée de vie du JWT access token |
| `auth.jwt_refresh_token_lifetime_days` | INTEGER | 7 | Durée de vie du JWT refresh token |
| `auth.totp_required` | BOOLEAN | false | TOTP obligatoire pour tous les utilisateurs |
| `auth.totp_required_for_roles` | JSON | ["SUPER_ADMIN","ADMIN"] | Rôles pour lesquels le TOTP est obligatoire |
| `auth.max_login_attempts` | INTEGER | 5 | Tentatives de connexion avant blocage |
| `auth.lockout_duration_minutes` | INTEGER | 15 | Durée du blocage après trop de tentatives |
| `auth.password_min_length` | INTEGER | 12 | Longueur minimale du mot de passe |
| `auth.password_require_uppercase` | BOOLEAN | true | Majuscule obligatoire |
| `auth.password_require_number` | BOOLEAN | true | Chiffre obligatoire |
| `auth.password_require_special` | BOOLEAN | true | Caractère spécial obligatoire |
| `auth.password_history_count` | INTEGER | 5 | Nombre de mots de passe précédents interdits |
| `auth.password_max_age_days` | INTEGER | 90 | Durée de validité max du mot de passe |

### 5.3 Catégorie SMTP

| Clé | Type | Valeur par défaut | Description |
|-----|------|-------------------|-------------|
| `smtp.host` | STRING | "localhost" | Hôte du serveur SMTP |
| `smtp.port` | INTEGER | 587 | Port SMTP |
| `smtp.use_tls` | BOOLEAN | true | Activer TLS |
| `smtp.use_ssl` | BOOLEAN | false | Activer SSL (mutuellement exclusif avec TLS) |
| `smtp.username` | STRING | "" | Identifiant SMTP |
| `smtp.password` | SECRET | "" | Mot de passe SMTP (masqué) |
| `smtp.from_email` | EMAIL | "noreply@administration.fr" | Email expéditeur par défaut |
| `smtp.from_name` | STRING | "Système d'Archivage" | Nom expéditeur par défaut |
| `smtp.reply_to` | EMAIL | "" | Email reply-to par défaut |
| `smtp.timeout_seconds` | INTEGER | 10 | Timeout de connexion SMTP |

### 5.4 Catégorie STORAGE

| Clé | Type | Valeur par défaut | Description |
|-----|------|-------------------|-------------|
| `storage.main_path` | PATH | "/var/archivage/media" | Chemin principal des fichiers |
| `storage.backup_path` | PATH | "/var/archivage/backups" | Chemin des sauvegardes |
| `storage.reports_path` | PATH | "/var/archivage/reports" | Chemin des rapports générés |
| `storage.temp_path` | PATH | "/tmp/archivage" | Chemin fichiers temporaires |
| `storage.warning_threshold_pct` | INTEGER | 80 | Alerte orange à X% d'utilisation |
| `storage.critical_threshold_pct` | INTEGER | 95 | Alerte rouge à X% d'utilisation |

### 5.5 Catégorie OCR

| Clé | Type | Valeur par défaut | Description |
|-----|------|-------------------|-------------|
| `ocr.tesseract_path` | PATH | "/usr/bin/tesseract" | Chemin de l'exécutable Tesseract |
| `ocr.default_languages` | JSON | ["fra","eng"] | Langues OCR par défaut |
| `ocr.min_quality_score` | INTEGER | 60 | Score qualité minimum (0-100) en dessous duquel une correction manuelle est suggérée |
| `ocr.max_parallel_pages` | INTEGER | 4 | Pages traitées en parallèle |
| `ocr.dpi_target` | INTEGER | 300 | DPI cible pour la numérisation |

### 5.6 Catégorie WORKFLOW

| Clé | Type | Valeur par défaut | Description |
|-----|------|-------------------|-------------|
| `workflow.default_approval_delay_days` | INTEGER | 5 | Délai par défaut pour approuver (jours ouvrés) |
| `workflow.escalation_delay_hours` | INTEGER | 24 | Délai avant escalade de niveau 1 |
| `workflow.max_escalation_levels` | INTEGER | 3 | Nombre maximum de niveaux d'escalade |
| `workflow.weekend_days` | JSON | [5,6] | Jours considérés non ouvrés (5=sam, 6=dim) |

### 5.7 Catégorie RGPD

| Clé | Type | Valeur par défaut | Description |
|-----|------|-------------------|-------------|
| `rgpd.data_retention_default_years` | INTEGER | 5 | Durée de conservation par défaut |
| `rgpd.anonymization_delay_days` | INTEGER | 30 | Délai avant anonymisation effective |
| `rgpd.dpo_email` | EMAIL | "" | Email du Délégué à la Protection des Données |
| `rgpd.privacy_policy_url` | URL | "" | URL de la politique de confidentialité |

---

## 6. MATRICE DES PERMISSIONS

| Action | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|--------|:-----------:|:-----:|:----------:|:-----------:|:-----:|:--------:|
| Consulter tous les paramètres | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Modifier paramètres généraux | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Modifier paramètres AUTH | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Modifier paramètres SMTP | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Tester configuration SMTP | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Modifier paramètres STOCKAGE | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Modifier paramètres OCR | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Gérer templates d'emails | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Gérer jours fériés | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Planifier fenêtre maintenance | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Activer mode maintenance | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Voir logs système | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Déclencher VACUUM/REINDEX | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Consulter monitoring ressources | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Effectuer health check | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Voir historique configuration | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ (lecture) |
| Effectuer rollback configuration | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Exporter configuration (YAML/JSON) | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Importer configuration | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Gérer clés API intégrations | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Modifier paramètres RGPD | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

---

## 7. SÉQUENCES DÉTAILLÉES

### Séquence 1 — Modification d'un paramètre avec validation

```
Admin                    Backend                     Redis / PostgreSQL
  |                         |                               |
  |-- PATCH /api/v1/admin/settings/smtp.host/ -->|          |
  |   { "value": "smtp.nouveau-prestataire.fr" } |          |
  |                         |                               |
  |                         |-- Vérifie droits Admin        |
  |                         |-- Charge SystemSetting(key)   |
  |                         |-- Valide type (STRING) ✅     |
  |                         |-- Valide format (hostname) ✅ |
  |                         |-- Détecte incohérences -------|
  |                         |   (port 587 + TLS=true → OK) |
  |                         |                               |
  |<-- 200 Preview ---------|                               |
  |   {                     |                               |
  |     old_value: "smtp.actuel.fr",                        |
  |     new_value: "smtp.nouveau-prestataire.fr",           |
  |     requires_totp: false,                               |
  |     warnings: []                                        |
  |   }                     |                               |
  |                         |                               |
  |-- POST /confirm/ ------>|                               |
  |   { "confirmed": true } |                               |
  |                         |                               |
  |                         |-- Crée ConfigChangeLog ------>|
  |                         |-- Update SystemSetting ------->|
  |                         |-- Invalide cache Redis ------->|
  |                         |   (key: system_setting:smtp.host)
  |                         |-- Journalise (MOD 11) ------->|
  |                         |                               |
  |<-- 200 Applied ---------|                               |
  |   { "status": "applied", "applied_at": "..." }          |
```

---

### Séquence 2 — Health check complet

```
Admin                    Backend                    Composants
  |                         |                           |
  |-- GET /api/v1/admin/health/ -->|                    |
  |                         |                           |
  |                         |-- Checks en parallèle --->|
  |                         |   (asyncio.gather)        |
  |                         |                           |
  |                         |-- Check PostgreSQL ------->|
  |                         |   SELECT 1 + stats query  |
  |                         |<-- OK (23ms) -------------|
  |                         |                           |
  |                         |-- Check Redis ------------>|
  |                         |   PING + INFO memory      |
  |                         |<-- OK (2ms) --------------|
  |                         |                           |
  |                         |-- Check Celery ----------->|
  |                         |   app.control.inspect()   |
  |                         |<-- 3 workers actifs -------|
  |                         |                           |
  |                         |-- Check Stockage --------->|
  |                         |   os.statvfs() + write test
  |                         |<-- OK, 67% utilisé --------|
  |                         |                           |
  |                         |-- Check SMTP ------------->|
  |                         |   socket connect (10s TO)  |
  |                         |<-- OK (145ms) -------------|
  |                         |                           |
  |                         |-- Check Sauvegardes ------>|
  |                         |   (Dernière backup réussie)|
  |                         |<-- WARNING (36h ago) ------|
  |                         |                           |
  |                         |-- Calcule statut global   |
  |                         |   → WARNING (backup)      |
  |                         |                           |
  |                         |-- Met en cache (TTL 60s)  |
  |                         |                           |
  |<-- 200 Health Report ---|                           |
  |   { "status": "WARNING", "components": [...] }      |
```

---

### Séquence 3 — Export et import de configuration

```
Super Admin              Backend                    Système de fichiers
     |                      |                              |
     |-- GET /api/v1/admin/config/export/ -->|             |
     |   { "format": "yaml", "categories": ["SMTP","AUTH"]}|
     |                      |                              |
     |                      |-- Collecte SystemSetting     |
     |                      |   (filtré par catégories)    |
     |                      |-- Masque les SECRET          |
     |                      |   (remplace par "***MASKED***")
     |                      |-- Sérialise en YAML          |
     |                      |-- Ajoute métadonnées :       |
     |                      |   version, date, checksum    |
     |                      |                              |
     |<-- 200 (attachment) --|                             |
     |   config_export_2026-02-18.yaml                     |
     |                      |                              |
     |  [Plus tard : import]|                              |
     |                      |                              |
     |-- POST /api/v1/admin/config/import/ -->|            |
     |   (fichier YAML uploadé)               |            |
     |                      |                              |
     |                      |-- Valide format YAML         |
     |                      |-- Vérifie checksum           |
     |                      |-- Valide chaque paramètre    |
     |                      |   (type, plage, cohérence)   |
     |                      |                              |
     |<-- 200 Preview -------|                             |
     |   { "changes": [     |                             |
     |     { key: "smtp.host",                            |
     |       old: "old.smtp.fr",                          |
     |       new: "new.smtp.fr" }                         |
     |   ], "errors": [] }  |                             |
     |                      |                              |
     |-- POST /confirm/ ---->|                             |
     |   (TOTP requis)       |                             |
     |                      |                              |
     |                      |-- Applique les changements   |
     |                      |-- Crée ConfigChangeLog       |
     |                      |   (type=IMPORT) par param.   |
     |                      |-- Invalide cache Redis       |
     |                      |-- Journalise (MOD 11)        |
     |                      |                              |
     |<-- 200 Applied -------|                             |
```

---

## 8. ENDPOINTS API

### 8.1 Paramètres système

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/admin/settings/` | Lister tous les paramètres (organisés par catégorie) | JWT + Admin |
| `GET` | `/api/v1/admin/settings/{key}/` | Détail d'un paramètre avec historique | JWT + Admin |
| `PATCH` | `/api/v1/admin/settings/{key}/preview/` | Prévisualiser le changement (validation sans application) | JWT + Admin |
| `PATCH` | `/api/v1/admin/settings/{key}/apply/` | Appliquer un changement validé | JWT + Admin |
| `POST` | `/api/v1/admin/settings/{key}/reset/` | Réinitialiser à la valeur par défaut | JWT + Super Admin |
| `GET` | `/api/v1/admin/settings/category/{category}/` | Paramètres d'une catégorie | JWT + Admin |

### 8.2 Historique de configuration

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/admin/config-history/` | Historique de toutes les modifications | JWT + Admin |
| `GET` | `/api/v1/admin/config-history/{id}/` | Détail d'une modification | JWT + Admin |
| `POST` | `/api/v1/admin/config-history/{id}/rollback/` | Rollback vers cette valeur (TOTP requis) | JWT + Super Admin |
| `GET` | `/api/v1/admin/settings/{key}/history/` | Historique d'un paramètre spécifique | JWT + Admin |

### 8.3 SMTP et templates emails

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `POST` | `/api/v1/admin/smtp/test/` | Envoyer un email de test | JWT + Admin |
| `GET` | `/api/v1/admin/email-templates/` | Lister les templates d'emails | JWT + Admin |
| `GET` | `/api/v1/admin/email-templates/{id}/` | Détail d'un template | JWT + Admin |
| `PATCH` | `/api/v1/admin/email-templates/{id}/` | Modifier un template | JWT + Admin |
| `POST` | `/api/v1/admin/email-templates/{id}/preview/` | Prévisualiser avec données de test | JWT + Admin |
| `POST` | `/api/v1/admin/email-templates/{id}/reset/` | Restaurer le template système par défaut | JWT + Admin |

### 8.4 Jours fériés

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/admin/holidays/` | Lister les jours fériés (filtrables par année) | JWT + Admin |
| `POST` | `/api/v1/admin/holidays/` | Ajouter un jour férié | JWT + Admin |
| `GET` | `/api/v1/admin/holidays/{id}/` | Détail d'un jour férié | JWT + Admin |
| `PATCH` | `/api/v1/admin/holidays/{id}/` | Modifier | JWT + Admin |
| `DELETE` | `/api/v1/admin/holidays/{id}/` | Supprimer (soft delete) | JWT + Admin |
| `POST` | `/api/v1/admin/holidays/import-france/` | Importer les jours fériés FR pour une année | JWT + Admin |

### 8.5 Maintenance

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/admin/maintenance/` | Lister les fenêtres de maintenance | JWT + Admin |
| `POST` | `/api/v1/admin/maintenance/` | Planifier une maintenance | JWT + Admin |
| `GET` | `/api/v1/admin/maintenance/{id}/` | Détail | JWT + Admin |
| `PATCH` | `/api/v1/admin/maintenance/{id}/` | Modifier une maintenance planifiée | JWT + Admin |
| `POST` | `/api/v1/admin/maintenance/{id}/activate/` | Activer immédiatement (TOTP requis) | JWT + Super Admin |
| `POST` | `/api/v1/admin/maintenance/{id}/deactivate/` | Désactiver la maintenance | JWT + Super Admin |
| `GET` | `/api/v1/admin/maintenance/current/` | Maintenance active en cours (si existante) | Public (pour la page maintenance) |

### 8.6 Santé et monitoring

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/admin/health/` | Health check complet de tous les composants | JWT + Admin |
| `GET` | `/api/v1/admin/health/quick/` | Health check rapide (PostgreSQL + Redis) | JWT + Admin |
| `GET` | `/api/v1/admin/monitoring/` | Métriques ressources (CPU, RAM, disque) | JWT + Admin |
| `GET` | `/api/v1/admin/monitoring/database/` | Statistiques PostgreSQL (tables, index, connexions) | JWT + Admin |
| `GET` | `/api/v1/admin/monitoring/celery/` | État des workers et file d'attente Celery | JWT + Admin |
| `GET` | `/api/v1/admin/logs/` | Logs système paginés avec filtres | JWT + Admin |

### 8.7 Opérations de maintenance base de données

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `POST` | `/api/v1/admin/database/vacuum/` | Déclencher un VACUUM ANALYZE (asynchrone) | JWT + Super Admin |
| `POST` | `/api/v1/admin/database/reindex/` | Déclencher un REINDEX (asynchrone) | JWT + Super Admin |
| `GET` | `/api/v1/admin/database/table-stats/` | Taille et fragmentation des tables | JWT + Admin |
| `GET` | `/api/v1/admin/database/slow-queries/` | Requêtes lentes récentes (pg_stat_statements) | JWT + Admin |
| `GET` | `/api/v1/admin/database/locks/` | Verrous actifs en base | JWT + Admin |

### 8.8 Import/Export configuration

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/admin/config/export/` | Exporter la configuration (YAML ou JSON) | JWT + Super Admin |
| `POST` | `/api/v1/admin/config/import/preview/` | Prévisualiser l'impact d'un import | JWT + Super Admin |
| `POST` | `/api/v1/admin/config/import/apply/` | Appliquer l'import (TOTP requis) | JWT + Super Admin |

---

## 9. FORMAT DES RÉPONSES API

### Exemple : Health check (GET `/api/v1/admin/health/`)

```json
{
  "success": true,
  "data": {
    "status": "WARNING",
    "checked_at": "2026-02-18T10:30:00Z",
    "cached_until": "2026-02-18T10:31:00Z",
    "components": {
      "postgresql": {
        "status": "OK",
        "response_time_ms": 23,
        "details": {
          "version": "PostgreSQL 16.2",
          "connections_active": 12,
          "connections_max": 100,
          "database_size_gb": 4.7,
          "invalid_indexes": 0,
          "blocked_queries": 0
        }
      },
      "redis": {
        "status": "OK",
        "response_time_ms": 2,
        "details": {
          "memory_used_mb": 156,
          "memory_max_mb": 1024,
          "hit_rate_pct": 94.3,
          "connected_clients": 8
        }
      },
      "celery": {
        "status": "OK",
        "details": {
          "workers_active": 3,
          "queue_length": 7,
          "oldest_task_age_seconds": 45
        }
      },
      "celery_beat": {
        "status": "OK",
        "details": {
          "last_heartbeat": "2026-02-18T10:29:55Z",
          "next_scheduled_task": "2026-02-18T10:35:00Z"
        }
      },
      "storage": {
        "status": "OK",
        "details": {
          "path": "/var/archivage/media",
          "total_gb": 2000,
          "used_gb": 1340,
          "used_pct": 67.0,
          "writable": true
        }
      },
      "smtp": {
        "status": "OK",
        "response_time_ms": 145
      },
      "ocr_tesseract": {
        "status": "OK",
        "details": {
          "version": "5.3.3",
          "installed_languages": ["fra", "eng", "ara"]
        }
      },
      "backups": {
        "status": "WARNING",
        "details": {
          "last_successful_backup": "2026-02-17T02:00:00Z",
          "hours_since_last_backup": 32.5,
          "expected_max_hours": 25,
          "warning_reason": "Dernière sauvegarde réussie il y a 32.5h (seuil : 25h)"
        }
      }
    }
  }
}
```

### Exemple : Paramètre système (GET `/api/v1/admin/settings/smtp.host/`)

```json
{
  "success": true,
  "data": {
    "key": "smtp.host",
    "category": "SMTP",
    "label": "Hôte du serveur SMTP",
    "description": "Adresse du serveur SMTP pour l'envoi des emails. Peut être un nom de domaine ou une adresse IP.",
    "value": "smtp.prefectures.fr",
    "value_type": "STRING",
    "default_value": "localhost",
    "is_sensitive": false,
    "requires_super_admin": false,
    "requires_totp": false,
    "is_locked": false,
    "updated_at": "2026-01-15T09:30:00Z",
    "updated_by": {
      "id": "...",
      "full_name": "Jean-Pierre Admin"
    },
    "recent_history": [
      {
        "id": "...",
        "old_value": "smtp.ancien.fr",
        "new_value": "smtp.prefectures.fr",
        "changed_by": "Jean-Pierre Admin",
        "changed_at": "2026-01-15T09:30:00Z",
        "change_type": "UPDATE"
      }
    ]
  }
}
```

---

## 10. SERVICE DE MONITORING SYSTÈME

```python
# apps/administration/services/monitoring_service.py

import os
import subprocess
import psutil
from django.db import connection
from django.core.cache import cache
from celery.app.control import Control


class SystemMonitoringService:
    """Service de collecte des métriques système pour le health check et le monitoring."""

    def get_full_health_report(self) -> dict:
        """
        Exécute tous les checks et retourne un rapport complet.
        Les checks sont exécutés en parallèle avec un timeout individuel de 10s.
        """
        import concurrent.futures
        checks = {
            'postgresql': self.check_postgresql,
            'redis': self.check_redis,
            'celery': self.check_celery,
            'celery_beat': self.check_celery_beat,
            'storage': self.check_storage,
            'smtp': self.check_smtp,
            'ocr_tesseract': self.check_tesseract,
            'backups': self.check_backups,
        }
        results = {}
        with concurrent.futures.ThreadPoolExecutor(max_workers=8) as executor:
            futures = {executor.submit(check_fn): name for name, check_fn in checks.items()}
            for future in concurrent.futures.as_completed(futures, timeout=15):
                name = futures[future]
                try:
                    results[name] = future.result(timeout=10)
                except Exception as e:
                    results[name] = {'status': 'CRITICAL', 'error': str(e)}

        global_status = 'OK'
        for component in results.values():
            if component.get('status') == 'CRITICAL':
                global_status = 'CRITICAL'
                break
            if component.get('status') == 'WARNING':
                global_status = 'WARNING'

        return {'status': global_status, 'components': results}

    def check_postgresql(self) -> dict:
        import time
        start = time.time()
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
            cursor.execute("""
                SELECT
                    pg_database_size(current_database()) AS db_size,
                    (SELECT count(*) FROM pg_stat_activity WHERE state = 'active') AS active_conns,
                    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_conns,
                    (SELECT count(*) FROM pg_stat_user_indexes WHERE NOT indisvalid) AS invalid_idx
            """)
            row = cursor.fetchone()
        elapsed_ms = (time.time() - start) * 1000
        return {
            'status': 'OK' if elapsed_ms < 100 else 'WARNING',
            'response_time_ms': round(elapsed_ms, 2),
            'details': {
                'database_size_gb': round(row[0] / 1024**3, 2),
                'connections_active': row[1],
                'connections_max': row[2],
                'invalid_indexes': row[3],
            }
        }

    def check_redis(self) -> dict:
        import time
        start = time.time()
        cache.set('health_check', '1', timeout=5)
        result = cache.get('health_check')
        elapsed_ms = (time.time() - start) * 1000
        if result != '1':
            return {'status': 'CRITICAL', 'error': 'Redis lecture/écriture échouée'}
        return {
            'status': 'OK' if elapsed_ms < 50 else 'WARNING',
            'response_time_ms': round(elapsed_ms, 2),
        }

    def check_storage(self) -> dict:
        from apps.administration.models import SystemSetting
        storage_path = SystemSetting.get('storage.main_path', '/var/archivage/media')
        warning_threshold = SystemSetting.get('storage.warning_threshold_pct', 80)
        critical_threshold = SystemSetting.get('storage.critical_threshold_pct', 95)

        if not os.path.exists(storage_path):
            return {'status': 'CRITICAL', 'error': f"Chemin inexistant : {storage_path}"}

        stat = os.statvfs(storage_path)
        total = stat.f_blocks * stat.f_frsize
        free = stat.f_bfree * stat.f_frsize
        used = total - free
        used_pct = (used / total * 100) if total > 0 else 0

        # Test d'écriture
        test_file = os.path.join(storage_path, '.health_check_write_test')
        try:
            with open(test_file, 'w') as f:
                f.write('ok')
            os.remove(test_file)
            writable = True
        except Exception:
            writable = False

        status = 'OK'
        if used_pct >= critical_threshold:
            status = 'CRITICAL'
        elif used_pct >= warning_threshold:
            status = 'WARNING'
        if not writable:
            status = 'CRITICAL'

        return {
            'status': status,
            'details': {
                'path': storage_path,
                'total_gb': round(total / 1024**3, 2),
                'used_gb': round(used / 1024**3, 2),
                'used_pct': round(used_pct, 1),
                'writable': writable,
            }
        }

    def check_tesseract(self) -> dict:
        from apps.administration.models import SystemSetting
        tesseract_path = SystemSetting.get('ocr.tesseract_path', '/usr/bin/tesseract')
        try:
            result = subprocess.run(
                [tesseract_path, '--version'],
                capture_output=True, text=True, timeout=5
            )
            version_line = result.stdout.split('\n')[0] if result.stdout else result.stderr.split('\n')[0]

            langs_result = subprocess.run(
                [tesseract_path, '--list-langs'],
                capture_output=True, text=True, timeout=5
            )
            languages = [
                l.strip() for l in langs_result.stderr.split('\n')
                if l.strip() and l.strip() != 'List of available tessdata languages:'
            ]
            return {
                'status': 'OK',
                'details': {
                    'version': version_line.strip(),
                    'installed_languages': languages
                }
            }
        except FileNotFoundError:
            return {'status': 'CRITICAL', 'error': f"Tesseract non trouvé : {tesseract_path}"}
        except subprocess.TimeoutExpired:
            return {'status': 'WARNING', 'error': 'Tesseract timeout (>5s)'}
```

---

## 11. SERVICE DE CALCUL DES JOURS OUVRÉS

```python
# apps/administration/services/working_days_service.py

from datetime import date, timedelta
from apps.administration.models import Holiday, SystemSetting


class WorkingDaysService:
    """
    Service de calcul des délais en jours ouvrés.
    Utilisé par les modules Workflow et Partage pour calculer les échéances.
    """

    def __init__(self, scope: str = 'NATIONAL'):
        self.scope = scope
        self._load_config()

    def _load_config(self):
        weekend_days = SystemSetting.get('workflow.weekend_days', [5, 6])
        self.weekend_days = set(weekend_days)

    def add_working_days(self, start_date: date, working_days: int) -> date:
        """
        Calcule la date résultant de l'ajout de N jours ouvrés à une date de départ.
        Exclut les week-ends (configurables) et les jours fériés (de la base).
        """
        current = start_date
        remaining = working_days

        while remaining > 0:
            current += timedelta(days=1)
            if self.is_working_day(current):
                remaining -= 1

        return current

    def count_working_days(self, start_date: date, end_date: date) -> int:
        """Compte le nombre de jours ouvrés entre deux dates (inclusif)."""
        count = 0
        current = start_date
        while current <= end_date:
            if self.is_working_day(current):
                count += 1
            current += timedelta(days=1)
        return count

    def is_working_day(self, check_date: date) -> bool:
        """Retourne True si la date est un jour ouvré."""
        if check_date.weekday() in self.weekend_days:
            return False
        if Holiday.objects.filter(
            date=check_date,
            is_active=True,
            is_deleted=False
        ).filter(
            models.Q(scope='NATIONAL') | models.Q(scope=self.scope)
        ).exists():
            return False
        return True
```

---

## 12. RÈGLES MÉTIER CRITIQUES

### RB-015-01 — Validation avant application, jamais de modification directe

Toute modification de paramètre passe obligatoirement par deux étapes : une étape de prévisualisation (qui valide et affiche l'impact) et une étape de confirmation. Il n'existe pas d'endpoint permettant de modifier directement un paramètre sans confirmation explicite. Cette règle est non contournable même pour le Super Admin.

---

### RB-015-02 — Paramètres sensibles masqués dans toutes les réponses

Les paramètres de type `SECRET` (mots de passe SMTP, tokens, clés API) ne sont jamais retournés en clair dans les réponses API. Ils sont remplacés par `"***MASKED***"` dans toutes les sérialisations. Pour modifier un secret, l'utilisateur saisit la nouvelle valeur sans jamais voir l'ancienne.

---

### RB-015-03 — Export de configuration sans secrets

L'export de configuration (YAML/JSON) ne contient jamais les valeurs des paramètres marqués `is_sensitive = True`. Ces valeurs sont remplacées par `"***MASKED***"`. L'import d'un fichier contenant `"***MASKED***"` ne modifie pas ces paramètres (valeur ignorée). Cela permet de partager des configurations de référence entre environnements sans risque de fuite de credentials.

---

### RB-015-04 — Mode maintenance uniquement via Super Admin + TOTP

L'activation du mode maintenance est l'action la plus critique du système. Elle nécessite : un compte Super Admin actif, une validation TOTP, et une confirmation explicite du message utilisateur. Un compte Admin seul ne peut jamais activer ce mode. Cette règle protège contre une mise en maintenance accidentelle ou malveillante.

---

### RB-015-05 — Rollback disponible pour toute modification

Chaque modification de paramètre génère systématiquement un enregistrement `ConfigChangeLog` archivant la valeur précédente. Il est donc toujours possible de revenir à n'importe quelle valeur historique en moins de 30 secondes. Le rollback lui-même génère un nouveau `ConfigChangeLog` (type `ROLLBACK`) pour maintenir la traçabilité complète.

---

### RB-015-06 — Health check mis en cache, non répétable en boucle

Le health check complet est mis en cache Redis avec un TTL de 60 secondes. Un utilisateur ne peut pas déclencher plus de 1 health check complet par minute (rate limiting). Le health check rapide (PostgreSQL + Redis uniquement) a un TTL de 10 secondes. Cette règle prévient une saturation des checks qui consommeraient des ressources système.

---

### RB-015-07 — Jours fériés et impact sur les workflows existants

Quand un nouveau jour férié est ajouté (ou un existant supprimé), le système recalcule automatiquement les `approval_deadline` de tous les workflows actifs (`WorkflowInstance.status = 'IN_PROGRESS'`) et les demandes d'accès en attente (`AccessRequest.status = 'PENDING'`). Cette recalculation est effectuée en tâche Celery pour ne pas bloquer la réponse HTTP.

---

## 13. TÂCHES CELERY PLANIFIÉES

### TASK-015-01 : Health check automatique périodique
```python
@shared_task(name='administration.auto_health_check')
# Fréquence     : Toutes les 15 minutes
# Description   : Exécute le health check complet et met à jour le cache
# Alerte        : Si un composant passe en CRITICAL → notification Super Admin + Admin (MODULE 12)
# Stockage      : Résultat stocké en Redis ET dans SystemLog pour historique
```

### TASK-015-02 : Activation automatique des maintenances planifiées
```python
@shared_task(name='administration.activate_scheduled_maintenance')
# Fréquence     : Toutes les 5 minutes
# Description   : Vérifie les MaintenanceWindow dont scheduled_start <= now() et status=SCHEDULED
#                 Déclenche automatiquement le mode maintenance
# Notification  : Utilisateurs actifs notifiés (délai de grâce configuré)
```

### TASK-015-03 : Recalcul des délais après modification du calendrier
```python
@shared_task(name='administration.recalculate_deadlines_after_holiday_change')
# Déclenchement : Via signal post_save sur Holiday
# Description   : Recalcule approval_deadline pour tous les AccessRequest PENDING
#                 et les WorkflowInstance IN_PROGRESS affectés
# Traitement    : Par lot de 200 pour éviter les locks
```

### TASK-015-04 : Purge des logs système anciens
```python
@shared_task(name='administration.purge_old_system_logs')
# Fréquence     : Hebdomadaire (dimanche 4h00)
# Description   : Soft delete des SystemLog > rétention configurée (défaut 90 jours)
# Note          : ConfigChangeLog n'est JAMAIS purgé (rétention illimitée)
```

### TASK-015-05 : Génération quotidienne du rapport de monitoring
```python
@shared_task(name='administration.daily_monitoring_report')
# Fréquence     : Quotidienne à 6h00
# Description   : Agrège les métriques des 24h écoulées (disponibilité, erreurs, performance)
#                 Stocke un snapshot JSON dans SystemLog (type=MONITORING_SNAPSHOT)
# Utilité       : Permet de tracer l'évolution des performances dans le temps
```

### TASK-015-06 : Vérification des paramètres de sécurité AUTH
```python
@shared_task(name='administration.check_auth_security_settings')
# Fréquence     : Hebdomadaire (lundi 7h00)
# Description   : Vérifie que les paramètres d'authentification respectent
#                 les recommandations ANSSI (longueur mdp, session timeout, etc.)
# Alerte        : Si un paramètre est en dessous des recommandations → WARNING pour Super Admin
```

---

## 14. ÉVÉNEMENTS JOURNALISÉS (MODULE 11)

| Code événement | Description | Sévérité | Données clés |
|----------------|-------------|----------|--------------|
| `CONFIG_PARAMETER_CHANGED` | Paramètre système modifié | WARNING | key, old_value (masqué si sensitive), new_value, changed_by_id |
| `CONFIG_PARAMETER_RESET` | Paramètre réinitialisé à la valeur par défaut | WARNING | key, old_value, default_value, reset_by_id |
| `CONFIG_ROLLBACK_PERFORMED` | Rollback de configuration effectué | WARNING | key, restored_value, rollback_by_id, original_change_id |
| `CONFIG_EXPORT_PERFORMED` | Export de configuration effectué | INFO | categories_exported, exported_by_id |
| `CONFIG_IMPORT_PERFORMED` | Import de configuration effectué | WARNING | parameters_count, changed_count, imported_by_id |
| `SMTP_TEST_PERFORMED` | Test SMTP effectué | INFO | destination_email, success, performed_by_id |
| `EMAIL_TEMPLATE_MODIFIED` | Template email modifié | INFO | template_slug, modified_by_id |
| `HOLIDAY_CREATED` | Jour férié ajouté | INFO | holiday_name, date, created_by_id |
| `HOLIDAY_DELETED` | Jour férié supprimé | INFO | holiday_name, date, deleted_by_id |
| `MAINTENANCE_SCHEDULED` | Maintenance planifiée | INFO | scheduled_start, estimated_duration, created_by_id |
| `MAINTENANCE_ACTIVATED` | Mode maintenance activé | ALERT | activated_by_id, whitelist_ips_count |
| `MAINTENANCE_DEACTIVATED` | Mode maintenance désactivé | INFO | deactivated_by_id, actual_duration_minutes |
| `DATABASE_VACUUM_TRIGGERED` | VACUUM déclenché manuellement | INFO | triggered_by_id |
| `DATABASE_REINDEX_TRIGGERED` | REINDEX déclenché manuellement | INFO | triggered_by_id |
| `HEALTH_CHECK_CRITICAL` | Health check : composant CRITICAL | CRITICAL | component_name, error_details |
| `HEALTH_CHECK_WARNING` | Health check : composant WARNING | WARNING | component_name, warning_details |
| `SECURITY_SETTINGS_WARNING` | Paramètres AUTH non conformes ANSSI | WARNING | non_compliant_params |

---

## 15. NOTIFICATIONS GÉNÉRÉES (MODULE 12)

| Déclencheur | Destinataire(s) | Canal | Priorité | Template |
|-------------|----------------|-------|----------|----------|
| Composant système en CRITICAL | Super Admin + Admin | In-app + Email | CRITIQUE | `system_component_critical` |
| Composant système en WARNING | Super Admin + Admin | In-app | HAUTE | `system_component_warning` |
| Mode maintenance activé (délai de grâce) | Tous utilisateurs actifs | In-app | CRITIQUE | `maintenance_starting_soon` |
| Mode maintenance désactivé | Super Admin + Admin | In-app | NORMALE | `maintenance_completed` |
| Stockage critique (> seuil critical) | Super Admin + Admin | In-app + Email | CRITIQUE | `storage_critical_threshold` |
| Stockage warning (> seuil warning) | Super Admin + Admin | In-app + Email | HAUTE | `storage_warning_threshold` |
| Export configuration effectué | Super Admin | In-app | NORMALE | `config_export_performed` |
| Import configuration effectué | Super Admin | In-app + Email | HAUTE | `config_import_performed` |
| Paramètres AUTH non conformes ANSSI | Super Admin | In-app + Email | HAUTE | `auth_security_non_compliant` |

---

## 16. INITIALISATION DES PARAMÈTRES (MIGRATION DJANGO)

Les paramètres système sont initialisés via une migration Django de données (data migration), garantissant que le système est opérationnel dès le premier déploiement sans configuration manuelle.

```python
# apps/administration/migrations/0002_initial_settings.py

def create_initial_settings(apps, schema_editor):
    SystemSetting = apps.get_model('administration', 'SystemSetting')

    settings_data = [
        # Général
        {
            'key': 'general.organization_name',
            'category': 'GENERAL',
            'label': 'Nom de l\'organisation',
            'value': 'Administration',
            'default_value': 'Administration',
            'value_type': 'STRING',
            'display_order': 1,
        },
        {
            'key': 'general.timezone',
            'category': 'GENERAL',
            'label': 'Fuseau horaire',
            'value': 'Europe/Paris',
            'default_value': 'Europe/Paris',
            'value_type': 'STRING',
            'allowed_values': ['Europe/Paris', 'Europe/London', 'UTC'],
            'display_order': 2,
        },
        # Authentification
        {
            'key': 'auth.session_timeout_minutes',
            'category': 'AUTH',
            'label': 'Durée de session inactive (minutes)',
            'value': 30,
            'default_value': 30,
            'value_type': 'INTEGER',
            'min_value': 5,
            'max_value': 480,
            'requires_super_admin': True,
            'display_order': 1,
        },
        {
            'key': 'auth.totp_required',
            'category': 'AUTH',
            'label': 'TOTP obligatoire pour tous',
            'value': False,
            'default_value': False,
            'value_type': 'BOOLEAN',
            'requires_super_admin': True,
            'requires_totp': True,
            'display_order': 2,
        },
        # SMTP
        {
            'key': 'smtp.host',
            'category': 'SMTP',
            'label': 'Hôte SMTP',
            'value': 'localhost',
            'default_value': 'localhost',
            'value_type': 'STRING',
            'display_order': 1,
        },
        {
            'key': 'smtp.password',
            'category': 'SMTP',
            'label': 'Mot de passe SMTP',
            'value': '',
            'default_value': '',
            'value_type': 'SECRET',
            'is_sensitive': True,
            'display_order': 5,
        },
        # ... (tous les paramètres listés en section 5)
    ]

    for setting_data in settings_data:
        SystemSetting.objects.get_or_create(
            key=setting_data['key'],
            defaults=setting_data
        )
```

---

## 17. STRUCTURE DES FICHIERS

```
apps/
└── administration/
    ├── __init__.py
    ├── admin.py
    ├── apps.py
    ├── models/
    │   ├── __init__.py
    │   ├── system_setting.py
    │   ├── config_change_log.py
    │   ├── email_template.py
    │   ├── holiday.py
    │   └── maintenance_window.py
    ├── serializers/
    │   ├── __init__.py
    │   ├── setting_serializers.py
    │   ├── template_serializers.py
    │   ├── holiday_serializers.py
    │   └── maintenance_serializers.py
    ├── views/
    │   ├── __init__.py
    │   ├── settings_views.py
    │   ├── smtp_views.py
    │   ├── template_views.py
    │   ├── holiday_views.py
    │   ├── maintenance_views.py
    │   ├── health_views.py
    │   ├── monitoring_views.py
    │   └── config_io_views.py          # Import/Export
    ├── urls.py
    ├── permissions.py
    ├── tasks.py
    ├── signals.py
    ├── services/
    │   ├── __init__.py
    │   ├── monitoring_service.py
    │   ├── working_days_service.py
    │   ├── config_export_service.py
    │   └── config_import_service.py
    ├── migrations/
    │   ├── 0001_initial.py
    │   └── 0002_initial_settings.py    # Data migration : paramètres par défaut
    └── tests/
        ├── __init__.py
        ├── test_settings.py
        ├── test_health_check.py
        ├── test_working_days.py
        ├── test_maintenance.py
        └── test_config_io.py

docs/
└── administration/
    └── MODULE_15_Administration_Configuration.md
```

---

## ✅ RÉCAPITULATIF MODULE 15

| Critère | Valeur |
|---------|--------|
| Cas d'utilisation | 25 UC |
| Modèles de données | 5 tables |
| Paramètres système prédéfinis | 35+ paramètres (7 catégories) |
| Endpoints API | 38 endpoints |
| Tâches Celery planifiées | 6 tâches |
| Événements journalisés (MODULE 11) | 17 événements |
| Notifications générées (MODULE 12) | 9 types |
| Services techniques | 4 services |

---

**Prochain module :** MODULE 16 — Sauvegarde, Restauration & Résilience
Ce module implémente la stratégie de sauvegarde 3-2-1 (3 copies, 2 supports, 1 hors site), les procédures de restauration à différents niveaux de granularité (document unique, base complète, point-in-time), le chiffrement GPG des archives, et les tests automatiques mensuels de restauration.
