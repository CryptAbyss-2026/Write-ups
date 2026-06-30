# 🚩 Flag n°00XX

## Propagande Alien

Un fichier PDF nommé `propagande_alien.pdf` est fourni. À première vue, il ne contient qu’une page de texte étrange et une mise en page volontairement incompréhensible.

L’objectif du challenge est de comprendre comment les informations sont dissimulées dans le document et de reconstruire le flag final.

---

## Thème du Flag

Forensics / PDF Analysis — Métadonnées + chiffrement simple

---

## I - Pourquoi ce flag ?

Ce challenge introduit une approche classique en forensics : les métadonnées PDF comme point d’entrée.

Contrairement à un simple document lisible, ici l’information utile n’est pas dans le contenu visible, mais dans les propriétés internes du fichier.

Le joueur doit :
- Inspecter les métadonnées du PDF
- Identifier une structure de flag partiellement encodée
- Comprendre qu’une étape de chiffrement supplémentaire est nécessaire

---

## II - Analyse initiale

Commande utilisée :

exiftool propagande_alien.pdf

Résultat :

ExifTool Version Number         : 13.25  
File Name                       : propagande_alien.pdf  
Directory                       : .  
File Size                       : 5.5 MB  
File Type                       : PDF  
PDF Version                     : 1.4  
Page Count                      : 1  
Tagged PDF                      : Yes  
Language                        : fr  
XMP Toolkit                     : Image::ExifTool 13.25  
Subject                         : Alien Communication  
Title                           : Transmission 07  
Author                          : Unknown  
Producer                        : Skia/PDF m151 Google Docs Renderer  
Keywords                        : CryptAbyss{mot13_mot45_mot8}, puis, decodage_caesar  

---

## III - Observation importante

Le champ Keywords contient une information cruciale :

CryptAbyss{mot13_mot45_mot8}, puis, decodage_caesar  

On en déduit :
- Le flag est fragmenté (mot13, mot45, mot8)
- Une étape de reconstruction est nécessaire
- Un chiffrement César doit être appliqué ensuite

---

## IV - Reconstruction du flag

En analysant le contenu du PDF, les éléments sont remis dans l’ordre logique.

On obtient :

moru_shor_vaïm  

Donc le flag intermédiaire est :

CryptAbyss{moru_shor_vaïm}

---

## V - Deuxième étape : César

L’indice indique clairement :

decodage_caesar  

On teste un bruteforce classique (0 à 25).

Le bon décalage est :

16  

---

## VI - Résolution du chiffrement

Application du César (+16) sur :

moru_shor_vaïm  

Résultat :

cehk_ixeh_lqyc  

---

## VII - Flag final

CryptAbyss{cehk_ixeh_lqyc}

---

## VIII - Méthodologie de résolution

Étape 1 — Analyse des métadonnées  
exiftool propagande_alien.pdf  

↓

Découverte :
CryptAbyss{mot13_mot45_mot8}

---

Étape 2 — Extraction du contenu PDF  
moru_shor_vaïm  

---

Étape 3 — Test César  
shift = 16  

---

Étape 4 — Flag final  
CryptAbyss{cehk_ixeh_lqyc}

---

## IX - Indices

Indice léger :
Les métadonnées contiennent plus que des informations classiques

Indice intermédiaire :
Le flag doit être reconstruit

Indice final :
Une substitution César est nécessaire

---

## X - Ce qu’il fallait apprendre

- Analyse de métadonnées avec exiftool  
- Extraction d’information cachée dans un PDF  
- Reconstruction de flag  
- Attaque par chiffrement César  
- Bruteforce de décalage  

---

## XI - Difficulté

Facile → Moyen

---

## XII - Résumé

PDF  
↓  
exiftool  
↓  
Keywords : CryptAbyss{mot13_mot45_mot8}  
↓  
Extraction contenu PDF  
↓  
moru_shor_vaïm  
↓  
César (shift 16)  
↓  
cehk_ixeh_lqyc  
↓  
FLAG FINAL  
CryptAbyss{cehk_ixeh_lqyc}
