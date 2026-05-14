# 🚩 Flag n°2 — File Carving

## Nom du Flag
**Interception radio — File Carving**

## Thème du Flag
Forensic / Intermédiaire — Univers narratif **Dossier Agent X** (transmission radio interceptée, fréquence H I 1420 MHz, Roswell 1947).

---

## I - Pourquoi ce flag ?

Faire pratiquer la **récupération de fichiers embarqués dans un conteneur binaire** (file carving) — technique de base en forensique disque, en analyse de malwares (payloads cachés dans des images), et en steganographie. Les participants doivent enchaîner plusieurs couches : extraction → métadonnées → mot de passe → archive chiffrée → flag. Cette mécanique « poupées russes » est un grand classique CTF qu'il faut savoir détecter.

---

## II - Explication du flag

Un fichier `intercept_package.bin` (~85 Ko) est construit par concaténation :

```
[en-tête texte] + junk + [PNG indice] + junk + [ZIP AES] + junk + [marqueur fin]
```

- Le **PNG d'indice** contient un message visible *"NOTE DE L'AGENT X"* qui évoque le film *Alien* (1979). Mais surtout, ses chunks `tEXt` (métadonnées PNG) renferment :
  ```
  Comment = "Password du ZIP chiffre : xenomorph"
  ```
- Le **ZIP est chiffré en AES-256** (via `pyzipper`) avec ce mot de passe `xenomorph` et contient `transmission_dechiffree.jpg` qui affiche le flag.

Le challenge croise donc **3 compétences** : carving binaire, lecture de métadonnées, déchiffrement ZIP AES.

---

## III - Solutions pour résoudre le flag

**Étape 1 — Identifier les fichiers embarqués :**

```bash
$ binwalk intercept_package.bin
2122    0x84A     PNG image, 600 x 400, 8-bit/color RGB
34370   0x8642    Zip archive data, AES-encrypted
```

**Étape 2 — Extraire automatiquement :**

```bash
$ binwalk -e intercept_package.bin
$ ls _intercept_package.bin.extracted/
84A.png   8642.zip
```

**Étape 3 — Lire les métadonnées du PNG :**

```bash
$ exiftool 84A.png
Author      : Agent X
Comment     : Password du ZIP chiffre : xenomorph
```

Mot de passe : **`xenomorph`** (clin d'œil au film *Alien*, 1979).

**Étape 4 — Ouvrir le ZIP AES (le `unzip` natif ne suffit pas, utiliser 7z) :**

```bash
$ 7z x -pxenomorph 8642.zip
$ ls
transmission_dechiffree.jpg
```

**Étape 5 — Lire le JPG :** l'image affiche directement le flag.

**Outils utilisables :** `binwalk`, `foremost`, `scalpel` (carving) ; `exiftool`, `Pillow` (métadonnées) ; `7z`, `pyzipper` (ZIP AES).

---

## IV - Indices

- **Indice léger :**
  Un fichier binaire peut en contenir d'autres, empilés « en oignon ». Regarde s'il n'y aurait pas des signatures de formats connus dissimulées à l'intérieur.

- **Indice intermédiaire :**
  Le métier de ce défi porte un nom : **file carving**. Des outils comme `binwalk` ou `foremost` détectent et extraient automatiquement tout ce qui ressemble à un fichier connu, à partir des signatures.

- **Indice final :**
  Deux fichiers sont embarqués. L'un est chiffré, l'autre ne l'est pas — mais il contient un secret qui permet d'ouvrir le premier. Analyse ses **métadonnées** (chunks PNG `tEXt`) avec `exiftool`. Pour le ZIP, attention au chiffrement **AES** (le `unzip` classique ne suffit pas, utiliser `7z`).

---

## V - Durée approximative
~15-20 minutes. Difficulté ⭐⭐ (intermédiaire, 200 pts).

---

## VI - Flag
`CryptAbyss{signal_intercepte_roswell_1947}`
