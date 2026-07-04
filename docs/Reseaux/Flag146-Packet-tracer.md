

# 🚩 Flag n°146


Le réseau de CryptAbyss a été frappé par une perturbation inconnue après l’apparition d’un signal hostile dans l’infrastructure.

Un poste utilisateur doit normalement accéder à un serveur Web interne, mais la communication est impossible.  
La panne ne vient pas d’un seul équipement : plusieurs couches du réseau ont été sabotées.

L’objectif du challenge est de rétablir la connectivité complète entre `PC0` et le serveur Web, puis d’accéder à la page HTTP pour récupérer le flag.

---

## Thème du Flag

**Network Troubleshooting — VLAN — Trunk — Port-Security — Router-on-a-stick — OSPF — ACL — HTTP**

---

## Vue rapide du challenge

| Élément | Détail |
|---|---|
| **Catégorie** | Réseaux |
| **Nom du challenge** | Blackout Xeno-Core |
| **Type** | Dépannage réseau avancé |
| **Outil** | Cisco Packet Tracer |
| **Fichier fourni** | `CTF-CryptAbyss2026-.pka` |
| **Client** | `PC0 — 192.168.10.10/24` |
| **Serveur Web** | `172.16.50.10/24` |
| **Accès final** | `http://172.16.50.10` |
| **Protocoles / notions** | VLAN, trunk, OSPF, ACL, HTTP |
| **Difficulté** | ⭐⭐⭐⭐⭐ Très difficile |
| **Durée estimée** | 60 à 90 minutes |

---

## I - Pourquoi ce flag ?

L’objectif de ce challenge est de tester une vraie méthodologie de dépannage réseau.

Le participant doit analyser le réseau couche par couche, en partant du poste client jusqu’au serveur Web.

Ce challenge permet de travailler plusieurs compétences :

| Compétence | Objectif |
|---|---|
| **Couche 2** | Vérifier les VLAN, trunks et ports de switch |
| **Port-security** | Comprendre pourquoi un port peut être bloqué malgré un lien physique actif |
| **Router-on-a-stick** | Diagnostiquer une sous-interface mal taguée |
| **Routage dynamique** | Vérifier les voisinages OSPF et les annonces réseau |
| **Filtrage réseau** | Identifier une ACL qui bloque un service |
| **Service applicatif** | Comprendre qu’un serveur peut être joignable mais que HTTP peut être désactivé |
| **Méthodologie** | Avancer progressivement sans corriger au hasard |


## II - Architecture du challenge

### Schéma physique

```text
PC0 Fa0
  |
  | Copper Straight-Through
  |
SW1 Fa0/1
SW1 Gi0/1
  |
  | Trunk
  |
SW2 Gi0/1
SW2 Gi0/2
  |
  | Trunk vers R1
  |
R1 Gi0/0
R1 S0/0/0
  |
  | Serial
  |
R2 S0/0/0
R2 S0/0/1
  |
  | Serial
  |
R3 S0/0/0
R3 S0/0/1
  |
  | Serial
  |
R4 S0/0/0
R4 Gi0/1
  |
  | Copper Straight-Through
  |
Serveur Fa0
```

---

## Plan d’adressage

| Équipement | Interface | Adresse IP | Rôle |
|---|---|---|---|
| `PC0` | `FastEthernet0` | `192.168.10.10/24` | Client |
| `PC0` | Gateway | `192.168.10.1` | Passerelle |
| `R1` | `Gi0/0.10` | `192.168.10.1/24` | Gateway VLAN 10 |
| `R1` | `S0/0/0` | `10.0.12.1/30` | Vers R2 |
| `R2` | `S0/0/0` | `10.0.12.2/30` | Vers R1 |
| `R2` | `S0/0/1` | `10.0.23.1/30` | Vers R3 |
| `R3` | `S0/0/0` | `10.0.23.2/30` | Vers R2 |
| `R3` | `S0/0/1` | `10.0.34.1/30` | Vers R4 |
| `R4` | `S0/0/0` | `10.0.34.2/30` | Vers R3 |
| `R4` | `Gi0/1` | `172.16.50.1/24` | LAN serveur |
| `Serveur` | `FastEthernet0` | `172.16.50.10/24` | Serveur Web |
| `Serveur` | Gateway | `172.16.50.1` | Passerelle serveur |

