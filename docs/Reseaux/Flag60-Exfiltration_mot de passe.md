
---
Flag n°0060 — Exfiltration mot de passe
description: Write-up du challenge réseau basé sur un canal caché ICMP, l’extraction du champ IP ID et un déchiffrement XOR.
---

# 🚩 Flag n°0060

## Exfiltration mot de passe — Canal caché ICMP + XOR


Un fichier PCAP nommé `EnqueteSurMoi.pcap` est fourni aux participants.

À première vue, le trafic réseau semble contenir de simples paquets ICMP. Cependant, certains champs des en-têtes IP sont utilisés pour cacher une information sensible.

Le challenge consiste à analyser le trafic réseau, identifier le canal caché, extraire les valeurs utiles, retrouver la clé XOR, puis reconstruire le flag.

---

## Thème du Flag

**Network Forensics — Covert Channel — ICMP — XOR — Known Plaintext Attack**

---

## Vue rapide du challenge

| Élément | Détail |
|---|---|
| **Catégorie** | Réseaux |
| **Nom du challenge** | Exfiltration mot de passe |
| **Numéro** | Flag n°0060 |
| **Fichier fourni** | `EnqueteSurMoi.pcap` |
| **Outils recommandés** | Wireshark, tshark, Python |
| **Technique principale** | Canal caché dans le champ IP ID |
| **Méthode utilisée** | XOR |
| **Difficulté** | ⭐⭐ Moyen |
| **Durée estimée** | 20 à 30 minutes |

---

## I - Pourquoi ce flag ?

L’objectif de ce challenge est d’apprendre aux participants à analyser un fichier PCAP en allant plus loin que le contenu visible des paquets.

Dans beaucoup d’analyses réseau, on regarde d’abord le payload. Ici, le flag n’est pas directement caché dans les données visibles, mais dans un champ de l’en-tête IP : le champ **Identification**, aussi appelé **IP ID**.

Ce challenge permet donc de travailler plusieurs compétences importantes :

- utiliser Wireshark pour analyser du trafic ICMP ;
- comprendre qu’un champ réseau peut être détourné ;
- repérer une anomalie dans les en-têtes IP ;
- extraire un champ précis avec `tshark` ;
- comprendre le principe du chiffrement XOR ;
- retrouver une clé grâce à un début de flag connu ;
- reconstruire un flag à partir de valeurs chiffrées.

!!! note "Idée principale"

    Le trafic semble normal au départ, mais les valeurs du champ **IP ID** cachent en réalité les caractères du flag.

---

## II - Explication du flag

Le fichier `EnqueteSurMoi.pcap` contient du trafic ICMP.

Lorsqu’on ouvre le fichier dans Wireshark, les paquets ressemblent à de simples échanges de type ping. Pourtant, certains paquets contiennent une chaîne suspecte dans les données ICMP :

```text
chk_user:sysadmin
```

Cette chaîne sert de première piste. Elle indique que le trafic ICMP doit être analysé plus attentivement.

En observant les paquets concernés, on remarque que les valeurs du champ **Identification (IP ID)** ne sont pas normales. Elles ne suivent pas une progression classique et semblent choisies volontairement.

Le champ **IP ID** a donc été utilisé comme canal caché.

---

## Schéma de fonctionnement

Le flag a été caché selon cette logique :

```text
Flag en clair
        │
        ▼
Conversion des caractères en ASCII
        │
        ▼
Chiffrement avec XOR
        │
        ▼
Insertion des valeurs chiffrées dans le champ IP ID
        │
        ▼
Génération du fichier PCAP
```

Pour résoudre le challenge, il faut faire l’opération inverse :

```text
Extraction des valeurs IP ID
        │
        ▼
Récupération de la clé XOR
        │
        ▼
Déchiffrement des valeurs
        │
        ▼
Reconstruction du flag
```

!!! important "Point important"

    Le début du flag est connu : `CryptAbyss{`  
    Cela permet de retrouver la clé XOR grâce à une attaque par texte clair connu, appelée **Known Plaintext Attack**.

---

## III - Solution détaillée

### Étape 1 — Accéder au challenge


Le participant récupère le fichier PCAP fourni pour l’analyse.

Fichier à analyser :

```text
EnqueteSurMoi.pcap
```

---

### Étape 2 — Ouvrir le fichier PCAP

Ouvrir le fichier avec Wireshark :

```text
EnqueteSurMoi.pcap
```

Appliquer ensuite un filtre pour afficher uniquement le trafic ICMP :

```text
icmp
```

Dans certains paquets ICMP, on remarque la présence de la chaîne suivante :

```text
chk_user:sysadmin
```

Cette information montre que le trafic ICMP n’est probablement pas anodin.

!!! tip "Première piste"

    Le payload donne une indication, mais le flag n’est pas directement écrit dans le contenu visible des paquets.

---

### Étape 3 — Repérer le canal caché

En observant les paquets ICMP plus en détail, il faut regarder les champs de l’en-tête IP.

Le champ intéressant est :

```text
Identification
```

Dans Wireshark ou `tshark`, ce champ correspond à :

```text
ip.id
```

Les valeurs de ce champ semblent anormales :

- elles ne sont pas séquentielles ;
- elles ne ressemblent pas à des valeurs générées naturellement ;
- elles changent d’une manière volontaire ;
- elles apparaissent dans des paquets ICMP spécifiques.

Cela indique que le champ **IP ID** est utilisé comme canal caché.

| Élément observé | Interprétation |
|---|---|
| Paquets ICMP | Trafic qui semble normal |
| Payload | Présence d’une chaîne suspecte |
| Champ IP ID | Valeurs anormales |
| Valeurs non séquentielles | Possible canal caché |
| Objectif | Extraire et déchiffrer les valeurs |

