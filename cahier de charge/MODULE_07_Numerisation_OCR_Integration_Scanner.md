# MODULE 07 — Numérisation, OCR & Intégration Scanner

## Projet : Système d'Archivage Numérique des Documents
**Version :** 1.0  
**Dépendances :** MODULE 01 — Architecture | MODULE 02 — Authentification | MODULE 06 — Documents  
**Statut :** Module technique — le MODULE 09 (Recherche) et le MODULE 10 (IA) dépendent de ce module

---

## 1. Présentation du module

La numérisation et l'extraction du contenu textuel (OCR — Optical Character Recognition) sont des fonctionnalités différenciantes majeures du système. Ce module transforme des documents papier scannés ou des images en documents numériques pleinement exploitables, indexables et recherchables.

L'intégration native avec les scanners via Tauri permet à l'application desktop d'accéder directement au matériel de numérisation sans passer par des logiciels tiers. L'OCR, basé sur Tesseract, extrait le texte des documents scannés avec un prétraitement intelligent pour maximiser la qualité de reconnaissance.

Ce module ne se contente pas d'un OCR basique : il intègre un contrôle qualité, une correction manuelle assistée, la détection automatique de la langue, et un système de scoring de confiance qui permet à l'archiviste d'identifier rapidement les documents nécessitant une révision.

---

## 2. Architecture technique du module

### 2.1 Composants principaux

**Côté Desktop (Tauri)**
- **Scanner Bridge** : Interface Rust Tauri communiquant avec les pilotes scanner (TWAIN sur Windows, SANE sur Linux)
- **Scan Preview** : Prévisualisation en temps réel avant scan définitif
- **Scan Settings Manager** : Configuration des paramètres de scan (résolution, couleur, format)

**Côté Backend (Django)**
- **OCR Queue Manager** : Gestion de la file d'attente des tâches OCR via Celery
- **Image Preprocessor** : Prétraitement des images avant OCR (deskew, noise reduction, contrast enhancement)
- **Tesseract Wrapper** : Interface Python avec Tesseract OCR
- **Quality Scorer** : Évaluation de la qualité du résultat OCR
- **Text Corrector** : Interface de correction manuelle assistée

**Bibliothèques utilisées**
- `pytesseract` : Wrapper Python pour Tesseract
- `Pillow` (PIL Fork) : Manipulation d'images
- `opencv-python` (cv2) : Prétraitement avancé d'images
- `pdf2image` : Conversion PDF vers images pour OCR
- `langdetect` : Détection automatique de la langue du texte extrait

---

## 3. Cas d'utilisation — Vue d'ensemble

| Code | Cas d'utilisation | Acteur principal |
|---|---|---|
| UC-OCR-01 | Scanner un document directement depuis l'application (mode desktop) | Archiviste / Agent |
| UC-OCR-02 | Configurer les paramètres de scan (résolution, couleur, recto-verso) | Archiviste / Agent |
| UC-OCR-03 | Prévisualiser le document avant scan définitif | Archiviste / Agent |
| UC-OCR-04 | Scanner un lot de documents (scan multiple) | Archiviste |
| UC-OCR-05 | Déclencher manuellement l'OCR sur un document existant | Archiviste |
| UC-OCR-06 | Consulter le statut de traitement OCR d'un document | Tout utilisateur (selon permissions) |
| UC-OCR-07 | Consulter le texte extrait par OCR | Tout utilisateur (selon permissions) |
| UC-OCR-08 | Corriger manuellement le texte OCR | Archiviste / Responsable |
| UC-OCR-09 | Valider le résultat OCR | Archiviste / Responsable |
| UC-OCR-10 | Relancer l'OCR après échec | Archiviste |
| UC-OCR-11 | Consulter les erreurs OCR | Archiviste / Admin |
| UC-OCR-12 | Configurer les langues supportées pour l'OCR | Admin |
| UC-OCR-13 | Consulter le score de qualité OCR | Archiviste |
| UC-OCR-14 | Filtrer les documents par qualité OCR | Archiviste |
| UC-OCR-15 | Exporter le texte OCR (TXT, PDF avec couche texte) | Tout utilisateur (selon permissions) |
| UC-OCR-16 | Annoter des zones à ignorer lors de l'OCR | Archiviste |
| UC-OCR-17 | Définir des zones prioritaires pour l'OCR | Archiviste |
| UC-OCR-18 | Consulter les statistiques OCR globales | Admin / Archiviste |

