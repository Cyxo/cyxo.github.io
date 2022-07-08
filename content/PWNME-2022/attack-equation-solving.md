---
title: "🇫🇷 PWNME - Attack Equation Solving"
date: 2022-07-06
draft: false
tags: [reverse engineering]
# image: "/image/blog-pic.jpg"
description: "Reverse (statique) d'un serveur Web en Go"
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
> I love vir**GIN** tonic
>
> *Vous avez accès au code compilé du site web*
>
> Le flag est sous le format suivant: PWNME{flag}
>
> ps: afin de valider le code sur le formulaire web il ne faut pas mettre PWNME{}.
>
> https://attack-equation-solving.pwnme.fr/
>
> **Attachments:**
> - [attack_solving.rar](/files/pwnme22/attack_solving.rar)

*Je sais que la solution la plus simple était en dynamique, mais comme mon debugger ne fonctionnait pas, je vous propose une solution en statique.*

Ce challenge est assez original puisque pour commencer il nous présente une interface web. Le binaire à reverse est donc le serveur web de ce site.

![Page web](/image/pwnme22/aes_web.png)

Sans plus attendre, j'ai ouvert le binaire dans mon désassembleur préféré. La première chose qu'on remarque c'est que le programme nous emmène à une fonction qui s'appelle `main_main` (ou `main.main` selon le désassembleur). On peut donc déjà identifier qu'il s'agit d'un binaire écrit en Go (c'est la sépcificité du Go). Un peu plus loin, on a la confirmation avec l'appel à une fonction comme `github_com_gin_gonic_gin___RouterGroup__handle` par exemple. En effet, les modules Go sont des repository Git et souvent distribués via Github, donc il n'est pas rare de retrouver "github.com" dans les programmes en Go.

![C'est du Go !](/image/pwnme22/aes_main.png)

On peut également à partir de ce nom de fonction se renseigner sur le fonctionnement [Gin Gonic](https://github.com/gin-gonic/gin), une librairie pour créer un serveur web en Go. On peut notamment y voir que l'appelle à la fonction `RouterGroup.handle` prend en arguments une méthode HTTP (par exemple `"POST"`), un chemin dans l'URL (par exemple `"/verify"`) et une ou plusieurs fonctions appelées "handlers" qui vont être exécutées lorsqu'on se rend sur cette URL.

L'assembleur généré par Go est vraiment simple et on peut facilement trouver les 3 arguments qui lui sont passés. Et comme cet endpoint `/verify` nous permet de vérifier notre flag, on va s'intéresser à son handler (ici sous le nom de `off_994D18`).

![C'est du Go !](/image/pwnme22/aes_handler.png)

Cette adresse pointe vers la fonction `main.verifyCode`, qui elle-même appelle une autre fonction `main.IsValidCode`. On va donc s'intéresser plus en détail à cette dernière.

Une première partie appelle la fonction `main.GetAESKey` dont voici l'algorithme :

![C'est du Go !](/image/pwnme22/aes_key.png)

Concrètement, on charge un tableau de bytes (à partir des trois nombres hexadécimaux qu'on voit ici) en mémoire, puis sur chaque byte on soustrait 32 et on effectue un xor avec l'index de la boucle. Enfin, le hash md5 de ce tableau est calculé et l'hexadécimal de ce hash sera retourné comme clé.

La suite de `IsValidCode` charge une chaîne hexadécimale qui est décodée et déchiffrée en AES ECB (d'où le nom **A**ttack **E**quation **S**olving, j'ai mis du temps à comprendre). Si on veut faire un équivalent en Python de la fonction `IsValidCode`, on a donc :

```py
from binascii import unhexlify
from Crypto.Cipher import AES
from hashlib import md5

# GetAESKey
key = list(
    0x538D51947F538894.to_bytes(8, byteorder="little") +
    0x538E8F8D7F93517F.to_bytes(8, byteorder="little") +
    0x7F99.to_bytes(2, byteorder="little")
)

for i in range(18):
    key[i] -= 32
    key[i] ^= i

key = md5(bytes(key)).hexdigest().encode()

# Charge et décode l'hexa
enc = unhexlify("59E2267F61917A7832D3608F6DDB2C6146E68ABBE82D0904CE10FE27BEBE4A54")
# Le déchiffre
cipher = AES.new(key, AES.MODE_ECB)
text = cipher.decrypt(enc)
print(text)
```

Et à l'arrivée, on obtient le clear text suivant :

```
b'pr0_cr4ck3r_g000\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10'
```

Qui permet bien de valider sur la page web !

![Verification input](/image/pwnme22/aes_yay.png)

[Go back to write-ups list](../)