---

### Étape 4 — Extraire les valeurs IP ID

Pour extraire les valeurs utiles, on peut utiliser `tshark`.

```bash
tshark -r EnqueteSurMoi.pcap \
-Y 'icmp and data.data contains "echo_request_routine"' \
-T fields -e ip.id
```

Explication de la commande :

| Élément | Rôle |
|---|---|
| `-r EnqueteSurMoi.pcap` | Lit le fichier PCAP |
| `-Y` | Applique un filtre d’affichage |
| `icmp` | Garde uniquement les paquets ICMP |
| `data.data contains "echo_request_routine"` | Sélectionne les paquets utiles |
| `-T fields` | Affiche seulement les champs demandés |
| `-e ip.id` | Extrait le champ IP ID |

Les valeurs obtenues correspondent aux caractères chiffrés du flag.

---

### Étape 5 — Retrouver la clé XOR

Le flag commence par :

```text
CryptAbyss{
```

La première lettre du flag est donc :

```text
C
```

La valeur ASCII de `C` est :

```text
67
```

La première valeur extraite du champ IP ID est :

```text
105
```

Pour retrouver la clé XOR, on effectue :

```python
67 ^ 105
```

Résultat :

```text
42
```

La clé XOR utilisée est donc :

```text
42
```

!!! success "Clé retrouvée"

    La clé XOR est `42`.

Cette étape est possible car on connaît le début du flag.  
C’est le principe de la **Known Plaintext Attack**.

---

### Étape 6 — Déchiffrer les valeurs extraites

On utilise ensuite un script Python pour appliquer l’opération XOR inverse sur toutes les valeurs IP ID extraites.

```python
valeurs = [
    105, 88, 83, 90, 94, 107, 72, 83, 89, 89, 81, 29, 78, 19, 31, 24, 18,
    75, 76, 19, 79, 25, 24, 29, 79, 29, 73, 26, 76, 75, 79, 24, 31, 18,
    18, 75, 31, 78, 73, 78, 73, 73, 79, 24, 79, 75, 18, 72, 76, 28, 75,
    79, 75, 30, 18, 27, 25, 72, 75, 25, 29, 26, 25, 19, 78, 75, 27, 18,
    78, 19, 31, 27, 30, 75, 78, 87
]

cle = 42

flag = "".join(chr(v ^ cle) for v in valeurs)

print(flag)
```

Résultat obtenu :

```text
CryptAbyss{7d9528af9e327e7c0fae2588a5dcdcce2ea8bf6aea4813ba37039da18d9514ad}
```

---

## IV - Récupération du flag

Le flag récupéré est :

```text
CryptAbyss{7d9528af9e327e7c0fae2588a5dcdcce2ea8bf6aea4813ba37039da18d9514ad}
```

---

## V - Indices

### Indice léger

Les paquets ICMP ne contiennent pas forcément l’information uniquement dans leur payload.

### Indice intermédiaire

Certains champs de l’en-tête IP peuvent être utilisés pour cacher des données. Le champ **Identification** mérite d’être observé.

### Indice final

Le flag est caché dans les valeurs du champ `ip.id`. Le début connu `CryptAbyss{` permet de retrouver la clé XOR.

---

## VI - Ce qu’il fallait apprendre

À travers ce challenge, le participant devait apprendre à :

- analyser un fichier PCAP avec Wireshark ;
- utiliser des filtres ICMP ;
- comprendre le rôle du champ IP ID ;
- détecter un canal caché dans un en-tête réseau ;
- extraire des champs précis avec `tshark` ;
- comprendre le principe du XOR ;
- utiliser une attaque par texte clair connu ;
- écrire un script Python pour reconstruire le flag.

---

## VII - Durée approximative

**20 à 30 minutes**

La durée peut varier selon le niveau du participant en :

- analyse réseau ;
- utilisation de Wireshark ;
- extraction avec `tshark` ;
- compréhension du chiffrement XOR.

---

## VIII - Résumé de la résolution

```text
Accès au challenge local
        │
        ▼
Ouverture du fichier PCAP
        │
        ▼
Filtrage du trafic ICMP
        │
        ▼
Observation de la chaîne chk_user:sysadmin
        │
        ▼
Analyse des en-têtes IP
        │
        ▼
Détection d’anomalies dans le champ IP ID
        │
        ▼
Extraction des valeurs ip.id avec tshark
        │
        ▼
Récupération de la clé XOR avec CryptAbyss{
        │
        ▼
Déchiffrement avec Python
        │
        ▼
Récupération du flag
```

---

## IX - Commandes utiles

### Filtrer les paquets ICMP dans Wireshark

```text
icmp
```

### Extraire les valeurs IP ID avec tshark

```bash
tshark -r EnqueteSurMoi.pcap \
-Y 'icmp and data.data contains "echo_request_routine"' \
-T fields -e ip.id
```

### Retrouver la clé XOR

```python
ord("C") ^ 105
```

### Déchiffrer le flag

```python
valeurs = [
    105, 88, 83, 90, 94, 107, 72, 83, 89, 89, 81, 29, 78, 19, 31, 24, 18,
    75, 76, 19, 79, 25, 24, 29, 79, 29, 73, 26, 76, 75, 79, 24, 31, 18,
    18, 75, 31, 78, 73, 78, 73, 73, 79, 24, 79, 75, 18, 72, 76, 28, 75,
    79, 75, 30, 18, 27, 25, 72, 75, 25, 29, 26, 25, 19, 78, 75, 27, 18,
    78, 19, 31, 27, 30, 75, 78, 87
]

cle = ord("C") ^ valeurs[0]

flag = "".join(chr(v ^ cle) for v in valeurs)

print(flag)
```

---


