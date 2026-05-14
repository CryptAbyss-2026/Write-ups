# 🚩 Flag n°3 — Memory Dump

## Nom du Flag
**Dump mémoire Windows — Agent X**

## Thème du Flag
Forensic / Avancé — Univers narratif **Dossier Agent X** (capture RAM du poste `WS-XFILES-01` quelques minutes avant interpellation).

---

## I - Pourquoi ce flag ?

Sensibiliser à la **forensique mémoire** : un dump RAM contient des artefacts vivants qu'aucun autre support ne révèle (presse-papiers, historiques de console, processus running, clés de chiffrement en clair). Le challenge force à comprendre que **les chaînes Windows ne sont pas toutes en ASCII** — le clipboard et beaucoup de structures Win32 utilisent **UTF-16 Little-Endian**, et `strings` sans option les rate complètement. C'est un piège typique en forensique Windows.

---

## II - Explication du flag

Le fichier `memoire.raw` (64 MiB) est un dump synthétique imitant la structure d'une capture mémoire Windows. Il contient des régions plausibles disséminées dans du bruit type *heap* :

- En-tête `Win10_x64_MemoryDump`, table de processus factice (`lsass.exe`, `chrome.exe`, `VeraCrypt.exe`…)
- Variables d'environnement, miettes navigateur, registre paths
- **Zone clipboard** au format Windows = **UTF-16LE**, contenant la **partie 1 du flag** : `CryptAbyss{contact_alien_`
- **Zone historique PowerShell** (`ConsoleHost_history.txt` style PSReadLine) en ASCII, contenant la **partie 2** : `wormhole_2026_confirmed}`

Le flag est **scindé entre deux encodages différents** dans deux artefacts système typiques. Il faut savoir les chercher tous les deux.

---

## III - Solutions pour résoudre le flag

**Étape 1 — Extraire les chaînes ASCII et UTF-16LE :**

```bash
$ strings -a -n 8 memoire.raw > strings_ascii.txt
$ strings -a -e l -n 8 memoire.raw > strings_utf16le.txt
```

Le drapeau `-e l` = **little-endian 16-bit**, indispensable pour le clipboard Windows.

**Étape 2 — Récupérer la partie 1 (clipboard, UTF-16LE) :**

```bash
$ grep -i "CryptAbyss{" strings_utf16le.txt
N'oublie pas, debut du flag = CryptAbyss{contact_alien_
```

**Étape 3 — Récupérer la partie 2 (historique PowerShell, ASCII) :**

```bash
$ grep -iE "history|echo" strings_ascii.txt | head -20
## ConsoleHost_history.txt (PSReadLine) ##
...
echo "wormhole_2026_confirmed}" | clip
```

**Étape 4 — Recomposer :**

| Source | Contenu |
|:-------|:--------|
| Clipboard (UTF-16LE) | `CryptAbyss{contact_alien_` |
| PS history (ASCII)   | `wormhole_2026_confirmed}` |
| **Concaténation**    | `CryptAbyss{contact_alien_wormhole_2026_confirmed}` |

**Note Volatility :** sur un dump réel acquis via WinPmem/DumpIt, on utiliserait `vol -f memoire.raw windows.clipboard` et `windows.cmdhistory`. Ici le dump est synthétique (pas de KPCR/DTB), Volatility ne décodera rien — les outils Unix standards suffisent.

**Outils utilisables :** `strings -e l`, `grep`, optionnellement Volatility 3.

---

## IV - Indices

- **Indice léger :**
  Sur un dump mémoire, `strings` et `grep` permettent déjà beaucoup. Demande-toi **où** un utilisateur laisse des traces : que fait-il quand il copie du texte ? quand il tape des commandes ?

- **Indice intermédiaire :**
  En explorant la sortie de `strings`, regarde les zones « presse-papiers » et « historique de commandes ». Si tu tombes sur des séquences bizarres — caractères ASCII séparés par des octets nuls, espaces étranges entre chaque lettre — c'est qu'un autre **encodage** se cache là. `strings` a une option pour ça (voir `man`, drapeau `-e`).

- **Indice final :**
  - **Morceau A** : zone presse-papiers → `strings -a -e l memoire.raw | grep CryptAbyss`
  - **Morceau B** : historique PowerShell (ASCII) → `strings -a memoire.raw | grep -i history`
  - Concatène les deux dans l'ordre A puis B.

---

## V - Durée approximative
~30 minutes. Difficulté ⭐⭐⭐ (avancé, 350 pts).

---

## VI - Flag
`CryptAbyss{contact_alien_wormhole_2026_confirmed}`
