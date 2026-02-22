# MODULE 09 — Moteur de Recherche Avancée & Indexation

## Projet : Système d'Archivage Numérique des Documents
**Version :** 1.0  
**Dépendances :** MODULE 01 — Architecture | MODULE 02 — Authentification | MODULE 06 — Documents | MODULE 07 — OCR  
**Statut :** Module fondamental — condition sine qua non de l'exploitabilité du système

---

## 1. Présentation du module

Un système d'archivage sans recherche performante est comme une bibliothèque sans catalogue : les documents existent mais sont introuvables. Ce module transforme l'entrepôt documentaire en une base de connaissances exploitable où chaque document peut être retrouvé en quelques secondes, quelle que soit la taille du fonds d'archives.

La recherche s'appuie sur trois dimensions complémentaires :
- **Recherche full-text** : recherche dans le contenu textuel (OCR) et les métadonnées
- **Recherche par facettes** : filtres combinables (catégorie, type, date, confidentialité, etc.)
- **Recherche sémantique** : compréhension du sens au-delà des mots-clés exacts

Le module utilise **PostgreSQL Full-Text Search** (natif, performant jusqu'à plusieurs millions de documents) avec possibilité de migration vers **Elasticsearch** si les volumes deviennent très importants (>10 millions de documents).

---

## 2. Technologies et architecture

### 2.1 Moteur de recherche principal

**PostgreSQL Full-Text Search (FTS)**
- Index GIN (Generalized Inverted Index) sur les champs textuels
- Support du français avec stemming (racinisation) et stop words
- Pondération des champs (titre > contenu > métadonnées)
- Ranking natif avec `ts_rank` et `ts_rank_cd`
- Performances excellentes jusqu'à 10M de documents
- Pas de dépendance externe supplémentaire

**Migration optionnelle vers Elasticsearch**
- Pour des volumes >10M de documents
- Recherche distribuée et scalable
- Meilleure gestion du multi-langue
- Facettes et agrégations plus puissantes
- L'architecture du module permet cette migration sans changement d'API

### 2.2 Stratégie d'indexation

| Champ | Source | Pondération | Exemple |
|---|---|---|---|
| Titre du document | `documents.title` | A (poids max) | "Contrat de travail" |
| Contenu OCR | `ocr_results.extracted_text` | B (poids élevé) | "... clause de confidentialité..." |
| Métadonnées standard | `documents.description`, `author` | C (poids moyen) | "Émis par le service RH" |
| Métadonnées personnalisées | `documents.custom_metadata` | C (poids moyen) | "Numéro de contrat : CT-2026-001" |
| Tags | `tags.name` | B (poids élevé) | "urgent", "confidentiel" |
| Commentaires | `document_comments.content` | D (poids faible) | Commentaires des utilisateurs |

---

## 3. Cas d'utilisation — Vue d'ensemble

### 3.1 Recherche de base

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-SRCH-01 | Recherche simple par mots-clés | Tout utilisateur |
| UC-SRCH-02 | Recherche dans le titre uniquement | Tout utilisateur |
| UC-SRCH-03 | Recherche dans le contenu OCR uniquement | Tout utilisateur |
| UC-SRCH-04 | Recherche par numéro de référence exact | Tout utilisateur |
| UC-SRCH-05 | Recherche par plage de dates | Tout utilisateur |

### 3.2 Recherche avancée

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-SRCH-06 | Recherche avec filtres combinés | Tout utilisateur |
| UC-SRCH-07 | Recherche avec opérateurs booléens (AND, OR, NOT) | Tout utilisateur |
| UC-SRCH-08 | Recherche par proximité de mots | Tout utilisateur |
| UC-SRCH-09 | Recherche avec caractères joker (wildcards) | Tout utilisateur |
| UC-SRCH-10 | Recherche phonétique (similarité sonore) | Tout utilisateur |
| UC-SRCH-11 | Recherche floue (tolérance aux fautes) | Tout utilisateur |
| UC-SRCH-12 | Recherche par plage de valeurs numériques | Tout utilisateur |

### 3.3 Facettes et filtres

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-SRCH-13 | Filtrer par catégorie | Tout utilisateur |
| UC-SRCH-14 | Filtrer par type de document | Tout utilisateur |
| UC-SRCH-15 | Filtrer par niveau de confidentialité | Tout utilisateur |
| UC-SRCH-16 | Filtrer par statut du cycle de vie | Tout utilisateur |
| UC-SRCH-17 | Filtrer par service émetteur | Tout utilisateur |
| UC-SRCH-18 | Filtrer par tags | Tout utilisateur |
| UC-SRCH-19 | Combiner plusieurs filtres simultanément | Tout utilisateur |

### 3.4 Recherche sémantique et suggestions

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-SRCH-20 | Suggestions automatiques pendant la frappe | Tout utilisateur |
| UC-SRCH-21 | Correction automatique des fautes de frappe | Tout utilisateur |
| UC-SRCH-22 | Recherche de documents similaires à un document donné | Tout utilisateur |
| UC-SRCH-23 | Recherche par entités nommées (personnes, lieux, organisations) | Tout utilisateur |

### 3.5 Gestion des recherches

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-SRCH-24 | Sauvegarder une recherche | Tout utilisateur |
| UC-SRCH-25 | Consulter ses recherches sauvegardées | Tout utilisateur |
| UC-SRCH-26 | Exécuter une recherche sauvegardée | Tout utilisateur |
| UC-SRCH-27 | Supprimer une recherche sauvegardée | Tout utilisateur |
| UC-SRCH-28 | Consulter l'historique de ses recherches | Tout utilisateur |
| UC-SRCH-29 | Créer une alerte de recherche (notification sur nouveaux résultats) | Tout utilisateur |

---

## 4. Description détaillée des cas d'utilisation

### UC-SRCH-01 — Recherche simple par mots-clés

**Acteur :** Tout utilisateur authentifié  
**Préconditions :** L'utilisateur est connecté  
**Postconditions :** Les résultats pertinents sont affichés, filtrés selon les permissions de l'utilisateur

**Scénario principal :**
1. L'utilisateur saisit un ou plusieurs mots-clés dans la barre de recherche principale (ex: "contrat travail")
2. Le système lance la recherche dès la validation (Enter ou clic sur bouton)
3. Le système recherche dans tous les champs indexés (titre, contenu OCR, métadonnées, tags, commentaires)
4. Le système applique automatiquement le filtre de confidentialité selon le rôle de l'utilisateur (un Agent ne verra pas les documents Secret)
5. Le système calcule un score de pertinence pour chaque résultat
6. Le système affiche les résultats triés par pertinence décroissante
7. Pour chaque résultat, l'utilisateur voit :
   - Titre du document
   - Extrait du contenu avec les mots-clés surlignés (snippet)
   - Métadonnées principales (type, catégorie, date)
   - Score de pertinence (sous forme d'étoiles ou pourcentage)
   - Badge de confidentialité
8. L'utilisateur peut cliquer sur un résultat pour consulter le document complet

**Règles métier :**
- La recherche est insensible à la casse (MAJUSCULES/minuscules)
- Les accents sont normalisés (e = é = è = ê)
- Le stemming français est appliqué : "travaille", "travaillé", "travailleur" trouvent tous "travail"
- Les stop words courants sont ignorés ("le", "la", "de", "et", etc.)
- Seuls les documents auxquels l'utilisateur a accès sont retournés

---

### UC-SRCH-06 — Recherche avec filtres combinés

**Acteur :** Tout utilisateur  
**Préconditions :** L'utilisateur a effectué une recherche  
**Postconditions :** Les résultats sont filtrés selon les critères sélectionnés

**Scénario principal :**
1. L'utilisateur effectue une recherche initiale (UC-SRCH-01)
2. Le système affiche les résultats et un panneau de filtres sur le côté
3. Le système affiche les facettes disponibles avec le nombre de documents par facette :
   - **Catégorie** : Administratif (45), Juridique (12), Financier (8)...
   - **Type de document** : Contrat (23), Facture (15), Rapport (7)...
   - **Année** : 2026 (30), 2025 (20), 2024 (5)...
   - **Confidentialité** : Public (10), Interne (35), Confidentiel (10)
   - **Service** : RH (25), Finance (15), Juridique (10)...
   - **Tags** : urgent (5), important (12), archivé (8)...
4. L'utilisateur sélectionne un ou plusieurs filtres (ex: Catégorie = Juridique + Année = 2026)
5. Le système applique les filtres immédiatement sans recharger la page (AJAX)
6. Le système met à jour le nombre de résultats et recalcule les facettes
7. L'utilisateur peut ajouter ou retirer des filtres dynamiquement
8. L'utilisateur peut réinitialiser tous les filtres d'un clic

**Règles métier :**
- Les filtres sont cumulatifs (ET logique entre différentes catégories, OU logique au sein de la même catégorie)
- Les facettes sont recalculées dynamiquement après chaque filtre
- Les filtres inaccessibles (ex: documents Secret pour un Agent) n'apparaissent pas dans les facettes

---

### UC-SRCH-07 — Recherche avec opérateurs booléens

**Acteur :** Tout utilisateur  
**Préconditions :** L'utilisateur connaît la syntaxe des opérateurs  
**Postconditions :** Les résultats correspondent à la requête booléenne

**Scénario principal :**
1. L'utilisateur saisit une requête avec opérateurs booléens dans la barre de recherche
2. **Opérateur AND** (ET) — tous les termes doivent être présents :
   - Syntaxe : `contrat AND travail` ou `contrat travail` (AND implicite)
   - Résultat : documents contenant "contrat" ET "travail"
3. **Opérateur OR** (OU) — au moins un terme doit être présent :
   - Syntaxe : `contrat OR convention`
   - Résultat : documents contenant "contrat" OU "convention" OU les deux
4. **Opérateur NOT** (SAUF) — exclure un terme :
   - Syntaxe : `contrat NOT interim`
   - Résultat : documents contenant "contrat" mais PAS "interim"
5. **Parenthèses** pour grouper :
   - Syntaxe : `(contrat OR convention) AND (travail OR emploi)`
   - Résultat : documents contenant (contrat ou convention) ET (travail ou emploi)
6. **Guillemets** pour recherche de phrase exacte :
   - Syntaxe : `"clause de non-concurrence"`
   - Résultat : documents contenant exactement cette phrase
7. Le système parse la requête, construit la requête PostgreSQL FTS correspondante
8. Le système retourne les résultats

**Règles métier :**
- Les opérateurs booléens sont insensibles à la casse (and = AND)
- En cas d'erreur de syntaxe, le système affiche un message d'aide avec des exemples
- Les opérateurs peuvent être combinés avec les filtres

---

### UC-SRCH-20 — Suggestions automatiques pendant la frappe

**Acteur :** Tout utilisateur  
**Préconditions :** L'utilisateur commence à taper dans la barre de recherche  
**Postconditions :** Des suggestions pertinentes sont affichées en temps réel

**Scénario principal :**
1. L'utilisateur commence à taper dans la barre de recherche (ex: "cont")
2. Après 3 caractères minimum, le système déclenche une recherche de suggestions
3. Le système consulte plusieurs sources de suggestions :
   - **Titres de documents** commençant par "cont" : "Contrat de travail", "Contrat de prestation"...
   - **Tags populaires** : "contrat", "contentieux"...
   - **Termes fréquents** dans le corpus OCR
   - **Recherches récentes de l'utilisateur** contenant "cont"
4. Le système affiche un dropdown de 5-10 suggestions maximum, triées par pertinence
5. Pour chaque suggestion, le système affiche le nombre de résultats attendus
6. L'utilisateur peut cliquer sur une suggestion pour lancer la recherche
7. L'utilisateur peut continuer à taper et les suggestions se mettent à jour en temps réel
8. L'utilisateur peut ignorer les suggestions et valider sa propre recherche

**Optimisation des performances :**
- Les suggestions sont calculées côté backend mais avec un cache Redis (TTL 1h)
- Le frontend attend 300ms après la dernière frappe avant d'envoyer la requête (debounce)
- Maximum 10 suggestions retournées pour limiter le payload

---

### UC-SRCH-24 — Sauvegarder une recherche

**Acteur :** Tout utilisateur  
**Préconditions :** L'utilisateur a effectué une recherche avec des filtres  
**Postconditions :** La recherche est sauvegardée et réutilisable

**Scénario principal :**
1. L'utilisateur effectue une recherche complexe avec plusieurs filtres
2. L'utilisateur clique sur "Sauvegarder cette recherche"
3. Le système affiche un formulaire de sauvegarde :
   - Nom de la recherche (obligatoire) : ex: "Contrats RH 2026 urgents"
   - Description (optionnelle)
   - Visibilité : privée (moi uniquement) ou partagée (avec mon équipe)
   - Créer une alerte : recevoir une notification quotidienne/hebdomadaire si de nouveaux documents correspondent
4. L'utilisateur remplit et valide
5. Le système sauvegarde les paramètres de recherche (mots-clés + tous les filtres actifs)
6. La recherche apparaît dans "Mes recherches sauvegardées"
7. L'utilisateur peut exécuter cette recherche d'un clic à tout moment

**Règles métier :**
- Une recherche sauvegardée capture l'état complet : requête, filtres, tri
- Si une alerte est activée, le système exécute la recherche quotidiennement et notifie si de nouveaux documents apparaissent
- Les recherches partagées sont visibles par tous les membres du département de l'utilisateur

---

## 5. Stratégie d'indexation et architecture

### 5.1 Structure de l'index PostgreSQL FTS

**Champ tsvector composite** :
```sql
CREATE INDEX idx_documents_search ON documents 
USING GIN (
  to_tsvector('french', 
    coalesce(title, '') || ' ' || 
    coalesce(description, '') || ' ' || 
    coalesce(author, '') || ' ' ||
    coalesce((SELECT string_agg(extracted_text, ' ') FROM ocr_results WHERE document_id = documents.id), '') || ' ' ||
    coalesce((SELECT string_agg(name, ' ') FROM tags JOIN document_tags ON tags.id = document_tags.tag_id WHERE document_tags.document_id = documents.id), '')
  )
);
```

### 5.2 Pondération des champs

La pondération est définie selon 4 niveaux (A, B, C, D) :

```sql
setweight(to_tsvector('french', title), 'A') ||
setweight(to_tsvector('french', coalesce(ocr_text, '')), 'B') ||
setweight(to_tsvector('french', coalesce(description, '')), 'C') ||
setweight(to_tsvector('french', coalesce(tags, '')), 'B')
```

Le calcul de pertinence favorise les documents où les termes apparaissent dans le titre.

### 5.3 Calcul du score de pertinence

PostgreSQL FTS utilise deux fonctions de ranking :

**`ts_rank`** : basé sur la fréquence des termes
```sql
ts_rank(search_vector, to_tsquery('french', 'contrat & travail'))
```

**`ts_rank_cd`** : prend en compte la proximité des termes (Cover Density)
```sql
ts_rank_cd(search_vector, to_tsquery('french', 'contrat & travail'))
```

Le système utilise une combinaison pondérée :
```sql
(ts_rank(search_vector, query) * 0.4) + (ts_rank_cd(search_vector, query) * 0.6)
```

---

## 6. Modèles de données

### 6.1 Modèle `SearchQuery` (Recherche sauvegardée)

**Table :** `search_queries`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `user_id` | UUID | FK → users.id, NOT NULL, INDEX | Propriétaire |
| `name` | VARCHAR(200) | NOT NULL | Nom de la recherche |
| `description` | TEXT | NULL | Description |
| `query_text` | TEXT | NOT NULL | Texte de la requête |
| `filters` | JSONB | NULL | Filtres appliqués (catégorie, type, dates, etc.) |
| `sort_by` | VARCHAR(50) | NOT NULL, DEFAULT 'relevance' | Tri : `relevance`, `date_desc`, `date_asc`, `title_asc` |
| `is_alert_enabled` | BOOLEAN | NOT NULL, DEFAULT FALSE | Alerte activée |
| `alert_frequency` | VARCHAR(20) | NULL | Fréquence : `daily`, `weekly` |
| `last_alert_sent_at` | TIMESTAMP | NULL | Dernière alerte envoyée |
| `is_shared` | BOOLEAN | NOT NULL, DEFAULT FALSE | Recherche partagée avec l'équipe |
| `result_count_snapshot` | INTEGER | NULL | Nombre de résultats lors de la dernière exécution |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `updated_at` | TIMESTAMP | NOT NULL, AUTO | Date de mise à jour |

**Exemple de `filters` JSONB :**
```json
{
  "categories": ["uuid-cat-1", "uuid-cat-2"],
  "document_types": ["uuid-type-1"],
  "date_from": "2025-01-01",
  "date_to": "2026-12-31",
  "confidentiality_levels": ["internal", "public"],
  "tags": ["urgent", "important"]
}
```

**Index :**
- `idx_search_queries_user_id` sur `user_id`
- `idx_search_queries_is_alert_enabled` sur `is_alert_enabled`

---

### 6.2 Modèle `SearchHistory` (Historique des recherches)

**Table :** `search_history`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `user_id` | UUID | FK → users.id, NOT NULL, INDEX | Utilisateur |
| `query_text` | TEXT | NOT NULL | Texte recherché |
| `filters_applied` | JSONB | NULL | Filtres appliqués |
| `result_count` | INTEGER | NOT NULL | Nombre de résultats |
| `search_duration_ms` | INTEGER | NULL | Durée de la recherche (ms) |
| `clicked_document_id` | UUID | FK → documents.id, NULL | Document cliqué (si applicable) |
| `searched_at` | TIMESTAMP | NOT NULL, AUTO | Date et heure de la recherche |

**Index :**
- `idx_search_history_user_id` sur `user_id`
- `idx_search_history_searched_at` sur `searched_at`

**Règle de rétention :** L'historique de recherche est conservé 90 jours puis supprimé automatiquement (RGPD).

---

### 6.3 Modèle `DocumentSearchIndex` (Index de recherche — vue matérialisée)

**Vue matérialisée :** `document_search_index`

Cette vue matérialisée est rafraîchie automatiquement après chaque modification de document, résultat OCR, ou tag.

| Champ | Type | Description |
|---|---|---|
| `document_id` | UUID | ID du document |
| `search_vector` | TSVECTOR | Vecteur de recherche full-text |
| `title` | TEXT | Titre (pour affichage) |
| `snippet` | TEXT | Extrait du contenu (300 premiers caractères) |
| `category_id` | UUID | Catégorie (pour filtres) |
| `document_type_id` | UUID | Type (pour filtres) |
| `confidentiality_level_id` | UUID | Niveau de confidentialité (pour filtres) |
| `lifecycle_status` | VARCHAR | Statut (pour filtres) |
| `document_date` | DATE | Date du document (pour filtres) |
| `created_at` | TIMESTAMP | Date de création (pour tri) |
| `tags` | ARRAY TEXT | Liste des tags (pour filtres) |

**Index :**
- Index GIN sur `search_vector` (index de recherche principal)
- Index B-tree sur `category_id`, `document_type_id`, `confidentiality_level_id`
- Index B-tree sur `document_date`

---

### 6.4 Modèle `SearchSuggestion` (Suggestions précalculées)

**Table :** `search_suggestions`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `suggestion_text` | VARCHAR(200) | UNIQUE, NOT NULL | Texte de la suggestion |
| `suggestion_type` | VARCHAR(30) | NOT NULL | Type : `document_title`, `tag`, `frequent_term` |
| `frequency` | INTEGER | NOT NULL, DEFAULT 1 | Fréquence d'utilisation |
| `result_count` | INTEGER | NOT NULL | Nombre de résultats attendus |
| `last_used_at` | TIMESTAMP | NOT NULL | Dernière utilisation |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |

**Index :**
- `idx_search_suggestions_text` sur `suggestion_text` (index GIN trigram pour recherche partielle)
- `idx_search_suggestions_frequency` sur `frequency DESC`

**Règle :** Les suggestions sont mises à jour quotidiennement par une tâche Celery qui analyse les recherches les plus fréquentes.

---

## 7. Matrice des permissions du module

| Permission code | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `search.basic` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `search.advanced` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `search.save_query` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `search.create_alert` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `search.view_history` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `search.export_results` | ✅ | ✅ | ✅ | ✅ | ✅ (limité) | ✅ |
| `search.view_all_users_history` | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |

**Note :** Les résultats de recherche sont toujours filtrés selon les permissions de confidentialité de l'utilisateur (MODULE 05).

---

## 8. Endpoints API du module

### 8.1 Recherche

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/search/` | Recherche principale | Oui |
| POST | `/api/v1/search/` | Recherche avec corps JSON (requêtes complexes) | Oui |
| GET | `/api/v1/search/suggest/` | Suggestions automatiques | Oui |
| GET | `/api/v1/search/facets/` | Récupérer les facettes disponibles | Oui |

**Paramètres de `/api/v1/search/` :**
```
?q=contrat travail
&category_id=uuid-cat
&document_type_id=uuid-type
&date_from=2025-01-01
&date_to=2026-12-31
&confidentiality_level=internal,public
&tags=urgent,important
&sort_by=relevance
&page=1
&per_page=20
```

### 8.2 Recherches sauvegardées

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/search/saved-queries/` | Liste des recherches sauvegardées | Oui |
| POST | `/api/v1/search/saved-queries/` | Sauvegarder une recherche | Oui |
| GET | `/api/v1/search/saved-queries/{id}/` | Détail d'une recherche sauvegardée | Oui |
| PATCH | `/api/v1/search/saved-queries/{id}/` | Modifier | Oui |
| DELETE | `/api/v1/search/saved-queries/{id}/` | Supprimer | Oui |
| POST | `/api/v1/search/saved-queries/{id}/execute/` | Exécuter | Oui |

### 8.3 Historique

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/search/history/` | Historique de mes recherches | Oui |
| DELETE | `/api/v1/search/history/` | Effacer l'historique | Oui |
| GET | `/api/v1/search/history/{id}/` | Détail d'une recherche passée | Oui |

---

## 9. Optimisations et performances

### 9.1 Cache des résultats de recherche

Les résultats de recherche sont mis en cache dans Redis avec une clé basée sur le hash de la requête + filtres :
- **TTL** : 5 minutes
- **Invalidation** : lors de la création/modification/suppression d'un document

Cela permet de servir instantanément les recherches identiques effectuées par plusieurs utilisateurs.

### 9.2 Pagination efficace

La pagination utilise des **cursors** plutôt que des offsets pour de meilleures performances sur les grands résultats :
```sql
WHERE id > last_seen_id ORDER BY relevance_score DESC LIMIT 20
```

### 9.3 Indexation différée

L'indexation des documents n'est pas effectuée de manière synchrone à la création :
1. Document créé → enregistrement en base
2. Tâche Celery asynchrone d'indexation déclenchée
3. Indexation effectuée en arrière-plan (1-5 secondes)

Pour les documents urgents, une indexation synchrone peut être forcée.

---

## 10. Tâches Celery planifiées

| Tâche | Fréquence | Description |
|---|---|---|
| `search.reindex_modified_documents` | Toutes les 5 minutes | Réindexe les documents modifiés depuis la dernière exécution |
| `search.update_suggestions` | Quotidien (2h) | Met à jour les suggestions précalculées |
| `search.send_search_alerts` | Quotidien (9h) | Exécute les recherches sauvegardées avec alerte activée |
| `search.cleanup_old_history` | Quotidien (3h) | Supprime l'historique de recherche >90 jours |
| `search.refresh_materialized_view` | Horaire | Rafraîchit la vue matérialisée `document_search_index` |
| `search.optimize_indexes` | Hebdomadaire (dimanche 4h) | VACUUM et REINDEX PostgreSQL |

---

## 11. Événements journalisés (MODULE 11 — Audit)

| Événement | Niveau | Détails |
|---|---|---|
| Recherche effectuée | INFO | user_id, query_text, result_count, duration_ms |
| Recherche sauvegardée créée | INFO | user_id, query_name, is_shared |
| Alerte de recherche déclenchée | INFO | query_id, user_id, new_results_count |
| Document cliqué depuis résultats | INFO | user_id, document_id, search_query, position_in_results |

---

## 12. Notifications générées (MODULE 12)

| Déclencheur | Destinataire | Canal | Message |
|---|---|---|
| Alerte de recherche (nouveaux résultats) | Utilisateur | Email + In-app | [N] nouveaux documents correspondent à votre recherche "[Nom]" |
| Aucun résultat trouvé (suggestion) | Utilisateur | In-app | Aucun résultat. Essayez : [Suggestions] |

---

*Fin du MODULE 09 — Moteur de Recherche Avancée & Indexation*  
*Nous avons maintenant complété 9 des 18 modules du cahier des charges. Il reste encore 9 modules à créer.*

**Statut actuel : 50% du cahier des charges complet**

Souhaites-tu que je poursuive avec le MODULE 10 — Intelligence artificielle & Automatisation ?
