# 🚩 Flag n°00XX

## Chall Patch

Un binaire Linux nommé `memory_leak` est fourni. Lors de son exécution, il affiche une prétendue adresse mémoire :

```text
Memory address obtained : Ap{rvC`{qqyu2u][2w]fkf]r6vaj1F]o1
```

Cependant, cette sortie ne ressemble pas à une adresse mémoire valide. Le titre du challenge est volontairement trompeur.

L'objectif est d'analyser le programme, comprendre son fonctionnement et corriger une erreur logique directement dans le binaire afin de révéler le flag.

---

## Thème du Flag

**Reverse Engineering — Analyse de code et Binary Patching**

---

## I - Pourquoi ce flag ?

L'objectif de ce challenge est d'initier les participants au patching de binaire.

Dans de nombreux cas réels, il n'est pas nécessaire de casser une protection complexe : il suffit parfois de modifier quelques instructions pour changer le comportement d'un programme.

Ce challenge permet d'apprendre à :

* Lire du pseudo-code dans Ghidra.
* Comprendre le fonctionnement d'un XOR.
* Identifier un paramètre inutilisé.
* Repérer une incohérence logique dans le code.
* Modifier directement une instruction dans un binaire.
* Exporter un exécutable patché.

---

## II - Explication du flag

Le programme appelle une fonction nommée `mall0c` :

```c
mall0c(0x42);
```

Le nom de la fonction est volontairement trompeur et ressemble à l'appel standard `malloc`.

En analysant la fonction, on constate que :

* Le paramètre `0x42` est bien passé à la fonction.
* Ce paramètre n'est jamais utilisé.
* Les données affichées sont décodées à l'aide d'un XOR avec la valeur `0x40`.

Le développeur semble avoir voulu utiliser la valeur `0x42`, mais a codé `0x40` en dur dans la fonction.

Le challenge consiste donc à corriger cette erreur en patchant le binaire afin d'utiliser `0x42` à la place de `0x40`.

Une fois le patch appliqué, le texte décodé devient le flag.

---

## III - Solution détaillée

### Étape 1 — Exécuter le programme

```bash
chmod +x memory_leak
./memory_leak
```

**Résultat :**

```text
Memory address obtained : Ap{rvC`{qqyu2u][2w]fkf]r6vaj1F]o1
```

Cette sortie ne ressemble pas à une adresse mémoire.

Le nom du challenge est donc probablement une fausse piste.

---

### Étape 2 — Analyser le programme avec Ghidra

Ouvrir le binaire dans Ghidra puis examiner la fonction `main`.

```c
undefined8 main(void)
{
    mall0c(0x42);
    return 0;
}
```

On remarque immédiatement qu'une valeur `0x42` est passée à la fonction.

Poursuivre l'analyse dans `mall0c`.

---

### Étape 3 — Examiner la fonction mall0c

La fonction contient un tableau d'octets et les affiche via une boucle :

```c
for (local_c = 0; local_c < local_10; local_c++) {
    putchar(local_38[local_c] ^ 0x40);
}
```

L'élément important est ici :

```c
^ 0x40
```

Chaque caractère est décodé par un XOR avec la valeur `0x40`.

Or, le programme reçoit également la valeur :

```c
0x42
```

dans `main`, mais celle-ci n'est jamais utilisée.

Cette incohérence est probablement volontaire.

---

### Étape 4 — Identifier le bug logique

Le développeur semble vouloir utiliser la valeur :

```text
0x42
```

mais la fonction applique en réalité :

```text
0x40
```

Le résultat affiché est donc incorrect.

L'objectif du challenge est de corriger cette erreur.

---

### Étape 5 — Patcher le binaire

Dans Ghidra :

1. Ouvrir la vue Assembleur.
2. Localiser l'instruction contenant la constante `0x40`.
3. Modifier cette valeur pour `0x42`.
4. Sauvegarder la modification.
5. Exporter le binaire patché.

Le programme utilisera désormais la bonne clé XOR.

---

### Étape 6 — Exécuter le binaire patché

```bash
./memory_leak
```

**Résultat :**

```text
Memory address obtained : CryptAbyss{w0w_Y0u_did_p4tch3D_m3}
```

Le flag apparaît désormais en clair.

---

## IV - Récupération du flag

```text
CryptAbyss{w0w_Y0u_did_p4tch3D_m3}
```

---

## V - Indices

### Indice léger

La sortie affichée ne ressemble pas vraiment à une adresse mémoire.

### Indice intermédiaire

Une valeur intéressante est passée à une fonction, mais semble ignorée.

### Indice avancé

Le programme reçoit `0x42`, mais utilise une autre constante lors du décodage.

### Indice final

Le challenge ne consiste pas à retrouver une clé : il faut modifier le programme pour qu'il utilise la bonne.

---

## VI - Ce qu'il fallait apprendre

* Les noms de fonctions peuvent être trompeurs.
* Tous les paramètres passés à une fonction ne sont pas forcément utilisés.
* Un XOR est souvent utilisé pour masquer des données.
* Le reverse engineering ne consiste pas uniquement à lire du code.
* Le patching permet de modifier directement le comportement d'un exécutable.
* Ghidra peut être utilisé pour exporter un binaire corrigé.

---

## VII - Durée approximative

**15 à 30 minutes**

---

## VIII - Résumé de la résolution

```text
Exécution du programme
        │
        ▼
Fausse "adresse mémoire"
        │
        ▼
Analyse de main()
        │
        ▼
Découverte du paramètre 0x42
        │
        ▼
Analyse de mall0c()
        │
        ▼
XOR réalisé avec 0x40
        │
        ▼
Incohérence détectée
        │
        ▼
Patch 0x40 → 0x42
        │
        ▼
Export du binaire
        │
        ▼
Exécution du binaire patché
        │
        ▼
CryptAbyss{w0w_Y0u_did_p4tch3D_m3}
```
