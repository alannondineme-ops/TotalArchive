# MODULE 14 — Tableaux de bord & Reporting

**Système d'Archivage Numérique — Administration Publique**  
**Version :** 1.0  
**Statut :** Spécification technique complète  
**Dépendances :** MODULE 02 (Auth), MODULE 04 (Cycle de vie), MODULE 05 (Confidentialité), MODULE 06 (Documents), MODULE 08 (Workflow), MODULE 11 (Audit), MODULE 12 (Notifications), MODULE 13 (Partage)

---

## 1. PRÉSENTATION DU MODULE

### 1.1 Contexte et justification

Un système d'archivage numérique pour administration publique génère un volume considérable de données opérationnelles. Sans outillage de pilotage adapté, ces données restent inexploitées : les responsables ne savent pas combien de documents arrivent, les archivistes ne voient pas les goulots d'étranglement dans les workflows, et les DSI ne peuvent pas anticiper la saturation du stockage.

Ce module répond à trois besoins distincts mais complémentaires :

1. **Pilotage opérationnel :** Donner aux responsables une vision en temps quasi-réel de l'activité du système (documents entrants, workflows en cours, alertes actives).
2. **Conformité réglementaire :** Produire automatiquement les rapports exigés par les textes (registre RGPD, bordereau d'élimination, rapport d'archivage pour autorités de contrôle).
3. **Anticipation et planification :** Fournir des projections (croissance du stockage, charge sur les archivistes) permettant d'anticiper les besoins en ressources.

### 1.2 Philosophie du module

Les dashboards ne sont pas de simples affichages de chiffres. Chaque widget doit répondre à une question métier précise. Le principe directeur est : **"chaque indicateur déclenche une action possible"**. Un widget "Documents en retard de validation" doit permettre, en un clic, d'accéder à la liste des documents concernés et d'escalader.

Les rapports, quant à eux, sont des documents formels. Ils sont générés en arrière-plan (Celery), stockés temporairement, et envoyés par email ou disponibles en téléchargement. Ils ne sont jamais générés en temps réel lors de la requête HTTP (risque de timeout).

### 1.3 Périmètre fonctionnel

- Dashboards configurables avec widgets personnalisables par rôle
- 6 dashboards thématiques prédéfinis (activité, sécurité, workflow, cycle de vie, utilisateurs, stockage)
- Rapports exportables en PDF, Excel et CSV
- Planification automatique des rapports (Celery Beat)
- Comparaisons temporelles (mois vs mois, année vs année)
- Drill-down : cliquer sur un chiffre pour voir le détail
- Envoi automatique par email selon planification

---

## 2. CAS D'UTILISATION — VUE D'ENSEMBLE

| Code | Cas d'utilisation | Acteur principal | Priorité |
|------|-------------------|-----------------|----------|
| UC-014-01 | Consulter le dashboard principal d'activité | Tout utilisateur authentifié | HAUTE |
| UC-014-02 | Consulter le dashboard de sécurité | Admin, Super Admin, Auditeur | HAUTE |
| UC-014-03 | Consulter le dashboard des workflows | Archiviste, Responsable, Admin | HAUTE |
| UC-014-04 | Consulter le dashboard du cycle de vie | Archiviste, Responsable, Admin | HAUTE |
| UC-014-05 | Consulter le dashboard des utilisateurs | Admin, Super Admin | HAUTE |
| UC-014-06 | Consulter le dashboard de stockage | Admin, Super Admin | HAUTE |
| UC-014-07 | Créer un dashboard personnalisé | Responsable, Admin | MOYENNE |
| UC-014-08 | Ajouter/retirer des widgets d'un dashboard | Responsable, Admin | MOYENNE |
| UC-014-09 | Configurer un widget (période, filtres) | Responsable, Admin | MOYENNE |
| UC-014-10 | Partager un dashboard avec d'autres utilisateurs | Responsable, Admin | BASSE |
| UC-014-11 | Effectuer un drill-down sur un indicateur | Tout utilisateur autorisé | HAUTE |
| UC-014-12 | Comparer deux périodes sur un indicateur | Responsable, Admin | MOYENNE |
| UC-014-13 | Générer un rapport d'activité mensuel | Admin, Responsable | HAUTE |
| UC-014-14 | Générer un rapport de conformité RGPD | Admin, Super Admin | HAUTE |
| UC-014-15 | Générer un rapport d'audit pour autorité de contrôle | Admin, Super Admin, Auditeur | HAUTE |
| UC-014-16 | Générer un rapport d'éliminations | Archiviste, Admin | HAUTE |
| UC-014-17 | Générer un rapport de versements aux archives nationales | Archiviste, Admin | HAUTE |
| UC-014-18 | Créer un rapport personnalisé avec filtres | Responsable, Admin | MOYENNE |
| UC-014-19 | Planifier l'envoi automatique d'un rapport | Admin | HAUTE |
| UC-014-20 | Consulter l'historique des rapports générés | Admin, Auditeur | HAUTE |
| UC-014-21 | Télécharger un rapport précédemment généré | Tout utilisateur autorisé | HAUTE |
| UC-014-22 | Exporter les données d'un widget en CSV | Responsable, Admin | MOYENNE |
| UC-014-23 | Configurer les seuils d'alerte des widgets | Admin | MOYENNE |
| UC-014-24 | Consulter les projections de stockage | Admin, Super Admin | HAUTE |
| UC-014-25 | Visualiser la carte des accès par service | Admin, Super Admin | BASSE |
| UC-014-26 | Configurer le fuseau horaire des rapports | Admin | BASSE |

---

## 3. CAS D'UTILISATION DÉTAILLÉS

### UC-014-01 — Consulter le dashboard principal d'activité

**Acteur principal :** Tout utilisateur authentifié (contenu filtré selon le rôle)
**Pré-conditions :** Utilisateur authentifié avec session JWT valide

**Flux principal :**
1. L'utilisateur accède à l'interface et atterrit sur le dashboard par défaut (ou son dashboard personnalisé si configuré)
2. Le frontend émet une requête `GET /api/v1/dashboards/activity/`
3. Le backend agrège les métriques depuis les vues matérialisées et le cache Redis
4. La réponse est retournée en < 500ms (SLA obligatoire pour les dashboards)
5. Les widgets s'affichent avec leurs données

