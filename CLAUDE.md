## Règles

1. **Régle d'or : à chaque fois que je te fais un retour / précise ma vision pour ce projet, update ce document CLAUDE.MD**
2. Tu n'as pas la permission de modifier / supprimer des documents de ce dossier. Lecture seule.
3. À chaque réponse, relis mentalement toutes les features du dossier `Features/` pour faire les meilleures suggestions d'ajouts / d'amélioration.
4. Challenger Julien pour que les specs soient précises — insister même quand elle contredit.

## Informations complémentaires

- Les aspects monétaires sont volontairement ignorés dans les specs pour l'instant.
- Les features en *italique commenté* dans les docs = abandonnées, ne pas les mentionner dans les specs.
- Si quelque chose de techniquement similaire existe hors éducation, le signaler.
- Tu as accès à un tool pour lire les fichiers Excalidraw visuellement — utilise-le systématiquement.

---

## Contexte du projet : Prométhium (PM)

### Vision

**Prométhium** est une _everything app_ pour étudiants en prépa scientifique. Inspirée d'Obsidian (modulaire, tout au clavier) mais construite pour les usages spécifiques des études scientifiques.

**4 problèmes résolus :**
1. Difficulté à coopérer sans friction (compétition + manque de temps en prépa)
2. Manque d'optimisation data-driven des parcours d'étude
3. Rétention des ressources pédagogiques par les établissements (problème de théorie des jeux)
4. Fragmentation des workflows (cours papier, PDFs, Anki, échanges...)

**Métriques cibles :** quantité d'exercices travaillés, interactions productives entre pairs, temps d'organisation réduit.

**Audiences :** élèves de prépa scientifique (cible principale), professeurs, établissements.

---

### Principe fondamental : tout est un fichier + fork network

**Abandon de l'abstraction "Node".** Tout contenu est un fichier dans un repo git. Les métadonnées vont dans le frontmatter. Le graphe émerge des liens entre fichiers.

**Abandon du modèle "overlay repos séparés".** Remplacé par un **fork network** : un seul repo git partagé (même object database), des namespaces distincts par peer — inspiré de Radicle + GitHub fork networks. Voir `Core Technical Features/Modèle de données — Inspiration Radicle.md` pour le design complet.

Les types de fichiers principaux :
- **Ressource** (`.md` avec frontmatter `type: td | lesson | exercise | course`) — contenu du prof
- **Anchor** (COB git — Collaborative Object) — overlay de l'étudiant/guilde sur une ressource, stocké comme un DAG de commits dans `refs/cobs/pm.promethium.anchor/<id>`
- **Link COB** (`refs/cobs/pm.promethium.link/<id>`) — lien typé entre deux ressources sans position
- **Binaires** (`.pdf`, `.mp4`, `.png`) — avec sidecar `.md` obligatoire contenant `binary: ./td1.pdf`

**Insight central : toute feature sociale est un Anchor COB.** Il n'y a pas de système social "en plus".

Exemple de ressource :
```markdown
---
type: td
title: TD Algèbre Linéaire 1
tags: [algèbre, L1]
---
```

Exemple d'anchor COB (commit JSON dans le DAG git) :
```json
{
  "op": "create",
  "anchor_type": "note",
  "source": {
    "file": "td1.md",
    "commit": "sha1:b3e7f1a2c9...",
    "position": {
      "kind": "text",
      "text_snapshot": "Soit E un espace vectoriel",
      "char_start": 142, "char_end": 189
    }
  },
  "target": { "file": "notes/td1-notes.md", "block_id": "bloc-42" },
  "layer": "personal",
  "author": "did:key:z6MkAlice..."
}
```

---

### Architecture décentralisée — Fork Network

Chaque dossier pédagogique = **un repo git partagé** avec une object database commune et des **namespaces par peer** :

```
.promethium/storage/<RID>/
├── objects/pack/objects.pack     ← ODB commune (zéro duplication)
└── refs/namespaces/
    ├── <NID-prof>/refs/heads/main    ← contenu officiel
    ├── <NID-alice>/refs/             ← fork élève Alice
    │   ├── heads/main
    │   └── cobs/pm.promethium.anchor/<id>   ← ses notes/anchors
    └── <NID-guilde>/refs/            ← fork de la guilde
        └── cobs/pm.promethium.anchor/<id>   ← annotations collectives
```

