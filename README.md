# 🎨 Charte Graphique — Système d'Archivage Numérique

> Design system complet pour une application d'archivage documentaire de l'administration publique.

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](https://github.com)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-production--ready-success.svg)](https://github.com)

---

## 📖 Vue d'ensemble

Cette charte graphique définit l'identité visuelle complète d'un système d'archivage numérique destiné à l'administration publique. Elle couvre tous les aspects du design, de la couleur à l'animation, en passant par les composants UI et l'accessibilité.

### 🎯 Philosophie

- **Précise** : Chaque pixel a une raison d'être
- **Discrète** : L'interface s'efface devant le contenu
- **Fiable** : Cohérence absolue, partout, toujours

---

## 📚 Structure du projet

```
.
├── charte graphique/
│   ├── index.html                                    # Page d'accueil de la charte
│   ├── CG-01-Identite-Palette.html                  # Identité & couleurs
│   ├── CG-02-Systeme-Couleurs-Tokens.html           # Tokens CSS complets
│   ├── CG-03-Typographie-Echelle.html               # Typographie & échelle
│   ├── CG-04-Espacement-Grille-Layout.html          # Espacement & grille
│   ├── CG-05-Effets-Grain-Verre-Ombres-Bordures.html # Effets visuels
│   ├── CG-06-Composants-Boutons-Inputs-Badges-Cards.html # Composants UI
│   ├── CG-07-Navigation-Sidebar-Topbar-Tabs.html    # Navigation
│   ├── CG-08-Feedback-Toasts-Modals-Etats.html      # Feedback utilisateur
│   ├── CG-09-Graphiques-DataViz.html                # Visualisation de données
│   ├── CG-10-Motion-Design-Animations.html          # Animations
│   ├── CG-11-Accessibilite-RGAA.html                # Accessibilité RGAA
│   └── CG-12-Tokens-CSS-Complets-Guide.html         # Guide d'implémentation
│
├── cahier de charge/
│   ├── MODULE_01_Architecture_Generale.md
│   ├── MODULE_02_Authentification_Utilisateurs_Roles_Permissions.md
│   ├── MODULE_03_Taxonomie_Documentaire.md
│   ├── MODULE_04_Cycle_de_vie_Politique_de_conservation.md
│   ├── MODULE_05_Confidentialite_Classification_Securite.md
│   ├── MODULE_06_Gestion_Documents_Metadonnees_Partie1.md
│   ├── MODULE_06_Gestion_Documents_Metadonnees_Partie2.md
│   ├── MODULE_07_Numerisation_OCR_Integration_Scanner.md
│   ├── MODULE_08_Workflow_Validation_Documentaire.md
│   ├── MODULE_09_Moteur_Recherche_Indexation.md
│   ├── MODULE_10_Intelligence_Artificielle_Automatisation.md
│   ├── MODULE_11_Audit_Tracabilite_Journalisation.md
│   ├── MODULE_12_Notifications_Alertes_Intelligentes.md
│   ├── MODULE_13_Demandes_Acces_Partage_Controle.md
│   ├── MODULE_14_Tableaux_de_Bord_Reporting.md
│   ├── MODULE_15_Administration_Configuration.md
│   ├── MODULE_16_Sauvegarde_Restauration_Resilience.md
│   ├── MODULE_17_Desktop_Tauri_Synchronisation_Offline.md
│   └── MODULE_18_API_Interoperabilite_Integrations.md
│
└── README.md
```

---

## 🎨 Modules de la charte graphique

### 🔷 Fondations

| Module | Description | Statut |
|--------|-------------|--------|
| **CG-01** | Identité visuelle & palette de couleurs | ✅ Complet |
| **CG-02** | Système de couleurs & tokens CSS | ✅ Complet |
| **CG-03** | Typographie & échelle | ✅ Complet |
| **CG-04** | Espacement, grille & layout | ✅ Complet |
| **CG-05** | Effets : grain, verre, ombres, bordures | ✅ Complet |

### 🔷 Composants

| Module | Description | Statut |
|--------|-------------|--------|
| **CG-06** | Boutons, inputs, badges, cards | ✅ Complet |
| **CG-07** | Navigation : sidebar, topbar, tabs | ✅ Complet |
| **CG-08** | Feedback : toasts, modals, états | ✅ Complet |
| **CG-09** | Graphiques & data visualization | ✅ Complet |

### 🔷 Patterns & Implémentation

| Module | Description | Statut |
|--------|-------------|--------|
| **CG-10** | Motion design & animations | ✅ Complet |
| **CG-11** | Accessibilité RGAA | ✅ Complet |
| **CG-12** | Tokens CSS complets & guide d'implémentation | ✅ Complet |

---

## 🎯 Caractéristiques principales

### ✨ Design System

- **Palette d'accent** : Sage Slate (#179E8E) — 10 nuances calibrées
- **Gris froids** : Base bleutée inspirée de GitHub
- **Typographie** : Plus Jakarta Sans (UI) + DM Mono (code)
- **Espacement** : Système base-4 (multiples de 4px)
- **Grille** : 12 colonnes responsive
- **Effets** : Grain SVG, glassmorphisme, ombres Z-depth

### 🌓 Thème clair/sombre

- Mode dark par défaut
- Mode light avec tokens sémantiques
- Transitions fluides (200ms)
- Bouton de changement de thème sur tous les modules

### 🧭 Navigation

- Breadcrumb avec fil d'Ariane
- Menu dropdown avec liste complète des modules
- Footer avec liens précédent/suivant
- Raccourcis clavier : `←` `→` `i`

### ♿ Accessibilité

- Conforme RGAA 4.1
- Contraste minimum 4.5:1 (texte) / 3:1 (UI)
- Navigation au clavier complète
- Attributs ARIA appropriés
- Focus visible sur tous les éléments interactifs

---

## 🚀 Démarrage rapide

### Visualiser la charte

1. Ouvrir `charte graphique/index.html` dans un navigateur
2. Naviguer entre les modules via le menu ou les liens
3. Tester le changement de thème avec le bouton "Thème"

### Intégrer dans un projet

1. Copier le fichier `CG-12-Tokens-CSS-Complets-Guide.html`
2. Extraire le bloc CSS des tokens (section `globals.css`)
3. Copier dans votre fichier `globals.css` ou équivalent
4. Utiliser les tokens CSS dans vos composants

```css
/* Exemple d'utilisation */
.button {
  background: var(--accent-500);
  color: var(--text-primary);
  padding: var(--space-3) var(--space-4);
  border-radius: var(--radius-sm);
  transition: all var(--dur-fast) var(--ease-spring);
}
```

---

## 📋 Cahier des charges

Le dossier `cahier de charge/` contient 18 modules détaillant les spécifications fonctionnelles complètes du système d'archivage :

### 🏗️ Architecture & Sécurité
- Architecture générale
- Authentification & permissions
- Confidentialité & classification

### 📄 Gestion documentaire
- Taxonomie documentaire
- Cycle de vie & conservation
- Métadonnées & indexation
- Numérisation & OCR

### 🔄 Workflows & Automatisation
- Workflow de validation
- Intelligence artificielle
- Notifications intelligentes

### 📊 Analyse & Administration
- Moteur de recherche
- Tableaux de bord & reporting
- Audit & traçabilité
- Administration & configuration

### 🔌 Intégration & Résilience
- API & interopérabilité
- Application desktop (Tauri)
- Sauvegarde & restauration

---

## 🛠️ Technologies recommandées

### Frontend
- **Framework** : React / Vue.js / Svelte
- **Styling** : CSS Variables (tokens) + Tailwind CSS (optionnel)
- **Animations** : Framer Motion / GSAP
- **Charts** : Chart.js / Recharts / D3.js

### Backend
- **API** : Node.js (Express) / Python (FastAPI) / Go
- **Base de données** : PostgreSQL + Elasticsearch
- **Stockage** : S3-compatible (MinIO)
- **OCR** : Tesseract / Google Cloud Vision

### Desktop
- **Framework** : Tauri (Rust + Web)
- **Sync** : WebSocket + IndexedDB

---

## 📐 Conventions de code

### CSS
- Utiliser les tokens CSS (`var(--token-name)`)
- Jamais de valeurs en dur (couleurs, espacements)
- Nommage BEM pour les classes custom
- Mobile-first responsive design

### Accessibilité
- Toujours inclure `aria-label` sur les boutons icônes
- Contraste minimum respecté
- Navigation au clavier testée
- Lecteur d'écran compatible

### Performance
- Lazy loading des images
- Code splitting des composants
- Optimisation des animations (GPU)
- Compression des assets

---

## 📄 Licence

Ce projet est sous licence MIT. Voir le fichier [LICENSE](LICENSE) pour plus de détails.

---

## 👥 Contribution

Les contributions sont les bienvenues ! Pour contribuer :

1. Fork le projet
2. Créer une branche (`git checkout -b feature/amelioration`)
3. Commit les changements (`git commit -m 'Ajout d'une fonctionnalité'`)
4. Push vers la branche (`git push origin feature/amelioration`)
5. Ouvrir une Pull Request

---



---

## 🙏 Remerciements

Inspirations et références :
- **Linear** — Système d'espacement et animations
- **GitHub** — Palette de gris froids
- **Notion** — Hiérarchie typographique
- **Tailwind CSS** — Tokens et conventions

---

<div align="center">

**Charte Graphique v1.0.0** — Système d'Archivage Numérique


</div>