---


---

## III - Explication du flag

Le réseau a été saboté à plusieurs niveaux.

Les pannes ne sont pas toutes visibles immédiatement. Certaines empêchent la connectivité de base, d’autres apparaissent seulement après avoir corrigé les premières erreurs.

| Niveau | Sabotage |
|---|---|
| **Couche 2** | Port-security incorrect sur le port de `PC0` |
| **Couche 2** | VLAN `10` absent sur `SW2` |
| **Couche 2** | Trunk `SW1 ↔ SW2` qui ne transporte pas le VLAN utilisateur |
| **Couche 2** | Native VLAN différente entre les switches |
| **Inter-VLAN** | Sous-interface R1 mal taguée |
| **Routage** | OSPF bloqué par une interface passive |
| **Routage** | Mismatch d’area OSPF entre R2 et R3 |
| **Routage** | Réseau serveur non annoncé dans OSPF |
| **Filtrage** | ACL qui autorise le ping mais bloque HTTP |
| **Applicatif** | Service HTTP désactivé sur le serveur |

---

## IV - Solution détaillée

### Étape 1 — Vérifier la configuration IP de PC0

Sur `PC0`, ouvrir :

```text
Desktop > IP Configuration
```

La configuration attendue est :

```text
IP Address      : 192.168.10.10
Subnet Mask     : 255.255.255.0
Default Gateway : 192.168.10.1
```

Ensuite, tester la passerelle :

```text
ping 192.168.10.1
```

Si le ping échoue, il faut commencer par diagnostiquer la couche 2 et la liaison jusqu’à R1.

---

### Étape 2 — Diagnostiquer le port de PC0 sur SW1

Sur `SW1`, vérifier l’état du port connecté à `PC0`.

```bash
show interfaces status
```

Puis vérifier le port-security :

```bash
show port-security
show port-security interface FastEthernet0/1
```

Le port peut être en état bloqué ou err-disabled à cause d’une ancienne adresse MAC autorisée.

Configuration sabotée :

```bash
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address 00AA.BBCC.DDEE
 switchport port-security violation shutdown
```

Correction :

```bash
enable
configure terminal

interface FastEthernet0/1
 shutdown
 no switchport port-security mac-address 00AA.BBCC.DDEE
 no switchport port-security
 switchport mode access
 switchport access vlan 10
 no shutdown
exit

end
write memory
```

!!! warning "Pourquoi c’est bloquant ?"
    Le lien physique peut sembler correct, mais le port-security peut empêcher le trafic de passer.

---

### Étape 3 — Vérifier le VLAN utilisateur sur les switches

Le poste `PC0` doit être dans le VLAN `10`.

Sur `SW1` et `SW2`, utiliser :

```bash
show vlan brief
```

Sur `SW2`, le VLAN `10` est absent.

Correction sur `SW2` :

```bash
enable
configure terminal

vlan 10
 name USERS
exit

end
write memory
```

---

### Étape 4 — Vérifier le trunk entre SW1 et SW2

Sur `SW1` et `SW2`, vérifier les trunks :

```bash
show interfaces trunk
```

Le lien trunk existe, mais côté `SW1`, le VLAN `10` n’est pas autorisé.

Configuration sabotée sur `SW1` :

```bash
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 99
```

Correction sur `SW1` :

```bash
enable
configure terminal

interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,99
 no shutdown
exit

end
write memory
```

---

### Étape 5 — Corriger la native VLAN entre SW1 et SW2

Le trunk entre `SW1` et `SW2` utilise une native VLAN différente de chaque côté.

Sur `SW1` :

```bash
switchport trunk native vlan 99
```

Sur `SW2`, la native VLAN est restée sur `1`.

