---
title: "🇫🇷 PWNME - Reverser don't like Rustacer"
date: 2022-07-08
draft: false
tags: [reverse engineering]
# image: "/image/blog-pic.jpg"
description: "Reverse d'un binaire en Rust"
showDate: true          # to enable/disable showing dates
math: true              # to enable showing equations (katex)
---

[Go back to write-ups list](../)

*This write-up is in French because it's a French CTF, sowwy*

> **Category:** Reverse
> 
> **Creator:** Bryton
> 
> **Description:**
> 
> Simpler than it seems
>
> Flag format: PWNME{flag}
>
> **Attachments:**
> - [Reverser.exe](/files/pwnme22/Reverser.exe)

## Introduction

Ce programme est un crack-me ordinaire où l'on doit entrer un mot de passe pour obtenir le flag.

![Rustacer usage](/image/pwnme22/rust_usage.png)

Cependant, lorsqu'on l'ouvre dans un désassembleur, on constate qu'en plus d'être écrit en Rust (comme indiqué par le titre du chall), le programme a été compilé sans les symboles : impossible donc de trouver main ou encore le nom d'une fonction qui vérifie notre mot de passe. On va donc devoir ruser 😏

## Recherche d'éléments intéressants

J'ai commencé par chercher l'adresse de la chaîne de caractères `"Usage: [...]"` ainsi que son utilisation dans le binaire :

![Rustacer string](/image/pwnme22/rust_str.png)

Cela nous mène vers une grosse fonction, qui d'ailleurs n'est pas bien analysée par IDA. Mais le plus gros de la vérification est quand même tout à fait compréhensible. On a d'abord une première partie qui vérifie la longueur du mot de passe entré :

![Password length](/image/pwnme22/rust_len.png)

On sait donc que la longueur doit être de 12. Puis une deuxième partie un peu plus longue qui exécute une fonction de vérification et qui en fonction du résultat affiche "Incorrect flag" ou appelle une autre fonction qui calcule puis affiche le flag.

![Comparison](/image/pwnme22/rust_cmp.png)

## Solution

Attendez... calcule puis affiche le flag ? Le plus simple donc, plutôt que de se perdre de le dédale de fonctions sans nom, ce serait de modifier le flux d'exécution du programme pour inverser la comparaison, et ainsi passer à l'affichage du flag lorsque notre mot de passe est faux !

On va donc patcher le binaire pour remplacer l'instruction `jnz` par son opposée : `jz`.

![Patch](/image/pwnme22/rust_patch.png)

Et ainsi, lorsqu'on lance le programme avec une string de longueur 12 en argument, on obtient le flag !

![Flag](/image/pwnme22/rust_flag.png)

[Go back to write-ups list](../)