# 🎯 Codebase Assistant v2

> **Génère des prompts enrichis depuis ton codebase local — sans API, sans agent.**  
> Colle le résultat dans Claude ou ChatGPT et obtiens des réponses précises, ancrées dans ton vrai code.

[![Python](https://img.shields.io/badge/Python-3.11%2B-blue?logo=python&logoColor=white)](https://python.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![No API Key Required](https://img.shields.io/badge/LLM%20API-not%20required-brightgreen)](#)
[![TF-IDF](https://img.shields.io/badge/Retrieval-TF--IDF%20%2B%20AST-purple)](#architecture)

---

## Pourquoi cet outil ?

Les LLMs répondent bien mieux quand on leur fournit **le bon contexte au bon moment**. Mais coller l'intégralité d'un projet dans une fenêtre de contexte, c'est :

- **trop long** (limite de tokens),
- **trop bruyant** (code non pertinent noyé dedans),
- **impossible** sur de grands projets.

**Codebase Assistant** résout ça. Il analyse ton projet, identifie les chunks de code les plus pertinents pour ta question (fonctions, classes, modules), et génère un prompt structuré prêt à coller dans ton LLM préféré — avec architecture, dépendances et scores de pertinence inclus.

**C'est un générateur de prompts, pas un agent.** Aucun appel API, aucune clé requise.

---

## ✨ Fonctionnalités

- **Retrieval sémantique TF-IDF** — bien supérieur au simple keyword matching, avec pondération IDF, tokénisation camelCase/snake_case et normalisation cosinus
- **Chunking AST intelligent** — découpe au niveau fonctions et classes (pas juste fichier entier) pour Python, JS/TS, Go, Rust
- **Graphe de dépendances** — détecte qui importe qui, identifie les hubs critiques, propage la pertinence aux fichiers voisins
- **Cache persistant** — index TF-IDF sérialisé sur disque (`.codebase_index/`), rechargé instantanément si le projet n'a pas changé
- **Interface web locale** — UI dark mode avec affichage des scores, des dépendances et copie en un clic
- **CLI complète** — sortie fichier, métadonnées JSON, focus forcé sur des fichiers spécifiques

---

## 🚀 Démarrage rapide

### Prérequis

```bash
pip install numpy
# Python 3.11+ requis (pour ast.parse amélioré)
```

### Mode CLI

```bash
# Question simple
python codebase_assistant.py --path ./mon_projet --question "Comment est gérée l'authentification ?"

# Sauvegarder le prompt dans un fichier
python codebase_assistant.py -p . -q "Architecture du système" -o prompt.txt --top-k 15

# Forcer des fichiers spécifiques dans le contexte
python codebase_assistant.py -p . -q "Pipeline de traitement" --focus src/pipeline.py src/config.py

# Forcer la reconstruction du cache
python codebase_assistant.py -p . -q "..." --rebuild

# Afficher les métadonnées de sélection
python codebase_assistant.py -p . -q "..." --json-meta
```

### Mode Web (recommandé)

```bash
python web_ui.py --path ./mon_projet
# → ouvre automatiquement http://localhost:7842
```

```bash
# Options avancées
python web_ui.py --path ./mon_projet --port 8080 --no-browser --rebuild
```

---

## 🏗️ Architecture

```
codebase_assistant.py   ← Orchestrateur principal + CLI + builder de prompt
chunker.py              ← Découpe sémantique du code (AST + regex + sliding window)
indexer.py              ← Index TF-IDF vectorisé + cache persistant
dep_graph.py            ← Graphe de dépendances inter-fichiers
web_ui.py               ← Interface web (serveur HTTP standalone, zéro dépendance)
```

### Pipeline de traitement

```
Projet local
    │
    ▼
[scan_codebase]          → liste des fichiers filtrés (ext, taille, dirs ignorés)
    │
    ├──▶ [analyze_architecture]   → arbre de modules + rôles inférés
    │
    ├──▶ [build_dependency_graph] → edges, adjacency, reverse, hubs, orphans
    │
    ├──▶ [chunk_all_files]        → chunks AST/regex/sliding-window par fichier
    │
    └──▶ [build_or_load_index]    → TF-IDF matrix (depuis cache ou reconstruction)
              │
              ▼
         [score_chunks]           → top-K chunks scorés + dep boost
              │
              ▼
         [build_enriched_prompt]  → prompt structuré Markdown prêt à coller
```

---

## 📦 Modules en détail

### `chunker.py` — Découpe sémantique

Stratégie adaptée par langage :

| Langage | Méthode | Granularité |
|---|---|---|
| Python | `ast.parse()` | Classes + fonctions top-level + code module |
| JS / TS | Regex sur `class`, `function`, `const =>` | Blocs fonctionnels |
| Go / Rust | Regex sur `func`, `fn`, `impl` | Fonctions et implémentations |
| Autres | Sliding window (80 lignes, step 60) | Blocs chevauchants |

Chaque chunk expose : `id`, `path`, `kind`, `name`, `content`, `start`, `end`, `signature`, `docstring`, `imports`.

Les chunks trop courts (< 3 lignes) sont ignorés. Les mastodontes (> 200 lignes) sont tronqués avec marqueur `[N lignes omises]`.

### `indexer.py` — Retrieval TF-IDF

- **Pondération** : chemin × 3, nom × 3, signature × 2, docstring × 2, contenu (800 chars)
- **Tokénisation** : split snake_case + camelCase + suppression stopwords code (`import`, `self`, `return`…)
- **IDF lissé** : `log((N+1)/(df+1)) + 1` — résistant aux termes omniprésents
- **Similarité cosinus** via produit matriciel NumPy sur vecteurs L2-normalisés
- **Dependency boost** : les fichiers voisins (dans le graphe) des top-5 résultats gagnent +0.15

**Cache** : index sérialisé en `pickle` dans `.codebase_index/`, invalidé automatiquement si un fichier est modifié (hash SHA-256 sur chemins + tailles + mtimes).

> 💡 Pour passer aux embeddings neuronaux, remplacez `TFIDFIndex` par `SentenceTransformer('all-MiniLM-L6-v2')` — l'interface `score_chunks` reste identique.

### `dep_graph.py` — Graphe de dépendances

Extraction des imports par langage :

| Langage | Méthode |
|---|---|
| Python | `ast.walk()` sur `Import` / `ImportFrom` |
| JS / TS | Regex `import from`, `require()`, `import()` |
| Go | Regex sur blocs `import (...)` |
| Rust | Regex `use crate::` + `mod` |

Résolution relative vers chemins projet (ignore `node_modules` et packages externes). Produit :
- `edges` / `adjacency` / `reverse` — graphe orienté complet
- `hubs` — top-10 fichiers les plus importés (fichiers critiques)
- `orphans` — fichiers isolés

### `web_ui.py` — Interface web

Serveur HTTP stdlib pure Python (zéro dépendance externe). Deux panneaux :
- **Gauche** : arbre du projet, rôles des modules, hubs de dépendances
- **Droite** : champ question, résultat du prompt, liste des chunks sélectionnés avec scores visuels et graphe de dépendances

API interne :

| Endpoint | Méthode | Description |
|---|---|---|
| `/` | GET | UI HTML embarquée |
| `/api/architecture` | GET | Vue d'ensemble du projet |
| `/api/files` | GET | Liste des fichiers scannés |
| `/api/generate` | POST | Génération du prompt (question + top_k) |

---

## ⚙️ Configuration

Dans `codebase_assistant.py` :

```python
MAX_FILE_SIZE_KB  = 200      # fichiers plus lourds ignorés
MAX_TOTAL_TOKENS  = 40_000   # limite tokens estimés dans le prompt final
TOP_K_DEFAULT     = 12       # nombre de chunks sélectionnés par défaut
```

Extensions supportées : `.py`, `.js`, `.ts`, `.jsx`, `.tsx`, `.go`, `.rs`, `.java`, `.cpp`, `.c`, `.rb`, `.php`, `.cs`, `.swift`, `.sh`, `.yaml`, `.toml`, `.json`, `.md`, `.html`, `.css`, + `Makefile`, `Dockerfile`…

Répertoires ignorés automatiquement : `__pycache__`, `.git`, `node_modules`, `.venv`, `dist`, `build`, `.next`, `.pytest_cache`…

---


---

## 🗂️ Structure du cache

```
mon_projet/
└── .codebase_index/
    ├── .gitignore      ← exclut le cache du dépôt automatiquement
    ├── index.pkl       ← index TF-IDF sérialisé (matrice + vocabulaire)
    └── meta.json       ← version, hash, stats (fichiers, chunks, vocab)
```

Le cache est invalidé automatiquement dès qu'un fichier est modifié, ajouté ou supprimé.

---


