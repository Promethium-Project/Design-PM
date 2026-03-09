# Feature Spec — Nœud Composite & DisplayTemplate

**Statut :** Proposition
**Auteur :** Claude
**Date :** 2026-02-25
**Dépendances :** Système git, Gestion des tags, Création de decks Anki, Gestion des exercices d'oraux, Analytics autour d'un dossier, Système de partage des dossiers

---

## 1. Résumé du concept

Dans Prométhium, la distinction entre *dossier* et *document* est abolie. Tout objet dans l'arborescence est un **Nœud** (`Node`). Un Node peut contenir du contenu propre, des enfants (d'autres Nodes), ou les deux simultanément.

Chaque Node est associé à un **DisplayTemplate** : une méthode de rendu qui définit comment le Node et ses enfants sont composés en une représentation lisible. Par analogie avec Python, le DisplayTemplate est le `__repr__` du Node — il décrit non pas *ce que* le Node est, mais *comment il se montre*.

Un Node sans enfants est une feuille (un exercice, une carte Anki, un fragment de cours). Un Node avec enfants est un Nœud Composite (un TD, un chapitre, un cours entier). La structure est récursive : un Nœud Composite peut contenir d'autres Nœuds Composites.

---

## 2. Modèle de données

### 2.1 Node

```
Node {
  id: UUID
  type: NodeType               // "exercice" | "cours" | "td" | "chapitre" | "oral" | "deck_anki" | "custom"
  title: string
  content: BlockList           // contenu propre du Node (en-tête, intro, corps...)
  children: Node[]             // liste ordonnée d'enfants
  template_id: UUID            // référence au DisplayTemplate de classe
  instance_overrides: Override[]  // surcharges spécifiques à cette instance
  metadata: NodeMetadata
  git_repo_id: UUID            // référence au repo git associé
  created_by: UserID
  created_at: timestamp
  updated_at: timestamp
}
```

### 2.2 NodeMetadata

```
NodeMetadata {
  subject: string              // matière (Maths, Physique, Info...)
  tags: TagID[]                // tags hiérarchiques communautaires
  difficulty: 1..5 | null
  estimated_duration: minutes | null
  prerequisites: NodeID[]      // Nodes prérequis
  year: string | null          // année scolaire ou concours
  establishment: string | null
  visibility: "private" | "guild" | "public"
}
```

### 2.3 DisplayTemplate

```
DisplayTemplate {
  id: UUID
  name: string
  target_type: NodeType        // le type de Node auquel ce template s'applique par défaut
  blocks: TemplateBlock[]      // liste ordonnée de blocs composant le rendu
  is_system_default: boolean   // template fourni par la plateforme
  is_shared: boolean           // template publié et réutilisable par la communauté
  author: UserID
  parent_template_id: UUID | null  // héritage de template
  export_variants: ExportVariant[] // variantes pour export (PDF, CSV, Markdown...)
}
```

### 2.4 TemplateBlock

Un bloc est soit **statique** (contenu fixe rédigé dans le template), soit un **slot** (contenu dynamique injecté à la compilation).

```
TemplateBlock {
  id: UUID
  block_type: "static" | "slot" | "conditional"

  // si static :
  content: RichText            // texte, titre, callout, séparateur...

  // si slot :
  slot_config: SlotConfig

  // si conditional :
  condition: SlotCondition
  then_blocks: TemplateBlock[]
  else_blocks: TemplateBlock[] | null
}
```

### 2.5 SlotConfig

```
SlotConfig {
  source: "children" | "metadata.*" | "stats.*" | "analytics.*"
  filter: FilterExpression | null  // ex: tag = "difficile", type = "exercice"
  sort_by: SortExpression | null   // ex: difficulty asc, position asc
  limit: number | null
  render_mode: "inline" | "card" | "compact" | "title_only"
  collapsible: boolean
  show_metadata: boolean           // affiche difficulté, durée estimée, tags sous chaque enfant
}
```

### 2.6 Override d'instance

L'override permet d'injecter du contenu à un emplacement précis du rendu, **sans modifier le template de classe**.

```
Override {
  id: UUID
  position: "before" | "after"
  target: "child:{NodeID}" | "slot:{SlotID}" | "block:{BlockID}"
  content: TemplateBlock[]
}
```

---

## 3. Modes de rendu

Chaque Node Composite expose trois vues accessibles depuis l'interface.

### 3.1 Vue Explorateur

Affiche les enfants sous forme de liste ou de grille de cartes. C'est la vue de *navigation*. Elle n'applique pas le DisplayTemplate : elle montre la structure brute.

Interactions disponibles : réorganiser les enfants par drag-and-drop, ouvrir un enfant, modifier les métadonnées, accéder au template.

