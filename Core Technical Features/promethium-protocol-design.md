# PromethiumChain — Design d'un protocole décentralisé pour Promethium

**Statut :** Proposition technique  
**Date :** 2026-03-11  
**Dépendances :** Node/Anchor primitives, Système git, Modèle de permissions, Système de tags collaboratif

---

## 0. Résumé exécutif

Ce document propose l'architecture d'un **protocole décentralisé** remplaçant git comme couche de versioning et de distribution pour Promethium. L'idée centrale est une séparation nette entre :

- **L1 (la chaîne de données)** : un log append-only, content-addressed, qui constitue la source de vérité pour tout l'état de Promethium (Nodes, Anchors, tags, permissions, identités). Toute mutation est un **commit signé** cryptographiquement, immuable une fois publié. C'est le "settlement layer".
- **L2 (les services temps réel)** : des backends interchangeables (chat, notifications, présence, collaboration live) qui opèrent en mémoire/temps réel puis **ancrent** périodiquement leur état sur la L1. N'importe qui peut fournir un L2 alternatif ; la donnée finit toujours sur la L1.

Ce modèle s'inspire de Radicle (P2P git avec gossip protocol et identités cryptographiques), d'IPFS (content-addressing via CID, Merkle DAGs), et de l'architecture L1/L2 des blockchains (séparation settlement/execution).

---

## 1. Pourquoi pas git tel quel ?

Git est déjà au cœur du design actuel de Promethium (versioning des dossiers, branches pour les profs, PRs pour les élèves). Mais git ne couvre pas nativement plusieurs besoins critiques :

**1.1 Pas de modèle d'identité natif.** Git repose sur `user.name` + `user.email`, qui sont auto-déclarés et non vérifiés. Pour un système social avec permissions par guilde, par couche d'Anchor, et par rôle (prof/élève), il faut des identités cryptographiques vérifiables — des keypairs Ed25519 comme dans Radicle.

**1.2 Pas de contrôle d'accès natif.** Git est "all or nothing" sur un repo. Les permissions fines (un élève peut poser un Anchor de type `forum` mais pas modifier le cours source ; une guilde peut voir les Anchors de ses membres mais pas ceux d'une autre guilde) nécessitent une couche d'autorisation intégrée au protocole, pas ajoutée par un serveur centralisé.

**1.3 Pas de social primitives.** Les issues, PRs, et discussions de GitHub sont des features propriétaires côté serveur, pas des objets git. Radicle a résolu ça avec ses **Collaborative Objects (COBs)** — des artefacts sociaux stockés comme objets git. Promethium peut aller plus loin : puisque "tout est un Node", les artefacts sociaux (threads de forum, intents, notes de guilde) sont déjà des Nodes versionnés.

**1.4 Pas de support pour les formats binaires riches.** Git gère mal les diffs de PDFs, d'images OCR, de vidéos. Promethium manipule des documents hétérogènes où le content-addressing (hash du contenu → CID) est plus adapté que le SHA-1 d'un blob git.

**1.5 Pas de mécanisme pour les données temps réel.** Le chat de guilde, la présence, les notifications d'intent — ces flux doivent opérer en temps réel puis être consolidés. Git n'a pas de concept de "state channel" ou de "rollup".

---

## 2. Modèle d'identité

### 2.1 Identité de nœud (NodeIdentity)

Chaque participant au réseau possède une paire de clés Ed25519. La clé publique encodée en DID (`did:key:z6Mk...`) constitue l'identité souveraine. Aucun serveur central, aucun email requis.

```
NodeIdentity {
  did: DID                    // did:key:z6Mk... (Ed25519 public key encodée)
  signing_key: Ed25519Pub     // clé publique de signature
  encryption_key: X25519Pub   // clé publique de chiffrement (dérivée ou séparée)
  display_name: string        // nom affiché (non unique, non vérifié)
  created_at: timestamp
  metadata: Map<string, string>  // bio, avatar CID, etc.
}
```