---

## 4. Description détaillée des cas d'utilisation

### UC-OCR-01 — Scanner un document directement depuis l'application (mode desktop)

**Acteur :** Archiviste ou Agent  
**Préconditions :** L'application desktop Tauri est lancée. Un scanner est connecté et détecté par le système  
**Postconditions :** Le document est scanné, uploadé et enregistré dans le système avec déclenchement automatique de l'OCR

**Scénario principal :**
1. L'acteur place le document physique dans le scanner
2. L'acteur accède à la fonction "Numériser un document" dans l'application desktop
3. Le système détecte automatiquement le(s) scanner(s) disponible(s) et affiche la liste
4. L'acteur sélectionne le scanner à utiliser
5. L'acteur configure les paramètres de scan :
   - Résolution (150, 300, 600 dpi) — défaut : 300 dpi
   - Mode couleur (Couleur, Niveaux de gris, Noir et blanc)
   - Format de sortie (PDF, JPEG, PNG, TIFF)
   - Recto-verso (si supporté par le scanner)
   - Taille de page (A4, A3, Lettre, Auto-detect)
6. L'acteur clique sur "Prévisualiser"
7. Le système effectue un scan rapide basse résolution (preview scan)
8. L'acteur voit la prévisualisation et peut ajuster la zone de découpe
9. L'acteur valide et clique sur "Scanner"
10. Le système lance le scan haute résolution
11. Le système affiche une barre de progression
12. Le scan est terminé, le système affiche l'image scannée
13. L'acteur peut :
    - Scanner une page supplémentaire (pour documents multi-pages)
    - Recommencer le scan
    - Valider et créer le document
14. L'acteur remplit les métadonnées du document (titre, type, catégorie, etc.)
15. Le système uploade le fichier scanné vers le backend via l'API
16. Le système crée le document (UC-DOC-01 du MODULE 06)
17. Le système déclenche automatiquement une tâche Celery de traitement OCR
18. L'acteur reçoit une notification : "Document créé, OCR en cours de traitement"

**Scénarios alternatifs :**
- **3a.** Aucun scanner n'est détecté → afficher un message d'erreur et un lien vers la configuration des scanners du système
- **10a.** Le scan échoue (bourrage papier, scanner débranché) → afficher l'erreur et permettre de réessayer
- **13a.** L'acteur scanne plusieurs pages → le système assemble toutes les pages en un seul fichier PDF multi-pages

---

### UC-OCR-05 — Déclencher manuellement l'OCR sur un document existant

**Acteur :** Archiviste  
**Préconditions :** Le document existe. Le document n'a pas encore de texte OCR ou l'OCR a échoué précédemment  
**Postconditions :** Une nouvelle tâche OCR est créée et traitée

**Scénario principal :**
1. L'archiviste consulte le document
2. L'archiviste voit le statut OCR actuel (ex: "Non traité", "Échec", "Qualité insuffisante")
3. L'archiviste clique sur "Lancer l'OCR"
4. Le système vérifie que le format du fichier est compatible (PDF, JPEG, PNG, TIFF)
5. Le système demande confirmation si un OCR existe déjà (écrasement du texte existant)
6. L'archiviste confirme
7. Le système crée une nouvelle tâche OCR dans la file Celery
8. Le système affiche un message : "OCR planifié, traitement en cours"
9. L'archiviste peut continuer à travailler, il sera notifié à la fin du traitement

---

### UC-OCR-08 — Corriger manuellement le texte OCR

**Acteur :** Archiviste ou Responsable  
**Préconditions :** Le document possède un texte OCR extrait. L'acteur a la permission de corriger l'OCR  
**Postconditions :** Le texte OCR est corrigé et réindexé pour la recherche

**Scénario principal :**
1. L'archiviste consulte le document
2. L'archiviste accède à l'onglet "Texte OCR"
3. Le système affiche le texte extrait avec une interface d'édition côte-à-côte : à gauche l'image du document, à droite le texte OCR
4. L'archiviste identifie les erreurs de reconnaissance (caractères mal reconnus, mots incorrects)
5. L'archiviste corrige le texte directement dans l'éditeur
6. Le système peut suggérer des corrections automatiques basées sur un dictionnaire (soulignement des mots non reconnus)
7. L'archiviste peut accepter ou refuser les suggestions
8. L'archiviste sauvegarde les modifications
9. Le système marque le texte OCR comme "Corrigé manuellement"
10. Le système réindexe le document avec le nouveau texte (MODULE 09)
11. Le système enregistre une entrée dans l'historique : "Texte OCR corrigé manuellement par [Nom]"

