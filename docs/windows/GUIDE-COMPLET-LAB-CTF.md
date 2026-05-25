# 🛡️ Lab CTF Windows/AD — Guide complet

> **Environnement** : VMware Workstation · Réseau isolé Host-only · 3 VMs  
> **Domaine** : `ctf.local`  
> **Flags** : 7 challenges (8 flags au total avec Kerberos séparé)  
> **Compte de départ joueur** : `lowpriv` / `User@1234`

---

## Table des matières

- [[#Architecture]]
- [[#Comptes et groupes]]
- [[#Ordre de déploiement global]]
- [[#Phase 0 — Infrastructure VMware]]
- [[#Phase 1 — Structure AD de base]]
- [[#Phase 2 — Jonction des postes au domaine]]
- [[#Phase 3 — Déplacer les machines dans lOU Workstations]]
- [[#Checklist de vérification finale]]

---

## Architecture

```
┌──────────────────────────────────────────────────────┐
│               VMnet2 — 192.168.10.0/24               │
│                     Host-only                        │
│                                                      │
│  ┌─────────────┐   ┌──────────┐   ┌──────────────┐  │
│  │    DC01     │   │   WS01   │   │     WS02     │  │
│  │ WinSrv 2022 │   │ Win10/11 │   │  Win10/11    │  │
│  │ AD DS + DNS │   │          │   │  + KeePass   │  │
│  │ 10.10.10    │   │ 10.10.20 │   │  10.10.21    │  │
│  └─────────────┘   └──────────┘   └──────────────┘  │
└──────────────────────────────────────────────────────┘
```

| Machine | OS | Rôle | IP statique | DNS |
|---|---|---|---|---|
| DC01 | Windows Server 2022 | Contrôleur de domaine | 192.168.10.10 | 127.0.0.1 |
| WS01 | Windows 10/11 | Poste utilisateur | 192.168.10.20 | 192.168.10.10 |
| WS02 | Windows 10/11 | Poste utilisateur + KeePass | 192.168.10.21 | 192.168.10.10 |

---

## Comptes et groupes

### Utilisateurs AD

| Compte | Mot de passe | OU | Description |
|---|---|---|---|
| `lowpriv` | `User@1234` | Users | Point d'entrée joueur |
| `helpdesk1` | `Helpdesk@1234` | Users | Pivot Flag 3 |
| `svc-web` | `Svc!Web2024` | Service Accounts | Kerberoasting — Flag 7b |
| `svc-legacy` | `Legacy123` | Service Accounts | AS-REP Roasting — Flag 7b |
| `svc-backup` | `Backup@Secure99` | Service Accounts | Credentials exposés — Flag 3 & 4 |

### Groupes AD

| Groupe | Membres | Rôle |
|---|---|---|
| `IT-Helpdesk` | helpdesk1 | Droits ACL AD — Flag 3 |
| `CTF-Players` | lowpriv, helpdesk1 | Accès domaine de base |
| `LocalAdmins-WS01` | svc-web | Admin local WS01 via GPO — Flag 3 |
### Utilisateurs et mdp
| Compte              | Mot de passe      | Type                | Utilisé dans                           |
| ------------------- | ----------------- | ------------------- | -------------------------------------- |
| `CTF\Administrator` | lesbananes1350!   | Admin domaine       | Setup initial, jonction domaine        |
| `CTF\lowpriv`       | `User@1234`       | Utilisateur domaine | Point d'entrée joueur — tous les flags |
| `CTF\helpdesk1`     | `Helpdesk@1234`   | Utilisateur domaine | Flag 3 (ACL AD)                        |
| `CTF\svc-web`       | `Svc!Web2024`     | Compte de service   | Flag 7b (Kerberoasting)                |
| `CTF\svc-legacy`    | `Legacy123`       | Compte de service   | Flag 7b (AS-REP Roasting)              |
| `CTF\svc-backup`    | `Backup@Secure99` | Compte de service   | Flag 3 & 4 (credentials exposés)       |
| `WS01\admin-local`  | `AdminLocal@WS01` | Admin local WS01    | Flag 5 (pivot depuis backup.zip)       |
| `WS02\admin-local`  | `AdminLocal@WS02` | Admin local WS02    | Flag 5 (pivot)                         |
| KeePass master      | `keepass123`      | Mot de passe base   | Flag 7 (ouverture ctf.kdbx)            |
### Structure des OUs

```
ctf.local
└── CTF
    ├── Users               ← lowpriv, helpdesk1
    ├── Groups              ← IT-Helpdesk, CTF-Players, LocalAdmins-WS01
    ├── Service Accounts    ← svc-web, svc-legacy, svc-backup
    ├── Workstations        ← WS01, WS02
    └── Servers
```

---

## Ordre de déploiement global

```
1.  DC01  → Setup-AD-Base.ps1
2.  WS01  → Jonction domaine (manuel)
3.  WS02  → Jonction domaine (manuel)
4.  DC01  → Déplacer WS01 + WS02 dans OU=Workstations (manuel)
5.  WS01  → Flag2-Tache-Planifiee.ps1
6.  DC01  → Flag3-ACL-AD.ps1
7.  WS01  → ⚠️ Connexion lowpriv une fois (manuel) puis Flag4-Historique-PS.ps1
8.  DC01  → Flag5-SMB-Share.ps1
9.  WS01  → Flag6-LLMNR.ps1
10. WS02  → Flag6-LLMNR.ps1
11. WS02  → Flag7-KeePass.ps1 puis étapes manuelles KeePass
12. DC01  → Flag7b-Kerberos.ps1
```

> [!tip] Avant chaque script PowerShell
> Ouvre PowerShell en **Administrateur** et tape :
> ```powershell
> Set-ExecutionPolicy Bypass -Scope Process -Force
> ```

---

## Phase 0 — Infrastructure VMware

> [!check] Déjà effectué
> VMware installé, VMs créées, OS installés, domaine promu.

### Rappel IP statiques

**DC01** :
```powershell
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.10.10 -PrefixLength 24 -DefaultGateway 192.168.10.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 127.0.0.1
```

**WS01** :
```powershell
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.10.20 -PrefixLength 24 -DefaultGateway 192.168.10.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.10.10
```

**WS02** :
```powershell
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.10.21 -PrefixLength 24 -DefaultGateway 192.168.10.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.10.10
```

---

## Phase 1 — Structure AD de base

**Machine :** `DC01`  
**Script :** `Setup-AD-Base.ps1`

### Ce que le script fait

- Crée les 6 OUs : `CTF`, `Users`, `Groups`, `Service Accounts`, `Workstations`, `Servers`
- Crée les 5 comptes : `lowpriv`, `helpdesk1`, `svc-web`, `svc-legacy`, `svc-backup`
- Crée les 3 groupes : `IT-Helpdesk`, `CTF-Players`, `LocalAdmins-WS01`
### Script Setup-AD-Base.ps1
```powershell
# =============================================================================
# Setup-AD-Base.ps1
# Machine : DC01
# Rôle    : Crée toutes les OUs, utilisateurs et groupes nécessaires aux flags
# À lancer EN PREMIER, avant tous les scripts de flags
# =============================================================================
#Requires -RunAsAdministrator
Import-Module ActiveDirectory

$DomainDN = "DC=ctf,DC=local"

Write-Host "`n[SETUP BASE] Création de la structure AD" -ForegroundColor Cyan

# =============================================================================
# OUs
# =============================================================================
Write-Host "`n[*] OUs..." -ForegroundColor Yellow

$OUs = @(
    @{ Name = "CTF";              Path = $DomainDN },
    @{ Name = "Users";            Path = "OU=CTF,$DomainDN" },
    @{ Name = "Groups";           Path = "OU=CTF,$DomainDN" },
    @{ Name = "Service Accounts"; Path = "OU=CTF,$DomainDN" },
    @{ Name = "Workstations";     Path = "OU=CTF,$DomainDN" },
    @{ Name = "Servers";          Path = "OU=CTF,$DomainDN" }
)

foreach ($ou in $OUs) {
    $dn = "OU=$($ou.Name),$($ou.Path)"
    if (-not (Get-ADOrganizationalUnit -Filter "DistinguishedName -eq '$dn'" -ErrorAction SilentlyContinue)) {
        New-ADOrganizationalUnit -Name $ou.Name -Path $ou.Path -ProtectedFromAccidentalDeletion $false
        Write-Host "  [+] OU créée : $($ou.Name)" -ForegroundColor Green
    } else {
        Write-Host "  [~] OU existante : $($ou.Name)" -ForegroundColor Gray
    }
}

# =============================================================================
# UTILISATEURS
# =============================================================================
Write-Host "`n[*] Utilisateurs..." -ForegroundColor Yellow

$UsersOU = "OU=Users,OU=CTF,$DomainDN"
$SvcOU   = "OU=Service Accounts,OU=CTF,$DomainDN"

$Users = @(
    @{
        Sam         = "lowpriv"
        Name        = "Low Privilege User"
        Password    = "User@1234"
        OU          = $UsersOU
        Description = "Compte de départ CTF - bas privilèges"
        # Utilisé dans : Flag 1,2,3,4,5,6,7,8 (point d'entrée joueur)
    },
    @{
        Sam         = "helpdesk1"
        Name        = "Helpdesk User 1"
        Password    = "Helpdesk@1234"
        OU          = $UsersOU
        Description = "Compte helpdesk - droits AD délégués dangereux (Flag 3)"
        # Utilisé dans : Flag 3 (GenericAll sur LocalAdmins-WS01, ResetPassword sur svc-backup)
    },
    @{
        Sam         = "svc-web"
        Name        = "Service Web"
        Password    = "Svc!Web2024"
        OU          = $SvcOU
        Description = "Compte service web - SPN enregistré (Flag 8 Kerberoasting)"
        # Utilisé dans : Flag 8 (Kerberoasting via SPN HTTP/intranet.ctf.local)
    },
    @{
        Sam         = "svc-legacy"
        Name        = "Service Legacy"
        Password    = "Legacy123"
        OU          = $SvcOU
        Description = "Compte legacy - pré-auth Kerberos désactivée (Flag 8 AS-REP)"
        # Utilisé dans : Flag 8 (AS-REP Roasting)
    },
    @{
        Sam         = "svc-backup"
        Name        = "Service Backup"
        Password    = "Backup@Secure99"
        OU          = $SvcOU
        Description = "Compte service backup - credentials dans historique PS (Flag 4) et share SMB (Flag 5)"
        # Utilisé dans : Flag 4 (historique PS), Flag 5 (backup.ps1 dans le share)
    }
)

foreach ($u in $Users) {
    if (-not (Get-ADUser -Filter "SamAccountName -eq '$($u.Sam)'" -ErrorAction SilentlyContinue)) {
        $pwd = ConvertTo-SecureString $u.Password -AsPlainText -Force
        New-ADUser `
            -SamAccountName      $u.Sam `
            -Name                $u.Name `
            -AccountPassword     $pwd `
            -Enabled             $true `
            -Path                $u.OU `
            -Description         $u.Description `
            -PasswordNeverExpires $true
        Write-Host "  [+] $($u.Sam) créé  (pwd: $($u.Password))" -ForegroundColor Green
    } else {
        Write-Host "  [~] $($u.Sam) déjà existant" -ForegroundColor Gray
    }
}

# =============================================================================
# GROUPES
# =============================================================================
Write-Host "`n[*] Groupes..." -ForegroundColor Yellow

$GroupsOU = "OU=Groups,OU=CTF,$DomainDN"

$Groups = @(
    @{
        Name        = "IT-Helpdesk"
        Description = "Groupe helpdesk - droits délégués sur la GPO et objets AD (Flag 1, Flag 3)"
        Members     = @("helpdesk1")
        # Flag 1 : droits d'édition sur GPO-Runner
        # Flag 3 : GenericAll sur LocalAdmins-WS01
    },
    @{
        Name        = "CTF-Players"
        Description = "Groupe des joueurs - accès de base au domaine"
        Members     = @("lowpriv", "helpdesk1")
        # Aucun flag direct, sert à donner un accès domaine minimal propre
    },
    @{
        Name        = "LocalAdmins-WS01"
        Description = "Membres de ce groupe deviennent admins locaux sur WS01 via GPO (Flag 3)"
        Members     = @("svc-web")
        # Flag 3 : helpdesk1 a GenericAll dessus -> peut s'y ajouter -> admin local WS01
    }
)

foreach ($g in $Groups) {
    if (-not (Get-ADGroup -Filter "Name -eq '$($g.Name)'" -ErrorAction SilentlyContinue)) {
        New-ADGroup `
            -Name          $g.Name `
            -GroupScope    Global `
            -GroupCategory Security `
            -Path          $GroupsOU `
            -Description   $g.Description
        Write-Host "  [+] Groupe créé : $($g.Name)" -ForegroundColor Green
    } else {
        Write-Host "  [~] Groupe existant : $($g.Name)" -ForegroundColor Gray
    }

    foreach ($m in $g.Members) {
        try {
            Add-ADGroupMember -Identity $g.Name -Members $m -ErrorAction Stop
            Write-Host "      [+] $m -> $($g.Name)" -ForegroundColor Green
        } catch {
            Write-Host "      [~] $m déjà membre de $($g.Name)" -ForegroundColor Gray
        }
    }
}

# =============================================================================
# RÉSUMÉ
# =============================================================================
Write-Host "`n============================================================" -ForegroundColor Cyan
Write-Host " SETUP BASE TERMINÉ" -ForegroundColor Cyan
Write-Host "============================================================" -ForegroundColor Cyan

Write-Host "`n UTILISATEURS :" -ForegroundColor White
Write-Host "  lowpriv     / User@1234         OU=Users         -> compte de départ joueur" -ForegroundColor Gray
Write-Host "  helpdesk1   / Helpdesk@1234     OU=Users         -> pivot Flag 1 et 3" -ForegroundColor Gray
Write-Host "  svc-web     / Svc!Web2024       OU=Svc Accounts  -> Kerberoasting Flag 8" -ForegroundColor Gray
Write-Host "  svc-legacy  / Legacy123         OU=Svc Accounts  -> AS-REP Roasting Flag 8" -ForegroundColor Gray
Write-Host "  svc-backup  / Backup@Secure99   OU=Svc Accounts  -> credentials Flag 4 et 5" -ForegroundColor Gray

Write-Host "`n GROUPES :" -ForegroundColor White
Write-Host "  IT-Helpdesk      -> helpdesk1                  (droits GPO + ACL AD)" -ForegroundColor Gray
Write-Host "  CTF-Players      -> lowpriv, helpdesk1         (accès domaine de base)" -ForegroundColor Gray
Write-Host "  LocalAdmins-WS01 -> svc-web                    (admin local WS01 via GPO)" -ForegroundColor Gray

Write-Host "`n PROCHAINE ÉTAPE : lancer les scripts de flags dans l'ordre du guide" -ForegroundColor Yellow
Write-Host ""
```

### Lancement

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
.\Setup-AD-Base.ps1
```

### Actions manuelles

Aucune.

### Vérification

```powershell
Get-ADUser -Filter * -SearchBase "OU=Users,OU=CTF,DC=ctf,DC=local" | Select Name, SamAccountName
Get-ADGroup -Filter * -SearchBase "OU=Groups,OU=CTF,DC=ctf,DC=local" | Select Name
```

> [!success] Snapshot DC01
> **`DC01 — Structure AD de base`**

---

## Phase 2 — Jonction des postes au domaine

### WS01 — PowerShell Admin

```powershell
# Étape 1 — renommer
Rename-Computer -NewName "WS01" -Force -Restart

# Étape 2 — rejoindre le domaine (après redémarrage)
Add-Computer -DomainName "ctf.local" -Credential (Get-Credential) -Restart
# → Saisir : CTF\Administrator  /  P@ssw0rd!CTF2024
```

### WS02 — PowerShell Admin

```powershell
Rename-Computer -NewName "WS02" -Force -Restart

Add-Computer -DomainName "ctf.local" -Credential (Get-Credential) -Restart
# → Saisir : CTF\Administrator  /  P@ssw0rd!CTF2024
```

### Actions manuelles

Saisir les credentials `CTF\Administrator` quand demandé.

> [!success] Snapshots
> **`WS01 — Joint au domaine`**  
> **`WS02 — Joint au domaine`**

---

## Phase 3 — Déplacer les machines dans l'OU Workstations

**Machine :** `DC01`

```powershell
Get-ADComputer WS01 | Move-ADObject -TargetPath "OU=Workstations,OU=CTF,DC=ctf,DC=local"
Get-ADComputer WS02 | Move-ADObject -TargetPath "OU=Workstations,OU=CTF,DC=ctf,DC=local"

# Vérification
Get-ADComputer -Filter * -SearchBase "OU=Workstations,OU=CTF,DC=ctf,DC=local" | Select Name
# Attendu : WS01, WS02
```

## Checklist de vérification finale

### DC01

```powershell
# Structure AD
Get-ADOrganizationalUnit -Filter * -SearchBase "OU=CTF,DC=ctf,DC=local" | Select Name
Get-ADUser -Filter * -SearchBase "OU=CTF,DC=ctf,DC=local" -Properties Description | Select SamAccountName, Description
Get-ADGroup -Filter * -SearchBase "OU=CTF,DC=ctf,DC=local" | Select Name

# Flag 3 — ACL
Get-Acl "AD:\$(Get-ADGroup -Identity LocalAdmins-WS01 | Select -ExpandProperty DistinguishedName)" | Select -ExpandProperty Access | Where-Object { $_.IdentityReference -match "helpdesk" }

# Flag 5 — Shares
Get-SmbShare | Select Name, Path

# Flag 7b — SPNs et pre-auth
Get-ADUser -Identity "svc-web" -Properties ServicePrincipalNames | Select -ExpandProperty ServicePrincipalNames
Get-ADUser -Identity "svc-legacy" -Properties DoesNotRequirePreAuth | Select DoesNotRequirePreAuth
```

### WS01

```powershell
# Flag 2 — Tâche planifiée
Get-ScheduledTask -TaskName "HighTask" | Select TaskName, @{N="User";E={$_.Principal.UserId}}, State
Start-ScheduledTask -TaskName "HighTask"
Start-Sleep 5
Get-Content C:\Users\Public\flag2.txt

# Flag 4 — Historique PS
Test-Path "C:\Users\lowpriv\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"

# Flag 6 — LLMNR
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" -Name EnableMulticast
```

### WS02

```powershell
# Flag 6 — LLMNR (même vérification)
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" -Name EnableMulticast

# Flag 7 — KeePass
Test-Path "C:\Users\lowpriv\Documents\KeePass\ctf.kdbx"

# Flag 7 — Page IIS
Invoke-WebRequest http://intranet.ctf.local/flag7/ -UseBasicParsing | Select StatusCode
# Attendu : 200
```

### Snapshots finaux

> [!success] Snapshots définitifs — prendre après toutes les vérifications
> **`DC01 — Lab CTF complet`**  
> **`WS01 — Lab CTF complet`**  
> **`WS02 — Lab CTF complet`**

---

## Récapitulatif des flags

| # | Machine(s) | Vulnérabilité | Compte pivot | Flag |
|---|---|---|---|---|
| 2 | WS01 | Tâche planifiée SYSTEM | `lowpriv` | `FLAG{scheduled_task_priv_esc}` |
| 3 | DC01 | ACL AD — GenericAll / ResetPassword | `helpdesk1` | `FLAG{ad_acl_delegation_abuse}` |
| 4 | WS01 | Historique PSReadLine | `lowpriv` | via `\\DC01\Backups` |
| 5 | DC01 | Share SMB sensible | `lowpriv` | `FLAG{smb_sensitive_data_exposed}` |
| 6 | WS01+WS02 | LLMNR Poisoning | `lowpriv` | Hash NTLMv2 |
| 7 | WS02 | KeePass Auto-Type | `lowpriv` | `FLAG{keepass_autotype_credential_leak}` |
| 7b-K | DC01 | Kerberoasting | `lowpriv` | `FLAG{kerberoasting_svc_account_cracked}` |
| 7b-A | DC01 | AS-REP Roasting | `lowpriv` | `FLAG{asrep_roasting_no_preauth}` |
