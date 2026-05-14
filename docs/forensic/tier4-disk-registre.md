# 🚩 Flag n°4 — Disk Image + Registre Windows

## Nom du Flag
**Clé USB Agent X — Disque & Registre**

## Thème du Flag
Forensic / Expert — Univers narratif **Dossier Agent X** (deux artefacts saisis : sa clé USB FAT32 et une copie de son hive registre `NTUSER.DAT`).

---

## I - Pourquoi ce flag ?

Faire pratiquer **deux piliers de la forensique Windows** simultanément :

1. **Analyse de hives registre** (`NTUSER.DAT`, `SAM`, `SYSTEM`…) au format `regf` — qu'aucun éditeur de texte ne peut lire et qui demandent un parser dédié.
2. **Récupération de fichiers supprimés** sur un FS FAT — où l'octet `0xE5` qui marque la suppression ne détruit pas le contenu : les clusters restent intacts jusqu'à réutilisation.

Le flag est **scindé entre les deux artefacts** pour forcer le joueur à exploiter les deux pistes — aucune des deux ne suffit isolément.

---

## II - Explication du flag

**Artefact 1 — `NTUSER.DAT`** (~8 KiB) : généré par PowerShell via `reg.exe save` sur une clé `HKCU` temporaire. C'est un **vrai hive Windows** (magic `regf`), avec arborescence thématique :
- `\Software\Microsoft\Windows\CurrentVersion\Run` (clés Run plausibles + une suspecte `XenoBeacon`)
- `\Software\VeraCrypt`, `\Software\SimonTatham\PuTTY\Sessions\roswell-bastion` (artefacts plausibles)
- `\AlienContact\Observations\` → valeur `FlagPart_Registry = "verite_revelee_"` ← **moitié A**

**Artefact 2 — `cle_usb_agent_x.img`** (10 MiB) : image disque FAT32 brute construite manuellement (boot sector valide, signature `0x55AA`, deux FAT, root directory). Contient :
- Fichiers actifs leurres (`readme.txt`, `liste_courses.txt`, `photo_hangar.jpg`…)
- Une entrée de répertoire dont le 1er octet est `0xE5` (= supprimé) : `EVIDEN~1.TXT` → mais son **contenu reste dans les clusters** : un rapport d'observation HANGAR 18 incluant `xenomorph_parmi_nous}` ← **moitié B**

**Recomposition :** `CryptAbyss{` + `verite_revelee_` + `xenomorph_parmi_nous}` = `CryptAbyss{verite_revelee_xenomorph_parmi_nous}`.

---

## III - Solutions pour résoudre le flag

### Partie A — Hive `NTUSER.DAT`

```bash
$ xxd NTUSER.DAT | head -1
00000000: 7265 6766 ...   regf...
```

Magic `regf` confirmé. Parser avec **regipy** :

```python
from regipy.registry import RegistryHive
hive = RegistryHive("NTUSER.DAT")
ac = hive.get_key("AlienContact")
for sk in ac.iter_subkeys():
    if sk.name == "Observations":
        for v in sk.get_values():
            print(v.name, "=", v.value)
# FlagPart_Registry = verite_revelee_
```

**Moitié A : `verite_revelee_`**

### Partie B — Image disque

**Approche rapide :**

```bash
$ strings cle_usb_agent_x.img | grep -B2 -A 15 HANGAR
%%CTF_EVIDENCE_FILE%%
== RAPPORT D'OBSERVATION - HANGAR 18 ==
...
--- fragment ---
  xenomorph_parmi_nous}
```

**Approche académique (parser FAT) :**

```bash
$ xxd -s 0x24000 -l 256 cle_usb_agent_x.img
000240a0: e545 4944 454e 7e 31 54 58 54 ...   .EVIDEN~1TXT
                                              ↑ 0xE5 = supprimé
```

On lit le first cluster + size dans l'entrée, on calcule l'offset, on extrait avec `dd`.

**Approche carving :** `photorec /log /d recovered/ cle_usb_agent_x.img` (cocher type `txt`).

**Moitié B : `xenomorph_parmi_nous}`**

### Partie C — Recomposition

| Source | Contenu |
|:-------|:--------|
| Registre — `\AlienContact\Observations\FlagPart_Registry` | `verite_revelee_` |
| Fichier effacé — `evidence_hangar18.txt` | `xenomorph_parmi_nous}` |
| **Concaténation** | `CryptAbyss{verite_revelee_xenomorph_parmi_nous}` |

**Outils utilisables :** `regipy`, `RegRipper`, `Registry Explorer` (registre) ; `xxd`, `dd`, `photorec`, `foremost`, `sleuthkit` (disque).

---

## IV - Indices

- **Indice léger :**
  Un hive Windows (`NTUSER.DAT`, `SAM`, `SYSTEM`...) est un fichier binaire au format `regf`. Il n'est pas lisible tel quel — il faut un **parser dédié**. Du côté de la clé USB, un fichier « effacé » en FAT n'est pas nécessairement détruit.

- **Indice intermédiaire :**
  - **Registre** : outils standards → `regipy` (Python), `RegRipper`, ou `Registry Explorer` (Eric Zimmerman). Cherche une clé dont le nom colle au thème du CTF.
  - **Disque** : en FAT, quand un fichier est supprimé, le **premier octet** de son entrée de répertoire passe à `0xE5` mais ses **clusters de données** restent intacts tant qu'ils ne sont pas réalloués. Le contenu est récupérable.

- **Indice final :**
  - **Moitié A (registre)** : sous-clé `\AlienContact\Observations`, valeur `FlagPart_Registry`.
  - **Moitié B (fichier effacé)** : `photorec`, `foremost -t txt`, ou plus simplement `strings cle_usb_agent_x.img | grep -i hangar`.
  - **Concaténation** : `CryptAbyss{ moitié_A + moitié_B }`.

---

## V - Durée approximative
~60 minutes. Difficulté ⭐⭐⭐⭐⭐ (expert, 500 pts).

---

## VI - Flag
`CryptAbyss{verite_revelee_xenomorph_parmi_nous}`
