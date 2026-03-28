Ce document regroupent les spécifications fonctionnelles et techniques pour Prométhium. 

## Table des matières  
  
1. [Vision et objectifs](#1-vision-et-objectifs)  
	1.1 [Problème à résoudre](#11-problème-à-résoudre)  
	1.2 [Proposition de valeur](#12-proposition-de-valeur)  
	1.3 [Cas d’usage principaux](#13-cas-dusage-principaux)  
	1.4 [Périmètre du projet](#14-périmètre-du-projet)  
  
2. [Personas et parcours utilisateur](#2-personas-et-parcours-utilisateur)  
	2.1 [Personas](#21-personas)  
	2.2 [User journeys](#22-user-journeys)  
	2.3 [Scénarios critiques](#23-scénarios-critiques)  
  
3. [UX et interfaces](#3-ux-et-interfaces)  
	3.1 [Cartographie des écrans](#31-cartographie-des-écrans)  
	3.2 [Description détaillée des écrans](#32-description-détaillée-des-écrans)  
	3.3 [Règles UX et principes d’interaction](#33-règles-ux-et-principes-dinteraction)  
	3.4 [Design system](#34-design-system)  
  
4. [Objets métiers et modèle de données](#4-objets-métiers-et-modèle-de-données)  
	4.1 [Liste des entités principales](#41-liste-des-entités-principales)  
	4.2 [Description des entités](#42-description-des-entités)  
	4.3 [Relations et contraintes](#43-relations-et-contraintes)  
	4.4 [Schéma conceptuel](#44-schéma-conceptuel)  
  
5. [Architecture technique](#5-architecture-technique)  
	5.1 [Vue globale du système](#51-vue-globale-du-système)  
	5.2 [Architecture frontend](#52-architecture-frontend)  
	5.3 [Architecture backend](#53-architecture-backend)  
	5.4 [Services et microservices](#54-services-et-microservices)  
	5.5 [Interfaces et API](#55-interfaces-et-api)  
	5.6 [Flux de données](#56-flux-de-données)  
	  
6. [Architecture data](#6-architecture-data)  
	6.1 [Stockage](#61-stockage)  
	6.2 [Traitement](#62-traitement)  
	6.3 [Indexation et cache](#63-indexation-et-cache)  
	6.4 [Performance et scalabilité](#64-performance-et-scalabilité)  
  
7. [Sécurité et modèle de permissions](#7-sécurité-et-modèle-de-permissions)  
	7.1 [Authentification](#71-authentification)  
	7.2 [Autorisation et rôles](#72-autorisation-et-rôles)  
	7.3 [Protection des données](#73-protection-des-données)  
	7.4 [Audit et traçabilité](#74-audit-et-traçabilité)  
	  
8. [Roadmap et plan de développement](#8-roadmap-et-plan-de-développement)  
	8.1 [MVP](#81-mvp)  
	8.2 [Version 1](#82-version-1)  
	8.3 [Versions futures](#83-versions-futures)  
	8.4 [Jalons](#84-jalons)  
  
9. [Contraintes, coûts et risques](#9-contraintes-coûts-et-risques)  
	9.1 [Contraintes techniques](#91-contraintes-techniques)  
	9.2 [Contraintes organisationnelles](#92-contraintes-organisationnelles)  
	9.3 [Estimation des coûts](#93-estimation-des-coûts)  
	9.4 [Analyse des risques](#94-analyse-des-risques)  
  
10. [Qualité, tests et validation](#10-qualité-tests-et-validation)  
	10.1 [Stratégie de test](#101-stratégie-de-test)  
	10.2 [Critères d’acceptation](#102-critères-dacceptation)  
	10.3 [Suivi qualité](#103-suivi-qualité)  
  
11. [Déploiement et exploitation](#11-déploiement-et-exploitation)  
	11.1 [Environnements](#111-environnements)  
	11.2 [CI/CD](#112-cicd)  
	11.3 [Monitoring](#113-monitoring)  
	11.4 [Maintenance](#114-maintenance)  
  
12. [Annexes](#12-annexes)  
	12.1 [Glossaire](#121-glossaire)  
	12.2 [Références techniques](#122-références-techniques)  
	12.3 [Documents associés](#123-documents-associés)

# 1. Vision et objectifs 

## 1.1. Problème à résoudre 

### 1.1.1 Quel besoin précis?

On cherche à résoudre 4 problèmes distincts : 

- **La difficulté à coopérer efficacement dans les études**. Le problème est d'autant plus fort en classes préparatoires où le manque de temps et la compétitivité rendre l'entraide particulièrement difficile. Le manque de temps poussent des élèves initialement enclins à travailler en groupe à travailler seul car ils ne souhaitent pas se retrouver à envoyer des messages, à planifier des temps où ils peuvent travailler ensemble... La compétition poussent les étudiants à penser que aider un camarade les desservis plus que cela ne les aide.  La pratique semble leur donner raison, la plupart des élèves très haut dans le classement semblent travailler seuls (difficile à fact checker néanmoins). 

- **Le manque d'optimisation des processus.** Les classes préparatoires sont un exemple d'intérêt. En effet, il serait faux d'affirmer que les classes préparatoires ne sont pas optimisées. Les préfets ont un recours importants à des tableurs Excel pour optimiser les décisions liées à leurs établissement, voir quelles méthodes organisationnelles marchent le mieux etc... Mais le cycle d'itération est lent. En effet, la donnée c'est essentiellement les résultats du concours (à savoir 1 fois par an).  L'optimisation sur des échelles de temps inférieures est réalisée par les professeurs. C'est leur expérience qui guide leurs choix d'exercices, et ils affinent d'année en année leurs méthodes, en s'adaptant aussi à la classe. Mais ils manquent de données objectives pour comprendre quels sont les exercices clés dans la progression des élèves (bien qu'il soit clair que leur intuition suffit à faire de bon choix dans un grand nombre de cas). PM propose aux élèves des parcours optimisés pour leur profil. Ils permet aux algorithmes de ne pas se baser sur l'expérience d'un seul professeur mais de tout ceux dont le TD est sur la plateforme. La prise de décision dans son parcours d'étude est data driven. 

- **La rétention des ressources pédagogiques comme frein systémique.** Les établissements considèrent leurs ressources (TDs, annales, méthodes) comme un avantage concurrentiel et refusent de les partager — ce qui est rationnel d'un point de vue théorie des jeux : dans un système à somme nulle où le classement au concours est relatif, partager ses ressources revient à armer ses concurrents. Résultat : les meilleures ressources restent cloisonnées, inaccessibles aux élèves des établissements moins dotés, ce qui creuse les inégalités de résultats entre prépas. PM rompt cet équilibre en rendant le partage mutuellement avantageux : un établissement qui met ses ressources sur la plateforme bénéficie en retour des données d'usage agrégées (quels exercices font progresser, dans quel ordre, avec quel taux de réussite), données qu'il n'aurait jamais pu collecter seul. La valeur perçue du partage doit dépasser la valeur perçue de la rétention.

- **Fragmentation des workflows.** Un étudiant consciencieux jongle en permanence entre son cours papier, ses PDFs, ses notes numériques, ses fiches Anki, son moteur de recherche et ses échanges avec ses camarades. Chaque changement de contexte est une interruption. L'enjeu n'est pas seulement de trouver les bons exercices ou de coopérer — c'est de ne jamais avoir à sortir d'un seul environnement pour travailler. PM cherche à minimiser le nombre de ruptures pour la concentration. 

Quelle métrique PM cherche à optimiser? 
- La quantité d'exercices cherchées par les étudiants en classe préparatoire (on cherche donc à optimiser leurs temps, et augmenter leurs temps de recherche concentrée, sans recours à la correction).
- Le nombre d'interaction productive qu'ils ont avec leurs camarades (on vise plusieurs fois par session de travail).  (% d'étudiant ayant au moins une interaction en vue d'aider les autres par session de travail)
- On cherche à diminuer le temps qu'un étudiant passe à s'organiser (temps passé à rédiger une todo, à naviguer dans les interfaces...)

### 1.1.2 Pour qui?

Ce projet s'adresse aux étudiants en général. Cela étant dit, il est clair que l'audience cible est celle qui cherche à progresser en résolution de problème (le problème est bien moins pertinent en dehors des études scientifiques, à l'exception peut-être de la médecine mais que l'on connait mal). 

Dans un premier temps, s'axer sur la classe préparatoire scientifique que l'on connait bien est la meilleure solution. C'est aussi un milieu très exigent, si le produit ne réponds pas au besoin, c'est la manière la plus rapide de s'en rendre compte. 

Ce projet s'adresse également aux professeurs désireux d'optimiser leurs cours, d'avoir plus de visibilité sur le travail que fournissent leurs élèves ainsi que la possibilité de confronter son travail au reste des professeurs qui sont sur la plateforme. 

### 1.1.3 Pourquoi maintenant?

Plusieurs raisons : 
- La classe préparatoire est encore proche, c'est une manière de cristalliser nos apprentissages avant qu'ils ne soient trop lointains. 
- L'éducation évolue peu, beaucoup de projet edtech qui se lancent, mais beaucoup à l'internationale, ils n'ont pas la rigueur "à la française" qui nous rend monstrueux sur le plan théorique
- Le projet est plus facile à développer que jamais avec le développement des outils agentiques. 
- C'est un projet qui va récolter beaucoup beaucoup de données sur le workflow des étudiants, à l'ère des LLM, ce serait une base de donnée qui aurait beaucoup de valeur car ce genre de dataset est difficile à extraire (formats très hétérogènes, recours au papier...). 

## 1.2. Proposition de valeur 

### 1.2.1 Panorama 

PM est une _everything app_ pour l'étudiant en prépa scientifique, conçue pour ne jamais interrompre sa concentration. L'interface est modulaire et entièrement pilotable au clavier, inspirée d'Obsidian dans sa philosophie mais construite pour les usages spécifiques des études scientifiques.

Les axes de valeur sont les suivants :

**Pour l'étudiant :** un environnement de travail unifié qui absorbe tous les formats (PDF, manuscrit via OCR, vidéo, LaTeX), un système de partage de ressources avec versioning git, des algorithmes de recommandation d'exercices personnalisés, un time tracker comparé aux profils similaires, et une couche sociale structurée en guildes permettant la coopération sans friction.

**Pour le professeur :** un outil de création et de publication de cours avec analytics granulaires — distribution du temps sur chaque exercice, taux de réussite par item, ordre optimal empirique — que son seul jugement ne pourrait jamais produire.

**Pour l'établissement :** l'accès à des données agrégées sur la progression de ses classes, ainsi qu'une visibilité sur la plateforme qui transforme la rétention des ressources d'un avantage compétitif en un coût d'opportunité.

Les killer features sont : 
- Un partage de ressources optimisé avec un git interne. 
- La possibilité de mapper des documents de nature hétérogène entre eux (prise de note sur un pdf, sur une vidéo, deux notes mappées entre elles), via la création d'index sur des documents (spatial, temporel etc...)
- Un système de tags hiérarchiques communautaires pour structurer la connaissance disponible dans la base de donnée globale. 
- Des algorithmes qui analysent chaque interaction / réussite des élèves avec le contenu pour proposer les meilleurs parcours. 

### 1.2.2 Analyse concurrentielle

- **Applications de prise de note privacy first avec du bidirectional linking :**
	- **Obsidian :** Sur le plan de l'UX, c'est le plus gros concurrent. Obsidian est très modulaire : et c'est ce qu'il faut garder dans PM le plus possible. Certes on part de besoin spécifiques (la classe préparatoire) mais les objets que l'on crée doivent rester très généraux pour être appliqués à d'autre domaines (faculté, universités étrangères, médcine...). Les deux problèmes de l'application sont :
		-  Elle n'arrive pas à briser la frontière entre papier et numérique. La gestion de l'OCR, et de mécanisme d'UX pour fondre notes numériques / notes manuscrites.
		-  Elle ne donne pas une bonne UX pour la collaboration. 
	- **Roam Research & Logseq** : ces applications ont été tuées par Obsidian, donc cf ci-dessus.

- **Applications de prises de notes :**
	- **Amphi :** L'app est à ce jour bugée, mets en exergue la difficulté de créer une app de prise de note manuscrite. Ce qui manque c'est la gestion de la communication entre un Drive et la structuration des données manuscrites. Mais l'aspect interface modulaire est aussi très bon. 

- **Mathraining** : Le côté compétitif est ici une redondance. Tous ne veulent pas s'engager dans la compétition. Par ailleurs la classe préparatoire introduit déjà la compétition. PM vise à être un outil de support, pas l'instance qui orchestre la compétition entre les utilisateurs. La compétition sur PM doit être une feature émergente (exemple les users créent un Excel dans un repo pour faire un classement). 

- **Github :** Ne pourrait pas être utilisé pour ce cas d'usage car n'a pas assez de verticales pour gérer les formats de documents hétérogènes et n'abstrait pas assez la complexité de git avec une bonne UX, ce qui est essentiel pour les étudiants qui n'ont pas le temps d'intégrer ce type de complexité et doivent aller droit au but. 

- **Anki** : Aujourd'hui Anki ne propose pas aux étudiants comment passer à l'échelle plus vite. Il est peu utilisé dans les études scientifiques parce que les gens ont du mal à le détacher de l'image de l'outil pour apprendre une liste de vocabulaire. Pourtant on sait qu'il est un outil puissant pour apprendre son cours. Le vrai problème c'est juste la constitution du deck : Anki n'apporte aucune solution pour accélérer le processus. Les LLMs ne fonctionnent pas : chaque étudiant doit laisser son empreinte sur son deck pour que celui-ci soit réellement efficace. PM apporte une vrai solution, un document devient un deck rapidement, et c'est l'élève qui le fait.

- **Notion :** Le logiciel n'a pas vocation à faire une percée dans le monde de l'éducation à cause de son format propriétaire qui est un frein à l'adoption énorme. Ce n'est pas assez modulaire. 

- **Forums prepa.org / mathématiques.net ... :** Quand un élève ne parvient pas à faire un exercice il a 2 solutions :
	- Taper l'exercice sur internet et croiser les doigts pour le trouver. C'est chronophage.
	- Demander à un LLM, mais il ne connait pas le contexte, les connaissances de l'élève, et ça casse fait une cassure dans son workflow.

## 1.3. Cas d'usage principaux 

- Partage rapide entre étudiants au sein de groupes de travail. 
	- Partage de solutions à des problèmes. 
	- Partage de notes sur les cours, de correctifs et erratum. 
	- Partage d'itinéraire suivis dans l'exploration de ressources. 
- Insights sur sa progression et sur celle des autres en temps réel pour optimiser ses processus. 
- Tracking automatique (de temps, de % de couverture, de taux de réussite...). 
- Données d'une granularité très importantes pour les créateurs de ressources (les professeurs)
	- Distribution du temps passé sur un cours
	- Insights sur l'ordre optimal des exercices qui optimise le taux de succès. 

# 2. Personas & Parcours utilisateur 

## 2.1. Personas

### 2.1.1 Élève de prépa scientifique 

_TO DO : Dans cette partie il faudra ajouter des renvoies à la suite pour chaque friction / objectif que l'on résout_

C'est la cible principale. C'est le persona dont on a la meilleure connaissance des besoins.
C'est un profil motivé souvent à l'aise avec les outils numériques mais sans volonté d'y consacrer du temps d'apprentissage (il n'ira pas sur Github...). 

Il jongle en permanence entre cours papier, PDFs, flashcards Anki, échanges WhatsApp et recherches web. On veut résoudre son problème de fragmentation. 

Il est à la fois compétiteur (le classement compte) et conscient que la coopération peut l'aider — mais il n'a pas le temps de l'organiser, il faut que cela soit fait pour lui. 

  **Objectifs :**
  - Trouver rapidement les bons exercices pour progresser sur un point précis.
  - Ne pas avoir à quitter son environnement de travail pour poser une question, chercher une correction ou faire une fiche.
  - Savoir où il en est dans sa progression par rapport à ses objectifs et à ses camarades.
  - Coopérer avec son groupe de travail sans friction de coordination.

  **Points de friction actuels :**
  - Les ressources sont éparpillées sur des drives privés, des groupes WhatsApp, des forums mal indexés.
  - Il n'a aucun signal fiable pour savoir si un exercice est adapté à son niveau ou stratégiquement prioritaire.
  - Passer d'un cours papier à ses notes numériques casse sa concentration.
  - Aider un camarade coûte du temps et lui semble le désavantager dans la compétition.
  - Constituer un deck Anki manuellement est trop chronophage pour être fait sérieusement.

  **Valeur attendue de PM :**
  - Un environnement unique qui absorbe PDF, cours manuscrits (OCR), vidéos et notes — sans jamais avoir à switcher d'outil.
  - Des recommandations d'exercices basées sur les données réelles de progression de milliers d'élèves, pas sur l'intuition seule.
  - Une couche sociale intégrée : poser une question ancrée précisément dans un document, voir les annotations de sa guilde, partager une solution sans sortir de son flux.
  - La création rapide de flashcards depuis ses propres annotations sur un cours.
  - Un time tracker comparatif pour calibrer ses sessions et identifier ses points de blocage.

  **Critères de succès :**
  - Il réalise plus d'exercices par session sans augmenter son temps de travail.
  - Il a au moins une interaction productive avec un camarade à chaque session de travail.
  - Il passe moins de 5 minutes par session à s'organiser (navigation, TODO, recherche de ressource).



### 2.1.2 Professeur 

C'est le profil qui a le plus grand levier de croissance pour ce projet : convaincre un professeur de premier plan d'utiliser l'outil pour son cours est une nécessité. 

Un professeur a constitué au fil des années un corpus de TDs et d'exercices dont il est fier. Il adapte son cours chaque année à sa classe, mais cette adaptation repose sur son intuition — il sait que certains exercices font progresser, d'autres pas, mais n'a pas de données pour le valider. Il est **méfiant** vis-à-vis du partage de ses ressources, qu'il considère comme un **avantage concurrentiel pour ses élèves**.

**Objectifs :**
- Comprendre où ses élèves bloquent réellement, avec une granularité que les copies de colle ne lui donnent que sur une autre temporalité.
- Améliorer l'ordre et la sélection de ses exercices en s'appuyant sur des données empiriques.
- Maintenir son cours à jour et corriger les erreurs signalées par ses élèves efficacement.
- Potentiellement, avoir de la visibilité sur la qualité de ses ressources comparées à celles d'autres établissements.

**Points de friction actuels :**
- Le seul signal de feedback qu'il reçoit est le classement annuel au concours — délai d'un an, signal agrégé et peu actionnable.
- Il ne sait pas combien de temps ses élèves passent sur chaque exercice, ni lesquels ils abandonnent.
- Les erratas et corrections remontent par email ou à l'oral, sans traçabilité.
- Partager ses ressources revient à armer les établissements concurrents — c'est rationnel de ne pas le faire dans le
système actuel.

**Valeur attendue de PM :**
- Des analytics granulaires sur l'usage de son cours : temps passé par section, taux de réussite par exercice, ordre empiriquement optimal.
- Un flux de retours structurés de ses élèves (questions ancrées dans le document, errata proposés formellement).
- Un système de versioning de ses ressources : commits pour affiner son cours, PRs pour intégrer les contributions de ses élèves (exercices d'oraux, corrections alternatives).
- Un échange mutuellement avantageux : partager ses ressources lui donne accès à des données qu'il n'aurait jamais pu
collecter seul.

**Critères de succès :**
- Il peut identifier en moins de 10 minutes les passages les plus bloquants d'un TD après une session de travail de sa classe.
- Il reçoit et traite les errata de manière structurée, sans perte d'information.
- Il perçoit la valeur du partage supérieure à la valeur de la rétention.

## 2.2. User journeys

### 2.2.1 Un utilisateur qui travaille sur son propre cours 
- OCR / Upload de PDF [[Supérieur/Design PM/Features/Gestion de l'OCR]]
- Hover overlays pour avoir les variables 
- Eventuellement intégration LLM 
- Utilisation de la fonctionnalités pour poser des questions. 
- Ajoute un erratum
- Création de deck Anki / partage...
- Son temps a été tracké, il a un hoverlay pour voir sur le cours où est-ce qu'il a le plus trainé, et ça peut influer directement ses recommandations de cartes sur Anki
### 2.2.2 Un utilisateur qui travaille sur un poly d'exercice 
- Son parcours est recommandé
- Il peut poser des questions
- Il cherche
- Il propose une sol
- Il a accès à la sol des autres qu'il peut mettre en signet 
- Il a un impact sur l'algo 
- Il planifie sa session suivante en disant les exercices qu'il va aborder 
- Il regarde la progression de ses camarades dans les trees.
- Il peut faire une review plus tard.
### 2.2.3 Un utilisateur qui veut s'entrainer sur des exercices supplémentaires
- L'algo lui recommande
- il cherche / pose des questions aide à mapper mieux 
### 2.2.4 Un professeur qui propose son cours sur la plateforme
- Il pose son cours (un pdf n'importe quoi)
- Là il a un travail pour tagger correctement son cours
- mettre des prérequis
- ses élèves commencent à le tryhard, il peut suivre leurs progression 
- voir où sont les passages les plus bloquants
- il réponds aux questions 
- Régulièrement, il fait des commits pour affiner son cours 
- Ses élèves font des PR pour mettre des exercices d'oraux relatifs à sa matière par exemple

### 2.2.5 

# 3. UX et interfaces

## 3.1. Cartographie des écrans

## 3.2 Description des écrans 

## 3.3 Règles UX 


# 4. Objets métiers 

Problèmes : 
- Comment gérer les subfolders? 
	- Si un prof décide d'upload tout son cours, comment recommander des chapitres? 
		- Pas nécéssaire de faire des subgits? 
		- Juste faire sur le moteur de recherche? 
		- Quid des decks anki? 
- Les tags hiérarchiques ne sont pas très au point 


## 4.1 Liste des entités principales 

## 4.2 Modèle de données 

Pour chaque 


# 5. Architecture technique 
## 5.1 Vue globale

## 5.2 Composants

On fait la liste de tous les services dans le backend

## 5.3 Flux de données 

Des parcours de données 

# 6. Data Architecture 

## 6.1 Stockage
## 6.2 Traitement
## 6.3 Performance

# 7. Sécurité & modèle de permission 

## 7.1 Auth 

## 7.2 Autorisation 

## 7.3 Sécurité & Data 


# 8. Roadmap 


# 9. Contraintes & coûts 