Correction sur `SW2` :

```bash
enable
configure terminal

interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,99
 no shutdown
exit

end
write memory
```

!!! tip "Diagnostic"
    Utilisez `show interfaces trunk` pour vérifier les VLAN autorisés et la native VLAN.

---

### Étape 6 — Vérifier le trunk entre SW2 et R1

Le lien `SW2 Gi0/2 ↔ R1 Gi0/0` doit transporter le VLAN `10`.

Sur `SW2` :

```bash
show interfaces trunk
```

Correction si nécessaire :

```bash
enable
configure terminal

interface GigabitEthernet0/2
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,99
 no shutdown
exit

end
write memory
```

---

### Étape 7 — Corriger la sous-interface R1

Sur `R1`, la sous-interface du VLAN utilisateur est mal taguée.

Vérifier :

```bash
show running-config
show ip interface brief
```

Configuration sabotée :

```bash
interface GigabitEthernet0/0.10
 encapsulation dot1Q 110
 ip address 192.168.10.1 255.255.255.0
```

Le VLAN attendu est `10`, pas `110`.

Correction :

```bash
enable
configure terminal

interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
exit

end
write memory
```

Tester ensuite depuis `PC0` :

```text
ping 192.168.10.1
```

Si cette étape fonctionne, la partie LAN utilisateur est restaurée.

---

### Étape 8 — Vérifier les liens Serial

Les routeurs sont reliés par des interfaces Serial.

Vérifier les interfaces :

```bash
show ip interface brief
```

Vérifier le côté DCE si nécessaire :

```bash
show controllers serial 0/0/0
```

ou :

```bash
show controllers serial 0/0/1
```

Les interfaces doivent être en état :

```text
up/up
```

Les adresses attendues sont :

| Routeur | Interface | Adresse |
|---|---|---|
| `R1` | `S0/0/0` | `10.0.12.1/30` |
| `R2` | `S0/0/0` | `10.0.12.2/30` |
| `R2` | `S0/0/1` | `10.0.23.1/30` |
| `R3` | `S0/0/0` | `10.0.23.2/30` |
| `R3` | `S0/0/1` | `10.0.34.1/30` |
| `R4` | `S0/0/0` | `10.0.34.2/30` |

Si une interface est côté DCE, ajouter :

```bash
clock rate 64000
```

Exemple :

```bash
interface Serial0/0/0
 clock rate 64000
 no shutdown
```

---

### Étape 9 — Vérifier le voisinage OSPF

Sur chaque routeur, vérifier les voisins OSPF :

```bash
show ip ospf neighbor
```

Puis vérifier les routes :

```bash
show ip route
```

À ce stade, certaines routes sont absentes à cause des sabotages OSPF.

---

### Étape 10 — Corriger R2 : interface passive OSPF

Sur `R2`, l’interface vers `R3` est configurée en passive-interface.

Configuration sabotée :

```bash
router ospf 1
 passive-interface Serial0/0/1
```

Cela empêche la formation du voisinage OSPF entre `R2` et `R3`.

Correction :

```bash
enable
configure terminal

router ospf 1
 no passive-interface Serial0/0/1
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0
exit

end
write memory
```

---

### Étape 11 — Corriger R3 : mismatch d’area OSPF

Sur `R3`, le lien vers `R2` est annoncé dans la mauvaise area.

Configuration sabotée :

```bash
router ospf 1
 network 10.0.23.0 0.0.0.3 area 1
 network 10.0.34.0 0.0.0.3 area 0
```

Correction :

```bash
enable
configure terminal

router ospf 1
 no network 10.0.23.0 0.0.0.3 area 1
 network 10.0.23.0 0.0.0.3 area 0
 network 10.0.34.0 0.0.0.3 area 0
exit

end
write memory
```

Vérifier ensuite :

```bash
show ip ospf neighbor
show ip route
```

---

### Étape 12 — Corriger R4 : réseau serveur non annoncé

