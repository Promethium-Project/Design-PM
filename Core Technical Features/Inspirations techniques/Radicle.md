---
date: 2026-03-28
---

Réflexion : ce serait intéressant d'utiliser Radicle au début, de ne pas chercher à le réimplémenter from scratch. Et quand je l'aurais bien poncé au bout d'un moment je serais en mesure de modifier le code etc...

Prise de note sur le fonctionnement du Protocole Heartwood de Radicle [[https://radicle.xyz/guides/protocol]]

Extension que je vois : 
- Système de contrôle d'accès plus fin en s'inspirant d'initiatives comme git crypt


Il faut également lire les RIPs (Radicle Improvement Proposals) pour bien comprendre le protocol : 

**Entités :**
- Nodes : identifiées par des public keys 


- Les utilisateurs et les nodes ne sont pas distinguables au niveau du protocol (l'user est une node comme une autre). 
	- Les nodes (par extension les users) sont identifiés par **NID : ed25519 pub key**
- Repository identifié par un unique **RID**
- **Seeding Policy :** rules for 
	- List of repo the Node is interested in 
	- Data retention 
	- Synchronisation 
- **NID** : The public part is encoded ans shared as a Decentralized Identifier (DID) (méthode did:key)
	- A noter que l'on peut y associer un alias 
- Protocol : 
	- A gossip protocol 
	- 3 Messages types : 
		- **Node Annoucement :** quand une nouvelle node rejoint le réseau elle envoie ça pour dire aux pairs qu'elle connait, j'existe, voila comment me joindre. 
			- version
			- features
			- timestamp 
			- alias
			- addresses : IP par exemple
			- nonce (DoS protection) :
			- agent
		- **Inventory Announcements :** utlisé pour une node pour diffuser son inventaire. Cela permet au réseau de compléter sa routing table NID -> RID
			- inventory 
			- timestamp 
		- **Reference Annoucements :** permet de diffuser des updates sur des repos (relayés uniquement aux nodes intéressées)
			- rid
			- refs (signées cryptographiquement)
			- timestamp 
	- Structure d'un message (voir quel protocol est utilisé dans RIP-1 TCP? ) : 
		- Node ID 
		- Signature
		- Message 
	- Les échanges sont encryptés : Diffie-Hellman
- Dans Radicle on fait le choix de faire transiter les objets par le protocol Git natif (gossip est utilisé pour s'échanger les métadatas). Transferts Git sur la même connexion que le protocol Gossip. 
	- Il y a une idée de multi plexing, une même connexion est utilisée par 2 protocols (Git + Gossip)
- Il ne faut pas oublier d'ajouter des bootstrap nodes
- Repositories :
	- Instancier avec un identity document -> c'est lui qui permet de dériver le RID 
		- Permet de vérifier l'ownership et les permissions
		- Metadatas (nom + description)

Exemple 'Identity document : 
```json 
{
  "delegates": ["did:key:z6MknSLrJoTcukLrE435hVNQT4JUhbvWLX4kUzqkEStBU8Vi"],
  "threshold": 1,
  "payload": {
    "xyz.radicle.project": {
      "name": "heartwood",
      "description": "Radicle Heartwood Protocol & Stack ❤️🪵",
      "defaultBranch": "master"
    }
  }
}
```

il est stocké dans .git/refs/rad/id

On peut rendre un repo private : 
```json
{
  ...
  "visibility": {
    "type": "private",
    "allow": ["did:key:z6Mkt67GdsW7715MEfRuP4pSZxJRJh6kj6Y48WRqVv4N1tRk"]
  }
}
```

- C'est incomplet (voir début du document)
- RID ( bien développé dans RIP 2): 
	- on fait le hash du initial document (en utilisant la SHA1 de Git)
	- Encodage en base58
- Radicle utilise les namespaces pour gérer la collaboration : 
	- Git supports dividing the refs of a single repository into multiple namespaces, each of which has its own branches, tags, and HEAD. Git can expose each namespace as an independent repository to pull from and push to, while sharing the object store, and exposing all the refs to operations such as [git-gc[1]](https://git-scm.com/docs/git-gc).
		- Il faut le voir comme un soft fork
- Les namespaces sont nommés avec le NID

```
<storage>                     # Storage root containing all repositories
├─ <rid>                      # Storage for first repository
│  └─ refs                    # All Git references locally stored
│     └─ namespaces           # All peer source trees or "namespaces"
│        ├─ <nid>             # First node's source tree
│        │  └─ refs           # First node's Git references
│        │     ├─ heads       # First node's branches
│        │     │   └─ master  # First node's master branch
│        │     └─ tags        # First node's tags
│        │
│        └─ <nid>             # Second node's source tree
│           ├─ refs           # Second node's references
│           └─ ...
├─ <rid>                      # Storage for second repository
│   ...
└─ <rid>                      # etc.
    ...
```

- [[RIP3]] pour en apprendre plus sur le stockage
- Un repo est lié à un delegate (identifié par son DID) (ensemble de proprios)

Git URL scheme : 
- il est custom 

- Authentification :
	- Updating the canonical branch -> histoire de delegate
	- LE RID et l'identity document font office de preuve cryptographique 
	- Les delegates authentifient tous les changements dans les metadatas du repo (c'est donc le cas des refs)
	- les signatures sont stockées dans refs/rad/sigrefs
- https://theupdateframework.github.io/specification/latest/ : c'est un principe qui est documenté ici : pour vérifier la validité d'un fichier, on peut vérifier la validité de toutes les étapes qui l'ont constitué. 

- **Collaboratives objects**
	- Par défaut 'reverse domain name notation): 
		- xyz.radicle.issue
		- xyz.radicle.patch
		- xyz.radicle.id
- Concurrency & Consistency 
	- COBs = set of commits in a DAG 
	- Les cobs sont des objets qui vivent dans le namespace des personnes (Alice Bob). La reconstruction sous forme de thread par exemple est client side

 ```
  ~/.radicle/storage/rad:z3gqcJUoA1n9.../
  ├── objects/                          ← tous les objets Git (commits, blobs, trees)
  │   ├── pack/
  │   └── ...
  └── refs/
      └── namespaces/
          ├── <node-id-Alice>/           ← namespace d'Alice
          │   └── refs/
          │       ├── heads/
          │       │   └── main           ← sa branche de code
          │       └── cobs/
          │           ├── xyz.radicle.id/
          │           │   └── <hash>     ← document d'identité du repo
          │           ├── xyz.radicle.issue/
          │           │   ├── <hash-1>   ← issue #1
          │           │   └── <hash-2>   ← issue #2
          │           └── xyz.radicle.patch/
          │               └── <hash-3>   ← patch proposé
          │
          └── <node-id-Bob>/             ← namespace de Bob
              └── refs/
                  ├── heads/
                  │   └── main           ← sa version du code
                  └── cobs/
                      ├── xyz.radicle.id/
                      │   └── <hash>
                      └── xyz.radicle.issue/
                          ├── <hash-1>   ← même issue #1 (répliquée)
                          └── <hash-2>
 ```
