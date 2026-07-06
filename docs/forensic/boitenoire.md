# [0155] La Boîte Noire du Vaisseau — Write-up

**Catégorie** : Forensic  
**Difficulté** : Moyen-Difficile  
**Flag** : `CryptAbyss{bl4ck_b0x_z3phyr9_fl1ght_d4t4_r3c0v3r3d}`

## Description

La boîte noire d'un vaisseau détruit a été récupérée dans le secteur Omega-12. Le terminal de récupération permet de télécharger le dump binaire. Retrouvez les données classifiées.

## Résolution

### Étape 1 — Parser le format binaire

Le format est documenté sur la page web :
- Magic : `ABYSS7BB` (8 octets)
- Nombre de records : uint32 LE
- Chaque record : type (1 octet) + longueur (uint32 LE) + données

Script Python pour parser :
```python
import struct
data = open("black_box_dump.bin", "rb").read()
assert data[:8] == b"ABYSS7BB"
n = struct.unpack_from("<I", data, 8)[0]
offset = 12
for i in range(n):
    t = data[offset]
    l = struct.unpack_from("<I", data, offset+1)[0]
    d = data[offset+5:offset+5+l]
    print(f"Record {i}: type={t:#x}, len={l}")
    if t == 0x02:
        open("extracted.png", "wb").write(d)  # sauvegarder l'image
    offset += 5 + l
```

### Étape 2 — Extraire les métadonnées de l'image

Le record de type 0x02 est un PNG. En examinant les chunks tEXt (ou avec `exiftool`/`identify -verbose`) :

```
Author = Commandant Zephyr-9
```

### Étape 3 — Lire l'indice dans les logs

Le record CRITIQUE (type 0x01) indique :
> "Clé = MD5 étendu du nom complet de l'officier dans les métadonnées photo. AES-256-CBC."

### Étape 4 — Déchiffrer le payload

- `key = MD5("Commandant Zephyr-9") + MD5(MD5("Commandant Zephyr-9"))` → 32 octets
- IV = 16 premiers octets du blob chiffré (record type 0x03)
- Déchiffrement AES-256-CBC → flag avec padding PKCS7

## Outils

- Python + struct (parsing binaire)
- exiftool ou parsing PNG manuel (métadonnées)
- PyCryptodome ou cryptography (AES)