Sur `R4`, le réseau serveur `172.16.50.0/24` n’est pas annoncé dans OSPF.

Configuration incomplète :

```bash
router ospf 1
 network 10.0.34.0 0.0.0.3 area 0
```

Correction :

```bash
enable
configure terminal

router ospf 1
 network 10.0.34.0 0.0.0.3 area 0
 network 172.16.50.0 0.0.0.255 area 0
exit

end
write memory
```

Vérifier depuis `R1` :

```bash
show ip route
```

Le réseau suivant doit apparaître :

```text
O 172.16.50.0/24
```

---

### Étape 13 — Vérifier la configuration du serveur

Sur le serveur, vérifier :

```text
Desktop > IP Configuration
```

Configuration attendue :

```text
IP Address      : 172.16.50.10
Subnet Mask     : 255.255.255.0
Default Gateway : 172.16.50.1
```

Si la passerelle est incorrecte, le serveur peut recevoir les paquets mais ne pas répondre correctement.

---

### Étape 14 — Tester la connectivité IP complète

Depuis `PC0`, tester :

```text
ping 172.16.50.10
```

Résultat attendu :

```text
Reply from 172.16.50.10
```

Si le ping fonctionne, la couche 3 est correcte.

!!! success "Connectivité IP restaurée"
    À ce stade, le routage fonctionne.

    Mais cela ne garantit pas encore que le service Web est accessible.

---

### Étape 15 — Diagnostiquer l’accès HTTP

Depuis `PC0`, ouvrir le navigateur et tester :

```text
http://172.16.50.10
```

Si le ping fonctionne mais que le Web ne répond pas, il faut vérifier :

1. les ACL ;
2. le service HTTP du serveur.

---

### Étape 16 — Corriger l’ACL sur R4

Sur `R4`, une ACL laisse passer ICMP mais bloque TCP/80.

Vérifier :

```bash
show access-lists
show running-config
```

Configuration sabotée :

```bash
ip access-list extended XENO_FILTER
 permit icmp any any
 deny tcp any host 172.16.50.10 eq 80
 permit ip any any
```

Appliquée sur l’interface serveur :

```bash
interface GigabitEthernet0/1
 ip access-group XENO_FILTER out
```

Correction simple :

```bash
enable
configure terminal

interface GigabitEthernet0/1
 no ip access-group XENO_FILTER out
 no shutdown
exit

end
write memory
```

Ou correction plus fine :

```bash
enable
configure terminal

ip access-list extended XENO_FILTER
 no deny tcp any host 172.16.50.10 eq 80
 permit tcp any host 172.16.50.10 eq 80
exit

end
write memory
```

!!! 
    Un ping réussi prouve seulement que la connectivité IP fonctionne.

    Le service HTTP peut encore être bloqué par une ACL ou désactivé côté serveur.

### Étape 18 — Récupérer le flag

Depuis `PC0`, ouvrir :

```text
http://172.16.50.10
```

Le serveur affiche :

```text
CryptAbyss{X3n0_N3tw0rk_R3st0r3d}
```

---

## V - Récupération du flag

!!! success "Flag récupéré"
    ```text
    CryptAbyss{N3tw0rk9d1c5ddb7d15f5ab65145386effce0071a6e7a79}

    ```

---

## VI - Indices

### Indice léger — Couche par couche

Le problème ne se limite pas forcément au routage.  
Commencez par vérifier le chemin depuis le poste client, couche par couche.

### Indice intermédiaire — Les apparences trompent

Un équipement peut sembler correctement configuré tout en laissant passer un mauvais VLAN, une mauvaise encapsulation ou une mauvaise annonce réseau.

### Indice final — Le ping ne suffit pas

Même si la connectivité IP est rétablie, cela ne garantit pas que le service demandé est accessible.  
Pensez aussi au filtrage et au service applicatif.


## VIII - Durée approximative

"Temps estimé"
    **45 à 90 minutes**
Difficulté :

```text
⭐⭐⭐⭐⭐ Très difficile — 500 points
```
