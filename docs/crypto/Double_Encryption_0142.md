# 🚩 Flag n°0142

## Double Encryption
Après avoir infiltré le compte de Myrox, vous accédez à son profil personnel.
Dans son fil d’actualité, un rapport de mission attire votre attention. Cependant, le message est complètement illisible, comme s’il avait été volontairement chiffré pour éviter toute interception humaine.
Votre mission : déchiffrer ce rapport et découvrir le message caché.

## Thème du Flag
Cryptographie

---

## I - Pourquoi ce flag ?
L’objectif de ce challenge est d’amener les participants à identifier et combiner plusieurs couches de chiffrement. Le défi pousse à reconnaître un chiffrement de type Vigenère, à exploiter une information découverte précédemment (myrox), puis à poursuivre l’analyse lorsqu’une partie du message reste encore incompréhensible.

---

## II - Explication du flag
Le flag a été construit avec un double chiffrement successif afin de créer une difficulté progressive. Dans un premier temps, trois mots-clés ainsi que le flag ont été chiffrés avec un César ROT-7. Ensuite, l’ensemble du message a été chiffré une seconde fois avec Vigenère en utilisant la clé myrox.

---

## III - Solutions pour résoudre le flag

# Étape 1 - Chiffrement de Vigenère
Le message doit d’abord être déchiffré avec Vigenère en utilisant la clé : myrox
On remarque que certains mots restent encore chiffrés :
- ylkvbe
- qbnly
- hkqvpua
- JyfwaHifzz{K0bis3_3uj41wa10u}

# Étape 2 - Chiffrement César (ROT-7)
Les éléments restants sont chiffrés avec un César ROT-7 appliqué sur l’alphabet. Après déchiffrement on obtient :
- redoux
- juger
- adjoint
- CryptAbyss{D0ubl3_3nc41pt10n}

Les mots redoux, juger et adjoint doivent être utilisés dans le flag n°0145 what3words. 

---

## IV - Indices

- **Indice léger :**  
  Le nom d’un individu rencontré précédemment pourrait vous être utile.

- **Indice intermédiaire :**  
  Le message semble avoir été chiffré plusieurs fois.

- **Indice final :**  
  Analysez le texte chiffré avec l’indice de coïncidence

---

## V - Durée approximative
10 à 30 minutes.

---

## VI - Flag
CryptAbyss{D0ubl3_3nc41pt10n}
