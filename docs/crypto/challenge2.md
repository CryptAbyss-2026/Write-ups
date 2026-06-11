# Challenge 2 — Multi-layer + Caesar

## Difficulté
Medium

---

## Description

Un message codé est présent dans le fichier `challenge2.txt`.

Une clé encodée est fournie dans le fichier `key`.

---

## Fichiers fournis

- `challenge2.txt`
- `key`

---

## Étapes de résolution

### 1. Décodage du fichier principal

```bash
base64 -d challenge2.txt
```

Résultat :

```
Fausse piste : la clé est indispensable.

FubswDebvv{gF0g3_F3v@u_+E@v364}

Tu es proche... mais pas encore.
```

Le premier message est une fausse piste.

---

### 2. Analyse de la clé

```bash
base64 -d key
```

Résultat :

```text
Essaye de deviner le chiffrement utiliser dans le contenu de challenge.txt pour trouver le flag caché. clé = 3
```

La clé indique un décalage de 3, ce qui oriente vers un chiffrement de Caesar.

---

### 3. Déchiffrement Caesar

Message à traiter :

```text
RXNzYXllIGRlIGRldmluZXIgbGUgY2hpZmZyZW1lbnQgdXRpbGlzZXIgZGFucyBsZSBjb250ZW51IGRlIGNoYWxsZW5nZS50eHQgcG91ciB0cm91dmVyIGxlIGZsYWcgY2FjaMOpLiBjbMOpID0gMwo=
```

Appliquer un décalage de `-3`.

Résultat :

```text
CryptAbyss{dC0d3_C3s@r_+B@s364}
```

---

## Conclusion

Le challenge combine plusieurs étapes :

- Base64
- Fausse piste
- Chiffrement Caesar

---

## Indices

### Indice 1
Tous les résultats ne sont pas forcément le flag.

### Indice 2
Une clé est fournie, elle n’est pas là par hasard.

### Indice 3
Le message final semble avoir subi un décalage.

---

## Topologie

```text
challenge2.txt
        ↓
Base64 decode
        ↓
Fausse piste + message chiffré

key
        ↓
Base64 decode
        ↓
clé = 3

Message chiffré
        ↓
Caesar -3
        ↓
CryptAbyss{dC0d3_C3s@r_+B@s364}
```
