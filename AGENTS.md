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

### Principe fondamental : tout est un fichier

**Abandon de l'abstraction "Node".** Tout contenu est un fichier dans un repo git. Les métadonnées vont dans le frontmatter. Le graphe émerge des liens entre fichiers.

Les types de fichiers principaux :
- **Ressource** (`.md` avec frontmatter `type: td | lesson | exercise | course`) — contenu du prof
- **Anchor** (`.anchor.md` ou `.anchor.json`) — overlay de l'étudiant/guilde sur une ressource
- **Link** (`.link.md`) — lien typé entre deux ressources sans position (prérequis, références...)
- **Binaires** (`.pdf`, `.mp4`, `.png`) — ressources non-markdown (voir Q1 non résolue)

**Insight central : toute feature sociale est un Anchor.** Il n'y a pas de système social "en plus".

Exemple de ressource :
```markdown
---
type: td
title: TD Algèbre Linéaire 1
prerequisites:
  - ../cours/algebre-lineaire-1.md
tags: [algèbre, L1]
---
```

Exemple d'anchor (overlay étudiant sur une ressource prof) :
```markdown
---
type: anchor
anchor_type: note | definition | anki | forum | guild_annotation | signet | erratum
source:
  file: <URI vers le fichier prof — format non résolu, voir Q3>
  commit: sha1:b3e7f1
  position:
    kind: pdf | text | video | ocr
    text_snapshot: "Soit E un espace vectoriel"
target:
  file: notes/td1-notes.md
  block_id: bloc-42
layer: personal | guild
author: did:key:pubkey
---
```

---

### Architecture décentralisée

Chaque utilisateur et chaque prof a son **propre repo git**. Les overlays (anchors, links communautaires) vivent dans les repos de leurs auteurs — ils ne modifient jamais le repo source.

```
Repo du prof      Repo étudiant (overlay)    Repo guilde (overlay)
  td1.md    ←──────  .anchor/td1-note.md  ←── .anchor/td1-guild.md
  cours1.md          .link/prereq.md
```

**Stale detection** : chaque anchor stocke le `commit` du fichier source au moment de sa création. Quand le client détecte un nouveau commit dans le repo du prof, il compare localement — sans serveur central.

**Reverse lookup** (trouver tous les overlays sur un fichier source) :
- Overlays personnels : triviaux, repo local
- Overlays de guilde : repos des guildes dont on est membre
- Overlays publics (forum, errata) : relay avec index `source_file → [anchor_ids]` ou AppView d'établissement

**Chat** : ne rentre pas dans git. Modèle séparé — event stream append-only sur un relay. Le `Room ID` est un fichier `.md` dans le repo de la guilde qui pointe vers le relay. Deux usages distincts :
- **Threads forum ancrés** (semi-persistants, liés à un doc) → fichier anchor dans le repo
- **Chat temps réel / DMs** → relay event stream, hors git

---

### Anchor System & Mapping

Infrastructure centrale reliant des positions précises entre documents hétérogènes (PDF, OCR, vidéo, texte riche).

- **3 modes de vue :** split-screen (travail), overlays (lecture), marginalia (révision)
- **Layers :** personal / guild, togglables individuellement
- **Stale-ité :** quand le prof fait un commit sur la source, les anchors peuvent ne plus correspondre → interface de révision diff côté client
- **Clusters** : anchors superposés fusionnés en un marqueur cluster avec badge de compte

Types d'anchor : `note`, `definition`, `anki`, `forum`, `guild_annotation`, `signet`, `erratum`, `intent`, `inferred_prerequisite`

Types de link (sans position) : `prerequisite`, `references`, `corrects`, `follows`, `generalizes`, `specializes`, `inferred_prerequisite`

---

### Features → primitives (résumé)

| Feature | Implémentation |
|---|---|
| Notes de guilde | `.anchor.md` `guild_annotation`, layer `guild` |
| Erratum | `.anchor.md` `erratum` + rendu conditionnel |
| Questions / mini-forum | `.anchor.md` `forum` + fichier thread |
| Signet | `.anchor.md` `signet`, layer `personal` |
| Solutions | fichier `solution.md`, enfant du dossier exercice |
| Tags hiérarchiques collaboratifs | fichiers `tag.md` versionnés git + service consensus |
| Parcours recommandé | `.link.md` `inferred_prerequisite` + service ML |
| Time tracker | service analytics séparé |
| Intention de faire un exo | fichier `intent.md` + `.anchor.md` `intent` |
| Chat de guilde | relay event stream (hors git) + `room.md` pointeur |

---

### Questions architecturales ouvertes

**Q1 — Fichiers exotiques (PDF, vidéo)**
Option A (sidecar) : `td1.pdf` + `td1.md` (fiche metadata à côté) vs Option B (wrapper) : `td1.md` avec `binary: ./td1.pdf` dans le frontmatter. **Non résolu.**

**Q2 — Format des métadonnées d'anchor**
Frontmatter YAML uniforme (verbeux pour les positions complexes) vs `.anchor.json` pour les anchors avec `bbox` / coordonnées. **Non résolu.**

**Q3 — URI cross-repo**
Comment un anchor étudiant pointe-t-il vers un fichier dans le repo du prof ? Chemin relatif (intra-repo seulement), URL git (couplé hébergeur), CID IPFS (content-addressed), ou `at://did:key:prof/...` (DID-based). **Non résolu.**

**Q4 — DisplayTemplate**
Avec l'abandon de Node, qui contrôle le rendu d'un fichier ? Convention de frontmatter ? Fichier de config par dossier ? Ou concept abandonné au profit de rendus fixes par `type` ? **Non résolu.**

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