Le DID est l'identifiant universel. Toutes les opérations (commit, anchor, message, vote sur un tag) sont signées par la clé privée correspondante. Un pair qui reçoit un commit peut vérifier son authenticité sans serveur tiers.

### 2.2 Identité de projet (RepoIdentity)

Un repo Promethium (équivalent d'un dossier/cours) possède aussi une identité cryptographique, définie par un **identity document** signé par ses delegates (le prof créateur + les co-mainteneurs éventuels).

```
RepoIdentity {
  id: RepoID                  // hash du document d'identité initial
  name: string
  description: string
  delegates: DID[]            // qui peut publier sur la branche canonique
  threshold: int              // nombre de signatures requises pour un merge (1 = single owner, N = multisig)
  default_permissions: PermissionSet  // permissions par défaut pour les non-delegates
  created_at: timestamp
  identity_history: CommitID[]  // historique des changements du doc d'identité
}
```

**Pourquoi un threshold ?** Un prof seul a `threshold: 1`. Un département qui maintient un cours commun peut exiger `threshold: 2` (double validation). C'est le multi-sig appliqué à l'éducation.

### 2.3 Identité de guilde

La guilde étant un Node composite (cf. rapport Node+Anchor), son identité suit le même schéma qu'un repo :

```
GuildIdentity {
  id: GuildID                 // hash du doc d'identité
  name: string
  members: DID[]              // liste des membres
  admins: DID[]               // sous-ensemble de members avec droits admin
  permissions_policy: GuildPermissionPolicy
  created_at: timestamp
}
```

La liste des membres est elle-même versionnée — ajouter ou retirer un membre est un commit signé par un admin.

---

## 3. Architecture L1 — la chaîne de données

### 3.1 Principe fondamental

La L1 est un **DAG (Directed Acyclic Graph) content-addressed**. Chaque mutation de l'état de Promethium est un **PromCommit** — un objet immuable dont l'identifiant est le hash de son contenu (comme un commit git, mais étendu).

```
PromCommit {
  id: CID                       // hash(contenu) — identifiant content-addressed
  author: DID                   // qui a produit ce commit
  signature: Ed25519Sig          // signature du commit par l'auteur
  parents: CID[]                 // commits parents (forme le DAG)
  timestamp: timestamp
  repo_id: RepoID               // repo concerné
  operations: Operation[]        // liste des mutations atomiques
  state_root: CID               // Merkle root de l'état résultant après ce commit
}
```

### 3.2 Opérations (mutations)

Chaque opération dans un commit décrit une mutation atomique. Le typage est exhaustif :

```
Operation =
  | NodeCreate   { node: Node }
  | NodeUpdate   { node_id: NodeID, patch: NodePatch }
  | NodeDelete   { node_id: NodeID }
  | AnchorCreate { anchor: Anchor }
  | AnchorUpdate { anchor_id: AnchorID, patch: AnchorPatch }
  | AnchorDelete { anchor_id: AnchorID }
  | TagPropose   { tag: Tag, position: TagTreePosition }
  | TagVote      { tag_id: TagID, vote: up | down }
  | PermGrant    { target: DID, permission: Permission, scope: Scope }
  | PermRevoke   { target: DID, permission: Permission, scope: Scope }
  | BlobStore    { cid: CID, size: uint64, mime: string }
  | ThreadReply  { thread_id: NodeID, content: Node }
  | IntentDeclare { intent: Intent }
  | IntentResolve { intent_id: NodeID, outcome: completed | expired | cancelled }
```

**Propriété critique** : chaque opération inclut implicitement le `repo_id` et le `author` du commit parent. Les validateurs du réseau peuvent vérifier que l'auteur a bien le droit d'exécuter l'opération (cf. section 5).

### 3.3 Stockage content-addressed

Les blobs (PDFs, images, vidéos, scans OCR) sont stockés dans un store content-addressed séparé du DAG de commits. Le commit référence le blob par son CID.

```
BlobStore:
  CID → bytes

CommitStore:
  CID → PromCommit

StateStore:
  (RepoID, CID) → MerkleTree<NodeID, Node>  // état complet du repo à un commit donné
```

**Analogie IPFS** : le BlobStore fonctionne comme un nœud IPFS local. Les blobs sont découpés en chunks, hashés en Merkle DAG. Deux uploads du même PDF produisent le même CID → déduplication native.

**Analogie git** : le CommitStore est l'équivalent du `.git/objects` mais avec des CIDs multihash au lieu de SHA-1, et des opérations typées au lieu de tree/blob/commit.

### 3.4 Merkle State Tree

À chaque commit, l'état complet du repo (tous ses Nodes, Anchors, permissions) est représenté comme un **Merkle Patricia Tree** dont la racine est le `state_root` du commit.

Cela permet :
- **Vérification rapide** : deux pairs peuvent comparer leur `state_root` pour un même commit. S'il diffère, il y a une divergence (corruption ou fork malveillant).
- **Sync incrémental** : un pair qui rejoint le réseau n'a pas besoin de rejouer tous les commits. Il peut demander l'état courant (state tree) et vérifier son intégrité via la racine Merkle.
- **Preuves d'inclusion** : un pair peut prouver qu'un Node spécifique existe dans l'état du repo à un commit donné, sans révéler le reste de l'état. Utile pour les permissions (prouver qu'on est membre d'une guilde sans révéler les autres membres).

### 3.5 Résolution des conflits

Contrairement à une blockchain avec consensus global, Promethium n'a pas besoin de résoudre le double-spending. Deux cas de conflit :

**Cas 1 — commits concurrents sur le même repo.** Si deux delegates d'un même repo publient simultanément, le DAG a deux branches. La résolution suit le modèle git : merge automatique si pas de conflit sur les mêmes Nodes, merge manuel sinon. Le delegate fait un commit de merge signé.

**Cas 2 — fork malveillant.** Un pair essaie de publier un commit qui contredit l'historique (modifier un Node qu'il n'a pas le droit de toucher). Les autres pairs vérifient la signature + les permissions et rejettent le commit. Pas de consensus PoW/PoS nécessaire : la vérification est déterministe à partir du `RepoIdentity.delegates` et du `PermissionSet`.

**Règle de canonicité** : la branche canonique d'un repo est déterminée par les commits signés par les delegates qui satisfont le threshold. C'est exactement le modèle de Radicle.

---

## 4. Architecture L2 — les services temps réel

### 4.1 Le problème

Certaines interactions dans Promethium sont temps réel :
- Chat de guilde
- Notifications d'intent ("je vais faire ces exos ce soir")
- Présence (qui est en ligne, qui travaille sur quoi)
- Collaboration live (deux élèves annotent le même document)
- Système de correction (soumissions, échanges entre correcteurs)

Ces flux ne peuvent pas passer par la L1 : un commit signé + propagation gossip pour chaque message de chat serait absurde en latence et en volume.

### 4.2 Le pattern L2

Un service L2 est un **serveur éphémère** qui :
1. Gère un flux temps réel (WebSocket, WebRTC, CRDT, peu importe le protocole).
2. Maintient un **state buffer** en mémoire.
3. **Ancre** périodiquement (toutes les N minutes, ou à la fermeture de session) une snapshot de cet état sur la L1 sous forme de PromCommit.

```
L2Service {
  service_type: "chat" | "presence" | "live_collab" | "correction" | string
  endpoint: URL                // point d'accès WebSocket/HTTP
  provider: DID                // qui opère ce service
  target_repo: RepoID          // repo concerné (ou guild_id)
  anchor_frequency: duration   // fréquence d'ancrage sur L1
  state_schema: CID            // schema des données que ce L2 produit (permet la validation)
}
```

**L'ancrage** se fait en produisant un commit avec des opérations `NodeCreate` / `NodeUpdate` qui reflètent les messages échangés, les annotations live, etc.

### 4.3 Interchangeabilité des L2

C'est le point clé de ton idée : **n'importe qui peut fournir un L2 alternatif** à condition qu'il ancre ses données sur la L1 dans le format attendu.

Concrètement :
- Tu fournis un service de chat basé sur WebSocket + Redis. Il ancre les messages comme des Nodes de type `thread_message` dans le commit.
- Un tiers peut fournir un service de chat basé sur Matrix/XMPP. Tant qu'il ancre les messages avec les mêmes opérations (`ThreadReply`), les données sont compatibles.
- Un autre tiers peut fournir un service de collaboration live basé sur CRDTs (Yjs, Automerge). Il ancre les modifications résultantes comme des `AnchorUpdate` sur la L1.

```
┌─────────────────────────────────────────────────────┐
│                    CLIENTS                           │
│  (Web app, Desktop, Mobile, CLI)                     │
└──────────┬──────────────────┬───────────────────────┘
           │                  │
           │ WebSocket        │ P2P gossip
           ▼                  ▼
┌──────────────────┐  ┌──────────────────┐
│   L2 : Chat      │  │   L2 : Collab    │
│   (provider A)   │  │   (provider B)   │
│   WebSocket+Redis│  │   Yjs CRDT       │
└────────┬─────────┘  └────────┬─────────┘
         │  anchor             │  anchor
         ▼                     ▼
┌─────────────────────────────────────────┐
│              L1 : PromChain             │
│   DAG content-addressed + Merkle state  │
│   gossip P2P entre nœuds Promethium     │
└─────────────────────────────────────────┘
```

### 4.4 Garanties

- **Disponibilité** : si un L2 tombe, les données déjà ancrées sont sur la L1. Un autre L2 peut reprendre.
- **Vérifiabilité** : l'ancrage est un commit signé par le DID du provider L2. Les utilisateurs peuvent vérifier que le provider n'a pas altéré les messages.
- **Permissivité** : le L2 n'a pas de droits supérieurs à ceux de ses utilisateurs. Il relaie des opérations signées par les utilisateurs eux-mêmes. Le L2 est un facilitateur de latence, pas une autorité.

### 4.5 State channels pour les guildes

Le chat de guilde est un cas d'usage naturel pour un **state channel** :

1. La guilde ouvre un canal (un Node de type `state_channel` sur la L1, signé par un admin).
2. Les membres échangent des messages via le L2. Chaque message est signé par son auteur.
3. Périodiquement, le L2 ancre un batch de messages sur la L1.
4. À la fermeture du canal (ou en cas de litige), l'état complet est publié sur la L1.

Un membre qui quitte la guilde n'a plus accès au canal L2, mais les messages déjà ancrés sur la L1 sont soumis aux permissions du repo (cf. section 5).

---

## 5. Modèle de permissions intégré au protocole

### 5.1 Principes

Les permissions ne sont pas un service séparé — elles sont **vérifiées par chaque pair** à la réception d'un commit. Un commit qui contient une opération non autorisée est rejeté par le réseau.

### 5.2 Permission primitives

```
Permission =
  | Read          // voir le contenu d'un Node
  | Write         // modifier un Node existant
  | Create        // créer un nouveau Node/Anchor
  | Delete        // supprimer un Node/Anchor
  | Admin         // modifier les permissions, ajouter/retirer des delegates
  | Anchor        // poser un Anchor sur un document (peut être restreint par type)
  | Tag           // proposer/voter sur des tags
  | Delegate      // agir au nom du repo (merge, release)

Scope =
  | Repo(RepoID)
  | Guild(GuildID)
  | Node(NodeID)           // permission sur un Node spécifique
  | AnchorType(string)     // permission limitée à un type d'Anchor
  | Layer(LayerType)       // permission limitée à une couche
```

### 5.3 Rôles prédéfinis

Les rôles sont des ensembles nommés de permissions, définis dans le `RepoIdentity` :

```
Roles:
  owner:      [Read, Write, Create, Delete, Admin, Anchor, Tag, Delegate]
  maintainer: [Read, Write, Create, Delete, Anchor, Tag, Delegate]
  contributor:[Read, Create, Anchor, Tag]        // peut créer des Nodes (solutions, questions) mais pas modifier ceux des autres
  student:    [Read, Anchor(forum|signet|note|anki), Tag]  // peut lire, annoter personnellement, poser des questions
  viewer:     [Read]
```

Un prof qui crée un cours est `owner`. Ses élèves sont `student` par défaut. Un élève qui contribue activement (solutions validées, erratums acceptés) peut être promu `contributor`.

### 5.4 Permissions par couche (Layer)

Les Anchors opèrent dans des couches. Les permissions de couche sont additives :

- **personal** : tout utilisateur `student+` peut créer/modifier ses propres Anchors personnels.
- **guild** : tout membre de la guilde peut créer des Anchors de guilde. Seul l'auteur de l'Anchor peut le modifier.
- **public** : les Anchors forum sont visibles par tout utilisateur avec `Read` sur le repo. La création requiert `Anchor(forum)`.

### 5.5 Vérification

À la réception d'un commit, chaque pair exécute :

```
pour chaque operation dans commit.operations:
  author = commit.author
  repo = commit.repo_id
  role = lookup_role(author, repo)  // depuis le state tree courant
  if not has_permission(role, operation.required_permission, operation.scope):
    REJECT commit
    return
ACCEPT commit
```

Cette vérification est **déterministe** : tous les pairs arrivent à la même conclusion. Pas de consensus probabiliste nécessaire.

---

## 6. Réseau P2P — gossip et réplication

### 6.1 Topologie

Le réseau Promethium est composé de **nœuds** qui exécutent le protocole. Chaque nœud :
- Stocke les repos qu'il suit (seed).
- Propage les commits via un **protocole gossip**.
- Vérifie les signatures et permissions des commits reçus.

Trois types de nœuds :

| Type | Rôle | Stockage | Toujours en ligne |
|------|------|----------|-------------------|
| **Full node** | Stocke tout un repo + valide | Complet | Oui (serveurs, seeders dédiés) |
| **Light node** | Client utilisateur, stocke ses propres données | Partiel | Non |
| **Seed node** | Haute disponibilité, point d'entrée réseau | Complet, multi-repos | Oui |

### 6.2 Gossip protocol

Inspiré de Radicle et Secure Scuttlebutt. Les nœuds échangent des **annonces** :

```
Announcement {
  repo_id: RepoID
  latest_commit: CID
  state_root: CID
  peer_id: DID
  timestamp: timestamp
  signature: Ed25519Sig
}
```

Quand un nœud reçoit une annonce avec un `latest_commit` qu'il ne connaît pas, il initie un fetch des commits manquants (delta sync, comme git fetch).

### 6.3 Connexions sécurisées

Les connexions entre pairs utilisent le **Noise XK handshake** (identique à Radicle et au Lightning Network). Chaque pair authentifie l'autre via son DID avant tout échange de données.

### 6.4 Découverte de repos

Un annuaire décentralisé (lui-même un repo spécial sur le réseau) référence les repos publics avec leurs métadonnées (nom, tags, description, nombre de seeders). C'est l'équivalent du browsing HuggingFace que tu vises — mais sans serveur central.

Pour le MVP, un ou plusieurs seed nodes opérés par Promethium servent de point d'entrée. La décentralisation complète de la découverte est une étape ultérieure.

---

## 7. Mapping sur les primitives existantes

### 7.1 Node → objet dans le Merkle State Tree

Un Node Promethium (cours, exercice, solution, thread, carte Anki, profil utilisateur, guilde) est un objet dans le state tree du repo. Son `NodeID` est le CID de sa création initiale. Ses versions successives sont des `NodeUpdate` dans les commits.

### 7.2 Anchor → relation dans le state tree

Un Anchor est stocké dans le state tree comme une relation entre deux NodeIDs (ou un NodeID + une position). Son AnchorID est le CID de sa création.

### 7.3 Tag tree → repo spécial

Le tag tree global est un repo partagé avec des règles de permissions spécifiques : tout utilisateur peut proposer (`TagPropose`), le merge dans le tree global est décidé par un algorithme de consensus communautaire (cf. spec existante "merge des perceptions").

### 7.4 Git existant → migration

Les repos actuellement en git peuvent être importés en convertissant chaque commit git en `PromCommit`. L'historique est préservé. Les fichiers deviennent des Nodes avec `BlobStore` entries.

---

## 8. Comparaison avec les alternatives

| Critère | Git + serveur central | Radicle | IPFS + smart contract | PromChain (cette proposition) |
|---------|----------------------|---------|----------------------|-------------------------------|
| Identité | email (non vérifié) | DID Ed25519 | wallet Ethereum | DID Ed25519 |
| Contrôle d'accès | côté serveur (GitHub) | delegates sur le repo | smart contract | intégré au protocole, vérifié par tous les pairs |
| Social primitives | propriétaire (GitHub Issues) | COBs (issues, patches) | aucune native | Node/Anchor (tout est social nativement) |
| Formats riches | blobs opaques | blobs git | CID content-addressed | CID + AnchorPosition polymorphique |
| Temps réel | webhooks | aucun | aucun | L2 interchangeables |
| Consensus | aucun (confiance au serveur) | delegates + threshold | PoS/PoW | delegates + threshold (pas de blockchain coûteuse) |
| Token/crypto | non | RAD token (optionnel) | obligatoire (gas fees) | **aucun** (pas de token, pas de fees) |

**Point important** : cette proposition n'introduit pas de token ni de cryptocurrency. La "blockchain" ici est une structure de données (DAG content-addressed avec Merkle proofs), pas un réseau financier. L'incitation à participer est l'accès aux données et aux services de la plateforme, pas une récompense monétaire.

---

## 9. Workflow concret — exemple du prof qui publie un TD

1. Le prof génère sa keypair Ed25519 → obtient son DID `did:key:z6MkProf...`.
2. Il crée un repo `Analyse-MP2-2026` avec `threshold: 1`, `delegates: [son DID]`.
3. Il upload son PDF de TD → le PDF est chunké, hashé → `CID:QmTD...` stocké dans le BlobStore.
4. Il crée un Node `{type: "exercise_set", blob: CID:QmTD...}` → opération `NodeCreate` dans un `PromCommit` signé.
5. Il publie le commit sur le réseau gossip → les seed nodes le propagent.
6. Ses élèves (membres de la guilde `MP2-LLG-2026`) reçoivent le commit, vérifient la signature, stockent le repo.
7. Un élève travaille un exercice, pose un Anchor forum → opération `AnchorCreate(type: forum)` dans un commit signé par le DID de l'élève. Le réseau vérifie que l'élève a la permission `Anchor(forum)` sur ce repo (il l'a, car il est `student`).
8. Un autre élève répond via le L2 chat → la réponse est relayée en temps réel. Après 5 minutes, le L2 ancre le thread sur la L1.
9. Le prof fait un nouveau commit avec `NodeUpdate` pour corriger une erreur dans le TD → le système de stale-ité détecte les Anchors affectés.

---

## 10. Stack technique proposée

| Couche | Technologie | Justification |
|--------|------------|---------------|
| Identité | Ed25519 (libsodium) | Standard, rapide, utilisé par Radicle et SSH |
| Hashing | SHA-256 + Multihash (CIDv1) | Compatible IPFS, auto-descriptif |
| Sérialisation | CBOR (DAG-CBOR) | Standard IPLD, compact, schemaless |
| Transport P2P | Noise XK + TCP/QUIC | Authentifié, chiffré, éprouvé |
| Gossip | protocole custom inspiré Radicle | Adapté aux repos éducatifs (peu de throughput, haute intégrité) |
| State tree | Merkle Patricia Trie (en Rust ou Go) | Preuves d'inclusion, sync incrémental |
| L2 chat | WebSocket + CRDT (Yjs) | Temps réel, offline-first, merge automatique |
| L2 collab | Yjs + WebRTC | Édition collaborative P2P directe |
| Stockage local | SQLite + fichiers (blobs) | Simple, embarqué, cross-platform |
| Client | Rust core + bindings (WASM pour web, FFI pour desktop/mobile) | Performance, portabilité |

---

## 11. Questions ouvertes et risques

**11.1 — Scalabilité du Merkle state tree.** Un repo de cours actif avec des milliers d'Anchors de milliers d'étudiants va produire un state tree volumineux. Solutions possibles : pruning des Anchors personnels au-delà d'un horizon temporel, ou sharding du state tree par couche (personal, guild, public).

**11.2 — Disponibilité sans seed nodes.** En full P2P, si aucun pair n'est en ligne, le repo est inaccessible. En pratique, Promethium devra opérer des seed nodes pour garantir la disponibilité — ce qui reintroduit une forme de centralisation. Mitigation : tout utilisateur peut devenir seed node, et les données restent vérifiables même si le seed est malveillant.

**11.3 — Latence de propagation gossip.** Le gossip n'a pas la latence d'un serveur central. Un élève qui pose une question ne verra pas la réponse instantanément si elle passe par la L1. C'est précisément le rôle du L2 : offrir la latence basse, et ancrer ensuite.

**11.4 — Adoption.** La complexité du protocole est cachée derrière le client (l'UX d'Obsidian-like). L'utilisateur ne voit jamais de CID ni de DID. Mais les développeurs tiers qui veulent fournir un L2 doivent comprendre le protocole. Documentation et SDK sont critiques.

**11.5 — Révocation de clés.** Si un élève perd sa clé privée, comment révoquer son identité ? Solution : un mécanisme de rotation de clés inscrit dans le protocol (un nouveau commit avec une opération `KeyRotate` signée par l'ancienne clé, ou par un delegate du repo/de la guilde en cas de perte totale).

**11.6 — Conformité RGPD.** Les données sur la L1 sont immuables. Le droit à l'effacement est incompatible avec un log append-only. Solutions : chiffrement des données personnelles avec clé utilisateur (l'effacement = destruction de la clé), ou soft-delete (marquage, pas suppression physique) avec garbage collection après expiration.

**11.7 — Taille des blobs.** Les PDFs et vidéos sont volumineux. Le réseau gossip ne devrait pas propager les blobs, seulement les commits (qui référencent les blobs par CID). Le fetch des blobs se fait à la demande, comme IPFS Bitswap.

---

## 12. Roadmap d'implémentation

**Phase 0 — Fondations (semaines 1-8)**
- Librairie crypto : génération de keypairs, signatures, vérification.
- Format PromCommit : sérialisation CBOR, hashing CIDv1.
- BlobStore local : stockage content-addressed sur filesystem.
- CLI minimale : `prom init`, `prom commit`, `prom log`.

**Phase 1 — Réseau (semaines 9-16)**
- Protocole gossip : annonces, fetch de commits, Noise XK handshake.
- Seed node : réplication complète des repos suivis.
- Sync entre deux pairs : delta sync basé sur les CIDs manquants.

**Phase 2 — Permissions & état (semaines 17-24)**
- Merkle state tree : construction, vérification, preuves d'inclusion.
- Vérification des permissions à la réception des commits.
- RepoIdentity et GuildIdentity : création, modification, rotation de delegates.

**Phase 3 — L2 MVP (semaines 25-32)**
- L2 chat : WebSocket server + ancrage périodique sur L1.
- Spécification de l'interface L2 : schema d'ancrage, protocole d'enregistrement.
- SDK pour développeurs L2 tiers.

**Phase 4 — Intégration client (semaines 33+)**
- Binding WASM pour le client web Promethium.
- Migration des repos git existants vers PromChain.
- UX : toute la complexité crypto est invisible pour l'utilisateur.
