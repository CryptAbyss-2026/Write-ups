# Challenge 1 — Base64

## Difficulté
Easy

---

## Description
```` Bienvenue

Un message a été encodé dans le fichier challenge1.txt
Retrouvez le flag.

Indication: 
C’est un encodage très connu

bonne chance!!!!
```
---

## Fichier fourni

- `challenge1.txt`

---

## Étapes de résolution

### 1. Observer le contenu

```text
Q3J5cHRBYnlzc3tkQzBkM19CQHMzNjRfZkBjMWwzfQ==
```

---

### 2. Identifier l’encodage

La présence du caractère `=` indique probablement un encodage Base64.

---

### 3. Décoder le contenu

```bash
base64 -d challenge1.txt
```

---

### 4. Résultat

```text
CryptAbyss{dC0d3_B@s364_f@c1l3}
```

---

## Conclusion

Le message était simplement encodé en Base64.

---

## Indices

### Indice 1
Le format du texte peut donner un indice.

### Indice 2
Certains encodages se terminent par `=`.

### Indice 3
Essayez un décodage simple avec `base64`.

---

## Topologie

```text
challenge1.txt
        ↓
Base64 decode
        ↓
FLAG1
```