### 3.2 Vue Document

Applique le DisplayTemplate et rend le Node comme un document continu. C'est la vue de *lecture et de travail*.

Le rendu est composé dans l'ordre des blocs du template. Chaque enfant rendu inline dans un slot peut être :
- déplié (contenu visible)
- replié (seul le titre est visible)
- masqué (exclu du rendu pour cette session)

La vue Document est l'interface principale pour travailler un TD exercice par exercice, lire un cours chapitre par chapitre, ou réviser un deck complet.

### 3.3 Vue Séquentielle

Affiche un seul enfant à la fois, avec une navigation avant/arrière. Conçue pour les conditions d'examen ou de révision concentrée. Le rendu de chaque enfant utilise son propre DisplayTemplate (celui du type "exercice", "carte", etc.), pas le template du parent.

### 3.4 Vue Export

Déclenche la compilation vers un format externe (PDF, CSV, Markdown) en utilisant la variante export du DisplayTemplate. Voir section 6.

---

## 4. Le DisplayTemplate en détail

### 4.1 Slots disponibles

Les slots sont les points d'injection dynamique dans le template.

**Slots sur les enfants :**
```
{{children}}                              — tous les enfants, ordre courant
{{children | sort_by: position}}          — ordre manuel défini dans l'explorateur
{{children | sort_by: difficulty asc}}    — du plus facile au plus difficile
{{children | filter: tag = "lemme"}}      — enfants ayant le tag "lemme"
{{children | filter: type = "exercice"}}  — seulement les exercices
{{children | limit: 5}}                   — les 5 premiers enfants
```

**Slots sur les métadonnées :**
```
{{metadata.title}}
{{metadata.subject}}
{{metadata.difficulty}}
{{metadata.estimated_duration}}
{{metadata.tags}}
{{metadata.prerequisites}}
{{metadata.establishment}}
{{metadata.year}}
```

**Slots sur les statistiques agrégées :**
```
{{stats.total_children}}
{{stats.total_estimated_duration}}      — somme des durées estimées des enfants
{{stats.avg_difficulty}}
{{stats.completion_rate}}               — % d'enfants marqués réussis par l'utilisateur courant
{{stats.community_success_rate}}        — taux de réussite global sur la plateforme
```

**Slots sur les analytics :**
```
{{analytics.time_heatmap}}              — overlay de chaleur du temps passé sur les enfants
{{analytics.struggle_indicators}}       — indicateurs des enfants les plus bloquants
```

### 4.2 Conditions

Les blocs conditionnels permettent d'adapter le rendu au contexte.

```
{{if user.has_submitted(child)}}
  {{slot: community_solutions | for: child}}
{{else}}
  [Solutions accessibles après soumission]
{{end}}

{{if metadata.difficulty >= 4}}
  ⚠ Exercice classé difficile — temps estimé : {{metadata.estimated_duration}} min
{{end}}
```

### 4.3 Structure type d'un template TD

```
[STATIC] Titre H1 : {{metadata.title}}
[STATIC] Ligne de metadata : {{metadata.subject}} · {{metadata.year}} · {{metadata.establishment}}
[STATIC] Durée totale estimée : {{stats.total_estimated_duration}} min
[STATIC] Séparateur
[STATIC] Consignes générales (texte rédigé dans le template)
[SLOT]   {{children | sort_by: position | render_mode: inline | collapsible: true | show_metadata: true}}
[STATIC] Séparateur
[STATIC] Pied de page : {{stats.total_children}} exercices · Taux de réussite communautaire : {{stats.community_success_rate}}
```

### 4.4 Structure type d'un template Cours

```
[STATIC] Titre H1 : {{metadata.title}}
[STATIC] Prérequis : {{metadata.prerequisites}}
[STATIC] Objectifs pédagogiques (texte rédigé dans le template)
[SLOT]   {{children | filter: type = "chapitre" | render_mode: inline | collapsible: true}}
[STATIC] Résumé (texte rédigé dans le template)
[SLOT]   {{children | filter: tag = "exercice_associé" | render_mode: card}}
```

### 4.5 Structure type d'un template Deck Anki (export CSV)

```
[SLOT]   {{children | filter: type = "carte" | render_mode: csv_row}}
  — chaque carte produit une ligne : "recto;verso;tags"
```

### 4.6 Structure type d'un template Oral

```
[STATIC] En-tête PDF : {{metadata.title}} · {{metadata.year}}
[SLOT]   {{children | sort_by: position | render_mode: pdf_block | show_metadata: false}}
  — chaque exercice est rendu comme un bloc PDF avec un saut de page optionnel
```

---

