## Pourquoi

Les COBs git sont append-only et CRDT, mais la propagation P2P est lente. Pour les usages temps-réel (threads de guilde, annotations collaboratives simultanées), écrire un commit git par événement est impraticable.

**Solution** : un relay L2 reçoit les événements en temps réel. Toutes les N secondes, il produit un **checkpoint signé** qui est committé comme un seul commit COB sur L1 (git).

---

## Architecture

```
Clients (Alice, Bob, Carol)
        │  WebSocket (Phase 1) / libp2p (Phase 2)
        ▼
┌───────────────────┐
│   Relay L2        │  ← reçoit, valide, ordonne les événements
│   (event buffer)  │
└────────┬──────────┘
         │  toutes les 5s (ou 50 événements)
         ▼
┌───────────────────┐
│   Checkpoint      │  ← batch signé par delegates
│   builder         │
└────────┬──────────┘
         │
         ▼
   COB git (L1)      ← 1 commit = N événements
```

---

## Format d'un événement L2

Chaque événement est signé par son auteur **avant** d'être envoyé au relay. Le relay ne peut pas forger de messages.

```json
{
  "event_id": "sha256:a1b2c3...",
  "author": "did:key:z6MkAlice...",
  "timestamp": "2026-03-28T14:32:00.412Z",
  "seq": 47,
  "cob_id": "refs/cobs/abc123",
  "op": "thread.reply",
  "payload": {
    "body": "La démonstration du théorème 3 est fausse en page 2.",
    "parent_event_id": "sha256:f9e8d7..."
  },
  "sig": "ed25519:z7Xy..."
}
```

- `seq` : numéro de séquence local à l'auteur (détecte les doublons)
- `sig` : signature Ed25519 de l'auteur sur `{event_id, author, timestamp, seq, op, payload}`
- Le relay rejette tout événement dont la signature est invalide

---

## Format d'un checkpoint (commit COB L1)

```json
{
  "version": "1",
  "type": "pm.promethium.checkpoint",
  "op": "batch",
  "author": "did:key:z6MkRelay...",
  "timestamp": "2026-03-28T14:32:05Z",
  "payload": {
    "prev_checkpoint": "sha256:prev...",
    "events_root": "sha256:merkle_root...",
    "events": [ /* liste ordonnée des événements L2 */ ],
    "delegate_sigs": [
      { "delegate": "did:key:z6MkAlice...", "sig": "ed25519:..." },
      { "delegate": "did:key:z6MkBob...",  "sig": "ed25519:..." }
    ]
  }
}
```

- `events_root` : racine d'un arbre de Merkle sur les événements (permet de prouver qu'un événement est dans le batch sans tout télécharger)
- `delegate_sigs` : ≥ threshold delegates ont signé le Merkle root
- `prev_checkpoint` : chaîne les checkpoints entre eux — l'état L1 est une séquence de batches

---

## Phase 1 — Centralisé

Le serveur PM joue **les deux rôles** : relay L2 et signataire unique des checkpoints.

```
threshold = 1
delegate  = did:key:z6MkServeur...
```

Le serveur reçoit les événements, les ordonne par timestamp serveur, génère un checkpoint signé toutes les 5 secondes (ou si le buffer atteint 50 événements), et le committe dans le COB git.

**Pas de complexité multi-sig** : le serveur est le seul delegate qui signe. Simple à implémenter, migration transparente vers Phase 2.

---

## Phase 2 — P2P

Le relay devient distribué. Les delegates du repo (déjà définis dans `refs/rad/id`) cosignent les checkpoints.

**Problème de coordination** : comment les delegates se mettent-ils d'accord sur l'ordre des événements ?

Solution : **le relay propose, les delegates approuvent.**

```
1. Un relay P2P collecte les événements pendant 5s
2. Il publie un checkpoint candidat (non signé) sur le relay gossip
3. Les delegates vérifient chaque signature d'événement
4. Chaque delegate signe le Merkle root
5. Quand threshold signatures collectées → commit COB L1
```

Si un delegate est offline → le checkpoint attend jusqu'à ce que threshold soit atteint, ou timeout (fallback sur checkpoint partiel si ≥ threshold/2 disponibles, marqué `partial: true`).

---

## Garanties de sécurité

| Menace | Protection |
|---|---|
| Relay forge un message | Impossible — chaque événement est signé par son auteur |
| Relay omet des messages | Détectable — les `seq` par auteur sont continus, un gap = preuve d'omission |
| Relay réordonne malicieusement | Les timestamps auteur + seq limitent la manipulation |
| Relay tombe en panne | Le dernier checkpoint L1 est l'état autoritaire ; le relay repart de là |
| Double-spend d'événement | `event_id` = hash du contenu → idempotent |

---

## Ce qui change dans le modèle de données

**Avant (v2 actuelle)** : chaque message = 1 commit COB git.

**Après** : les COBs à haute fréquence (threads, annotations simultanées) utilisent `pm.promethium.checkpoint` comme type de commit. Les COBs à faible fréquence (erratum, patch, intent) continuent à écrire directement sur L1 — pas besoin de L2 pour eux.

Nouvelle règle dans le protocole client PM :

> Un client qui lit un thread COB doit savoir dépaqueter les commits de type `checkpoint` pour reconstruire la séquence d'événements ordonnés.

---

## Lien avec le LFS / BitTorrent

Le relay L2 gère les **événements** (petits, fréquents). Le swarm BitTorrent gère les **blobs** (gros, statiques). Les deux sont orthogonaux et complémentaires :

- Un message dans un thread qui référence un PDF → l'événement passe par le relay L2, le PDF est servi par le swarm
- Un checkpoint L1 peut contenir des `event_id` qui pointent vers des blobs dans le swarm via leur `sha256`
