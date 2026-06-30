# Write-up : Obfuscation de Dimensions

## Description
Le programme affiche un flux de données visuelles corrompu. Le flag est présent dans le buffer, mais les dimensions d'affichage sont incorrectes.

## Analyse
Au lancement, le programme indique : `Buffer de 357 bits détecté`. 
Cela signifie que le produit de la **Largeur (W)** par la **Hauteur (H)** doit être exactement égal à 357 pour que l'image soit alignée.

## Résolution
Le flag étant un texte horizontal, on cherche une largeur supérieure à la hauteur parmi les diviseurs de 357.
Calcul : $357 / 7 = 51$.

## Commandes
Modifier le fichier `config.py` avec les valeurs suivantes :
```python
RENDER_WIDTH = 51
RENDER_HEIGHT = 7
```
Puis relancer le script principal pour lire le flag.
