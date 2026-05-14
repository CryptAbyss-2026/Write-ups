# 🚩 Flag n°1 — Magic Bytes

## Nom du Flag
**Magic Bytes — Document classifié**

## Thème du Flag
Forensic / Initiation — Univers narratif **Dossier Agent X** (Zone 51, premier contact extraterrestre).

---

## I - Pourquoi ce flag ?

Initier les participants au principe fondamental de l'analyse forensique : **un fichier ne se définit pas par son extension mais par sa signature binaire**. Beaucoup de challenges plus avancés (carving, polyglottes, exfiltration via faux types MIME) reposent sur ce réflexe. C'est aussi le tier d'échauffement qui doit donner confiance avant les défis plus techniques.

---

## II - Explication du flag

Une image PNG (900×600) est générée avec Pillow. Elle représente une scène stylisée *"DOCUMENT CLASSIFIÉ – ZONE 51"* (soucoupe volante + faisceau lumineux) sur laquelle est imprimé visuellement le flag `CryptAbyss{area51_premier_contact}`.

Le fichier est ensuite **renommé en `.pdf`** (`shutil.move`). Aucun lecteur PDF ne peut l'ouvrir puisque les 8 premiers octets sont `89 50 4E 47 0D 0A 1A 0A` (signature PNG canonique) et non `25 50 44 46 2D` (signature PDF).

Le challenge est donc construit autour d'un **mismatch volontaire entre extension et magic bytes**.

---

## III - Solutions pour résoudre le flag

**Étape 1 — Identifier le vrai type via libmagic :**

```bash
$ file document_classifie.pdf
document_classifie.pdf: PNG image data, 900 x 600, 8-bit/color RGB, non-interlaced
```

**Étape 2 — Vérifier en hexadécimal :**

```bash
$ xxd document_classifie.pdf | head -1
00000000: 8950 4e47 0d0a 1a0a 0000 000d 4948 4452  .PNG........IHDR
```

**Étape 3 — Renommer et ouvrir :**

```bash
$ cp document_classifie.pdf document_classifie.png
$ start document_classifie.png      # Windows
$ xdg-open document_classifie.png   # Linux
```

L'image révèle visuellement le flag.

**Outils utilisables :** `file`, `xxd`, `hexdump`, HxD, Bless, 010 Editor, ImHex, ou la lib Python `filetype`/`python-magic`.

---

## IV - Indices

- **Indice léger :**
  L'apparence peut être trompeuse. Rien ne garantit qu'une extension de fichier reflète son véritable format.

- **Indice intermédiaire :**
  Tout fichier commence par une « signature » sur ses premiers octets, imposée par son format réel. Plusieurs outils la révèlent immédiatement.

- **Indice final :**
  `file`, `xxd` ou un éditeur hexadécimal sont tes amis. Les octets `89 50 4E 47` sont la signature d'un format d'image très connu.

---

## V - Durée approximative
~5 minutes. Difficulté ⭐ (initiation, 100 pts).

---

## VI - Flag
`CryptAbyss{area51_premier_contact}`
