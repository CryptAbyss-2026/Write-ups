# Write-up : Blocs Compressés (Partie 1)

## Description
Le challenge consiste à récupérer un payload fragmenté et obfusqué à travers plusieurs archives ZIP simulant des vecteurs de tests techniques.

## Analyse
En inspectant les fichiers `test_vector.txt` dans les archives (`tests_x86.zip`, `tests_arm.zip`, etc.), on observe :
1. **Marqueurs d'ordre** : Des balises comme `[PART:1]`, `[PART:2]` indiquent l'ordre de reconstruction.
2. **Obfuscation Hexadécimale** : Les caractères utilisent des pipes `|` au lieu de backslashes `\` (ex: `|x1f|x8b`).
3. **Corruption de structure** : Des doubles espaces sont présents entre chaque octet pour empêcher une exécution directe.
4. **Signature GZIP** : Le début de la chaîne nettoyée (`\x1f\x8b`) révèle une archive GZIP.

## Résolution
Pour résoudre ce flag, il faut :
1. Extraire tous les fichiers des archives ZIP.
2. Réassembler les blocs dans l'ordre numérique des balises `[PART:X]`.
3. Nettoyer la chaîne : remplacer les `|` par des `\` et supprimer tous les espaces/sauts de ligne.
4. Convertir cette chaîne de caractères en binaire brut.
5. Décompresser le fichier `.gz` obtenu pour extraire le script Bash final.

## Commandes
```bash
# Extraction et nettoyage
unzip -q "*.zip"
cat out_*/test_vector.txt | sort | sed 's/\[PART:[0-9]\] //g' | sed 's/|/\\/g' | tr -d ' \n' > payload_hex.txt

# Conversion et exécution
echo -ne $(cat payload_hex.txt) > secret_payload.gz
gunzip -f secret_payload.gz
chmod +x secret_payload
./secret_payload
```
