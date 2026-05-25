# Flag 7 — KeePass Auto-Type

> **Technique :** KeePass Auto-Type credential extraction
> **Machine de déploiement :** WS02
> **Machine d'exploitation :** WS02
> **Compte de départ :** `CTF\lowpriv` / `User@1234`
> **Difficulté :** Facile / Intermédiaire

---

## Déploiement

### Prérequis — Transférer KeePass sur WS02 sans internet

> [!warning] À faire sur ta machine hôte AVANT de lancer le script

**Étape 1 — Télécharger KeePass sur ta machine hôte :**

```
https://keepass.info/download.html
```

Télécharger **KeePass 2.x** — fichier `KeePass-2.xx-Setup.exe`.

**Étape 2 — Transférer vers WS02** (choisir une méthode) :

**Méthode A — Glisser-déposer VMware Tools :**
1. S'assurer que VMware Tools est installé sur WS02
2. Glisser-déposer le fichier `.exe` directement dans la fenêtre VMware de WS02
3. Placer le fichier dans `C:\Temp\` sur WS02

**Méthode B — Via le share DC01 :**
1. Copier le fichier dans `C:\Shares\Sensitive\` sur DC01
2. Depuis WS02 PowerShell Admin :

```powershell
net use S: \\DC01\Sensitive
New-Item -ItemType Directory -Path C:\Temp -Force | Out-Null
Copy-Item S:\KeePass-2.xx-Setup.exe C:\Temp\KeePass-Setup.exe
net use S: /delete
```

**Méthode C — Dossier partagé VMware :**
1. **VM → Settings → Options → Shared Folders → Add**
2. Partager le dossier contenant l'installeur
3. Depuis WS02 :

```powershell
Copy-Item "\\vmware-host\Shared Folders\KeePass-Setup.exe" C:\Temp\KeePass-Setup.exe
```

---

### Script — Flag7-KeePass.ps1

> [!warning] Lancer en PowerShell Admin sur WS02
> L'installeur KeePass doit être présent dans `C:\Temp\KeePass-Setup.exe` avant de lancer ce script

```powershell
#Requires -RunAsAdministrator

Write-Host "`n[FLAG 7] KeePass Auto-Type" -ForegroundColor Cyan

# --- 1. Verifier que l'installeur KeePass est present ---
$keepassInstaller = "C:\Temp\KeePass-Setup.exe"
if (-not (Test-Path $keepassInstaller)) {
    Write-Host "  [!] ERREUR : installeur KeePass introuvable." -ForegroundColor Red
    Write-Host "  [!] Placer l'installeur dans C:\Temp\KeePass-Setup.exe" -ForegroundColor Red
    Write-Host "  [!] Voir section Prerequis du guide pour le transfert sans internet." -ForegroundColor Red
    exit 1
}

# --- 2. Installer KeePass en silent ---
Write-Host "  [*] Installation de KeePass..." -ForegroundColor Yellow
Start-Process -FilePath $keepassInstaller -ArgumentList "/SILENT /NORESTART" -Wait
Write-Host "  [+] KeePass installe" -ForegroundColor Green

