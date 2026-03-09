# Rapport — Implémentation des features sociales via Node + Anchor

**Date :** 2026-02-25
**Contexte :** Ce rapport analyse chaque feature sociale et collaborative de Prométhium et détermine dans quelle mesure elle se réduit aux deux primitives techniques établies : le **Nœud Composite + DisplayTemplate** et l'**Anchor System**.

---

## Rappel des deux primitives

**Node** : tout contenu dans l'app est un Node. Il peut avoir du contenu propre, des enfants (d'autres Nodes), des métadonnées, et un DisplayTemplate qui contrôle son rendu. Il est versionné via git.

**Anchor** : un lien bidirectionnel entre une position précise dans un document source et un Node cible. Il a un type, un display template (overlay, panel, marginalia, icon), et une layer (personal, guild).

---

## 1. L'insight central : l'utilisateur est un Node

Avant d'analyser les features, un constat architectural s'impose. Si le modèle affirme "tout est un Node", alors l'utilisateur lui-même devrait être un Node.

Un profil utilisateur est un document avec du contenu (bio, stats publiques, solutions publiées), une version history (git), et des relations avec d'autres Nodes (ses contributions). Si l'utilisateur est un Node, la **guilde est un Node Composite** dont les enfants sont des Nodes utilisateurs — et tout le contenu partagé dans la guilde est une couche d'Anchors sur ce Node guilde.

```
Node (type: "guild") {
  children: [
    Node (type: "user") { ... },
    Node (type: "user") { ... },
  ]
  shared_content: [NodeID, ...]      // cours, TDs partagés dans la guilde
  anchor_layers: [AnchorLayer]       // la couche d'annotations de guilde sur chaque document
}
```

Ce choix rend le modèle entièrement self-consistent : il n'y a pas de système social "en plus" — il n'y a que des Nodes et des Anchors. Les conséquences sont profondes pour chaque feature qui suit.

---

## 2. Features réductibles à 100% aux primitives

### 2.1 Notes de guilde (Coopération sur un dossier)

Une note de guilde est un Anchor de type `guild_annotation` avec `layer.type = "guild"`. Posée sur un document (cours, TD, exercice), elle est visible à tous les membres de la guilde via le toggle de layer dans le MappingView. Elle ne modifie ni le document source ni les notes personnelles de qui que ce soit.

Le "toggle en raccourci clavier pour activer/désactiver les notes de guilde" = toggle du layer guild dans le panneau de layers de l'Anchor System. Aucun nouveau mécanisme requis.

**→ Anchor (type: guild_annotation, layer: guild)**

---

### 2.2 Erratum

Un erratum est un Anchor de type `erratum` posé par n'importe quel utilisateur sur une position précise d'un cours. Il pointe vers un Node cible qui contient le texte de correction (manuscrit scanné, LaTeX, texte riche).

Le display template de l'erratum est `render_mode: overlay, trigger: click, marker_style: dot` — une icône dans la marge, qui s'expand au clic pour afficher la correction. Le prof peut ensuite valider l'erratum via une PR git qui intègre la correction directement dans le cours source.

Les polys à trou (zones où une réponse doit être complétée) sont un cas particulier : le DisplayTemplate du cours inclut un slot conditionnel qui injecte le contenu de l'Anchor erratum directement dans le trou quand il existe.

**→ Anchor (type: erratum) + DisplayTemplate conditionnel**

---

### 2.3 Questions / Mini-forum

Une question posée sur un passage de cours est un Anchor de type `forum` pointant vers un Node de type `thread`. Le thread contient la question + ses réponses comme Nodes enfants. L'ensemble des Anchors forum sur un document constitue le mini-forum de ce document.

Trier le forum par position dans le document = trier les Anchors forum par `source_position.char_start`. Cliquer sur un passage du cours pour voir les questions ancrées = activer les Anchors forum dans le viewport courant.

**→ Anchor (type: forum) + Node (type: thread)**

---

### 2.4 Signet sur une solution

Un signet sur une solution ou une réponse appréciée est un Anchor de type `signet` pointant vers le Node solution. Il est personnel (layer: personal). La collection de tous les signets d'un utilisateur = tous ses Anchors de type `signet`.

La social proof (une solution qui génère beaucoup de signets est mise en avant) = `stats.signet_count` sur le Node solution, exposé comme slot dans le DisplayTemplate de la vue solutions.

**→ Anchor (type: signet) + slot DisplayTemplate**

---

### 2.5 Solutions d'exercices

Une solution est un Node (type: `solution`), enfant du Node exercice. Sa visibilité est contrôlée par le DisplayTemplate du Node exercice via un conditionnel :

