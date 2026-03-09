# Feature Spec — Anchor System & Mapping

**Statut :** Proposition
**Auteur :** Claude
**Date :** 2026-02-25
**Dépendances :** Nœud Composite & DisplayTemplate, Système git, Création de decks Anki, Gestion des questions, Gestion de l'OCR, Coopération sur un dossier, Analytics autour d'un dossier

---

## 1. Résumé du concept

L'Anchor System est l'infrastructure qui permet de relier une **position précise dans un document source** à un ou plusieurs **Nodes cibles** (une note, une carte Anki, une question de forum, une annotation de guilde, une définition...).

Un anchor est directionnel mais bidirectionnel dans la navigation : depuis la source on atteint la cible, depuis la cible on peut revenir à la position source.

Un anchor n'impose pas un mode d'affichage fixe : il est associé à un **AnchorDisplayTemplate** qui détermine comment il se rend visuellement (overlay flottant, panneau latéral, marginalia, icône dans la marge). Ce template est configurable par type d'anchor, exactement comme le DisplayTemplate des Nodes.

L'Anchor System est la couche qui brise la frontière entre les formats : un timestamp vidéo, une région dans un PDF, une sélection dans un texte OCR'd, et une position dans un document riche sont tous des sources d'anchor interchangeables.

---

## 2. Modèle de données

### 2.1 Anchor

```
Anchor {
  id: UUID
  source_doc_id: NodeID           // document source
  source_position: AnchorPosition // position dans la source (polymorphique)
  target_id: NodeID               // Node cible (note, carte Anki, forum, définition...)
  anchor_type: AnchorType         // détermine le template par défaut
  display_template: AnchorDisplayTemplate
  visual: AnchorVisual
  layer: AnchorLayer
  created_by: UserID
  created_at: timestamp
  is_stale: boolean               // true si la source a été mise à jour et la position peut avoir glissé
}
```

### 2.2 AnchorPosition (polymorphique)

La position source est typée selon le format du document source.

```
// Document PDF text-based (LaTeX, Word exporté, etc.)
PDFPosition {
  page: int
  char_start: int       // offset dans le texte extrait de la page (via pdfplumber / pdf.js)
  char_end: int
  bbox: {               // bounding box pour le rendu visuel du highlight
    x: float            // coordonnées normalisées [0,1] sur la page
    y: float
    width: float
    height: float
  }
  text_snapshot: string // snapshot du texte sélectionné — sert à re-ancrer si le PDF change de version
}

// Note : pour les PDFs scannés (image-only), utiliser OCRPosition ci-dessous.
// Pour les PDFs avec texte vectoriel obfusqué (rare), fallback sur une position image simple.

// Vidéo
VideoPosition {
  start_ms: int
  end_ms: int | null  // null = anchor ponctuel (timestamp unique)
}

// Document texte riche
TextPosition {
  char_start: int
  char_end: int
}

// Document OCR (image + couche texte)
OCRPosition {
  image_x: float      // position sur l'image originale
  image_y: float
  image_width: float
  image_height: float
  char_start: int | null  // position dans la couche texte OCR, si disponible
  char_end: int | null
}
```

### 2.3 AnchorType

```
AnchorType: "note" | "definition" | "anki" | "forum" | "guild_annotation" | "signet" | "custom"
```

Chaque type a un AnchorDisplayTemplate par défaut. L'utilisateur peut override ce template pour un anchor spécifique ou pour tous les anchors de ce type dans un document donné.

### 2.4 AnchorDisplayTemplate

```
AnchorDisplayTemplate {
  render_mode: "panel" | "overlay" | "marginalia" | "icon"
  trigger: "hover" | "click" | "always"
  marker_style: "highlight" | "underline" | "dot" | "region" | "none"
  marker_color: string
  show_layer_badge: boolean     // indique visuellement la couche (personnel, guilde)
  overlay_size: "compact" | "expanded" | null
}
```

### 2.5 AnchorVisual