# --- 3. Installer IIS ---
Write-Host "  [*] Installation IIS..." -ForegroundColor Yellow
$features = @(
    "IIS-WebServerRole",
    "IIS-WebServer",
    "IIS-CommonHttpFeatures",
    "IIS-StaticContent",
    "IIS-DefaultDocument"
)
foreach ($f in $features) {
    Enable-WindowsOptionalFeature -Online -FeatureName $f -All -NoRestart `
        -ErrorAction SilentlyContinue | Out-Null
}
Write-Host "  [+] IIS installe" -ForegroundColor Green

# --- 4. Page de login intranet ---
$wwwFlag7 = "C:\inetpub\wwwroot\flag7"
New-Item -ItemType Directory -Path $wwwFlag7 -Force | Out-Null

@'
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Intranet CTF</title>
    <style>
        *{box-sizing:border-box;margin:0;padding:0}
        body{font-family:'Segoe UI',sans-serif;background:#0d1117;color:#c9d1d9;
             display:flex;justify-content:center;align-items:center;min-height:100vh}
        .card{background:#161b22;border:1px solid #30363d;border-radius:10px;
              padding:40px;width:360px}
        h2{color:#58a6ff;margin-bottom:6px;font-size:1.4rem}
        .sub{color:#8b949e;font-size:13px;margin-bottom:28px}
        label{display:block;font-size:13px;color:#8b949e;margin-bottom:5px}
        input{width:100%;padding:9px 12px;background:#0d1117;border:1px solid #30363d;
              border-radius:6px;color:#c9d1d9;font-size:14px;margin-bottom:16px}
        button{width:100%;padding:10px;background:#238636;color:#fff;border:none;
               border-radius:6px;font-size:14px;cursor:pointer}
        button:hover{background:#2ea043}
        #result{margin-top:20px;padding:14px;background:#0d1117;border-radius:6px;
                display:none;font-family:monospace;font-size:13px}
        .ok{color:#3fb950;border:1px solid #238636}
        .err{color:#f85149;border:1px solid #da3633}
    </style>
</head>
<body>
<div class="card">
    <h2>Intranet CTF</h2>
    <p class="sub">Portail d'administration interne</p>
    <label>Utilisateur</label>
    <input type="text" id="user" autocomplete="off"/>
    <label>Mot de passe</label>
    <input type="password" id="pass"/>
    <button onclick="tryLogin()">Se connecter</button>
    <div id="result"></div>
</div>
<script>
function tryLogin(){
    const u=document.getElementById('user').value;
    const p=document.getElementById('pass').value;
    const r=document.getElementById('result');
    if(u==='admin'&&p==='FLAG{keepass_autotype_credential_leak}'){
        r.className='ok';r.style.display='block';
        r.innerHTML='Connexion reussie !<br><br><b>FLAG{keepass_autotype_credential_leak}</b>';
    }else{
        r.className='err';r.style.display='block';
        r.innerHTML='Identifiants incorrects.';
    }
}
document.addEventListener('keydown',e=>{if(e.key==='Enter')tryLogin();});
</script>
</body>
</html>
'@ | Set-Content "$wwwFlag7\index.html" -Encoding UTF8
Write-Host "  [+] Page intranet creee : http://intranet.ctf.local/flag7/" -ForegroundColor Green

# --- 5. Entree hosts ---
$hostsFile = "C:\Windows\System32\drivers\etc\hosts"
if ((Get-Content $hostsFile) -notmatch "intranet\.ctf\.local") {
    Add-Content $hostsFile "`n127.0.0.1`tintranet.ctf.local"
    Write-Host "  [+] hosts : intranet.ctf.local -> 127.0.0.1" -ForegroundColor Green
} else {
    Write-Host "  [~] hosts : entree deja presente" -ForegroundColor Gray
}

# --- 6. Fichier XML KeePass a importer ---
# Le profil lowpriv doit exister (connexion prealable requise)
if (-not (Test-Path "C:\Users\lowpriv")) {
    Write-Host "  [!] ATTENTION : profil lowpriv inexistant." -ForegroundColor Magenta
    Write-Host "  [!] Connecte-toi une fois avec CTF\lowpriv, deconnecte-toi, puis relance." -ForegroundColor Magenta
}

$kpDir = "C:\Users\lowpriv\Documents\KeePass"
New-Item -ItemType Directory -Path $kpDir -Force | Out-Null

@"
<?xml version="1.0" encoding="utf-8"?>
<KeePassFile>
    <Meta><DatabaseName>CTF Passwords</DatabaseName></Meta>
    <Root>
        <Group>
            <Name>CTF</Name>
            <Entry>
                <String><Key>Title</Key>    <Value>Intranet CTF</Value></String>
                <String><Key>UserName</Key> <Value>admin</Value></String>
                <String><Key>Password</Key> <Value>FLAG{keepass_autotype_credential_leak}</Value></String>
                <String><Key>URL</Key>      <Value>http://intranet.ctf.local/flag7</Value></String>
                <String><Key>Notes</Key>    <Value>Compte admin portail intranet</Value></String>
                <AutoType>
                    <Enabled>True</Enabled>
                    <DataTransferObfuscation>0</DataTransferObfuscation>
                    <DefaultSequence>{USERNAME}{TAB}{PASSWORD}{ENTER}</DefaultSequence>
                </AutoType>
            </Entry>
        </Group>
    </Root>
</KeePassFile>
"@ | Set-Content "$kpDir\import.xml" -Encoding UTF8
Write-Host "  [+] XML KeePass cree : $kpDir\import.xml" -ForegroundColor Green

# --- 7. Verifications ---
Write-Host "`n  [*] KeePass installe :" -ForegroundColor Yellow
Test-Path "C:\Program Files\KeePass Password Safe 2\KeePass.exe"

Write-Host "`n  [*] Test page IIS :" -ForegroundColor Yellow
try {
    $r = Invoke-WebRequest http://intranet.ctf.local/flag7/ -UseBasicParsing
    Write-Host "  [+] Page accessible (HTTP $($r.StatusCode))" -ForegroundColor Green
} catch {
    Write-Host "  [!] Page inaccessible - verifier IIS" -ForegroundColor Red
}

Write-Host "`n  [*] Fichier XML :" -ForegroundColor Yellow
Test-Path $kpDir\import.xml

Write-Host "`n[FLAG 7] Script termine - Etapes manuelles requises (voir guide)" -ForegroundColor Green
Write-Host "  SNAPSHOT apres etapes manuelles : WS02 - Flag7 deploye" -ForegroundColor Yellow
```

---

### Étapes manuelles — dans l'ordre

> [!warning] À faire APRÈS le script, sur WS02

**Étape 1 — Se connecter en tant que `lowpriv`** sur WS02 (`CTF\lowpriv` / `User@1234`).

**Étape 2 — Créer la base KeePass** :

1. Ouvrir KeePass (`C:\Program Files\KeePass Password Safe 2\KeePass.exe`)
2. **File → New**
3. Enregistrer sous : `C:\Users\lowpriv\Documents\KeePass\ctf.kdbx`
4. Mot de passe maître : `Inf0Sec&Lab2024!`
5. Valider la création de la base

**Étape 3 — Importer le XML** :

1. **File → Import → KeePass XML 2.x**
2. Sélectionner : `C:\Users\lowpriv\Documents\KeePass\import.xml`
3. Confirmer l'import
4. **Ctrl+S** pour sauvegarder

**Étape 4 — Supprimer le XML source** (PowerShell Admin) :

```powershell
Remove-Item "C:\Users\lowpriv\Documents\KeePass\import.xml" -Force
```

**Étape 5 — Vérification finale** :

Ouvrir Microsoft Edge sur WS02 : `http://intranet.ctf.local/flag7/`
La page de login doit s'afficher.

> [!success] Snapshot WS02
> **`WS02 — Flag7 déployé`**

---

## Write-up

### Contexte

KeePass est un gestionnaire de mots de passe local. Sa fonction Auto-Type permet de saisir automatiquement les credentials dans la fenêtre active correspondant à une URL configurée. Un attaquant ayant accès à la session peut ouvrir la page cible et déclencher l'Auto-Type pour faire "taper" le mot de passe par KeePass — sans jamais voir le mot de passe directement dans la base.

---

### Reconnaissance

#### Étape 1 — Rechercher une base KeePass

Connecté en `CTF\lowpriv` sur WS02 :

```powershell
Get-ChildItem C:\Users\lowpriv\Documents\ -Recurse -Filter "*.kdbx"
```

**Résultat :**

```
Mode    Name
----    ----
-a----  ctf.kdbx
```

#### Étape 2 — Ouvrir la base KeePass

Ouvrir KeePass : **File → Open** → sélectionner `ctf.kdbx` → saisir le mot de passe maître `keepass123`.

#### Étape 3 — Explorer les entrées

Dans KeePass, on voit l'entrée **Intranet CTF** avec :

| Champ | Valeur |
|---|---|
| Titre | Intranet CTF |
| Utilisateur | admin |
| URL | http://intranet.ctf.local/flag7 |
| Mot de passe | `●●●●●●●●` (masqué) |

---

### Exploitation

#### Étape 4 — Ouvrir la page cible dans le navigateur

Ouvrir Microsoft Edge sur WS02 :

```
http://intranet.ctf.local/flag7/
```

La page de login s'affiche. Cliquer dans le champ **Utilisateur** pour le mettre en focus.

#### Étape 5 — Déclencher l'Auto-Type

Dans KeePass, avec l'entrée **Intranet CTF** sélectionnée :

- Clic droit → **Perform Auto-Type**
- Ou raccourci global : `Ctrl+Alt+A` (la fenêtre active doit être la page de login)

KeePass tape automatiquement :

1. `admin` dans le champ Utilisateur
2. `Tab` pour passer au champ Mot de passe
3. Le flag comme mot de passe
4. `Entrée` pour valider

**Résultat dans le navigateur :**

```
✅ Connexion réussie !
FLAG{keepass_autotype_credential_leak}
```

---

### Récupération du flag

```
FLAG{keepass_autotype_credential_leak}
```

---

### Aller plus loin

On peut aussi lire le mot de passe directement depuis KeePass sans passer par l'Auto-Type :

```
Clic droit sur l'entrée → Copy Password → coller dans un éditeur de texte
```

Ou afficher le mot de passe en clair :

```
Clic sur l'icône œil dans le champ mot de passe de l'entrée
```

---

### Remédiation

| Problème | Correction |
|---|---|
| Base KeePass accessible sans verrouillage de session | Activer le verrouillage automatique de KeePass après inactivité (`Tools → Options → Security`) |
| Auto-Type sans confirmation | Activer la confirmation avant Auto-Type dans les options KeePass |
| Session Windows non verrouillée | Politique de verrouillage d'écran automatique via GPO |
| Base KeePass sans protection supplémentaire | Utiliser un fichier clé en plus du mot de passe maître |

---

### Résumé de la chaîne d'exploitation

```
lowpriv
  └─→ Get-ChildItem → ctf.kdbx trouvé
        └─→ KeePass → File → Open → mot de passe maître : keepass123
              └─→ entrée "Intranet CTF" → URL : http://intranet.ctf.local/flag7
                    └─→ navigateur → http://intranet.ctf.local/flag7/
                          └─→ focus sur champ Utilisateur
                                └─→ KeePass → Perform Auto-Type (Ctrl+Alt+A)
                                      └─→ FLAG{keepass_autotype_credential_leak}
```