## 5. Héritage et override — classe vs instance

### 5.1 Héritage de template

Un DisplayTemplate peut hériter d'un template parent (`parent_template_id`). Il hérite de tous les blocs du parent et peut les surcharger ou en ajouter de nouveaux. Cela fonctionne comme l'héritage de classes.

Exemple : le template "TD Maths MP" hérite du template "TD" générique. Il surcharge le bloc de pied de page pour ajouter un lien vers les ressources de la filière.

### 5.2 Override d'instance

Un créateur de Node peut modifier le rendu d'une instance spécifique sans toucher au template de classe. Les overrides sont des blocs injectés *avant* ou *après* un enfant précis, ou *avant* ou *après* un slot.

Exemple : un prof veut ajouter un commentaire entre l'exercice 3 et l'exercice 4 de **ce TD précis**. Il crée un override `{position: "after", target: "child:{id_exercice_3}", content: [bloc texte "Remarque importante..."]}`.

L'override est stocké sur le Node, pas sur le template. Il est versionné via git comme le reste du contenu du Node.

### 5.3 Priorité de rendu

```
Override d'instance  (priorité maximale)
       ↓
Template spécifique au Node
       ↓
Template hérité (parent)
       ↓
Template système par défaut pour le type
```

---

## 6. Variantes d'export

Chaque DisplayTemplate peut définir plusieurs variantes d'export. Une variante est un rendu alternatif du même template pour un format cible différent.

```
ExportVariant {
  format: "pdf" | "csv" | "markdown" | "latex" | "html"
  blocks: TemplateBlock[]  // redéfinit ou surcharge les blocs pour ce format
}
```

Exemples de variantes :
- **PDF print** : supprime les métadonnées interactives, ajoute des marges, gère les sauts de page
- **CSV Anki** : produit une ligne par carte `recto;verso;tags`
- **Markdown** : rendu flat sans slots conditionnels, pour export vers Obsidian ou GitHub

L'export est déclenché depuis la vue Explorateur ou la vue Document via un raccourci clavier ou un menu contextuel.

---

## 7. UX

### 7.1 Accès aux vues

Depuis n'importe quel Node Composite, l'utilisateur bascule entre les vues via des onglets persistants en haut de l'écran :
- `[Explorer]` `[Document]` `[Séquentiel]`

Un raccourci clavier permet de basculer instantanément entre les vues (cohérent avec la philosophie keyboard-first de Prométhium).

### 7.2 Édition du template — vue "Template"

Un quatrième onglet `[Template]` est accessible uniquement aux utilisateurs ayant les droits d'édition sur le Node.

La vue Template est un éditeur de blocs (similaire à un éditeur de document ordinaire) avec deux types de blocs distincts visuellement :
- **Blocs statiques** : identiques à l'éditeur de document standard, fond blanc
- **Blocs de slot** : fond coloré (ex: bleu), avec une syntaxe de configuration inline

L'éditeur propose une palette de blocs accessible via `/` (slash command), incluant tous les slots disponibles avec leur description et un aperçu du rendu.

Un bouton `Prévisualiser` compile le template en temps réel avec le contenu actuel du Node.

### 7.3 Override d'instance — édition inline

Depuis la vue Document, un utilisateur ayant les droits peut survoler l'espace entre deux enfants rendus. Un `+` apparaît. En cliquant dessus, il ouvre un mini-éditeur de bloc qui crée un override d'instance à cet endroit précis.

L'override est visuellement distingué du contenu template par une couleur de fond différente et une icône "override" dans la marge.

### 7.4 Application d'un template existant

Lors de la création d'un Node Composite, un sélecteur propose :
1. Les templates systèmes pour le type détecté (TD, Cours, Oral, Deck...)
2. Les templates de la communauté (les plus utilisés, filtrés par matière/type)
3. "Commencer vide"

Un aperçu live du template est affiché dans le sélecteur avant confirmation.

### 7.5 Partage d'un template

Depuis la vue Template, le créateur peut publier son template sur la plateforme. Il choisit un nom, une description, les types de Nodes compatibles. Le template devient alors consultable et réutilisable par la communauté.

---

## 8. Interactions avec les autres features

### 8.1 Système git

Le DisplayTemplate d'un Node est versionné dans le même repo git que son contenu. Un commit peut donc toucher :
- Le contenu d'un enfant (exercice modifié)
- L'ordre des enfants (réorganisation du TD)
- Le template (nouvelle mise en forme)
- Les overrides d'instance (commentaire ajouté entre deux exercices)

Ces changements apparaissent séparément dans le diff, permettant au prof de distinguer "j'ai changé un exercice" de "j'ai changé la mise en forme de tout le TD".

### 8.2 Analytics