**Widgets présents sur le dashboard activité :**
- Documents entrants (aujourd'hui / 7 jours / 30 jours) avec courbe de tendance
- Documents par statut (actif, en validation, archivé, en retard) — donut chart
- Volume d'activité par heure (heatmap sur les 7 derniers jours)
- Top 5 des utilisateurs les plus actifs (documents créés/modifiés)
- Top 5 des catégories documentaires les plus alimentées
- Alertes actives (documents en retard, accès suspects, sauvegardes en échec)

**Filtrage selon le rôle :**
- Agent : voit uniquement les données relatives à ses propres documents
- Archiviste : voit les données de son service
- Responsable : voit les données de son service
- Admin / Super Admin : voit l'ensemble du système

**Post-conditions :** Dashboard affiché, données mises en cache Redis (TTL 5 minutes)

---

### UC-014-13 — Générer un rapport d'activité mensuel

**Acteur principal :** Admin, Responsable
**Pré-conditions :** Utilisateur authentifié avec les droits de génération de rapports

**Flux principal :**
1. L'utilisateur accède à "Rapports > Nouveau rapport > Rapport d'activité"
2. Il configure les paramètres :
   - Période : mois cible (sélecteur mois/année)
   - Périmètre : tout le système / un service / une catégorie documentaire
   - Format de sortie : PDF, Excel, ou les deux
   - Inclure les graphiques : oui/non (PDF uniquement)
3. Il clique sur "Générer"
4. Le système crée un enregistrement `Report` avec statut `PENDING`
5. Une tâche Celery est mise en file d'attente (`generate_monthly_activity_report`)
6. L'utilisateur reçoit un message "Le rapport est en cours de génération, vous serez notifié."
7. En arrière-plan, Celery génère le rapport :
   - Collecte des métriques pour la période
   - Génération du PDF via WeasyPrint ou ReportLab
   - Génération du Excel via openpyxl
   - Sauvegarde des fichiers générés (stockage temporaire, rétention 30 jours)
   - Mise à jour du `Report` avec statut `READY` et les chemins des fichiers
8. L'utilisateur reçoit une notification (MODULE 12) avec lien de téléchargement
9. L'utilisateur télécharge le rapport

**Flux alternatifs :**
- **7a.** Erreur lors de la génération → `Report.status = 'FAILED'`, utilisateur notifié, possibilité de relancer
- **9a.** Lien de téléchargement expiré (> 30 jours) → message d'erreur invitant à regénérer

**Post-conditions :**
- `Report` créé et stocké temporairement
- Utilisateur notifié et fichier disponible au téléchargement
- Événement `REPORT_GENERATED` journalisé (MODULE 11)

---

### UC-014-14 — Générer un rapport de conformité RGPD

**Acteur principal :** Admin, Super Admin
**Pré-conditions :** Utilisateur authentifié avec droits Admin minimum

**Flux principal :**
1. L'utilisateur accède à "Rapports > Conformité > Rapport RGPD"
2. Il configure :
   - Période de référence (par défaut : année civile en cours)
   - Type de rapport : complet (tous les éléments RGPD) ou partiel (thème spécifique)
3. Génération en arrière-plan via Celery

**Contenu du rapport RGPD généré :**
- **Registre des traitements :** Liste de tous les documents contenant des données personnelles (flag RGPD dans MODULE 05), catégorie de données, durée de conservation prévue vs réelle
- **Accès aux données personnelles :** Qui a accédé à quels documents avec données personnelles sur la période
- **Partages de données personnelles :** Documents avec données perso partagés (MODULE 13), avec qui, pour quelle durée
- **Suppressions et anonymisations :** Documents supprimés ou anonymisés sur la période (conformité droit à l'oubli)
- **Incidents de sécurité :** Événements de sécurité liés aux données personnelles (accès non autorisé, tentatives)
- **État de conformité :** Synthèse avec indicateurs verts/oranges/rouges (documents perso sans durée de conservation définie → rouge)

**Post-conditions :**
- Rapport RGPD généré, stocké 1 an (rétention légale)
- Événement `RGPD_REPORT_GENERATED` journalisé (MODULE 11)

---

### UC-014-19 — Planifier l'envoi automatique d'un rapport

**Acteur principal :** Admin
**Pré-conditions :** Utilisateur authentifié avec droits Admin

**Flux principal :**
1. L'utilisateur accède à "Rapports > Planification > Nouveau calendrier"
2. Il configure :
   - Type de rapport à planifier
   - Fréquence : quotidien, hebdomadaire, mensuel
   - Jour et heure d'envoi (selon fréquence)
   - Destinataires : liste d'emails (internes et/ou externes)
   - Format : PDF, Excel, CSV
   - Paramètres du rapport (périmètre, filtres)
3. Il valide
4. Le système crée un `ReportSchedule` actif
5. Celery Beat exécute automatiquement la génération et l'envoi à chaque échéance

**Post-conditions :**
- `ReportSchedule` créé
- Prochain déclenchement planifié

---

### UC-014-24 — Consulter les projections de stockage

**Acteur principal :** Admin, Super Admin
**Pré-conditions :** Utilisateur authentifié Admin ou Super Admin, au moins 30 jours de données historiques

**Flux principal :**
1. L'utilisateur accède à "Dashboard Stockage > Projections"
2. Le système analyse les données de croissance des 90 derniers jours (volume total, nombre de fichiers, croissance par catégorie)
3. Un modèle de régression linéaire simple calcule la projection sur 6, 12 et 24 mois
4. Le widget affiche :
   - Stockage actuel (utilisé / total disponible)
   - Projection à 6 mois (avec intervalle de confiance)
   - Projection à 12 mois
   - Date estimée de saturation (si tendance actuelle maintenue)
   - Recommandation : alerte orange si saturation < 12 mois, rouge si < 6 mois
5. Un graphique linéaire historique + projection est affiché

**Post-conditions :** Données de projection affichées, mises en cache Redis (TTL 1 heure)

---

## 4. MODÈLES DE DONNÉES

### 4.1 Dashboard — Tableaux de bord

```python
class Dashboard(BaseModel):
    """
    Tableau de bord : ensemble de widgets organisés dans une grille.
    Les dashboards système sont prédéfinis et non supprimables.
    Les dashboards utilisateurs sont entièrement personnalisables.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    name = models.CharField(
        max_length=200,
        help_text="Nom affiché dans l'interface (ex: 'Mon dashboard opérationnel')"
    )
    description = models.TextField(blank=True)

    # Type : système (prédéfini non modifiable) ou utilisateur (personnalisable)
    dashboard_type = models.CharField(
        max_length=20,
        choices=[
            ('SYSTEM', 'Dashboard système (prédéfini)'),
            ('USER', 'Dashboard utilisateur (personnalisé)'),
        ],
        default='USER'
    )

    # Dashboard système identifié par un slug fixe
    system_slug = models.CharField(
        max_length=50,
        unique=True,
        null=True,
        blank=True,
        help_text="Identifiant fixe pour les dashboards système : activity, security, workflow, lifecycle, users, storage"
    )

    # Propriétaire (null pour les dashboards système)
    owner = models.ForeignKey(
        'users.User',
        on_delete=models.CASCADE,
        null=True, blank=True,
        related_name='owned_dashboards'
    )

    # Partage avec d'autres utilisateurs
    shared_with = models.ManyToManyField(
        'users.User',
        blank=True,
        related_name='shared_dashboards',
        help_text="Utilisateurs pouvant voir ce dashboard en lecture seule"
    )

    # Rôles minimum requis pour voir ce dashboard
    required_roles = models.JSONField(
        default=list,
        help_text="Liste de rôles autorisés, ex: ['ADMIN', 'SUPER_ADMIN']"
    )

    # Configuration de la grille (layout)
    grid_columns = models.PositiveSmallIntegerField(
        default=12,
        help_text="Nombre de colonnes de la grille (standard 12 colonnes CSS)"
    )

    # Dashboard par défaut de l'utilisateur
    is_default = models.BooleanField(
        default=False,
        help_text="Dashboard affiché à la connexion"
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
        related_name='deleted_dashboards'
    )

    class Meta:
        db_table = 'dashboards'
        indexes = [
            models.Index(fields=['owner', 'is_default']),
            models.Index(fields=['system_slug']),
            models.Index(fields=['dashboard_type']),
        ]
        constraints = [
            models.UniqueConstraint(
                fields=['owner', 'is_default'],
                condition=models.Q(is_default=True, owner__isnull=False),
                name='unique_default_dashboard_per_user'
            ),
        ]
```

---

### 4.2 Widget — Composants de dashboard

```python
class Widget(BaseModel):
    """
    Widget individuel affiché dans un dashboard.
    Chaque widget est configuré pour requêter un endpoint de données spécifique.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    dashboard = models.ForeignKey(
        Dashboard,
        on_delete=models.CASCADE,
        related_name='widgets'
    )

    name = models.CharField(
        max_length=200,
        help_text="Titre affiché sur le widget"
    )

    # Type de visualisation
    widget_type = models.CharField(
        max_length=30,
        choices=[
            ('KPI_CARD', 'Carte KPI (chiffre unique + tendance)'),
            ('LINE_CHART', 'Graphique linéaire'),
            ('BAR_CHART', 'Graphique barres'),
            ('DONUT_CHART', 'Graphique donut/camembert'),
            ('HEATMAP', 'Carte de chaleur'),
            ('DATA_TABLE', 'Tableau de données'),
            ('ALERT_LIST', 'Liste d\'alertes'),
            ('PROGRESS_BAR', 'Barre de progression'),
            ('GAUGE', 'Jauge (stockage, quota)'),
            ('MAP_CHART', 'Carte géographique'),
        ]
    )

    # Source de données
    data_source = models.CharField(
        max_length=100,
        help_text="Identifiant de la source de données : ex: 'documents.by_status', 'audit.events_by_severity'"
    )

    # Configuration du widget (filtrages, période, etc.)
    config = models.JSONField(
        default=dict,
        help_text="Configuration JSON : période, filtres, couleurs, seuils d'alerte"
    )

    # Position dans la grille
    grid_x = models.PositiveSmallIntegerField(default=0, help_text="Colonne de départ (0-11)")
    grid_y = models.PositiveSmallIntegerField(default=0, help_text="Ligne de départ")
    grid_width = models.PositiveSmallIntegerField(default=4, help_text="Largeur en colonnes (1-12)")
    grid_height = models.PositiveSmallIntegerField(default=3, help_text="Hauteur en unités de grille")

    # Rafraîchissement automatique
    refresh_interval_seconds = models.PositiveIntegerField(
        default=300,
        help_text="Fréquence de rafraîchissement en secondes (min 60, défaut 300)"
    )

    # Seuils d'alerte visuels
    alert_threshold_warning = models.FloatField(
        null=True, blank=True,
        help_text="Seuil orange (ex: 80% stockage utilisé)"
    )
    alert_threshold_critical = models.FloatField(
        null=True, blank=True,
        help_text="Seuil rouge (ex: 95% stockage utilisé)"
    )

    # Drill-down : URL cible pour le clic sur le widget
    drill_down_url = models.CharField(
        max_length=500,
        blank=True,
        help_text="URL relative vers laquelle naviguer au clic, ex: /documents/?status=OVERDUE"
    )

    # Ordre d'affichage
    order = models.PositiveSmallIntegerField(default=0)

    # Horodatage
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='deleted_widgets'
    )

    class Meta:
        db_table = 'widgets'
        ordering = ['dashboard', 'order']
        indexes = [
            models.Index(fields=['dashboard', 'order']),
            models.Index(fields=['data_source']),
        ]
        constraints = [
            models.CheckConstraint(
                check=models.Q(refresh_interval_seconds__gte=60),
                name='widget_min_refresh_interval'
            ),
        ]
```

---

### 4.3 Report — Rapports générés

```python
class Report(BaseModel):
    """
    Instance d'un rapport généré (ou en cours de génération).
    La génération est toujours asynchrone via Celery.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    # Type de rapport
    report_type = models.CharField(
        max_length=50,
        choices=[
            ('MONTHLY_ACTIVITY', 'Rapport d\'activité mensuel'),
            ('RGPD_COMPLIANCE', 'Rapport de conformité RGPD'),
            ('AUDIT_AUTHORITY', 'Rapport d\'audit pour autorité de contrôle'),
            ('ELIMINATION', 'Rapport d\'éliminations'),
            ('TRANSFER', 'Rapport de versements aux archives'),
            ('SECURITY', 'Rapport de sécurité'),
            ('WORKFLOW', 'Rapport de workflows'),
            ('STORAGE', 'Rapport de stockage'),
            ('USER_ACTIVITY', 'Rapport d\'activité utilisateurs'),
            ('CUSTOM', 'Rapport personnalisé'),
        ],
        db_index=True
    )

    name = models.CharField(
        max_length=300,
        help_text="Nom descriptif généré automatiquement, ex: 'Rapport activité - Janvier 2026'"
    )

    # Demandeur
    requested_by = models.ForeignKey(
        'users.User',
        on_delete=models.PROTECT,
        related_name='requested_reports',
        null=True, blank=True,
        help_text="Null si généré automatiquement par planification"
    )
    report_schedule = models.ForeignKey(
        'ReportSchedule',
        on_delete=models.SET_NULL,
        null=True, blank=True,
        related_name='generated_reports',
        help_text="Lien vers la planification si génération automatique"
    )

    # Paramètres de génération (stockés pour pouvoir régénérer à l'identique)
    parameters = models.JSONField(
        default=dict,
        help_text="Tous les paramètres utilisés : période, périmètre, filtres, format"
    )

    # Statut de génération
    status = models.CharField(
        max_length=20,
        choices=[
            ('PENDING', 'En attente de génération'),
            ('PROCESSING', 'En cours de génération'),
            ('READY', 'Disponible au téléchargement'),
            ('FAILED', 'Échec de génération'),
            ('EXPIRED', 'Expiré (fichier supprimé)'),
        ],
        default='PENDING',
        db_index=True
    )

    # Tâche Celery associée (pour suivi et annulation éventuelle)
    celery_task_id = models.CharField(
        max_length=255,
        blank=True,
        help_text="ID de la tâche Celery pour suivi"
    )

    # Fichiers générés
    file_pdf_path = models.CharField(
        max_length=500, blank=True,
        help_text="Chemin relatif du fichier PDF généré"
    )
    file_excel_path = models.CharField(
        max_length=500, blank=True,
        help_text="Chemin relatif du fichier Excel généré"
    )
    file_csv_path = models.CharField(
        max_length=500, blank=True,
        help_text="Chemin relatif du fichier CSV généré"
    )

    # Métadonnées du rapport généré
    row_count = models.PositiveIntegerField(
        null=True, blank=True,
        help_text="Nombre de lignes/enregistrements dans le rapport"
    )
    generation_duration_seconds = models.FloatField(
        null=True, blank=True,
        help_text="Durée de génération en secondes"
    )
    error_message = models.TextField(
        blank=True,
        help_text="Message d'erreur en cas d'échec"
    )

    # Expiration des fichiers (nettoyage automatique)
    expires_at = models.DateTimeField(
        help_text="Date de suppression automatique des fichiers (défaut +30 jours, 1 an pour RGPD)"
    )

    # Statistiques d'accès
    download_count = models.PositiveIntegerField(default=0)
    last_downloaded_at = models.DateTimeField(null=True, blank=True)

    # Horodatage
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    completed_at = models.DateTimeField(null=True, blank=True)

    # Soft delete
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        'users.User', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='deleted_reports'
    )

    class Meta:
        db_table = 'reports'
        indexes = [
            models.Index(fields=['report_type', 'status']),
            models.Index(fields=['requested_by', 'created_at']),
            models.Index(fields=['status', 'expires_at']),
            models.Index(fields=['report_schedule']),
        ]
```

---

### 4.4 ReportSchedule — Planification automatique des rapports

```python
class ReportSchedule(BaseModel):
    """
    Planification d'un rapport récurrent.
    Celery Beat déclenche la génération selon la fréquence configurée.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)

    name = models.CharField(
        max_length=200,
        help_text="Nom de la planification (ex: 'Rapport mensuel DSI')"
    )

    report_type = models.CharField(
        max_length=50,
        choices=[
            ('MONTHLY_ACTIVITY', 'Rapport d\'activité mensuel'),
            ('RGPD_COMPLIANCE', 'Rapport de conformité RGPD'),
            ('AUDIT_AUTHORITY', 'Rapport d\'audit'),
            ('ELIMINATION', 'Rapport d\'éliminations'),
            ('TRANSFER', 'Rapport de versements'),
            ('SECURITY', 'Rapport de sécurité'),
            ('WORKFLOW', 'Rapport de workflows'),
            ('STORAGE', 'Rapport de stockage'),
            ('USER_ACTIVITY', 'Rapport d\'activité utilisateurs'),
            ('CUSTOM', 'Rapport personnalisé'),
        ]
    )

    # Paramètres du rapport (identiques au Report.parameters)
    parameters = models.JSONField(default=dict)

    # Fréquence
    frequency = models.CharField(
        max_length=20,
        choices=[
            ('DAILY', 'Quotidien'),
            ('WEEKLY', 'Hebdomadaire'),
            ('MONTHLY', 'Mensuel'),
            ('QUARTERLY', 'Trimestriel'),
            ('YEARLY', 'Annuel'),
        ]
    )
    scheduled_time = models.TimeField(
        help_text="Heure d'exécution (ex: 07:00)"
    )
    scheduled_day_of_week = models.PositiveSmallIntegerField(
        null=True, blank=True,
        help_text="Jour de la semaine pour WEEKLY (0=lundi, 6=dimanche)"
    )
    scheduled_day_of_month = models.PositiveSmallIntegerField(
        null=True, blank=True,
        help_text="Jour du mois pour MONTHLY (1-28, éviter 29-31 pour compatibilité)"
    )

    # Formats générés
    generate_pdf = models.BooleanField(default=True)
    generate_excel = models.BooleanField(default=False)
    generate_csv = models.BooleanField(default=False)

    # Destinataires de l'envoi automatique
    recipient_emails = models.JSONField(
        default=list,
        help_text="Liste d'emails pour l'envoi automatique"
    )
    recipient_users = models.ManyToManyField(
        'users.User',
        blank=True,
        related_name='report_subscriptions',
        help_text="Utilisateurs internes notifiés (in-app + email selon préférences)"
    )

    # Statut
    is_active = models.BooleanField(default=True)
    last_run_at = models.DateTimeField(null=True, blank=True)
    next_run_at = models.DateTimeField(
        null=True, blank=True,
        help_text="Calculé automatiquement après chaque exécution"
    )
    last_run_status = models.CharField(
        max_length=20,
        choices=[
            ('SUCCESS', 'Succès'),
            ('FAILED', 'Échec'),
        ],
        null=True, blank=True
    )

    # Créateur
    created_by = models.ForeignKey(
        'users.User',
        on_delete=models.PROTECT,
        related_name='created_report_schedules'
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
        related_name='deleted_report_schedules'
    )

    class Meta:
        db_table = 'report_schedules'
        indexes = [
            models.Index(fields=['is_active', 'next_run_at']),
            models.Index(fields=['frequency', 'is_active']),
        ]
        constraints = [
            models.CheckConstraint(
                check=models.Q(scheduled_day_of_week__lte=6),
                name='report_schedule_valid_day_of_week'
            ),
            models.CheckConstraint(
                check=models.Q(scheduled_day_of_month__gte=1) & models.Q(scheduled_day_of_month__lte=28),
                name='report_schedule_valid_day_of_month'
            ),
        ]
```

---

## 5. MATRICE DES PERMISSIONS

| Action | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|--------|:-----------:|:-----:|:----------:|:-----------:|:-----:|:--------:|
| Voir dashboard activité (périmètre global) | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Voir dashboard activité (périmètre service) | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Voir dashboard activité (ses données) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Voir dashboard sécurité | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Voir dashboard workflow | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Voir dashboard cycle de vie | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Voir dashboard utilisateurs | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Voir dashboard stockage | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Créer dashboard personnalisé | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| Partager un dashboard | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| Générer rapport d'activité | ✅ | ✅ | ❌ | ✅ (son service) | ❌ | ✅ |
| Générer rapport RGPD | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Générer rapport d'audit | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Générer rapport éliminations | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| Générer rapport versements | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| Créer rapport personnalisé | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| Planifier envoi automatique rapport | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Consulter historique des rapports | ✅ | ✅ | ❌ | ✅ (ses rapports) | ❌ | ✅ |
| Télécharger un rapport | ✅ | ✅ | ✅ (ses rapports) | ✅ (ses rapports) | ❌ | ✅ |
| Exporter données widget en CSV | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Configurer seuils d'alerte | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Consulter projections stockage | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |

---

## 6. SÉQUENCES DÉTAILLÉES

### Séquence 1 — Chargement d'un dashboard avec cache Redis

```
Utilisateur              Frontend              Backend (DRF)          Redis/PostgreSQL
     |                      |                       |                       |
     |-- Connexion -------->|                        |                       |
     |                      |-- GET /api/v1/dashboards/activity/ ---------->|
     |                      |                       |                       |
     |                      |                       |-- Vérifie rôle user   |
     |                      |                       |-- Clé cache : -------->|
     |                      |                       |   dashboard:activity:ADMIN:fr
     |                      |                       |                       |
     |                      |                       |<-- HIT cache (< 5ms) -|
     |                      |                       |   (si TTL non expiré) |
     |                      |<-- Réponse 200 (< 50ms) ----------------------|
     |<-- Dashboard affiché-|                       |                       |
     |                      |                       |                       |
     |                      |   [Si cache MISS]     |                       |
     |                      |                       |-- Requêtes agrégées -->|
     |                      |                       |   (vues matérialisées)|
     |                      |                       |<-- Données (< 400ms) -|
     |                      |                       |-- Stocke en cache ---->|
     |                      |                       |   (TTL 5 minutes)     |
     |                      |<-- Réponse 200 (< 500ms) --------------------|
     |<-- Dashboard affiché-|                       |                       |
```

---

### Séquence 2 — Génération asynchrone d'un rapport

```
Admin                    Backend                 Celery Worker          Stockage
  |                         |                        |                     |
  |-- POST /api/v1/reports/ -->|                     |                     |
  |   (type, période, format)|                       |                     |
  |                          |                       |                     |
  |                          |-- Valide paramètres   |                     |
  |                          |-- Crée Report (PENDING)|                    |
  |                          |-- Envoie tâche Celery ->|                   |
  |                          |                        |                    |
  |<-- 202 Accepted ---------|                        |                    |
  |   {report_id, status: PENDING}                    |                    |
  |                          |                        |                    |
  |  [Polling toutes les 3s] |                        |                    |
  |-- GET /api/v1/reports/{id}/ -->|                  |                    |
  |<-- {status: PROCESSING} --|                       |                    |
  |                          |                        |                    |
  |                          |     [Celery Worker]    |                    |
  |                          |                        |-- Collecte données ->|
  |                          |                        |-- Génère PDF ------->|
  |                          |                        |   (WeasyPrint/ReportLab)
  |                          |                        |-- Génère Excel ----->|
  |                          |                        |   (openpyxl)        |
  |                          |                        |-- Sauvegarde fichiers>|
  |                          |<-- Update Report ------|                     |
  |                          |   (status=READY,       |                     |
  |                          |    file_pdf_path,      |                     |
  |                          |    completed_at)       |                     |
  |                          |-- Notifie admin (MOD 12)                    |
  |                          |-- Journalise (MOD 11)  |                    |
  |                          |                        |                    |
  |  [Notification reçue]    |                        |                    |
  |-- GET /api/v1/reports/{id}/download/ -->|          |                   |
  |<-- Stream PDF watermarqué --|          |           |                   |
```

---

### Séquence 3 — Drill-down d'un widget

```
Responsable              Frontend                 Backend
     |                      |                         |
     |-- Clique sur widget ->|                         |
     |   "23 documents en    |                         |
     |    retard de valid."  |                         |
     |                       |                         |
     |                       |-- Navigation vers ------>|
     |                       |   /documents/?status=OVERDUE&service=RH
     |                       |                         |
     |                       |                         |-- Filtre documents
     |                       |                         |   (workflow en retard,
     |                       |                         |    service RH uniquement)
     |                       |<-- Liste paginée --------|
     |<-- Vue documents ----  |                         |
     |   avec actions possibles                         |
     |   (escalader, assigner, voir détails)            |
```

---

## 7. VUES MATÉRIALISÉES POSTGRESQL

Les dashboards reposent sur des vues matérialisées rafraîchies périodiquement pour éviter des requêtes lourdes en temps réel.

```sql
-- Vue matérialisée : statistiques documents quotidiennes
CREATE MATERIALIZED VIEW mv_document_stats_daily AS
SELECT
    DATE_TRUNC('day', created_at) AS day,
    COUNT(*) AS total_created,
    COUNT(*) FILTER (WHERE processing_status = 'PENDING') AS pending_count,
    COUNT(*) FILTER (WHERE processing_status = 'VALIDATED') AS validated_count,
    COUNT(*) FILTER (WHERE processing_status = 'REJECTED') AS rejected_count,
    COUNT(*) FILTER (WHERE is_deleted = false AND lifecycle_status = 'ACTIVE') AS active_count,
    SUM(file_size_bytes) AS total_size_bytes,
    AVG(file_size_bytes) AS avg_size_bytes
FROM documents
WHERE is_deleted = false
GROUP BY DATE_TRUNC('day', created_at)
WITH NO DATA;

CREATE UNIQUE INDEX ON mv_document_stats_daily (day);


-- Vue matérialisée : activité par utilisateur (30 derniers jours)
CREATE MATERIALIZED VIEW mv_user_activity_30d AS
SELECT
    user_id,
    COUNT(*) AS actions_count,
    COUNT(DISTINCT DATE_TRUNC('day', created_at)) AS active_days,
    MAX(created_at) AS last_action_at
FROM audit_logs
WHERE created_at >= NOW() - INTERVAL '30 days'
  AND is_deleted = false
GROUP BY user_id
WITH NO DATA;

CREATE UNIQUE INDEX ON mv_user_activity_30d (user_id);


-- Vue matérialisée : workflows en retard
CREATE MATERIALIZED VIEW mv_overdue_workflows AS
SELECT
    wi.id AS instance_id,
    wi.document_id,
    wi.current_step,
    wi.started_at,
    EXTRACT(EPOCH FROM (NOW() - wi.started_at)) / 86400 AS days_elapsed,
    wt.name AS template_name,
    wt.max_duration_days
FROM workflow_instances wi
JOIN workflow_templates wt ON wi.template_id = wt.id
WHERE wi.status = 'IN_PROGRESS'
  AND wi.is_deleted = false
  AND (NOW() - wi.started_at) > (wt.max_duration_days * INTERVAL '1 day')
WITH NO DATA;


-- Vue matérialisée : stockage par catégorie
CREATE MATERIALIZED VIEW mv_storage_by_category AS
SELECT
    tc.id AS category_id,
    tc.name AS category_name,
    COUNT(d.id) AS document_count,
    SUM(d.file_size_bytes) AS total_bytes,
    AVG(d.file_size_bytes) AS avg_bytes
FROM documents d
JOIN taxonomy_categories tc ON d.category_id = tc.id
WHERE d.is_deleted = false
GROUP BY tc.id, tc.name
WITH NO DATA;

CREATE UNIQUE INDEX ON mv_storage_by_category (category_id);
```

---

## 8. ENDPOINTS API

### 8.1 Dashboards

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/dashboards/` | Lister les dashboards accessibles | JWT |
| `POST` | `/api/v1/dashboards/` | Créer un dashboard personnalisé | JWT + Perm |
| `GET` | `/api/v1/dashboards/{id}/` | Détail et configuration d'un dashboard | JWT |
| `PATCH` | `/api/v1/dashboards/{id}/` | Modifier nom, description, layout | JWT + Owner |
| `DELETE` | `/api/v1/dashboards/{id}/` | Soft delete (impossible sur dashboards système) | JWT + Owner |
| `POST` | `/api/v1/dashboards/{id}/set-default/` | Définir comme dashboard par défaut | JWT + Owner |
| `POST` | `/api/v1/dashboards/{id}/share/` | Partager avec d'autres utilisateurs | JWT + Owner |

### 8.2 Dashboards système (données)

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/dashboards/activity/` | Données dashboard activité | JWT |
| `GET` | `/api/v1/dashboards/security/` | Données dashboard sécurité | JWT + Admin |
| `GET` | `/api/v1/dashboards/workflow/` | Données dashboard workflows | JWT + Perm |
| `GET` | `/api/v1/dashboards/lifecycle/` | Données dashboard cycle de vie | JWT + Perm |
| `GET` | `/api/v1/dashboards/users/` | Données dashboard utilisateurs | JWT + Admin |
| `GET` | `/api/v1/dashboards/storage/` | Données dashboard stockage | JWT + Admin |

### 8.3 Widgets

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/dashboards/{dashboard_id}/widgets/` | Lister les widgets d'un dashboard | JWT |
| `POST` | `/api/v1/dashboards/{dashboard_id}/widgets/` | Ajouter un widget | JWT + Owner |
| `PATCH` | `/api/v1/dashboards/{dashboard_id}/widgets/{id}/` | Modifier config, position, taille | JWT + Owner |
| `DELETE` | `/api/v1/dashboards/{dashboard_id}/widgets/{id}/` | Supprimer un widget | JWT + Owner |
| `POST` | `/api/v1/dashboards/{dashboard_id}/widgets/reorder/` | Réorganiser l'ordre des widgets | JWT + Owner |
| `GET` | `/api/v1/widgets/{id}/data/` | Obtenir les données d'un widget spécifique | JWT |
| `GET` | `/api/v1/widgets/{id}/export/` | Exporter les données du widget en CSV | JWT + Perm |

### 8.4 Sources de données widget

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/widget-data/documents/by-status/` | Documents groupés par statut | JWT |
| `GET` | `/api/v1/widget-data/documents/by-category/` | Documents groupés par catégorie | JWT |
| `GET` | `/api/v1/widget-data/documents/trend/` | Tendance de création sur période | JWT |
| `GET` | `/api/v1/widget-data/workflows/overdue/` | Workflows en retard | JWT + Perm |
| `GET` | `/api/v1/widget-data/workflows/completion-rate/` | Taux de validation | JWT + Perm |
| `GET` | `/api/v1/widget-data/audit/events-by-severity/` | Événements d'audit par sévérité | JWT + Perm |
| `GET` | `/api/v1/widget-data/storage/usage/` | Utilisation stockage actuelle | JWT + Admin |
| `GET` | `/api/v1/widget-data/storage/projections/` | Projections de croissance | JWT + Admin |
| `GET` | `/api/v1/widget-data/users/top-active/` | Top utilisateurs actifs | JWT + Admin |
| `GET` | `/api/v1/widget-data/lifecycle/expiring-soon/` | Documents dont la conservation expire | JWT + Perm |

### 8.5 Rapports

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/reports/` | Lister les rapports (filtrables par type, statut) | JWT + Perm |
| `POST` | `/api/v1/reports/` | Demander la génération d'un rapport | JWT + Perm |
| `GET` | `/api/v1/reports/{id}/` | Statut et détail d'un rapport | JWT + Perm |
| `GET` | `/api/v1/reports/{id}/download/` | Télécharger le rapport (PDF, Excel, CSV) | JWT + Perm |
| `POST` | `/api/v1/reports/{id}/retry/` | Relancer la génération (si statut FAILED) | JWT + Perm |
| `DELETE` | `/api/v1/reports/{id}/` | Supprimer un rapport (soft delete) | JWT + Admin |
| `GET` | `/api/v1/reports/templates/` | Types de rapports disponibles avec paramètres | JWT + Perm |

### 8.6 Planifications

| Méthode | URL | Description | Auth |
|---------|-----|-------------|------|
| `GET` | `/api/v1/report-schedules/` | Lister les planifications | JWT + Admin |
| `POST` | `/api/v1/report-schedules/` | Créer une planification | JWT + Admin |
| `GET` | `/api/v1/report-schedules/{id}/` | Détail d'une planification | JWT + Admin |
| `PATCH` | `/api/v1/report-schedules/{id}/` | Modifier paramètres ou fréquence | JWT + Admin |
| `POST` | `/api/v1/report-schedules/{id}/activate/` | Activer | JWT + Admin |
| `POST` | `/api/v1/report-schedules/{id}/deactivate/` | Désactiver | JWT + Admin |
| `DELETE` | `/api/v1/report-schedules/{id}/` | Supprimer (soft delete) | JWT + Admin |
| `POST` | `/api/v1/report-schedules/{id}/run-now/` | Déclencher immédiatement hors planification | JWT + Admin |

---

## 9. FORMAT DES RÉPONSES API

### Exemple : Dashboard activité (GET `/api/v1/dashboards/activity/`)

```json
{
  "success": true,
  "data": {
    "generated_at": "2026-02-18T10:30:00Z",
    "cache_ttl_seconds": 300,
    "period": "last_30_days",
    "scope": "SYSTEM",
    "kpis": {
      "documents_total": 14382,
      "documents_today": 47,
      "documents_this_week": 312,
      "documents_this_month": 1204,
      "documents_pending_validation": 83,
      "documents_overdue": 12,
      "active_workflows": 156,
      "active_users_today": 28
    },
    "trends": {
      "documents_by_day": [
        {"date": "2026-01-20", "count": 38},
        {"date": "2026-01-21", "count": 52}
      ],
      "documents_by_category": [
        {"category": "Courrier entrant", "count": 4821, "percentage": 33.5},
        {"category": "Délibérations", "count": 2104, "percentage": 14.6}
      ]
    },
    "alerts": {
      "critical": 2,
      "warning": 7,
      "info": 14
    }
  }
}
```

### Exemple : Demande de génération rapport (POST `/api/v1/reports/`)

**Requête :**
```json
{
  "report_type": "MONTHLY_ACTIVITY",
  "parameters": {
    "year": 2026,
    "month": 1,
    "scope": "SYSTEM",
    "formats": ["PDF", "EXCEL"],
    "include_charts": true,
    "language": "fr"
  }
}
```

**Réponse (202 Accepted) :**
```json
{
  "success": true,
  "data": {
    "id": "a1b2c3d4-...",
    "name": "Rapport d'activité - Janvier 2026",
    "report_type": "MONTHLY_ACTIVITY",
    "status": "PENDING",
    "celery_task_id": "task-uuid-here",
    "estimated_duration_seconds": 45,
    "status_url": "/api/v1/reports/a1b2c3d4-.../",
    "created_at": "2026-02-18T10:30:00Z"
  }
}
```

---

## 10. CONTENU DES RAPPORTS PRÉDÉFINIS

### 10.1 Rapport d'activité mensuel

**Sections :**
1. **Résumé exécutif** — KPIs clés du mois vs mois précédent (documents créés, temps moyen validation, taux de conformité)
2. **Activité documentaire** — Volume entrant par semaine, par catégorie, par service
3. **Performance des workflows** — Taux de validation, délais moyens, documents en retard
4. **Activité des utilisateurs** — Top 10 utilisateurs actifs, connexions, documents traités
5. **État du stockage** — Utilisation, croissance mensuelle, projection
6. **Incidents de sécurité** — Alertes du mois, anomalies détectées (sans détail sensible)
7. **Comparaison N-1** — Graphiques d'évolution sur 12 mois glissants

---

### 10.2 Rapport de conformité RGPD

**Sections :**
1. **Registre des traitements** — Tableau de tous les documents contenant des données personnelles, avec : catégorie de données, base légale, durée de conservation prévue, responsable de traitement
2. **Cartographie des accès** — Qui a accédé à des données personnelles sur la période, depuis quel endpoint/module
3. **Partages de données personnelles** — Documents avec données perso partagés via MODULE 13, avec identité des destinataires et dates
4. **Suppressions et exercice des droits** — Documents supprimés ou anonymisés suite à une demande de droit à l'oubli
5. **Durées de conservation** — Documents dont la durée légale est dépassée (non conformes → rouge), en cours (orange), conformes (vert)
6. **Incidents** — Violations potentielles de données personnelles détectées par le MODULE 11
7. **Plan d'action** — Liste des points de non-conformité avec recommandations priorisées

---

### 10.3 Rapport d'audit pour autorité de contrôle

**Sections :**
1. **Identification du système** — Nom, version, période auditée, périmètre
2. **Architecture de sécurité** — Résumé des mécanismes (authentification MFA, chiffrement, contrôle d'accès)
3. **Journal d'audit** — Extrait des événements significatifs sur la période (filtrés par sévérité WARNING/ALERT/CRITICAL)
4. **Gestion des accès** — Matrice des habilitations actives, modifications des droits sur la période
5. **Intégrité documentaire** — Résultats des vérifications de hash (MODULE 06), signatures numériques actives
6. **Sauvegardes** — Calendrier des sauvegardes, résultats des tests de restauration (MODULE 16)
7. **Incidents de sécurité** — Liste détaillée des alertes avec actions correctives prises
8. **Attestation** — Date, périmètre, responsable du système (signature numérique possible)

---

### 10.4 Rapport d'éliminations

**Sections :**
1. **Bordereau d'élimination** — Liste des documents éliminés sur la période, avec : référence, titre, dates, durée de conservation appliquée, motif d'élimination
2. **Conformité légale** — Vérification que toutes les durées minimales légales ont bien été respectées
3. **Autorisations** — Qui a approuvé chaque élimination, date de décision
4. **Certificats de destruction** — Référence aux certificats signés numériquement (MODULE 13 + MODULE 04)
5. **Statistiques** — Volume éliminé (nombre de documents, volumétrie en Go)

---

## 11. GÉNÉRATEUR DE RAPPORTS — ARCHITECTURE

### 11.1 Choix technique : WeasyPrint vs ReportLab

Le module utilise **WeasyPrint** comme générateur PDF principal pour les rapports avec mise en page riche (CSS/HTML → PDF). ReportLab est utilisé en fallback pour les rapports à génération très rapide sans mise en page complexe.

Pour Excel : **openpyxl** (données tabulaires avec styles et graphiques).
Pour CSV : bibliothèque standard Python `csv`.

### 11.2 Structure du service de génération

```python
# apps/reporting/services/report_generator.py

import csv
import io
from datetime import datetime
from django.template.loader import render_to_string
import weasyprint
import openpyxl
from openpyxl.styles import Font, PatternFill, Alignment
from openpyxl.chart import BarChart, Reference


class ReportGenerator:
    """
    Générateur de rapports multi-format.
    Toujours instancié et appelé depuis une tâche Celery, jamais depuis une vue HTTP.
    """

    def generate(self, report: 'Report') -> dict:
        """
        Point d'entrée unique de génération.
        Retourne un dictionnaire avec les chemins des fichiers générés.
        """
        params = report.parameters
        report_type = report.report_type
        formats = params.get('formats', ['PDF'])
        generated_files = {}

        # Collecter les données selon le type de rapport
        data = self._collect_data(report_type, params)

        if 'PDF' in formats:
            pdf_path = self._generate_pdf(report, data)
            generated_files['pdf'] = pdf_path

        if 'EXCEL' in formats:
            excel_path = self._generate_excel(report, data)
            generated_files['excel'] = excel_path

        if 'CSV' in formats:
            csv_path = self._generate_csv(report, data)
            generated_files['csv'] = csv_path

        return generated_files

    def _collect_data(self, report_type: str, params: dict) -> dict:
        """Délègue la collecte des données au collecteur adapté au type."""
        collectors = {
            'MONTHLY_ACTIVITY': MonthlyActivityCollector,
            'RGPD_COMPLIANCE': RgpdComplianceCollector,
            'AUDIT_AUTHORITY': AuditAuthorityCollector,
            'ELIMINATION': EliminationCollector,
            'TRANSFER': TransferCollector,
            'SECURITY': SecurityCollector,
            'WORKFLOW': WorkflowCollector,
            'STORAGE': StorageCollector,
            'USER_ACTIVITY': UserActivityCollector,
        }
        collector_class = collectors.get(report_type)
        if not collector_class:
            raise ValueError(f"Type de rapport inconnu : {report_type}")
        return collector_class(params).collect()

    def _generate_pdf(self, report: 'Report', data: dict) -> str:
        """Génère le PDF via WeasyPrint à partir d'un template HTML Django."""
        template_name = f"reports/{report.report_type.lower()}.html"
        html_content = render_to_string(template_name, {
            'report': report,
            'data': data,
            'generated_at': datetime.now(),
            'organization_name': SystemSetting.get('ORGANIZATION_NAME'),
        })
        pdf_bytes = weasyprint.HTML(string=html_content).write_pdf()
        path = self._save_file(report, pdf_bytes, 'pdf')
        return path

    def _generate_excel(self, report: 'Report', data: dict) -> str:
        """Génère un fichier Excel structuré avec openpyxl."""
        wb = openpyxl.Workbook()
        ws = wb.active
        ws.title = "Données"

        # En-tête avec style
        header_fill = PatternFill(start_color="1F3864", end_color="1F3864", fill_type="solid")
        header_font = Font(color="FFFFFF", bold=True)

        headers = data.get('excel_headers', [])
        for col, header in enumerate(headers, start=1):
            cell = ws.cell(row=1, column=col, value=header)
            cell.fill = header_fill
            cell.font = header_font
            cell.alignment = Alignment(horizontal='center')

        # Données
        for row_idx, row_data in enumerate(data.get('excel_rows', []), start=2):
            for col_idx, value in enumerate(row_data, start=1):
                ws.cell(row=row_idx, column=col_idx, value=value)

        # Auto-dimensionner les colonnes
        for col in ws.columns:
            max_length = max(len(str(cell.value or '')) for cell in col)
            ws.column_dimensions[col[0].column_letter].width = min(max_length + 2, 50)

        output = io.BytesIO()
        wb.save(output)
        path = self._save_file(report, output.getvalue(), 'xlsx')
        return path

    def _generate_csv(self, report: 'Report', data: dict) -> str:
        """Génère un CSV simple encodé UTF-8 avec BOM (compatibilité Excel FR)."""
        output = io.StringIO()
        writer = csv.writer(output, delimiter=';', quoting=csv.QUOTE_ALL)

        writer.writerow(data.get('excel_headers', []))
        for row in data.get('excel_rows', []):
            writer.writerow(row)

        csv_bytes = ('\ufeff' + output.getvalue()).encode('utf-8')
        path = self._save_file(report, csv_bytes, 'csv')
        return path

    def _save_file(self, report: 'Report', content: bytes, extension: str) -> str:
        """Sauvegarde un fichier généré dans le stockage temporaire des rapports."""
        filename = f"reports/{report.id}/{report.report_type.lower()}_{datetime.now().strftime('%Y%m%d_%H%M%S')}.{extension}"
        # Sauvegarde via le système de fichiers configuré (local ou NAS)
        with open(f"{settings.REPORTS_STORAGE_PATH}/{filename}", 'wb') as f:
            f.write(content)
        return filename
```

---

### 11.3 Collecteur de données exemple (Rapport mensuel)

```python
# apps/reporting/collectors/monthly_activity_collector.py

from django.db import connection
from django.utils import timezone
import calendar


class MonthlyActivityCollector:

    def __init__(self, params: dict):
        self.year = params.get('year', timezone.now().year)
        self.month = params.get('month', timezone.now().month)
        self.scope = params.get('scope', 'SYSTEM')
        self.service_id = params.get('service_id')

    def collect(self) -> dict:
        """Collecte toutes les données nécessaires au rapport mensuel."""
        start_date = timezone.datetime(self.year, self.month, 1, tzinfo=timezone.utc)
        _, last_day = calendar.monthrange(self.year, self.month)
        end_date = timezone.datetime(self.year, self.month, last_day, 23, 59, 59, tzinfo=timezone.utc)

        return {
            'period': {'start': start_date, 'end': end_date, 'label': f"{self.month:02d}/{self.year}"},
            'kpis': self._collect_kpis(start_date, end_date),
            'documents_by_day': self._collect_daily_trend(start_date, end_date),
            'documents_by_category': self._collect_by_category(start_date, end_date),
            'workflow_stats': self._collect_workflow_stats(start_date, end_date),
            'top_users': self._collect_top_users(start_date, end_date),
            'storage_stats': self._collect_storage_stats(),
            'comparison_previous_month': self._collect_comparison(start_date),
            'excel_headers': [
                'Date', 'Documents créés', 'Documents validés',
                'Documents rejetés', 'Workflows ouverts', 'Workflows clôturés'
            ],
            'excel_rows': self._collect_excel_rows(start_date, end_date),
        }

    def _collect_kpis(self, start_date, end_date) -> dict:
        with connection.cursor() as cursor:
            cursor.execute("""
                SELECT
                    COUNT(*) FILTER (WHERE created_at BETWEEN %s AND %s) AS created,
                    COUNT(*) FILTER (WHERE processing_status = 'VALIDATED'
                                   AND updated_at BETWEEN %s AND %s) AS validated,
                    COUNT(*) FILTER (WHERE processing_status = 'REJECTED'
                                   AND updated_at BETWEEN %s AND %s) AS rejected,
                    COUNT(*) FILTER (WHERE is_deleted = false
                                   AND lifecycle_status = 'ACTIVE') AS total_active
                FROM documents
                WHERE is_deleted = false
            """, [start_date, end_date, start_date, end_date, start_date, end_date])
            row = cursor.fetchone()
        return {
            'created': row[0],
            'validated': row[1],
            'rejected': row[2],
            'total_active': row[3],
        }
```

---

## 12. RÈGLES MÉTIER CRITIQUES

### RB-014-01 — SLA de chargement des dashboards

Tous les endpoints de dashboards système (`/api/v1/dashboards/*/`) doivent répondre en moins de 500ms dans 95% des cas. Ce SLA est garanti par la combinaison des vues matérialisées PostgreSQL et du cache Redis. Si le cache est absent (premier appel ou expiration), la requête aux vues matérialisées ne doit pas dépasser 400ms. Au-delà de 500ms, une alerte est loguée (INFO) et la réponse est servie sans mise en cache.

---

### RB-014-02 — Génération asynchrone obligatoire des rapports

Aucun rapport n'est jamais généré de manière synchrone dans une requête HTTP. Tout rapport, y compris les plus simples, passe obligatoirement par Celery. Cette règle évite les timeouts HTTP (rapports complexes peuvent prendre plusieurs minutes) et garantit la scalabilité. Les endpoints de génération retournent toujours `202 Accepted` avec l'ID du rapport et une URL de polling.

---

### RB-014-03 — Rétention des fichiers de rapports

Les fichiers de rapports générés sont conservés selon leur type :
- Rapports d'activité, workflow, stockage : **30 jours**
- Rapports RGPD, rapports d'audit pour autorités : **1 an** (obligation légale)
- Rapports d'éliminations et de versements : **5 ans** (traçabilité archivistique)

Un Celery Beat quotidien purge les fichiers expirés et met à jour le statut des `Report` correspondants à `EXPIRED`.

---

### RB-014-04 — Filtrage des données selon le rôle

Les données affichées dans les dashboards sont filtrées selon le rôle et le périmètre de l'utilisateur. Un Responsable ne voit que les données de son service. Un Archiviste voit son service. Ce filtrage est appliqué au niveau du backend (jamais côté frontend uniquement) et est non contournable.

---

### RB-014-05 — Fraîcheur des vues matérialisées

Les vues matérialisées PostgreSQL sont rafraîchies selon les schedules suivants :
- `mv_document_stats_daily` : toutes les heures
- `mv_user_activity_30d` : toutes les 4 heures
- `mv_overdue_workflows` : toutes les 30 minutes (données opérationnelles critiques)
- `mv_storage_by_category` : toutes les 6 heures

Un indicateur de fraîcheur est affiché sur chaque widget ("Données mises à jour il y a X minutes").

---

### RB-014-06 — Encodage UTF-8 avec BOM pour les CSV

Tous les exports CSV utilisent l'encodage UTF-8 avec BOM (`\ufeff`) et le séparateur point-virgule (`;`) pour assurer la compatibilité avec Microsoft Excel en locale française. Cette règle est non négociable pour garantir la lisibilité des caractères accentués par les agents.

---

### RB-014-07 — Sécurité des téléchargements de rapports

Les rapports sont servis via des URLs signées temporaires (TTL 1 heure) et non directement depuis le système de fichiers. Chaque téléchargement est journalisé (MODULE 11) avec l'identité du téléchargeur. Un rapport RGPD ou d'audit ne peut être téléchargé que par les rôles Admin/Super Admin ou Auditeur, indépendamment du créateur du rapport.

---

## 13. TÂCHES CELERY PLANIFIÉES

### TASK-014-01 : Rafraîchissement vues matérialisées — Documents quotidiens
```python
@shared_task(name='reporting.refresh_mv_document_stats')
# Fréquence     : Toutes les heures
# Description   : REFRESH MATERIALIZED VIEW CONCURRENTLY mv_document_stats_daily
# Note          : CONCURRENTLY évite le lock de lecture pendant le refresh
# Durée typique : 5-15 secondes selon volume
```

### TASK-014-02 : Rafraîchissement vues matérialisées — Workflows en retard
```python
@shared_task(name='reporting.refresh_mv_overdue_workflows')
# Fréquence     : Toutes les 30 minutes
# Description   : REFRESH MATERIALIZED VIEW CONCURRENTLY mv_overdue_workflows
# Criticité     : HAUTE (données opérationnelles affichées sur dashboard)
```

### TASK-014-03 : Rafraîchissement vues matérialisées — Activité utilisateurs
```python
@shared_task(name='reporting.refresh_mv_user_activity')
# Fréquence     : Toutes les 4 heures
# Description   : REFRESH MATERIALIZED VIEW CONCURRENTLY mv_user_activity_30d
```

### TASK-014-04 : Rafraîchissement vues matérialisées — Stockage par catégorie
```python
@shared_task(name='reporting.refresh_mv_storage_by_category')
# Fréquence     : Toutes les 6 heures
# Description   : REFRESH MATERIALIZED VIEW CONCURRENTLY mv_storage_by_category
```

### TASK-014-05 : Génération de rapport à la demande
```python
@shared_task(name='reporting.generate_report', bind=True, max_retries=2)
# Déclenchement : À la demande (via POST /api/v1/reports/)
# Description   : Génère les fichiers PDF/Excel/CSV selon les paramètres du Report
# Retry         : 2 tentatives avec délai exponentiel en cas d'erreur
# Timeout       : 10 minutes (abort si dépassé, status=FAILED)
```

### TASK-014-06 : Exécution des rapports planifiés (Celery Beat)
```python
@shared_task(name='reporting.run_scheduled_reports')
# Fréquence     : Toutes les 15 minutes (vérifie les planifications dues)
# Description   : Pour chaque ReportSchedule actif dont next_run_at <= now(),
#                 crée un Report et lance TASK-014-05
# Post-action   : Met à jour last_run_at et calcule next_run_at
```

### TASK-014-07 : Purge des fichiers de rapports expirés
```python
@shared_task(name='reporting.purge_expired_reports')
# Fréquence     : Quotidienne à 3h00
# Description   : Supprime physiquement les fichiers dont expires_at < now()
#                 Met à jour Report.status = 'EXPIRED'
# Traitement    : Par lot de 100, avec vérification de l'existence du fichier
# Rapport       : Volume supprimé journalisé en audit (INFO)
```

### TASK-014-08 : Calcul et mise en cache des projections de stockage
```python
@shared_task(name='reporting.compute_storage_projections')
# Fréquence     : Quotidienne à 5h00
# Description   : Analyse la croissance des 90 derniers jours, calcule la régression
#                 linéaire, stocke les projections en cache Redis (TTL 25h)
# Alerte        : Si saturation prévue < 6 mois → notification Admin (MODULE 12)
```

### TASK-014-09 : Invalidation du cache dashboard après modifications significatives
```python
@shared_task(name='reporting.invalidate_dashboard_cache')
# Déclenchement : Via signal Django (post_save sur Document, AuditLog, WorkflowInstance)
# Description   : Invalide les clés Redis des dashboards affectés
# Sélectivité   : Invalide uniquement les dashboards pertinents (pas de flush global)
```

---

## 14. ÉVÉNEMENTS JOURNALISÉS (MODULE 11)

| Code événement | Description | Sévérité | Données clés |
|----------------|-------------|----------|--------------|
| `REPORT_REQUESTED` | Rapport demandé par un utilisateur | INFO | report_id, report_type, requested_by_id |
| `REPORT_GENERATED` | Rapport généré avec succès | INFO | report_id, duration_seconds, file_sizes |
| `REPORT_GENERATION_FAILED` | Échec de génération de rapport | WARNING | report_id, error_message, attempt |
| `REPORT_DOWNLOADED` | Rapport téléchargé | INFO | report_id, downloaded_by_id, format |
| `REPORT_EXPIRED` | Fichiers de rapport supprimés (expiration) | INFO | report_id, report_type, files_deleted |
| `RGPD_REPORT_GENERATED` | Rapport RGPD généré (sensible) | INFO | report_id, period, generated_by_id |
| `AUDIT_REPORT_GENERATED` | Rapport d'audit généré | INFO | report_id, period, generated_by_id |
| `REPORT_SCHEDULE_CREATED` | Planification de rapport créée | INFO | schedule_id, frequency, report_type |
| `REPORT_SCHEDULE_MODIFIED` | Planification modifiée | INFO | schedule_id, changes |
| `REPORT_SCHEDULE_FAILED` | Exécution planifiée en échec | WARNING | schedule_id, error_message |
| `DASHBOARD_CREATED` | Dashboard personnalisé créé | INFO | dashboard_id, owner_id |
| `DASHBOARD_SHARED` | Dashboard partagé avec des utilisateurs | INFO | dashboard_id, shared_with_count |
| `MV_REFRESH_FAILED` | Échec du rafraîchissement d'une vue matérialisée | WARNING | view_name, error |
| `STORAGE_SATURATION_ALERT` | Alerte saturation stockage prévue | ALERT | estimated_saturation_date, current_usage_pct |

---

## 15. NOTIFICATIONS GÉNÉRÉES (MODULE 12)

| Déclencheur | Destinataire(s) | Canal | Priorité | Template |
|-------------|----------------|-------|----------|----------|
| Rapport prêt au téléchargement | Demandeur | In-app + Email | NORMALE | `report_ready` |
| Échec de génération de rapport | Demandeur + Admin | In-app + Email | HAUTE | `report_failed` |
| Rapport planifié expiré (2ème échec consécutif) | Admin | In-app + Email | HAUTE | `report_schedule_failing` |
| Rapport RGPD généré | Admin + Super Admin | In-app | NORMALE | `rgpd_report_ready` |
| Alerte saturation stockage (< 12 mois) | Admin + Super Admin | In-app + Email | HAUTE | `storage_saturation_warning` |
| Alerte saturation stockage (< 6 mois) | Admin + Super Admin | In-app + Email | CRITIQUE | `storage_saturation_critical` |
| Rapport planifié envoyé aux destinataires | Créateur de la planification | In-app | BASSE | `scheduled_report_sent` |

---

## 16. VARIABLES D'ENVIRONNEMENT SPÉCIFIQUES

```env
# .env — ne jamais versionner

# Stockage des rapports générés (hors du dossier media Django)
REPORTS_STORAGE_PATH=/var/archivage/reports

# Rétentions fichiers rapports (en jours)
REPORTS_RETENTION_DEFAULT_DAYS=30
REPORTS_RETENTION_RGPD_DAYS=365
REPORTS_RETENTION_AUDIT_DAYS=365
REPORTS_RETENTION_ELIMINATION_DAYS=1825

# Cache dashboard
DASHBOARD_CACHE_TTL_SECONDS=300

# Projection stockage
STORAGE_WARNING_MONTHS=12
STORAGE_CRITICAL_MONTHS=6

# Génération PDF
WEASYPRINT_FONT_PATH=/usr/share/fonts/truetype/
```

---

## 17. DÉPENDANCES PYTHON SUPPLÉMENTAIRES

```
# requirements.txt — ajouts spécifiques à ce module

# Génération PDF (rapports HTML → PDF)
weasyprint==62.*

# Génération Excel
openpyxl==3.1.*

# Déjà présents dans la stack
# reportlab   → watermarking (MODULE 13)
# celery      → tâches asynchrones et planifiées
# redis       → cache dashboards
# psycopg2    → requêtes PostgreSQL directes (vues matérialisées)
```

---

## 18. STRUCTURE DES FICHIERS

```
apps/
└── reporting/
    ├── __init__.py
    ├── admin.py
    ├── apps.py
    ├── models/
    │   ├── __init__.py
    │   ├── dashboard.py
    │   ├── widget.py
    │   ├── report.py
    │   └── report_schedule.py
    ├── serializers/
    │   ├── __init__.py
    │   ├── dashboard_serializers.py
    │   ├── widget_serializers.py
    │   └── report_serializers.py
    ├── views/
    │   ├── __init__.py
    │   ├── dashboard_views.py
    │   ├── dashboard_data_views.py    # Sources de données widget
    │   ├── widget_views.py
    │   └── report_views.py
    ├── urls.py
    ├── permissions.py
    ├── tasks.py
    ├── services/
    │   ├── __init__.py
    │   ├── report_generator.py
    │   ├── storage_projection_service.py
    │   └── dashboard_cache_service.py
    ├── collectors/
    │   ├── __init__.py
    │   ├── monthly_activity_collector.py
    │   ├── rgpd_compliance_collector.py
    │   ├── audit_authority_collector.py
    │   ├── elimination_collector.py
    │   ├── transfer_collector.py
    │   ├── security_collector.py
    │   ├── workflow_collector.py
    │   ├── storage_collector.py
    │   └── user_activity_collector.py
    ├── templates/
    │   └── reports/
    │       ├── base_report.html
    │       ├── monthly_activity.html
    │       ├── rgpd_compliance.html
    │       ├── audit_authority.html
    │       ├── elimination.html
    │       └── transfer.html
    └── tests/
        ├── __init__.py
        ├── test_dashboard_views.py
        ├── test_report_generation.py
        ├── test_report_schedules.py
        └── test_collectors.py

docs/
└── reporting/
    └── MODULE_14_Tableaux_de_Bord_Reporting.md
```

---

## ✅ RÉCAPITULATIF MODULE 14

| Critère | Valeur |
|---------|--------|
| Cas d'utilisation | 26 UC |
| Modèles de données | 4 tables |
| Endpoints API | 40 endpoints |
| Tâches Celery planifiées | 9 tâches |
| Vues matérialisées PostgreSQL | 4 vues |
| Types de rapports prédéfinis | 9 types |
| Collecteurs de données | 9 collecteurs |
| Événements journalisés (MODULE 11) | 14 événements |
| Notifications générées (MODULE 12) | 7 types |

---

**Prochain module :** MODULE 15 — Administration système & Configuration
Ce module fournit l'interface d'administration complète permettant de configurer tous les paramètres du système (SMTP, stockage, OCR, seuils d'alerte, durées de rétention, jours fériés) sans nécessiter de redéploiement. Toute modification de configuration est tracée et réversible.
