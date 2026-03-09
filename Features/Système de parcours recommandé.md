---
tags:
  - Algorithmes
  - UX
Spec rédigée: Non rédigée
---
Ajouter les recommanded path sur les TD, faire un truc visuel sympa

_S'appuie sur le système de prérequis que l'on peut mettre dans les métadatas avec un attribut parent pour chaque fichier par exemple_



Quand un mec réussit un exo, (qu’il soumet une correction par exemple ou qu’il clique j’ai réussi (à voir pour avoir le bon critère) ajouter une edge dans un graph pour dire que lexo précédent lui a permis de réussir celui-ci

En moyennant sur bcp de personnes (modulo fonctions de poids etc…) on peut construire un tri topologique du TD

On peut même personnaliser le tri pour des profils similaires

Ça c’est rendu possible par un bon système de collecte de données, et après la possibilité de développer des algos qui créels principaux objets du site en fonction de ces données

_ça c'est donc pas en dur dans les recommanded path, mais inter dossiers, et non supervisé !_


Dans la gestion des pré requis, on peut ajouter des lemmes, de sorte que un exo qui fait démontrer un lemme soit placé avant dans le tri topologique des exos qui sont utiles.

On déclare un objet lemme / ou ça peut être plus général (une méthode etc…)

_c'est des objets plus abstraits, donc plus difficiles à gérer mais si ils peuvent être intégrés dans l'algorithme c'est très cool_