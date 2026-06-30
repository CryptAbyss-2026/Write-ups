# Write-up : Falsification JWT (Partie 2)

## Description
Ce challenge fait suite au précédent. L'objectif est d'exploiter une vulnérabilité de signature sur un JSON Web Token (JWT) pour obtenir des privilèges administrateur.

## Analyse
Le jeton `token.txt` contient un payload encodé en Base64 : `{"user": "guest"}`. 
L'algorithme utilisé est le **HS256** (HMAC-SHA256). La sécurité repose entièrement sur la complexité de la clé secrète. Si cette clé est découverte, il est possible de signer un nouveau jeton avec l'utilisateur `admin`.

## Résolution
1. **Analyse du jeton** : Décoder le payload pour identifier le champ à modifier.
2. **Brute-force de la signature** : Utiliser un dictionnaire pour retrouver le secret `secret123`.
3. **Forge du jeton** : Créer un nouveau jeton avec `{"user": "admin"}` et le signer avec le secret trouvé.
4. **Validation** : Soumettre le jeton au binaire `check_token`.

## Commandes

### 1. Brute-force du secret
```bash
# Utilisation de Hashcat (Mode 16500 = JWT)
hashcat -m 16500 token.txt dico.txt
# Affichage du résultat
hashcat -m 16500 token.txt dico.txt --show
```

### 2. Génération du token Admin
```bash
python3 -c "
import hmac, hashlib, base64
s = b'secret123'
h = base64.urlsafe_b64encode(b'{\"alg\":\"HS256\",\"typ\":\"JWT\"}').decode().strip('=')
p = base64.urlsafe_b64encode(b'{\"user\":\"admin\"}').decode().strip('=')
data = f'{h}.{p}'.encode()
sig = base64.urlsafe_b64encode(hmac.new(s, data, hashlib.sha256).digest()).decode().strip('=')
print(f'{h}.{p}.{sig}')"
```

### 3. Obtention du flag
```bash
./check_token
```
