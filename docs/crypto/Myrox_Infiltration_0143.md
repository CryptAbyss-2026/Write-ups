# 🚩 Flag n°0143

## Myrox Infiltration
Après avoir découvert plusieurs secrets cachés sur ZETA//NET, les analystes pensent qu’il doit être possible d’aller encore plus loin…
La page de connexion semble classique, mais les extraterrestres ne conçoivent pas leurs systèmes comme les humains. Une analyse plus poussée du comportement du formulaire pourrait révéler une faiblesse exploitable.
Votre mission est d’infiltrer le système alien et d’accéder au compte d’un utilisateur.

## Thème du Flag
Cryptographie - Timing Attack

---

## I - Pourquoi ce flag ?
L’objectif de ce challenge est d’initier les participants au principe des timing attacks, en leur montrant qu’une vulnérabilité peut être exploitée à partir du comportement d’un système et non uniquement d’une faille visible. Le défi pousse les joueurs à observer les temps de réponse d’un formulaire d’authentification afin de retrouver progressivement un mot de passe.

---

## II - Explication du flag
Le challenge a été construit en ajoutant volontairement une vulnérabilité de timing dans le fichier server.js. Lors de la vérification du mot de passe, une condition compare les caractères un par un et ajoute un temps de réponse supplémentaire lorsqu’un caractère est correct. À l’inverse, une réponse incorrecte est renvoyée presque immédiatement.

---

## III - Solutions pour résoudre le flag
La page de connexion ZETA//NET est vulnérable à une attaque par timing.
Lorsqu’un caractère du mot de passe est correct, le serveur met plus de temps à répondre (~1000 ms). En revanche, lorsqu’un caractère est incorrect, la réponse est beaucoup plus rapide (~50 ms).
Il est donc possible de brute force le mot de passe caractère par caractère en mesurant les temps de réponse. En testant les lettres une par une, on obtient progressivement le mot de passe. Le mot de passe complet est donc : myrox.
Après connexion, l’utilisateur accède au compte de l’alien Myrox.
Sur cette page, une notification apparaît en bas à droite dans les discussions privées, indiquée par un petit "1" rouge. En ouvrant cette discussion privée, le flag est visible directement dans la conversation.

---

## IV - Indices

- **Indice léger :**  
  Toutes les failles ne sont pas visibles… certaines se cachent dans le comportement du système.

- **Indice intermédiaire :**  
  Observez attentivement le temps de réponse du serveur lors des tentatives de connexion.

- **Indice final :**  
  Essayez de tester les caractères du mot de passe un par un et comparez les temps de réponse.

---

## V - Durée approximative
30 minutes.

---

## VI - Flag
CryptAbyss{T1m1ng_4l1en_D3c0d3r}