Les propriétés visuelles du marqueur affiché sur la source.

```
AnchorVisual {
  color: string
  opacity: float      // opacité du marqueur [0,1]
  icon: string | null // icône dans la marge pour les types "icon" ou "marginalia"
  pulse_on_active: boolean  // animation quand l'anchor entre dans le viewport
}
```

### 2.6 AnchorLayer

```
AnchorLayer {
  type: "personal" | "guild"
  guild_id: GuildID | null  // non null si type = "guild"
  is_visible: boolean       // togglé dans l'interface
}
```

### 2.7 AnchorGroup

Un ensemble d'anchors partageant la même source document et le même document cible forme un **AnchorGroup** : c'est le "mapping" entre deux documents. Un document source peut avoir N AnchorGroups (un par document cible, un par couche de guilde).

```
AnchorGroup {
  id: UUID
  source_doc_id: NodeID
  target_doc_id: NodeID
  layer: AnchorLayer
  anchors: AnchorID[]
  created_by: UserID
}
```

---

## 3. Templates d'anchor par défaut

### 3.1 Note (type: "note")

Relie une position source à un bloc dans un document de notes personnel.

```
render_mode: "panel"
trigger: "click"
marker_style: "highlight"
marker_color: bleu-doux
```

L'anchor note est le type principal créé en mode split-screen. Le panneau droit affiche le document de notes ; les highlights colorés dans le panneau gauche indiquent les positions ancrées.

### 3.2 Définition (type: "definition")

Relie un terme ou une variable à sa définition (issue du cours, de la base de données communautaire, ou générée par LLM avec contexte).

```
render_mode: "overlay"
trigger: "hover"
marker_style: "underline"
marker_color: gris-léger
overlay_size: "compact"
```

L'overlay s'affiche au survol, compact (3-5 lignes max), avec un lien vers le Node de définition complet si l'utilisateur veut aller plus loin.

### 3.3 Anki (type: "anki")

Relie une sélection de texte à une carte Anki.

```
render_mode: "overlay"
trigger: "click"
marker_style: "highlight"
marker_color: orange
overlay_size: "expanded"
```

Le marqueur orange indique qu'une carte Anki est ancrée ici. Au clic, un overlay affiche le recto/verso de la carte avec les statistiques SRS. La densité de couverture Anki est visible en un coup d'œil sur la source (zones orangées = couverture Anki existante).

### 3.4 Forum (type: "forum")

Relie une position source à un thread de questions/réponses.

```
render_mode: "icon"
trigger: "click"
marker_style: "dot"
icon: "💬"
overlay_size: "expanded"
```

Un point ou une icône dans la marge indique une question ancrée. Au clic, un overlay expanded affiche le thread avec la question et ses réponses. L'utilisateur peut répondre depuis l'overlay sans quitter la source.

### 3.5 Annotation de guilde (type: "guild_annotation")

Relie une position source à une annotation partagée dans la guilde.

```
render_mode: "marginalia"
trigger: "always"
marker_style: "region"
marker_color: vert-guilde
show_layer_badge: true
```

Visible en permanence (avec toggle global). Le badge de couche indique l'appartenance à la guilde. Les annotations de guilde ne bloquent pas la lecture : elles sont légèrement transparentes et se solidifient au survol.

### 3.6 Signet (type: "signet")

Marque une position source comme digne d'être retrouvée rapidement.

```
render_mode: "icon"
trigger: "hover"
marker_style: "dot"
icon: "🔖"
```

---

## 4. Interface principale — le MappingView

### 4.1 Modes de vue

Le MappingView est l'interface qui active la lecture des anchors. Il est accessible depuis n'importe quel Node document.

**Mode split-screen** (mode travail)
Deux panneaux côte à côte. Le panneau gauche affiche la source. Le panneau droit affiche un document de notes ou tout autre Node cible. Les anchors reliant ces deux documents sont visualisés dans les deux panneaux simultanément.