Le slot `{{analytics.time_heatmap}}` injecte dans la vue Document un overlay visuel superposé aux enfants, montrant la distribution du temps passé par les étudiants (feature analytics dossier). Le template contrôle si cet overlay est visible par défaut ou togglable par l'utilisateur.

Les analytics sont collectées **par enfant**, puis agrégées au niveau du Node Composite. Quel que soit le mode de rendu utilisé par l'étudiant (document ou séquentiel), le temps est toujours attribué à l'enfant actif, pas au parent.

### 8.3 Création de decks Anki

Le template "Deck Anki" transforme un Node Composite (ex: un chapitre de cours) en un export CSV compatible Anki en un clic. Le slot `{{children | filter: type = "carte"}}` en mode `csv_row` produit la structure `recto;verso;tags;deck_name`.

Chaque carte reste un Node feuille dans l'arborescence, avec un anchor vers son emplacement dans le cours source.

### 8.4 Exercices d'oraux

Le template "Oral" applique une variante PDF qui assemble les exercices d'un Node Composite en un document PDF paginé. Le prof glisse ses exercices dans le Node, applique le template Oral, et exporte en un clic. Les métadonnées (année, concours, matière) sont injectées automatiquement dans l'en-tête via les slots metadata.

### 8.5 Système de recommandation

Le Node Composite est l'unité que le système de recommandation peut suggérer dans son ensemble (ex : "ce TD de 8 exercices correspond à ton profil") ou partiellement (ex : "les exercices 3 et 7 de ce TD sont les plus pertinents pour toi maintenant"). Le template contrôle la vue "carte de recommandation" via un slot `{{children | recommended_for: current_user | limit: 3}}` disponible dans la vue Explorateur.

### 8.6 Coopération de guilde

Les notes de guilde (annotations collaboratives) sont des Overrides d'instance créés par les membres de la guilde, visibles uniquement aux membres. Ils se superposent au rendu du template sans le modifier. Un toggle dans l'interface permet de les afficher ou masquer.

---

## 9. Modèle de permissions

| Action | Créateur du Node | Membre de guilde | Utilisateur public |
|---|---|---|---|
| Modifier le template | ✅ | ❌ | ❌ |
| Créer un override d'instance | ✅ | (override de guilde seulement) | ❌ |
| Changer la variante d'export | ✅ | ❌ | ❌ |
| Lire la vue Document | ✅ | ✅ (si accès au Node) | ✅ (si public) |
| Publier un template | ✅ | ❌ | ❌ |
| Forker un template | ✅ | ✅ | ✅ |

---

## 10. Questions ouvertes

**Q1 — Conflit de templates dans un rendu imbriqué**
Quand un Node Composite est rendu inline dans le slot d'un parent (ex: un chapitre dans un cours), quel template régit le rendu de ce chapitre ? Son propre template, ou le template du parent qui le contient ? Une règle de priorité doit être définie. Proposition : le template du parent régit la mise en forme de *comment l'enfant s'insère* (taille, espacement, collapsible ou non), tandis que le template de l'enfant régit le rendu *interne* de l'enfant si on le déplie.

**Q2 — Granularité de l'historique des overrides**
Chaque override est versionné dans git. Pour un TD avec de nombreuses annotations d'instance, l'historique peut devenir dense. Doit-on regrouper les overrides dans un commit distinct de type "annotations" pour ne pas polluer l'historique du contenu ?

**Q3 — Template communautaire et maintenance**
Si 200 TDs utilisent un template communautaire publié par un utilisateur, et que cet utilisateur modifie son template, les TDs sont-ils mis à jour automatiquement (comme un package npm) ou figés à la version au moment de l'adoption (fork implicite) ? Les deux options ont des implications très différentes sur la stabilité et la confiance.

**Q4 — Progressive disclosure vs vue Document**
La vue Document affiche l'intégralité des exercices d'un TD. Cela entre en tension avec la feature de gestion de l'accès à la correction et la logique de progressive disclosure. Le template doit pouvoir conditionner l'affichage de chaque exercice à l'état de progression de l'utilisateur (ex: ne montrer l'exercice N+1 qu'une fois l'exercice N soumis). Cette logique conditionnelle doit être expressible dans la syntaxe des slots.

**Q5 — Éditeur de template et courbe d'apprentissage**
L'éditeur de template doit rester accessible à un professeur non-technique. La syntaxe des slots (`{{children | filter: tag = "difficile"}}`) peut être intimidante. Une interface visuelle de configuration des slots (dropdowns, toggles) serait plus inclusive mais moins puissante. Proposition : interface visuelle par défaut, avec un mode "raw" pour les utilisateurs avancés.
