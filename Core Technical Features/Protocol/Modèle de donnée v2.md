
> Restructuration par rapport à v1 : séparation nette entre **infrastructure** (primitives, agnostique au contenu) et **protocole client** (conventions du client PM).

**TO  DO :**
- Il faut faire un système à la Bittorrent pour gérer LFS
- Il faut un système de L2 pour des fonctionnalités chat et sociales pour ne pas avoir de latence. 

---

## Table des matières

**Partie I — Infrastructure**
1. [NID — Identité](#1-nid--identité)
2. [Repo](#2-repo)
3. [COB — Collaborative Object](#3-cob--collaborative-object)
4. [Stratégie de déploiement](#4-stratégie-de-déploiement)

**Partie II — Protocole client Prométhium**
5. [Format d'un commit COB](#5-format-dun-commit-cob)
6. [Types reconnus par le client PM](#6-types-reconnus-par-le-client-pm)
7. [Display Templates](#7-display-templates)

---

# Partie I — Infrastructure

> L'infrastructure ne connaît pas le contenu des fichiers, ni les types de COBs, ni les concepts applicatifs (note, erratum, thread...). Elle garantit uniquement les propriétés ci-dessous.

---

## 1. NID — Identité

```
NID = did:key:z6Mk<base58-pubkey>
```

- Paire de clés **Ed25519**, générée offline, sans serveur
- Un NID = un namespace = une clé privée
- Pas de compte, pas d'email, pas de mot de passe
- Deux entités distinctes ne peuvent pas partager un NID

**Ce que l'infra garantit :**
- Seul le détenteur de la clé privée peut écrire dans `refs/namespaces/<NID>/`
- Tout commit signé par un NID est vérifiable par n'importe quel peer

---

## 2. Repo

### 2.1 Structure

Un repo = **une ODB git partagée** + **un namespace par NID**.

```
<RID>/
├── objects/pack/objects.pack     ← ODB commune — zéro duplication de blobs
└── refs/
    ├── rad/id                    ← document identité du repo
    └── namespaces/
        ├── <NID-A>/refs/
        │   ├── heads/            ← branches de A
        │   └── cobs/             ← COBs de A
        ├── <NID-B>/refs/
        │   ├── heads/
        │   └── cobs/
        └── ...
```

Chaque NID écrit exclusivement dans son namespace. L'ODB est partagée — un blob présent dans un namespace est accessible à tous sans duplication.

### 2.2 RID — Repository Identity

```
RID = SHA1 du commit racine de refs/rad/id
```

Le RID est **immuable** — il identifie le repo pour toujours, indépendamment de l'évolution de son contenu ou de sa gouvernance.

### 2.3 Document identité (`refs/rad/id`)

DAG de commits sur `refs/rad/id`. Chaque commit contient un JSON :

```json
{
  "delegates": ["did:key:z6MkA...", "did:key:z6MkB..."],
  "threshold": 2,
  "payload": {}
}
```

- **`delegates`** : NIDs autorisés à faire avancer le canonical du repo
- **`threshold`** : nombre de delegates devant converger pour qu'un commit soit canonical
- **`payload`** : métadonnées libres — l'infra ne les interprète pas

Mettre à jour le document identité requiert que ≥ threshold delegates actuels signent le nouveau commit.

### 2.4 Canonicité

```
canonical(refs/heads/main) = commit C tel que
  |{ d ∈ delegates | refs/namespaces/<d>/refs/heads/main = C }| ≥ threshold
```

Calculé localement par chaque peer. Aucun coordinateur central.

**Cas simples :**
- Threshold 1, 1 delegate → ce que le delegate a poussé = canonical (repo individuel)
- Threshold 2, 3 delegates → majorité requise (repo collectif)

### 2.5 Politique d'accès

Stockée dans le payload du document identité — l'infra l'applique mais ne l'interprète pas :

```json
{
  "access": {
    "policy": "allowlist | open | institution",
    "allowlist": ["did:key:..."]
  }
}
```

### 2.6 Stale detection

Propriété **dérivée**, jamais stockée :

```
stale(ref_to_commit) = (ref_to_commit ≠ HEAD(refs/namespaces/<owner>/refs/heads/main))
```

L'infra expose les refs. Le client calcule la stale-ité à la volée.

---

## 3. COB — Collaborative Object

### 3.1 Définition

Un COB = **un ref pointant vers un DAG de commits git**, append-only, dans le namespace de son créateur.

```
refs/namespaces/<NID>/refs/cobs/<id>
                                  ↑
                    SHA1 du commit racine = identifiant stable du COB
```

**Ce que l'infra garantit :**
- Chaque commit est signé par son auteur (NID)
- L'union de deux DAGs est déterministe (CRDT natif git)
- Un COB est identifié de façon stable par le SHA de son commit racine

**Ce que l'infra ne connaît pas :**
- Le contenu des commits (JSON, texte, binaire — libre)
- La sémantique du COB (note ? PR ? tag ?)
- Les opérations valides

### 3.2 Merge de COBs

Quand deux peers se synchronisent sur le même COB :

```
1. Union des DAGs de commits
2. Topological sort → ordre causal garanti
3. Pas de conflit — chaque commit identifié par son SHA
4. Résultat identique sur tous les peers
```

### 3.3 Layers (convention de namespace)

L'infra ne connaît que des namespaces. La notion de "layer" est une convention :

| Convention | Namespace | Contrôle |
|---|---|---|
| Contenu personnel | `refs/namespaces/<NID-alice>/refs/cobs/` | Alice seule |
| Contenu collectif | Namespaces des membres d'un repo collectif | Threshold delegates |

Activer/désactiver un layer = toggle de visibilité d'un namespace dans le client.

---

## 4. Stratégie de déploiement

Le modèle de données est **identique** dans les deux phases. Seule la couche réseau change.

| | Phase 1 — Centralisé | Phase 2 — P2P |
|---|---|---|
| Hébergement du repo réseau | Serveurs PM | Local chez chaque user |
| Découverte | API REST | Gossip P2P (Noise XK) |
| Auth | Session/JWT serveur | Signature Ed25519 |
| Seed | Serveur PM unique | N seeds distribués |

**Ce qui rend la migration triviale :**
- NIDs générés offline dès Phase 1 → portables sans changement
- RIDs basés sur hash, pas sur URL serveur → stables
- COBs signés par NID → vérifiables sans serveur dès le premier jour
- Structure de namespaces identique en local et sur serveur

![[Architecture globale du réseau.excalidraw]]

![[Schéma fonctionnel du relay.excalidraw]]

---

# Partie II — Protocole client Prométhium

> Ce qui suit est une **convention du client officiel PM**. Un client tiers peut implémenter d'autres types sur la même infrastructure sans aucune modification côté infra.

---

## 5. Format d'un commit COB

Enveloppe JSON commune à tous les commits COB du client PM :

```json
{
  "version": "1",
  "type": "pm.promethium.<type>",
  "op": "<operation>",
  "author": "did:key:z6Mk...",
  "timestamp": "2026-03-28T14:32:00Z",
  "payload": {}
}
```

- **`type`** : namespace + nom du type, défini par convention client
- **`op`** : opération dans le cycle de vie du COB
- **`payload`** : contenu spécifique au type et à l'opération

Le `type` n'apparaît **pas** dans le chemin du ref — il est dans le commit. Cela permet à un même ref d'être interprété différemment par des clients différents.

---

## 6. Types reconnus par le client PM

### 6.1 `pm.promethium.anchor` — Overlay positionné

Relie une position dans un document source à un contenu cible.

**Sous-types (`anchor_type`) :**

| Type | Usage |
|---|---|
| `note` | Note personnelle sur un passage |
| `signet` | Signet sur un passage ou fichier entier |
| `definition` | Définition inline d'un terme |
| `anki` | Carte Anki liée à un passage |
| `forum` | Question/thread ancré sur un passage |
| `erratum` | Erreur signalée dans le contenu |
| `guild_annotation` | Annotation collective |
| `intent` | Intention déclarée sur un exercice |

**Opérations :** `create`, `update`, `update_position`, `react`, `resolve`, `delete`

**Schéma — `create` :**

```json
{
  "type": "pm.promethium.anchor",
  "op": "create",
  "author": "did:key:z6MkAlice...",
  "timestamp": "2026-03-28T14:32:00Z",
  "payload": {
    "anchor_type": "note",
    "source": {
      "file": "td1.md",
      "namespace": "<NID-prof>",
      "commit": "sha1:b3e7f1a2...",
      "position": { }
    },
    "target": {
      "file": "notes/td1.md",
      "block_id": "bloc-42"
    }
  }
}
```

**Positions (`source.position`) — polymorphique :**

```json
// Texte
{ "kind": "text", "char_start": 142, "char_end": 189,
  "text_snapshot": "Soit E un espace vectoriel" }

// PDF
{ "kind": "pdf", "page": 3,
  "bbox": { "x": 120, "y": 340, "w": 280, "h": 24 },
  "text_snapshot": "Soit E un espace vectoriel" }

// OCR
{ "kind": "ocr", "char_start": 1042, "char_end": 1089,
  "text_snapshot": "Soit E un espace vectoriel" }

// Vidéo
{ "kind": "video", "timestamp_start_ms": 312400, "timestamp_end_ms": 378000 }

// Fichier entier
null
```

**Lifecycle :**
```
create → active → stale (dérivé) → resolved
              ↑
        update / update_position
```

---

### 6.2 `pm.promethium.patch` — Proposition de changement

Équivalent d'une PR. Propose une modification dans le namespace d'un autre NID.

**Opérations :** `create`, `comment`, `revise`, `accept`, `reject`, `withdraw`

**Types de patch :** `erratum`, `solution`, `annotation_officielle`

**Machine d'états :** `open` → `merged` | `rejected` | `withdrawn`

**Schéma — `create` :**

```json
{
  "type": "pm.promethium.patch",
  "op": "create",
  "author": "did:key:z6MkAlice...",
  "timestamp": "2026-03-28T16:00:00Z",
  "payload": {
    "patch_type": "erratum",
    "title": "Erreur de signe ligne 42",
    "target": {
      "namespace": "<NID-prof>",
      "branch": "heads/main"
    },
    "diff": {
      "base_commit": "sha1:b3e7f1a2...",
      "patch_ref": "refs/namespaces/<NID-alice>/refs/heads/patch/erratum-42"
    },
    "linked_cob": "<id-anchor-erratum>"
  }
}
```

---

### 6.3 `pm.promethium.thread` — Fil de discussion

Conversation asynchrone, chaque message = un commit dans le DAG.

**Opérations :** `post`, `reply`, `react`, `mark_solution`, `edit`, `close`

**Machine d'états :** `open` → `resolved` | `closed`

**Schéma — `post` :**

```json
{
  "type": "pm.promethium.thread",
  "op": "post",
  "author": "did:key:z6MkAlice...",
  "timestamp": "2026-03-28T14:45:00Z",
  "payload": {
    "anchor_id": "<id-anchor-forum>",
    "body": "Je ne comprends pas pourquoi le noyau est non trivial.",
    "attachments": []
  }
}
```

---

### 6.4 `pm.promethium.intent` — Intention déclarée

Déclare qu'un élève travaille sur un exercice. Social proof + analytics.

**Opérations :** `declare`, `start`, `pause`, `resume`, `complete`, `abandon`

**Machine d'états :**
```
declare → declared → in_progress ⇄ paused → completed | abandoned
```

**Schéma — `declare` :**

```json
{
  "type": "pm.promethium.intent",
  "op": "declare",
  "author": "did:key:z6MkAlice...",
  "timestamp": "2026-03-28T14:00:00Z",
  "payload": {
    "target_file": "td1.md",
    "target_namespace": "<NID-prof>",
    "target_commit": "sha1:b3e7f1a2...",
    "scope": "guild"
  }
}
```

---

### 6.5 `pm.promethium.tag` — Tag collaboratif

Un concept du domaine avec des relations typées vers d'autres tags. Consensus via threshold.

**Opérations :** `create`, `rename`, `propose_relation`, `validate_relation`, `invalidate_relation`, `merge`

**Types de relation :** `prerequisite`, `sub_topic`, `related`

**État d'une relation :** `proposed` → `canonical` (quand `validations - invalidations ≥ threshold`)

**Schéma — `propose_relation` :**

```json
{
  "type": "pm.promethium.tag",
  "op": "propose_relation",
  "author": "did:key:z6MkAlice...",
  "timestamp": "2026-03-28T10:05:00Z",
  "payload": {
    "relation_id": "<uuid>",
    "relation_type": "prerequisite",
    "from_tag": "<cob-id-tag-A>",
    "to_tag": "<cob-id-tag-B>",
    "threshold": 3
  }
}
```

---

### 6.6 `pm.promethium.link` — Lien typé entre fichiers

Lien sans position entre deux fichiers. Peut venir d'un humain ou d'un service ML.

**Opérations :** `create`, `validate`, `invalidate`, `annotate`

**Types :** `prerequisite`, `inferred_prerequisite`, `follows`, `references`, `corrects`, `generalizes`, `specializes`

**État :** `proposed` → `canonical` | `rejected`

**Schéma — `create` (ML) :**

```json
{
  "type": "pm.promethium.link",
  "op": "create",
  "author": "did:key:z6MkServiceML...",
  "timestamp": "2026-03-28T10:00:00Z",
  "payload": {
    "link_type": "inferred_prerequisite",
    "source": { "file": "cours/algebre-1.md", "namespace": "<NID-prof>" },
    "target": { "file": "td/td1.md", "namespace": "<NID-prof>" },
    "origin": "ml",
    "confidence": 0.87,
    "threshold": 3
  }
}
```

---

## 7. Display Templates

Convention du client PM pour le rendu visuel des COBs. Stocké dans `.promethium/display.toml` à la racine du repo, surchargeable par dossier.

### 7.1 Templates d'anchor

```toml
[anchor.note]
inline    = "margin-icon"
icon      = "pencil"
color     = "#4A9EFF"
expanded  = "side-panel"

[anchor.signet]
inline    = "margin-bookmark"
icon      = "bookmark"
color     = "#F59E0B"
expanded  = "none"

[anchor.definition]
inline    = "inline-underline"
color     = "#10B981"
expanded  = "tooltip"

[anchor.anki]
inline    = "margin-icon"
icon      = "cards"
color     = "#8B5CF6"
expanded  = "card-preview"

[anchor.forum]
inline    = "margin-counter"
icon      = "chat"
color     = "#F59E0B"
expanded  = "thread-panel"

[anchor.erratum]
inline    = "inline-highlight"
color     = "#EF4444"
expanded  = "diff-panel"

[anchor.guild_annotation]
inline    = "margin-avatar-stack"
color     = "#6366F1"
expanded  = "side-panel"

[anchor.intent]
inline    = "margin-users"
icon      = "users"
color     = "#84CC16"
expanded  = "intent-list"
```

### 7.2 Templates de patch

```toml
[patch.erratum]
list_view    = "diff-summary"
detail_view  = "diff-panel"
status_colors = { open = "#F59E0B", merged = "#10B981", rejected = "#EF4444", withdrawn = "#6B7280" }

[patch.solution]
list_view    = "solution-preview"
detail_view  = "solution-full"
```

### 7.3 Templates de thread

```toml
[thread]
panel_width     = 380
show_avatars    = true
solution_style  = "pinned-top"
collapsed_after = 10
```

### 7.4 Modes inline disponibles

| Mode | Description |
|---|---|
| `margin-icon` | Icône dans la gouttière |
| `margin-counter` | Icône + badge numérique |
| `margin-bookmark` | Marqueur de signet |
| `margin-avatar-stack` | Pile d'avatars (max 3 + overflow) |
| `margin-users` | Icône groupe + count |
| `inline-underline` | Soulignement coloré |
| `inline-highlight` | Surlignage coloré |
| `none` | Invisible |

### 7.5 Modes expanded disponibles

| Mode | Description |
|---|---|
| `side-panel` | Panneau latéral |
| `thread-panel` | Panneau avec thread complet |
| `diff-panel` | Diff + statut |
| `tooltip` | Tooltip au hover |
| `card-preview` | Aperçu recto/verso Anki |
| `intent-list` | Liste d'intentions avec statuts |
| `none` | Pas d'interaction |

---

## Différences v1 → v2

| | v1 | v2 |
|---|---|---|
| Structure | Tout au même niveau | Infra / Protocole client séparés |
| Type dans le ref | `refs/cobs/pm.promethium.anchor/<id>` | `refs/cobs/<id>` — type dans le commit |
| Guilde | Faux `<NID-guilde>` namespace | Repo à part entière avec delegates + threshold |
| Infra | Connaît les types de COBs | Agnostique au contenu |
| Extensibilité | Types fixés | Client tiers peut ajouter ses types sans toucher l'infra |