```
┌──────────────────┬──────────────────┐
│   Source (PDF,   │   Notes (Node)   │
│   vidéo, OCR)    │                  │
│                  │                  │
│  [highlight] ────┼──▶ [bloc note]   │
│                  │                  │
│  [highlight] ────┼──▶ [bloc note]   │
│                  │                  │
└──────────────────┴──────────────────┘
```

**Mode document avec overlays** (mode lecture)
La source est affichée seule en plein écran. Les anchors dont le template est `overlay` ou `icon` sont actifs et réagissent au survol/clic. Les anchors `panel` sont désactivés ou réduits à une icône dans la marge.

**Mode marginalia** (mode révision)
La source occupe ~70% de l'écran. Les notes ancrées s'affichent dans la marge droite (~30%), alignées verticalement avec leur position source. Chaque note est compacte ; au clic, elle s'expand en overlay.

```
┌────────────────────────┬──────────────┐
│   Source               │  [note A]    │
│                        │              │
│  Lorem ipsum…          │  [note B]    │
│                        │              │
│  Consectetur…          │  [note C]    │
│                        │              │
└────────────────────────┴──────────────┘
```

Toggle entre les trois modes via raccourci clavier ou onglets en haut de l'écran.

### 4.2 Synchronisation du scroll (mode hybride)

Par défaut, les deux panneaux scrollent indépendamment.

Comportements actifs en permanence :
- Quand une position source ancrée entre dans le viewport du panneau gauche, le marqueur correspondant s'active visuellement (glow/pulse léger).
- Clic sur un marqueur actif dans le panneau gauche → les deux panneaux snappent simultanément pour centrer le anchor pair à l'écran.
- Clic sur un bloc ancré dans le panneau droit → le panneau gauche scroll et centre la position source correspondante.

Toggle "auto-sync" (accessible via raccourci clavier) :
- Activé → le scroll du panneau gauche entraîne automatiquement le panneau droit pour garder les anchors actifs alignés.
- Pour la vidéo → le scroll des notes entraîne le player à la position correspondante (et inversement).

### 4.3 Panneau de layers

Un panneau lateral (togglable, `L`) liste les layers actifs sur la source courante.

```
Layers actifs sur ce document :
  ☑  [Mes notes]            (14 anchors)
  ☑  [Mes cartes Anki]      (7 anchors)
  ☐  [Questions forum]      (3 anchors)
  ☑  [Guilde - MP⋆ 2025]    (5 anchors)
  ☐  [Définitions]          (auto)
```

Chaque layer est togglable individuellement. Les marqueurs sur la source se masquent/affichent en temps réel.

---

## 5. Création d'anchors — UX

### 5.1 Anchor de type "note" (split-screen)