```
{{if user.has_submitted OR user.time_on_exercise > threshold}}
  {{children | filter: type = "solution" | sort_by: signet_count desc}}
{{else}}
  [Solutions accessibles après soumission]
{{end}}
```

L'ordre des solutions (les plus signettées en premier) est un `sort_by` sur le slot. L'inspiration LeetCode = afficher plusieurs solutions d'auteurs différents dans la même vue, chacun étant un Node solution enfant.

**→ Node (type: solution) + DisplayTemplate conditionnel**

---

### 2.6 Review de TD (révision des exercices difficiles)

Revisiter les N exercices sur lesquels on a le plus galéré = un slot DisplayTemplate avec un filtre analytics :

```
{{children | sort_by: user_struggle_score desc | limit: N}}
```

Le slider "combien d'exercices reprendre" = le paramètre `N` passé dynamiquement au slot depuis l'UI. Aucun nouveau mécanisme : c'est une vue documentaire du Node TD avec un filtre analytics sur ses enfants.

**→ DisplayTemplate slot + analytics service**

---

### 2.7 Time tracker (affichage)

Le temps passé sur chaque Node est collecté par un service d'événements (service séparé). Son affichage dans le document utilise les slots analytics déjà définis :

```
{{stats.my_time}}
{{stats.avg_time_similar_profiles}}
{{analytics.time_heatmap}}
```

La comparaison avec les profils similaires est calculée par le service analytics et exposée comme un slot. Le DisplayTemplate contrôle où et comment ça s'affiche dans le document — bandeau en bas, overlay, ou section dédiée.

**→ DisplayTemplate slot + analytics service**

---

## 3. Features nécessitant une extension légère des primitives

### 3.1 Intention d'aborder des exercices (availability signal)

"Je vais aborder ce corpus d'exercices d'ici X jours" est une déclaration d'intention publique dans la guilde. Elle se modélise comme un Node de type `intent` :

```
Node (type: "intent") {
  author: UserID
  target_nodes: NodeID[]      // les exercices que je compte faire
  horizon: duration           // dans combien de temps (slider)
  layer: guild                // visible par la guilde
  expires_at: timestamp       // après l'échéance, le Node se marque "expiré"
}
```

Ce Node intent est un enfant du Node guilde, visible par tous ses membres. L'affichage sur le TD = un Anchor de type `intent` posé sur chaque exercice ciblé, avec `render_mode: icon`. Quand on survole un exercice, on voit les icônes des membres qui ont déclaré vouloir le faire bientôt.