**Stale detection** : propriété **dérivée**, jamais stockée.
```
stale = (anchor.source.commit ≠ HEAD de refs/namespaces/<prof>/refs/heads/main)
```

**Reverse lookup** : seed/AppView indexe `source_file → [anchor COB ids]`.

**Chat** : relay event stream hors git (inchangé). `room.md` dans namespace guilde = pointeur vers relay.

---

### Anchor System & Mapping

Infrastructure centrale reliant des positions précises entre documents hétérogènes (PDF, OCR, vidéo, texte riche).

- **3 modes de vue :** split-screen (travail), overlays (lecture), marginalia (révision)
- **Layers :** personal / guild — implémentés via namespaces (toggle = toggle de visibilité namespace)
- **Stale-ité :** propriété dérivée calculée à la volée (commit anchor vs HEAD prof)
- **Clusters** : anchors superposés fusionnés en un marqueur cluster avec badge de compte

Types d'anchor : `note`, `definition`, `anki`, `forum`, `guild_annotation`, `signet`, `erratum`, `intent`

Types de link COB (sans position) : `prerequisite`, `inferred_prerequisite`, `references`, `corrects`, `follows`, `generalizes`, `specializes`

Types de position : `text` (char offsets), `pdf` (bbox), `ocr` (texte extrait), `video` (timestamps), `null` (fichier entier)

---

### Features → primitives (résumé)

| Feature | Implémentation |
|---|---|
| Notes personnelles | Anchor COB `note`, layer personal |
| Signet | Anchor COB `signet`, position null si fichier entier |
| Notes de guilde | Anchor COB `guild_annotation`, namespace guilde |
| Erratum | Anchor COB `erratum` + Patch COB vers namespace prof |
| Questions / mini-forum | Anchor COB `forum` + Thread COB |
| Anki | Anchor COB `anki` + fichier `.anki.md` dans namespace élève |
| Solutions | Fichier `solution.md` dans namespace élève → Patch COB vers guilde/prof |
| Intentions | Intent COB `pm.promethium.intent` |
| Tags hiérarchiques collaboratifs | Tag COB `pm.promethium.tag` + consensus threshold |
| Parcours recommandé | Link COB `pm.promethium.link` `inferred_prerequisite` + service ML |
| Time tracker | Event stream hors git (service analytics) |
| Chat de guilde | Relay event stream (hors git) + `room.md` pointeur |

---

### Questions architecturales — État de résolution

**Q1 — Fichiers exotiques (PDF, vidéo)** : **Résolu** — sidecar `.md` avec `binary: ./td1.pdf` + `ocr_commit: sha1:...`. Anchor PDF = `position.kind: "pdf"` avec bbox. Anchor OCR = `position.kind: "ocr"`.

**Q2 — Format des métadonnées d'anchor** : **Résolu** — JSON dans les COB commits git (pas de frontmatter YAML).

**Q3 — URI cross-repo** : **Résolu** — namespace + path dans le même repo. `refs/namespaces/<NID-prof>/refs/heads/main:td1.md`.

**Q4 — DisplayTemplate** : **Proposé** — convention de frontmatter par `type` + config par dossier `.promethium/display.toml`.

---

### Git interne

Chaque document/dossier est versionnable. Profs font des commits, élèves font des PRs (erratum, exercices d'oraux...). Export `.promethium` pour conserver les liens hors plateforme.

---

### Architecture technique

- **Frontend** : Electron + React (SPA), pilotable au clavier, modulaire
- **Backend** : microservices — analytics, recommandation ML, consensus de tags, événements (time tracker), relay chat
- **Stack data** : repos git par utilisateur/guilde/prof, relay event streams pour le chat, AppView/index pour le reverse lookup des overlays publics

---

### Fichiers clés

- `Specs/Promethium - Spécification Fonctionnelles et Techniques (SFT).md` — spec principale
- `Core Technical Features/Mapping.md` — spec Anchor System complète
- `Core Technical Features/Rapport — Implémentation des features sociales via Node + Anchor.md` — mapping features → primitives
- `Core Technical Features/Système de tags.md` — tags collaboratifs
- `Features/` — toutes les features individuelles
- `Specs/` — specs détaillées et interfaces Excalidraw

---
