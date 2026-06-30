# Write-up : Polyglotte JPEG-ZIP

## Description
Une image JPEG contient une archive ZIP cachée après le marqueur de fin de fichier (EOF).

## Analyse
1. **Métadonnées** : L'analyse EXIF révèle un commentaire caché.
2. **Structure** : `binwalk` confirme la présence d'une structure ZIP en fin de fichier.

## Résolution
1. Extraire le mot de passe des métadonnées EXIF.
2. Extraire le contenu ZIP de l'image.
3. Utiliser le mot de passe pour ouvrir le fichier `flag.txt`.

## Commandes
```bash
# Lecture du mot de passe
exiftool alien_challenge.jpg | grep "mdp"

# Extraction de l'archive
unzip alien_challenge.jpg
# Saisir le mot de passe : GilUtardNegDiffAll

# Lecture du flag
cat flag.txt
```
