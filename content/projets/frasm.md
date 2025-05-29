+++
date = '2025-05-21T11:49:29+02:00'
draft = false
title = 'France Assembleur'
authors = "Wanako"
tags = ["Programmation"]
categories = ["Programmation"]
+++

# FRASM

**FRASM** (abréviation de *France Assembleur*) est un langage de programmation de type assembleur, minimaliste et entièrement en français. Il est conçu pour être simple, lisible, et exécuté à l’aide d’une machine virtuelle en Python.

---

## Objectif

### Objectif personnel

Créer un langage simple pour apprendre et comprendre :

* la création d’un langage interprété
* la gestion des variables, des instructions, des sauts et des conditions

### Objectif du langage

Créer un langage éducatif, accessible à un large public, en brisant la barrière de la langue.
Il s’agit d’une mise en œuvre simple et logique des opérations de base, proche du pseudocode.

---

## Syntaxe de base

Le programme est structuré en différentes sections, dont la section `Principal:`, qui contient la partie principale du programme.

### Tableau des instructions minimalistes

| Mot-clé                                        | Description                                                    |
| ---------------------------------------------- | -------------------------------------------------------------- |
| `definir a 10`                                 | Affecte une valeur numérique à une variable                    |
| `charger .nom_de_chaine. chaine de caractères` | Définit une chaîne de caractères                               |
| `ecrire a`                                     | Affiche la valeur d’une variable ou d’une chaîne de caractères |
| `si a == 10 (instruction)`                     | Condition                                                      |
| `aller label`                                  | Saut inconditionnel vers un label                              |
| `fin`                                          | Arrêt du programme                                             |

---

## Exemple de programme en FRASM

```plaintext
Principal:                    
definir a 10                     
definir b 5 
somme a b total 
ecrire total 
si total > 10 aller plus_10 
sinon aller moins_10

plus_10:
charger .afficher. 10 + 5 = 
ecrire .afficher. total
aller afficher_fin

moins_10:
charger .afficher. 10 - 5 = 
ecrire .afficher. total
aller afficher_fin

afficher_fin:
charger .fin_. Fin du programme
ecrire .fin_.
fin
```

---

## Où le trouver

Disponible sur mon GitHub : [Wanako : FRASM](https://github.com/Wanako0609/Frasm)

---

## Projets futurs

Création de **PyLang**, un langage plus accessible, inspiré de Python.

---

## Auteur

Projet personnel développé par **Wanako**, dans le but d’apprendre à concevoir un langage de programmation.

---
