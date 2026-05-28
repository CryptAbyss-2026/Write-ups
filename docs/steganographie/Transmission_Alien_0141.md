# 🚩 Flag n°0141

## Alien Transmission
Une station d'écoute terrestre a intercepté un signal audio étrange provenant de l'espace profond. Les scientifiques pensent qu'il s'agit d'une tentative de communication extraterrestre.
Le fichier audio nommé alien.mp3 semble être une discussion incompréhensible… mais les aliens ne communiquent pas toujours de manière conventionnelle.
Votre mission : analyser cette transmission pour découvrir le message caché laissé par la civilisation ZETA.

## Thème du Flag
Stéganographie 

---

## I - Pourquoi ce flag ?
L’objectif de ce challenge est de faire découvrir aux participants la stéganographie audio et de leur apprendre qu’un fichier audio peut contenir des informations cachées autrement qu’à l’écoute.

---

## II - Explication du flag
Le flag a été construit en intégrant une URL cachée dans le spectrogramme audio. Une image contenant le texte http://zetanet a d’abord été générée automatiquement en Python, puis transformée en spectrogramme pour être convertie en signal audio. Le résultat final est un fichier alien.mp3 qui semble être une discussion incompréhensible, mais dont l’analyse visuelle dans Audacity révèle l’URL menant au site ZETA//NET, où le flag est affiché.

---

## III - Solutions pour résoudre le flag
1. Télécharger et ouvrir Audacity
2. Importer le fichier alien.mp3
3. Cliquer sur les trois petits points à gauche de la piste audio
4. Sélectionner “Spectrogramme” au lieu de “Forme d’onde”
5. Un URL apparaît alors dans le spectrogramme
6. Ouvrir ce lien dans un navigateur
7. Le lien mène vers le réseau social des extraterrestres ZETA//NET
8. Le flag est visible en clair au-dessus du champ de mot de passe


---

## IV - Indices

- **Indice léger :**  
  Un fichier audio ne sert pas uniquement à être écouté. Parfois, l'information est cachée autrement.

- **Indice intermédiaire :**  
  Essayez d’utiliser un outil d’analyse audio pour examiner le signal plus en détail.

- **Indice final :**  
  Certaines représentations visuelles d’un signal audio peuvent révéler des informations invisibles autrement.

---

## V - Durée approximative
5 à 20 minutes.

---

## VI - Flag
CryptAbyss{H1dd3n_In_Pl4in_S0und}