Le score de fiabilité = `stats.intent_completion_rate` sur le Node utilisateur (ratio d'intents honorés dans le temps imparti). C'est un slot DisplayTemplate sur le profil utilisateur.

**→ Node (type: intent) + Anchor (type: intent) + slot DisplayTemplate sur le profil**

---

### 3.2 Parcours recommandé sur un TD

Le parcours recommandé inféré de manière non-supervisée (tri topologique des exercices selon les successions de réussite de la communauté) produit des **edges de préférence** entre exercices. Ces edges sont une nouvelle catégorie d'Anchor :

```
Anchor (type: "inferred_prerequisite") {
  source_doc_id: NodeID (exercice A)
  target_id: NodeID (exercice B)
  weight: float          // force statistique de la relation
  source: "community_inference"  // vs "manual" pour les prérequis définis par le prof
}
```

Ces Anchors inférés ne pointent pas vers une *position* dans un document, mais vers un autre Node entier. C'est une légère extension du modèle Anchor, où la `source_position` devient optionnelle quand l'Anchor relie deux Nodes entiers plutôt qu'une position à un Node.

L'affichage visuel du parcours recommandé sur le TD = DisplayTemplate slot `{{children | sort_by: inferred_prerequisite_order}}`. Le calcul est fait par le service de recommandation, le résultat est consommé par le template.

**→ Anchor (type: inferred_prerequisite, sans source_position) + DisplayTemplate slot**

---

### 3.3 Tags hiérarchiques collaboratifs

Chaque utilisateur a sa propre version du tag tree. Ce tag tree est un Node de type `tag_tree`, ce qui le rend versionnable (git) et éditable comme un document. Les enfants de ce Node sont des Nodes de type `tag`.

```
Node (type: "tag_tree") {
  owner: UserID
  children: [
    Node (type: "tag") { name: "Analyse", children: [
      Node (type: "tag") { name: "Séries entières" },
      Node (type: "tag") { name: "Intégration" }
    ]}
  ]
}
```

Le service de consensus communautaire compare les tag trees de tous les utilisateurs et produit un tag tree global agrégé (lui aussi un Node, mais de type `global_tag_tree`, en lecture seule pour les utilisateurs). La suggestion LLM lors du tagging = le LLM compare le contenu du Node à tagger avec le global_tag_tree et propose les tags les plus proches.

**→ Node (type: tag_tree / tag) + service de consensus séparé**

---

## 4. Features nécessitant un service dédié (primitives insuffisantes seules)

### 4.1 Algorithme de recommandation global

Le service de recommandation consomme en entrée : les Nodes et leurs métadonnées (tags, prérequis, difficulté), les Anchors inferred_prerequisite (edges statistiques), les événements analytics (quel utilisateur a réussi quoi et dans quel ordre). Il produit en sortie : une liste ordonnée de Nodes à suggérer à un utilisateur donné.

Ce service est entièrement séparé des primitives. Les primitives fournissent le graph de données qu'il consomme. Son output est exposé via des slots DisplayTemplate.

### 4.2 Voir les parcours de ses amis (colorful paths)

La version simple (liste de qui a fait quoi) = DisplayTemplate slot `{{analytics.guild_activity | for: current_node}}`. La version avancée (chemins colorés sur le TD selon les membres) = un composant de rendu spécialisé qui superpose des traces de couleur sur la vue explorateur du TD. Ce n'est pas un DisplayTemplate — c'est un renderer dédié qui consomme les données analytics de la guilde.

### 4.3 Superposition de graphs de connaissances

Le graph personnel de concepts explorés + sa superposition avec celui d'un ami nécessite un renderer graph (OpenGL/SVG/D3) qui consomme les données du Node graph et de l'analytics service. Les primitives fournissent les données (quels Nodes l'utilisateur a explorés, quels liens existent entre eux via prérequis et inferred_prerequisites) mais le rendu est un composant dédié.

---

## 5. Synthèse

| Feature | Node | Anchor | DisplayTemplate | Service séparé |
|---|:---:|:---:|:---:|:---:|
| Notes de guilde | | ✅ | | |
| Erratum | | ✅ | ✅ | |
| Questions / forum | ✅ | ✅ | | |
| Signet sur solution | | ✅ | | |
| Solutions d'exercices | ✅ | | ✅ | |
| Review TD (slider) | | | ✅ | ✅ analytics |
| Time tracker (affichage) | | | ✅ | ✅ analytics |
| Intention d'aborder des exos | ✅ | ✅ | ✅ | |
| Parcours recommandé inféré | | ✅ (étendu) | ✅ | ✅ ML |
| Tags collaboratifs | ✅ | | | ✅ consensus |
| Profil utilisateur | ✅ | | ✅ | |
| Guilde | ✅ | | | |
| Recommandation globale | | | ✅ (display) | ✅ ML |
| Parcours des amis (visuel simple) | | | ✅ | ✅ analytics |
| Parcours des amis (coloré) | | | | ✅ renderer |
| Superposition de graphs | | | | ✅ renderer |

---

## 6. Ce que ce mapping révèle

**Toute contribution sociale est soit un Node, soit un Anchor.** Une solution, une note de guilde, un erratum, une question, une intention d'aborder des exercices — tout produit social d'un utilisateur s'inscrit dans l'une des deux primitives. Il n'y a pas de "objet social" sui generis dans Prométhium.

**La social proof émerge naturellement du modèle.** Le nombre de signets sur une solution, la fiabilité d'un utilisateur, la "cote" d'un exercice difficile — tout ça est calculable à partir des métadonnées des Nodes et des Anchors. Ces métriques sont des agrégations sur le graph, pas des systèmes séparés.

**L'extension nécessaire est minimale.** La seule vraie extension du modèle Anchor est l'Anchor sans `source_position` (qui relie deux Nodes entiers plutôt qu'une position à un Node). Ça couvre les prérequis inférés, les relations inter-documents, et potentiellement d'autres liens sémantiques futurs. C'est une généralisation propre, pas une exception.

**Les services séparés ne touchent pas aux primitives.** L'algorithme de recommandation, le service analytics, le consensus de tags — ils lisent les Nodes et les Anchors, ils n'en définissent pas de nouveaux types. La frontière est nette.

**Le seul cas qui résiste vraiment** : les visualisations graph (superposition de graphs de connaissances, parcours colorés sur un TD). Ces features nécessitent un renderer dédié parce qu'elles sortent du paradigme documentaire. Mais elles consomment exactement les mêmes données que les vues documentaires — il n'y a pas de données supplémentaires à modéliser, seulement un composant de rendu différent.
