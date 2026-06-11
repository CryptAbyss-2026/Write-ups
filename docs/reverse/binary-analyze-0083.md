# 🚩 Flag n°0083

## Access Forbidden

Un binaire Linux nommé `chall` est fourni. Lors de son exécution, il affiche simplement un message :

```text
Access forbidden.
```

Le challenge consiste à comprendre pourquoi l'accès est refusé et quelles conditions doivent être remplies pour obtenir un comportement différent.

---

## Thème du Flag

**Reverse Engineering — Analyse de logique et arguments de programme**

---

## I - Pourquoi ce flag ?

L'objectif de ce challenge est d'initier les participants à l'analyse statique d'un binaire avec un décompilateur.

Contrairement au challenge précédent (0084) où le flag était directement visible avec `strings`, celui-ci nécessite de comprendre la logique du programme afin d'identifier les arguments attendus.

Le participant apprend à :

- Utiliser un outil de reverse engineering comme Ghidra.
- Lire un pseudo-code décompilé.
- Identifier des comparaisons de chaînes.
- Comprendre le comportement conditionnel d'un programme.

---

## II - Explication du flag

Le programme vérifie les arguments passés sur la ligne de commande.

Si les deux arguments attendus sont présents et correspondent exactement aux chaînes définies dans le code, le programme exécute un shell Bash avec les privilèges du binaire.

Dans le cas contraire, il affiche simplement :

```text
Access forbidden.
```

L'analyse du code permet de découvrir que les arguments attendus sont :

```text
printf
main
```

Une fois ces arguments fournis dans le bon ordre, le programme lance :

```c
execv("/bin/bash", ...)
```

et donne accès à un shell.

---

## III - Solution détaillée

### Étape 1 — Exécuter le programme

```bash
chmod +x chall
./chall
```

**Résultat :**

```text
Access forbidden.
```

Le programme refuse l'accès sans fournir davantage d'informations.

---

### Étape 2 — Analyser le binaire avec Ghidra

Importer le binaire dans Ghidra puis ouvrir la fonction `main`.

Le pseudo-code obtenu ressemble à ceci :

```c
if (((2 < argc) &&
    (strcmp(argv[1],"printf") == 0)) &&
    (strcmp(argv[2],"main") == 0))
{
    execv("/bin/bash", ...);
}

puts("Access forbidden.");
```

On remarque immédiatement plusieurs éléments intéressants :

- Le programme vérifie le nombre d'arguments.
- Le premier argument doit être `printf`.
- Le second argument doit être `main`.
- Si ces conditions sont remplies, un shell Bash est lancé.

Note : les arguments sont des mots communs pour éviter de les repérer via `strings`. 

---

### Étape 3 — Fournir les bons arguments

Exécuter le programme avec les valeurs découvertes lors de l'analyse :

```bash
./chall printf main
```

**Résultat :**

```text
bash-5.2#
```

Le programme ouvre alors un shell.

---

### Étape 4 — Vérifier les privilèges obtenus

```bash
whoami
```

**Résultat :**

```text
root
```

Le shell obtenu possède les privilèges root.

---

### Étape 5 — Récupérer le flag

Une fois dans le shell, il suffit de lire le fichier contenant le flag.

```bash
cat /root/flag.txt
```

**Résultat :**

```text
CryptAbyss{H0w_h4ve_yUo_d0n€_th1s??}
```

---

## IV - Récupération du flag

```text
CryptAbyss{H0w_h4ve_yUo_d0n€_th1s??}
```

---

## V - Indices

### Indice léger

Le message *"Access forbidden"* laisse penser que le programme attend quelque chose.

### Indice intermédiaire

Le programme vérifie des paramètres passés en ligne de commande.

### Indice final

Deux arguments précis doivent être fournis pour déclencher une fonction cachée.

---

## VI - Ce qu'il fallait apprendre

- Utiliser Ghidra pour analyser un exécutable.
- Lire un pseudo-code décompilé.
- Identifier des appels à `strcmp()`.
- Comprendre le rôle des arguments (`argv`).
- Repérer un appel à `execv()` permettant d'exécuter un programme externe.

---

## VII - Durée approximative

**10 à 15 minutes**

---

## VIII - Résumé de la résolution

```text
Exécution du programme
        │
        ▼
"Access forbidden."
        │
        ▼
Analyse avec Ghidra
        │
        ▼
Découverte des comparaisons strcmp()
        │
        ▼
Arguments requis :
printf main
        │
        ▼
./chall printf main
        │
        ▼
execv("/bin/bash")
        │
        ▼
Shell root
        │
        ▼
Lecture du flag
```