**Règles métier :**
- Un texte OCR corrigé manuellement ne peut plus être écrasé par un nouvel OCR automatique sans confirmation explicite
- Chaque correction manuelle augmente le score de qualité OCR à 100%

---

## 5. Processus de traitement OCR — Étapes détaillées

### 5.1 Pipeline complet

```
1. Réception de la tâche OCR
   ↓
2. Validation du fichier (format supporté, fichier existe)
   ↓
3. Conversion en images (si PDF multi-pages → une image par page)
   ↓
4. Prétraitement de chaque image
   ↓
5. Détection de la langue du document (si non spécifiée)
   ↓
6. Extraction du texte via Tesseract
   ↓
7. Post-traitement du texte (nettoyage, normalisation)
   ↓
8. Calcul du score de qualité OCR
   ↓
9. Détection automatique de métadonnées (dates, montants, références)
   ↓
10. Sauvegarde du résultat
   ↓
11. Indexation pour la recherche (MODULE 09)
   ↓
12. Notification de fin de traitement
```

---

### 5.2 Étape 4 — Prétraitement de l'image

Le prétraitement améliore considérablement la qualité de reconnaissance. Opérations appliquées séquentiellement :

**1. Conversion en niveaux de gris** (si l'image est en couleur)
```python
# Conversion via Pillow
image = image.convert('L')
```

**2. Redressement (Deskew)** — correction de l'inclinaison du scan
```python
# Détection de l'angle via OpenCV
# Rotation pour aligner le texte horizontalement
```

**3. Réduction du bruit (Denoising)**
```python
# Filtre médian pour supprimer le bruit de scan
cv2.medianBlur(image, 3)
```

**4. Augmentation du contraste**
```python
# Égalisation d'histogramme adaptatif (CLAHE)
clahe = cv2.createCLAHE(clipLimit=2.0, tileGridSize=(8,8))
enhanced = clahe.apply(image)
```

**5. Binarisation (Noir et blanc pur)** — méthode adaptative d'Otsu
```python
# Seuillage automatique pour obtenir un document noir sur blanc net
_, binary = cv2.threshold(image, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)
```

**6. Suppression des bordures noires** (crop automatique)
```python
# Détection des contours pour supprimer les marges inutiles
```

---

### 5.3 Étape 5 — Détection de la langue

Si la langue du document n'est pas spécifiée dans les métadonnées, le système :
1. Effectue un OCR rapide d'un échantillon (premier 1/4 de la première page)
2. Passe le texte extrait à `langdetect`
3. Obtient la langue détectée (ex: `fr`, `en`, `es`)
4. Relance Tesseract avec le pack de langue approprié

**Langues supportées par défaut :**
- Français (`fra`)
- Anglais (`eng`)
- Espagnol (`spa`)
- Allemand (`deu`)
- Arabe (`ara`)

D'autres langues peuvent être installées via les packs de données Tesseract.

---

### 5.4 Étape 6 — Extraction du texte via Tesseract

Configuration Tesseract optimale :
```python
custom_config = r'--oem 3 --psm 1 -l fra'
# oem 3 = LSTM neural network mode (meilleur)
# psm 1 = Automatic page segmentation with OSD (Orientation and Script Detection)
# -l fra = langue française
```

Le système extrait :
- Le texte brut (plein texte)
- Les coordonnées de chaque mot (bounding boxes) — pour l'interface de correction
- Le niveau de confiance par mot (0-100%)

---

### 5.5 Étape 8 — Calcul du score de qualité OCR

Le score de qualité est calculé selon plusieurs critères :

```python
score_qualite = (
    (confiance_moyenne_tesseract * 0.4) +
    (taux_mots_reconnus_dictionnaire * 0.3) +
    (ratio_caracteres_alphanumeriques * 0.2) +
    (absence_caracteres_bizarres * 0.1)
)
```

**Interprétation du score :**
- **90-100%** : Excellente qualité, aucune correction nécessaire
- **70-89%** : Bonne qualité, corrections mineures possibles
- **50-69%** : Qualité moyenne, révision recommandée
- **0-49%** : Qualité faible, révision manuelle obligatoire

---

### 5.6 Étape 9 — Détection automatique de métadonnées

Le système applique des regex pour extraire automatiquement :
- **Dates** : formats français (DD/MM/YYYY), ISO (YYYY-MM-DD)
- **Montants** : `123,45 €`, `$ 1,234.56`
- **Numéros de téléphone** : formats locaux
- **Emails** : adresses email valides
- **Numéros de référence** : patterns configurables (ex: contrat n° XX-YYYY-NNNN)

Ces métadonnées extraites sont proposées à l'archiviste lors de la saisie manuelle des métadonnées personnalisées.

---

## 6. Modèles de données

### 6.1 Modèle `OcrTask` (Tâche OCR)

**Table :** `ocr_tasks`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `document_id` | UUID | FK → documents.id, NOT NULL, INDEX | Document à traiter |
| `status` | VARCHAR(30) | NOT NULL, DEFAULT 'pending' | Statut : `pending`, `processing`, `completed`, `failed`, `cancelled` |
| `language` | VARCHAR(10) | NULL | Langue spécifiée (si NULL → détection auto) |
| `language_detected` | VARCHAR(10) | NULL | Langue détectée automatiquement |
| `page_count` | SMALLINT | NOT NULL, DEFAULT 1 | Nombre de pages à traiter |
| `pages_processed` | SMALLINT | NOT NULL, DEFAULT 0 | Nombre de pages traitées |
| `progress_percentage` | SMALLINT | NOT NULL, DEFAULT 0 | Progression (0-100%) |
| `quality_score` | DECIMAL(5,2) | NULL | Score de qualité (0-100) |
| `average_confidence` | DECIMAL(5,2) | NULL | Confiance moyenne Tesseract (0-100) |
| `word_count` | INTEGER | NULL | Nombre de mots extraits |
| `character_count` | INTEGER | NULL | Nombre de caractères extraits |
| `processing_time_seconds` | INTEGER | NULL | Durée du traitement (secondes) |
| `error_message` | TEXT | NULL | Message d'erreur (si échec) |
| `error_type` | VARCHAR(50) | NULL | Type d'erreur : `unsupported_format`, `file_not_found`, `tesseract_error`, `timeout` |
| `retry_count` | SMALLINT | NOT NULL, DEFAULT 0 | Nombre de tentatives |
| `celery_task_id` | VARCHAR(100) | NULL | ID de la tâche Celery |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `started_at` | TIMESTAMP | NULL | Début du traitement |
| `completed_at` | TIMESTAMP | NULL | Fin du traitement |
| `created_by_id` | UUID | FK → users.id, NULL | Qui a déclenché (NULL si automatique) |

**Index :**
- `idx_ocr_tasks_document_id` sur `document_id`
- `idx_ocr_tasks_status` sur `status`

---

### 6.2 Modèle `OcrResult` (Résultat OCR)

**Table :** `ocr_results`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `task_id` | UUID | FK → ocr_tasks.id, NOT NULL | Tâche OCR associée |
| `document_id` | UUID | FK → documents.id, NOT NULL, INDEX | Document |
| `page_number` | SMALLINT | NOT NULL | Numéro de page (1, 2, 3...) |
| `extracted_text` | TEXT | NULL | Texte brut extrait |
| `extracted_text_corrected` | TEXT | NULL | Texte corrigé manuellement |
| `is_manually_corrected` | BOOLEAN | NOT NULL, DEFAULT FALSE | Corrigé manuellement |
| `corrected_by_id` | UUID | FK → users.id, NULL | Qui a corrigé |
| `corrected_at` | TIMESTAMP | NULL | Date de correction |
| `word_data` | JSONB | NULL | Données détaillées par mot (position, confiance) |
| `metadata_extracted` | JSONB | NULL | Métadonnées détectées (dates, montants, emails) |
| `quality_score` | DECIMAL(5,2) | NULL | Score de qualité de cette page |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |

**Contrainte :** `UNIQUE (document_id, page_number)`

**Index :**
- `idx_ocr_results_document_id` sur `document_id`
- `idx_ocr_results_task_id` sur `task_id`
- Full-text index sur `extracted_text` pour la recherche

**Exemple de `word_data` JSONB :**
```json
{
  "words": [
    {
      "text": "Contrat",
      "confidence": 96.5,
      "bbox": {"x": 120, "y": 80, "width": 85, "height": 24}
    },
    {
      "text": "de",
      "confidence": 99.2,
      "bbox": {"x": 210, "y": 80, "width": 20, "height": 24}
    }
  ]
}
```

---

### 6.3 Modèle `ScannerConfiguration` (Configuration scanner — desktop)

**Table :** `scanner_configurations`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `user_id` | UUID | FK → users.id, NOT NULL | Utilisateur propriétaire |
| `scanner_name` | VARCHAR(200) | NOT NULL | Nom du scanner |
| `scanner_id` | VARCHAR(200) | NOT NULL | ID unique du scanner (fourni par le driver) |
| `default_resolution_dpi` | SMALLINT | NOT NULL, DEFAULT 300 | Résolution par défaut |
| `default_color_mode` | VARCHAR(20) | NOT NULL, DEFAULT 'color' | Mode couleur par défaut : `color`, `grayscale`, `bw` |
| `default_output_format` | VARCHAR(10) | NOT NULL, DEFAULT 'pdf' | Format par défaut : `pdf`, `jpeg`, `png`, `tiff` |
| `supports_duplex` | BOOLEAN | NOT NULL, DEFAULT FALSE | Supporte le recto-verso |
| `is_default` | BOOLEAN | NOT NULL, DEFAULT FALSE | Scanner par défaut pour cet utilisateur |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |
| `updated_at` | TIMESTAMP | NOT NULL, AUTO | Date de mise à jour |

**Index :**
- `idx_scanner_configurations_user_id` sur `user_id`

---

### 6.4 Modèle `OcrLanguage` (Langue OCR supportée)

**Table :** `ocr_languages`

| Champ | Type | Contraintes | Description |
|---|---|---|---|
| `id` | UUID v4 | PK, NOT NULL | Identifiant unique |
| `code` | VARCHAR(10) | UNIQUE, NOT NULL | Code ISO 639-3 (fra, eng, spa...) |
| `name` | VARCHAR(100) | NOT NULL | Nom de la langue |
| `tesseract_data_installed` | BOOLEAN | NOT NULL, DEFAULT FALSE | Pack de données Tesseract installé |
| `is_enabled` | BOOLEAN | NOT NULL, DEFAULT TRUE | Langue activée |
| `created_at` | TIMESTAMP | NOT NULL, AUTO | Date de création |

**Langues prédéfinies :**
| Code | Nom | Installé par défaut |
|---|---|---|
| `fra` | Français | ✅ |
| `eng` | Anglais | ✅ |
| `spa` | Espagnol | ✅ |
| `deu` | Allemand | ❌ |
| `ara` | Arabe | ❌ |

---

## 7. Matrice des permissions du module

| Permission code | Super Admin | Admin | Archiviste | Responsable | Agent | Auditeur |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `ocr.scan_document` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `ocr.trigger_manual` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `ocr.view_result` | ✅ | ✅ | ✅ | ✅ | ✅ (filtré) | ✅ (filtré) |
| `ocr.correct_text` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `ocr.validate_result` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `ocr.retry_failed` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `ocr.view_errors` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `ocr.configure_languages` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `ocr.view_stats` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `ocr.export_text` | ✅ | ✅ | ✅ | ✅ | ✅ (filtré) | ✅ (filtré) |

---

## 8. Séquences détaillées des flux principaux

### 8.1 Séquence — Scan et OCR automatique (Desktop Tauri)

```
Utilisateur         Tauri Desktop          API Django            Celery Worker         Tesseract
    |                     |                     |                       |                       |
    |-- place document -->|                     |                       |                       |
    |   dans scanner       |                     |                       |                       |
    |-- clique "Scanner" ->|                     |                       |                       |
    |                     |-- détecte scanner -->|                      |                       |
    |<-- config scanner ---|                     |                       |                       |
    |-- configure params -->|                    |                       |                       |
    |-- clique "Scan" ----->|                    |                       |                       |
    |                     |-- appel natif ------>|                      |                       |
    |                     |   TWAIN/SANE         |                       |                       |
    |                     |<-- image scannée ----|                      |                       |
    |<-- preview ----------|                     |                       |                       |
    |-- valide ------------>|                    |                       |                       |
    |                     |-- POST /documents/ ->|                       |                       |
    |                     |   (multipart upload) |-- INSERT document ---->|                      |
    |                     |                     |-- créer tâche OCR ----------------------->|
    |                     |<-- 201 + doc_id -----|                       |                       |
    |<-- notification -----|                     |                       |                       |
    |   "OCR en cours"     |                     |                       |-- démarre OCR ------->|
    |                     |                     |                       |<-- prétraitement ------|
    |                     |                     |                       |-- lancer Tesseract -->|
    |                     |                     |                       |<-- texte extrait ------|
    |                     |                     |                       |-- calculer qualité     |
    |                     |                     |                       |-- sauvegarder résultat |
    |                     |                     |                       |-- indexer (MODULE 09)  |
    |<-- notification -----|<-- webhook ---------|-----------------------|                       |
    |   "OCR terminé"      |                     |                       |                       |
```

### 8.2 Séquence — Correction manuelle du texte OCR

```
Archiviste          Frontend React         API Django            Base de données
    |                     |                     |                       |
    |-- consulte doc ----->|                     |                       |
    |-- onglet "OCR" ----->|                    |                       |
    |                     |-- GET /ocr/ -------->|                      |
    |                     |   documents/{id}/    |-- SELECT ocr_result ->|
    |                     |   results/           |<-- texte + word_data-|
    |<-- interface --------|                     |                       |
    |   split screen       |                     |                       |
    |   image | texte      |                     |                       |
    |                       |                     |                       |
    |-- corrige texte ----->|                    |                       |
    |-- sauvegarde -------->|                    |                       |
    |                     |-- PATCH /ocr/ ------>|                      |
    |                     |   results/{id}/      |-- UPDATE result ----->|
    |                     |   correct/           |   extracted_text_     |
    |                     |                     |   corrected           |
    |                     |                     |   is_manually_        |
    |                     |                     |   corrected = TRUE    |
    |                     |                     |   quality_score = 100 |
    |                     |                     |-- réindexer (MODULE 9)|
    |                     |                     |-- INSERT audit ------->|
    |                     |<-- 200 OK -----------|                       |
    |<-- confirmation -----|                     |                       |
```

---

## 9. Endpoints API du module

### 9.1 Tâches OCR

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/ocr/tasks/` | Liste des tâches OCR | Oui + Permission |
| POST | `/api/v1/ocr/tasks/` | Créer tâche OCR manuelle | Oui + Permission |
| GET | `/api/v1/ocr/tasks/{id}/` | Détail d'une tâche | Oui + Permission |
| POST | `/api/v1/ocr/tasks/{id}/retry/` | Relancer après échec | Oui + Permission |
| POST | `/api/v1/ocr/tasks/{id}/cancel/` | Annuler une tâche | Oui + Permission |
| GET | `/api/v1/ocr/documents/{doc_id}/task/` | Tâche OCR d'un document | Oui + Permission |

### 9.2 Résultats OCR

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/ocr/documents/{doc_id}/results/` | Résultats OCR (toutes pages) | Oui + Permission |
| GET | `/api/v1/ocr/results/{id}/` | Détail résultat page | Oui + Permission |
| PATCH | `/api/v1/ocr/results/{id}/correct/` | Corriger texte | Oui + Permission |
| POST | `/api/v1/ocr/results/{id}/validate/` | Valider résultat | Oui + Permission |
| GET | `/api/v1/ocr/documents/{doc_id}/export-text/` | Exporter texte (TXT) | Oui + Permission |

### 9.3 Configuration (Admin)

| Méthode | URL | Description | Auth |
|---|---|---|---|
| GET | `/api/v1/ocr/languages/` | Langues supportées | Oui |
| PATCH | `/api/v1/ocr/languages/{id}/` | Activer/désactiver langue | Oui + Admin |
| GET | `/api/v1/ocr/stats/` | Statistiques OCR globales | Oui + Permission |
| GET | `/api/v1/ocr/stats/quality-distribution/` | Répartition par qualité | Oui + Permission |

### 9.4 Scanners (Desktop uniquement — endpoints locaux Tauri)

Ces endpoints sont exposés par le serveur local Tauri, pas par le backend Django.

| Méthode | URL | Description |
|---|---|---|
| GET | `tauri://localhost/scanners/list` | Détecter scanners |
| POST | `tauri://localhost/scanners/preview` | Scan preview |
| POST | `tauri://localhost/scanners/scan` | Scan définitif |

---

## 10. Règles métier et optimisations

### 10.1 Stratégie de retry en cas d'échec

Si une tâche OCR échoue, le système applique une stratégie de retry exponentielle :
- 1ère tentative : immédiate
- 2ème tentative : après 5 minutes
- 3ème tentative : après 30 minutes
- Abandon après 3 échecs → notification à l'administrateur

---

### 10.2 Optimisation des performances OCR

**Pour les PDF multi-pages volumineux :**
- Traitement parallèle des pages (jusqu'à 4 pages simultanément)
- Chaque page est traitée dans un worker Celery séparé
- La progression globale est mise à jour en temps réel

**Cache des images prétraitées :**
- Les images prétraitées sont conservées 24h dans `/tmp/ocr-cache/`
- En cas de retry, le prétraitement n'est pas refait

---

### 10.3 Règle de priorisation des tâches OCR

Les tâches OCR sont priorisées selon :
1. Documents marqués "Urgent" (MODULE 06)
2. Documents déclenchés manuellement par un archiviste
3. Documents créés automatiquement (ordre chronologique)

---

## 11. Gestion des erreurs OCR

### 11.1 Types d'erreurs

| Code erreur | Description | Action automatique |
|---|---|---|
| `UNSUPPORTED_FORMAT` | Format de fichier non supporté | Notification à l'utilisateur |
| `FILE_NOT_FOUND` | Fichier physique introuvable | Alerte admin + vérification intégrité |
| `TESSERACT_ERROR` | Erreur interne Tesseract | Retry automatique |
| `TIMEOUT` | Traitement trop long (>10 min/page) | Annulation + notification |
| `LOW_QUALITY_IMAGE` | Image trop floue ou dégradée | Marquage qualité faible |
| `LANGUAGE_NOT_INSTALLED` | Pack de langue manquant | Notification admin |
| `OUT_OF_MEMORY` | Mémoire insuffisante | Retry avec paramètres allégés |

---

## 12. Tâches Celery du module

| Tâche | Fréquence | Description |
|---|---|---|
| `ocr.process_task` | On-demand | Traiter une tâche OCR complète |
| `ocr.process_page` | On-demand | Traiter une page spécifique (parallélisation) |
| `ocr.retry_failed_tasks` | Horaire (toutes les 30 min) | Relancer les tâches en échec selon stratégie |
| `ocr.cleanup_temp_files` | Quotidien (2h) | Supprimer les fichiers temp OCR de plus de 24h |
| `ocr.update_statistics` | Quotidien (1h) | Recalculer les statistiques OCR globales |

---

## 13. Événements journalisés (MODULE 11 — Audit)

| Événement | Niveau | Détails |
|---|---|---|
| Tâche OCR créée | INFO | document_id, task_id, created_by |
| OCR démarré | INFO | task_id, page_count, language |
| OCR terminé avec succès | INFO | task_id, quality_score, processing_time |
| OCR échoué | WARNING | task_id, error_type, error_message |
| Texte OCR corrigé | INFO | document_id, corrected_by, page_number |
| Résultat OCR validé | INFO | document_id, validated_by |
| Tâche OCR annulée | WARNING | task_id, cancelled_by |

---

## 14. Notifications générées (MODULE 12)

| Déclencheur | Destinataire | Canal | Message |
|---|---|---|
| OCR terminé avec succès | Créateur du document | In-app | OCR terminé : [Document] (qualité : [X]%) |
| OCR échoué | Créateur + Archivistes | In-app | Échec OCR : [Document] |
| Qualité OCR faible (<50%) | Archivistes | In-app | Révision requise : [Document] (qualité : [X]%) |
| Texte OCR corrigé | Propriétaire | In-app | Le texte OCR de [Document] a été corrigé |
| Pack langue manquant | Admins | Email | Le pack langue [X] est requis mais non installé |

---

*Fin du MODULE 07 — Numérisation, OCR & Intégration Scanner*  
*Prochain module : MODULE 08 — Workflow de validation documentaire*
