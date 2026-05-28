# 🚩 Flag n°0145

## Atterrissage d'urgence
En continuant l’exploration du profil de Myrox, un nouveau rapport de mission apparaît dans le fil d’actualité.
Les extraterrestres semblent avoir laissé une position réelle cachée dans leurs communications précédentes.
Votre mission : retrouver les trois mots-clés et localiser la balise.

## Thème du Flag
OSINT

---

## I - Pourquoi ce flag ?
L’objectif de ce challenge est d’amener les participants à réutiliser des informations obtenues dans un défi précédent afin de résoudre une énigme OSINT. Il pousse les joueurs à faire le lien entre plusieurs flags, à utiliser un service externe de géolocalisation et à se déplacer physiquement pour récupérer le flag final.

---

## II - Explication du flag
Le flag a été construit à partir du site what3words. Un emplacement précis devant l’UPJV a été sélectionné manuellement, puis les trois mots associés à cette position ont été récupérés (redoux.juger.adjoint). Ces mots ont ensuite été intégrés dans le challenge précédent (flag n°0142 Double Encryption) afin d’obliger les participants à les retrouver avant de pouvoir localiser la balise physique contenant le flag.

---

## III - Solutions pour résoudre le flag
Dans le défi précédent flag n°0142 Double Encryption, trois mots ont été obtenus après déchiffrement :
- redoux
- juger
- adjoint
Le rapport indique explicitement l’utilisation du site what3words. Le site https://what3words.com permet de localiser précisément un point du globe grâce à trois mots uniques. En entrant les trois mots : redoux.juger.adjoint. Une localisation précise apparaît sur la carte. Cette position correspond à un emplacement réel situé devant l’UPJV, où le flag est écrit et caché dans un ballon (un ballon par équipe).
Les participants doivent alors se rendre physiquement sur place pour récupérer l'information finale.

---

## IV - Indices

- **Indice léger :**  
  Déchiffre d’abord le rapport sur le profil de Myrox.

- **Indice intermédiaire :**  
  Trois mots ont été obtenus lors d’un défi précédent.

- **Indice final :**  
  Essayez d’utiliser ces trois mots sur un service de localisation basé sur trois mots.

---

## V - Durée approximative
20 à 30 minutes.

---

## VI - Flag
CryptAbyss{B4l1s3_D3t3ct3d}