1. L'utilisateur ouvre un document source.
2. Il ouvre le mode split-screen (`Cmd+\` ou raccourci) → choisit un document de notes existant ou en crée un.
3. Il sélectionne une portion de texte ou une région dans la source.
4. Un tooltip contextuel apparaît : `[Anchor ici]` avec les types disponibles.
5. Il choisit "Note" → un highlight apparaît sur la sélection source, le curseur saute dans le panneau droit à l'endroit correspondant.
6. Il rédige son bloc de note normalement dans le panneau droit.

### 5.2 Anchor de type "Anki"

1. Sélection dans la source → tooltip → "Créer une carte Anki".
2. Un overlay compact apparaît directement sur la source : champ recto (pré-rempli avec la sélection) + champ verso.
3. Validation → la carte Node est créée, l'anchor Anki est posé automatiquement, le rectangle orange apparaît sur la sélection.

### 5.3 Anchor de type "définition"

Deux modes de création :

**Manuel** : sélection → tooltip → "Définir" → l'utilisateur rédige une définition dans un mini-éditeur.

**Automatique** : le système peut suggérer des anchors de définition sur les termes connus (variables mathématiques, termes techniques) via un scan LLM du document. L'utilisateur valide ou ignore les suggestions. Les définitions auto-suggérées sont issues de la base de données communautaire de la plateforme ou générées avec le contexte du cours de l'utilisateur en preprompt.

### 5.4 Anchor de type "forum"

Se crée depuis la source par sélection → "Poser une question ici" → l'utilisateur rédige sa question. L'anchor est posé sur la sélection, la question est publiée dans le mini-forum du document source.

Inversement, une question existante dans le forum peut être ancrée à une position source a posteriori (drag-and-drop de la question vers la source dans le MappingView).

### 5.5 Source vidéo

La source vidéo (URL YouTube, fichier local, ou Node vidéo) s'affiche dans le panneau gauche avec son player standard.

La timeline du player affiche des marqueurs colorés aux positions des anchors existants.

Créer un anchor depuis la vidéo : pause → sélectionner un type dans le tooltip → le timestamp courant est capturé comme source position.

En auto-sync, le scroll du panneau droit (notes) ne déplace pas automatiquement le player, mais le player, quand il avance, scroll les notes pour centrer le bloc ancré au timestamp actuel.

### 5.6 Source OCR

Après OCR d'un document manuscrit, le document résultant est traité comme une source hybride : l'image originale sert de fond, la couche texte OCR est superposée de façon transparente.

Les anchors peuvent être placés sur la couche texte (comportement identique à un PDF) ou directement sur une région de l'image (pour les passages où l'OCR n'est pas fiable).

Un anchor sur une région image qui n'a pas de couche texte est de type `OCRPosition` sans `char_start/char_end`.

---

## 6. Gestion de la stale-ité

Quand le document source est mis à jour (nouveau commit git par le prof), les positions des anchors existants peuvent ne plus correspondre au contenu actuel.

Détection : à chaque nouvelle version du document source, le système compare les positions des anchors avec le nouveau contenu. Un anchor est marqué `is_stale: true` si sa position ne peut plus être réconciliée avec le nouveau contenu (texte supprimé, page réorganisée, timestamp hors durée...).

Résolution (UX) : les anchors stale sont affichés avec un marqueur rouge ou barré sur la source. Un bandeau en haut de l'interface prévient l'utilisateur : "X anchors sur ce document ne correspondent plus à la version actuelle. [Réviser]".

Le mode "Réviser les anchors stale" affiche une interface diff : à gauche l'ancienne position (surlignée dans l'ancienne version), à droite la nouvelle version. L'utilisateur re-sélectionne la nouvelle position ou supprime l'anchor.

---

## 7. Interactions avec les autres features

### 7.1 Création de decks Anki

Chaque carte Anki créée depuis la source via un anchor est automatiquement liée à sa position source. Le deck visible sur la source (coverage orange) montre quels passages sont couverts par des cartes. La statistique "% du document couvert par des cartes Anki" est calculée en agrégeant les longueurs de toutes les positions sources des anchors Anki sur le document.

### 7.2 Mini-forum

Le mini-forum d'un document est la collection de tous les anchors de type "forum" posés sur ce document. Chaque question a donc une position source précise et peut être explorée depuis la source ou depuis le forum. Trier le forum par "position dans le document" est l'ordre naturel.

### 7.3 Annotations de guilde

Les annotations de guilde sont des anchors avec `layer.type = "guild"`. Elles sont créées, éditées et supprimées uniquement par leurs auteurs. Tous les membres de la guilde voient les annotations de guilde des autres (si le toggle de layer est actif). Elles ne modifient ni la source ni les notes personnelles.

### 7.4 Git

Les anchors personnels sont versionnés dans le repo git du document de notes (pas du document source). Un commit dans le document de notes peut inclure des anchors créés, modifiés, ou supprimés.

Les anchors de guilde sont versionnés dans un repo git dédié au layer de guilde sur ce document.

Si le document source est updaté par le prof (commit dans le repo du cours), le système déclenche la vérification de stale-ité sur tous les AnchorGroups dont ce document est la source.

### 7.5 DisplayTemplate des Nodes

La vue marginalia du MappingView est en fait un cas particulier de DisplayTemplate d'un Node : le Node "document de notes" peut s'afficher avec un template "marginalia" qui positionne ses blocs en marge de leur source ancrée plutôt qu'en flux vertical. Le DisplayTemplate contrôle donc aussi la vue du document cible, pas seulement de la source.

### 7.6 Analytics

Le temps passé dans le MappingView est attribué au document source (pas au document de notes). Les analytics de la source peuvent inclure une heatmap d'activité des anchors : quelles positions ont le plus d'anchors posés par la communauté, quelles positions ont le plus de questions forum, quelles zones sont les moins couvertes en notes. Cette heatmap est utile pour le prof pour identifier les passages les moins bien compris ou les plus annotés.

### 7.7 Variables et retrieval

L'anchor de type "définition" est le mécanisme sous-jacent de la feature "passer sa souris sur une variable et avoir sa définition". La définition peut être :
- Rédigée manuellement par l'utilisateur (anchor personnel)
- Partagée dans la base communautaire (anchor public en lecture seule)
- Générée à la volée par LLM avec le cours en contexte (anchor éphémère, non persisté)

---

## 8. Modèle de permissions

| Action | Créateur du mapping | Membre de guilde | Utilisateur tiers |
|---|---|---|---|
| Créer un anchor personnel | ✅ | N/A | N/A |
| Voir ses propres anchors | ✅ | N/A | N/A |
| Créer un anchor de guilde | ✅ (si membre) | ✅ | ❌ |
| Voir les anchors de guilde | ✅ | ✅ | ❌ |
| Modifier un anchor de guilde | Auteur de l'anchor | ❌ | ❌ |
| Voir les anchors forum | ✅ | ✅ | ✅ (si doc public) |
| Re-anchor un anchor stale | Propriétaire de l'anchor | ❌ | ❌ |

---

## 9. Questions ouvertes

**Q1 — Anchors croisés entre deux documents non-source**
La spec couvre le cas source → note. Mais peut-on créer un anchor entre deux documents de notes (note A → note B) ? Ce serait du bidirectional linking à la Obsidian. Si oui, le système d'anchor gère-t-il tous les liens internes de la plateforme, ou est-ce un système séparé ?

**Q2 — Gestion des anchors sur PDF non-extractable**
Certains PDFs (scans non-OCR'd, PDFs protégés) ne permettent pas la sélection de texte. L'anchor doit alors se baser uniquement sur les coordonnées image. Si le PDF est re-uploadé en version OCR'd ultérieurement, les anchors image-seulement peuvent-ils être automatiquement réconciliés avec la couche texte ?

**Q3 — Limite du nombre de layers simultanés**
Avec des anchors personnels (notes, Anki, signets), des anchors de guilde, des anchors forum, et des anchors de définition, un document source très annoté peut devenir visuellement surchargé. Une règle de priorité d'affichage ou une limite de densité visuelle doit être définie. Proposition : quand plusieurs anchors se chevauchent sur la source, les fusionner en un seul marqueur "cluster" avec un badge de compte, qui s'expand au survol.

**Q4 — Anchor de type "définition" et base communautaire**
Si la base communautaire contient une définition pour un terme, est-elle proposée automatiquement à tous les utilisateurs qui ont ce terme dans leur document ? Cela implique un scan automatique de tous les documents pour matcher les termes connus — c'est une opération coûteuse et potentiellement intrusive. Définir un opt-in explicite vs opt-out.

**Q5 — Export des anchors**
Lors de l'export d'un document source en PDF (via le DisplayTemplate export), les anchors sont-ils inclus ? Si oui, sous quelle forme (annotations PDF standard, notes de bas de page, QR codes vers les Nodes cibles) ? La réponse détermine si les documents exportés de Prométhium restent "vivants" ou deviennent des copies mortes.
