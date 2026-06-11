# 🚩 Flag n°0084

## Pretty Easy

Un programme Linux nommé `code` est fourni. Lors de son exécution, il affiche simplement la date et l'heure du système.

---

## Thème du Flag

**Reverse Engineering — Reconnaissance basique**

---

## I - Pourquoi ce flag ?

L'objectif de ce challenge est d'apprendre aux participants qu'avant d'utiliser des outils avancés de reverse engineering, il est important d'effectuer une phase de reconnaissance simple.

De nombreux binaires contiennent encore des chaînes de caractères en clair qui peuvent être extraites directement avec des outils natifs du système.

Ce challenge introduit l'utilisation de la commande `strings`, incontournable lors des premières étapes d'analyse d'un exécutable.

---

## II - Explication du flag

Le binaire ne contient aucune protection particulière et le flag est directement embarqué dans les chaînes de caractères du programme.

Lorsque l'on exécute le programme, celui-ci affiche uniquement la date et l'heure courante afin de donner l'impression qu'une analyse plus poussée est nécessaire.

Cependant, une simple extraction des chaînes présentes dans le binaire permet de retrouver immédiatement le flag sans avoir besoin d'utiliser des outils de reverse engineering avancés.

---

## III - Solution détaillée

### Étape 1 — Exécuter le programme

Rendre le fichier exécutable puis le lancer :

```bash
chmod +x code
./code
```

**Résultat :**

```text
Sun Apr 26 21:31:50 2026
```

Le programme affiche uniquement la date système.

Aucune information supplémentaire n'est visible.

---

### Étape 2 — Analyser les chaînes du binaire

Avant d'ouvrir le programme dans Ghidra ou IDA, utiliser l'outil `strings` :

```bash
strings code
```

**Résultat :**

```text
/lib64/ld-linux-x86-64.so.2
ctime
__libc_start_main
__cxa_finalize
printf
libc.so.6
GLIBC_2.2.5
GLIBC_2.34
...
CryptAbyss{Pr3tTy_3a$y_d0nT_y0U_Th1nkk}
...
```

Parmi les différentes chaînes présentes dans le binaire, on repère immédiatement :

```text
CryptAbyss{Pr3tTy_3a$y_d0nT_y0U_Th1nkk}
```

Cette chaîne correspond au flag recherché.

---

## IV - Récupération du flag

```text
CryptAbyss{Pr3tTy_3a$y_d0nT_y0U_Th1nkk}
```

---

## V - Indices

### Indice léger

Tous les challenges de reverse engineering ne nécessitent pas forcément un désassembleur.

### Indice intermédiaire

Avant d'utiliser Ghidra, essayez d'examiner les chaînes de caractères contenues dans le binaire.

### Indice final

Une commande Linux très utilisée en forensic et reverse permet d'extraire les chaînes lisibles d'un exécutable.

---

## VI - Ce qu'il fallait apprendre

- Toujours commencer par une phase de reconnaissance simple.
- Utiliser `strings` avant de lancer des outils plus lourds.
- Identifier rapidement des informations sensibles laissées en clair dans un binaire.
- Comprendre qu'un exécutable peut contenir des données statiques accessibles sans désassemblage.

---

## VII - Durée approximative

**1 à 3 minutes**

---

## VIII - Résumé de la résolution

```text
Exécution du binaire
        │
        ▼
Affichage de la date
        │
        ▼
Analyse avec strings
        │
        ▼
Découverte du flag en clair
        │
        ▼
CryptAbyss{Pr3tTy_3a$y_d0nT_y0U_Th1nkk}
```
