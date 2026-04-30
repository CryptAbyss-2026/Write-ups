# MkDocs

## Prérequis
- Créer un répertoire github puis le cloner.
- `python` et `pip` installés

## Installer MkDocs
En local :
```bash
pip install mkdocs
```

## Initialiser le répertorie
Se positionner dans le répertoire puis :
```bash
mkdocs new .
```

## Déploiement local :
```bash
mkdocs serve
```
Le serveur est accessible à l'adresse http://127.0.0.1:8000/

## Déploiement en production
Lier le répertoire github et le projet (si non déjà fait):
```bash
git init
git add .
git commit -m "Initial docs"
git branch -M main
git remote add origin https://github.com/<USER>/<REPO>.git
git push -u origin main
```

Puis déployer avec :
```bash
mkdocs gh-deploy
```

Le site sera disponnible à l'adresse : https://USER.github.io/REPO/

## Mise à jour du site
Déposer le document dans `docs/` et mettre à jour [mkdocs.yml](mkdocs.yml)
Si vous avez des images les placer dans le dossier `assets/` et faire des références vers vos images (Exemple : `../assets/mon_image.png`)
Voir documentation : https://www.markdownlang.com/fr/basic/images.html   

Exemple :
```yml
site_name: CryptAbyss 2026
nav:
  - Catégorie: 
    - flag_name: write-up.md
```

Puis mettre à jour le répertoire et le site :
```bash
git add .
git commit -m "Mise à jour docs"
git push
mkdocs gh-deploy
```


## Notes 
- ⚠️ Ne pas supprimer la branche `gh-pages` si aucune autre branche n'a été configuréé
- Pour les utilisateurs ou organisations de Github Standard (pas de licence Entreprise) mettre le répertoire en public
