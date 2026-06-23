# Pixel Tracker Email — Cloudflare Worker + D1

> Déploiement d'un pixel de tracking 1×1 pour emails via Cloudflare Workers et base de données D1, sans serveur, sans hébergement.

---

## Principe

Un pixel de tracking est une image GIF transparente 1×1 px intégrée dans un email HTML. Quand le destinataire ouvre le mail, son client charge l'image → le Worker Cloudflare enregistre l'événement (token, IP, user-agent, heure, ville) de façon persistante dans une base D1.

**Avantages de cette approche :**
- Zéro serveur à maintenir
- Gratuit (100 000 requêtes/jour, 5 Go de base D1 en free tier Cloudflare)
- Déployé sur l'edge Cloudflare (rapide, fiable)
- Logs persistants consultables en SQL

---

## Prérequis

- Un compte Cloudflare (gratuit sur [cloudflare.com](https://cloudflare.com))
- PowerShell (Windows)

---

## Fichiers

### `index.js` — Code du Worker

```javascript
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url)
    const token   = url.searchParams.get('t') || ''
    const ip      = request.headers.get('CF-Connecting-IP') || ''
    const ua      = request.headers.get('User-Agent') || ''
    const country = request.cf?.country || ''
    const city    = request.cf?.city || ''
    const time    = new Date().toISOString()

    ctx.waitUntil(
      env.DB.prepare(
        "INSERT INTO opens (token, ip, ua, country, city, time) VALUES (?, ?, ?, ?, ?, ?)"
      ).bind(token, ip, ua, country, city, time).run()
    )

    const gif = new Uint8Array([
      0x47,0x49,0x46,0x38,0x39,0x61,0x01,0x00,
      0x01,0x00,0x80,0x00,0x00,0xff,0xff,0xff,
      0x00,0x00,0x00,0x21,0xf9,0x04,0x00,0x00,
      0x00,0x00,0x00,0x2c,0x00,0x00,0x00,0x00,
      0x01,0x00,0x01,0x00,0x00,0x02,0x02,0x44,
      0x01,0x00,0x3b
    ])

    return new Response(gif, {
      headers: {
        'Content-Type':  'image/gif',
        'Cache-Control': 'no-store',
      },
    })
  }
}
```

### `deploy.ps1` — Script PowerShell de déploiement

À conserver et adapter pour un futur redéploiement.

```powershell
$token      = "TON_API_TOKEN"
$accountId  = "TON_ACCOUNT_ID"
$workerName = "pixel-tracker"
$dbId       = "TON_DB_UUID"
```

---

## Procédure de déploiement (première installation)

### Étape 1 — Créer un API Token Cloudflare

1. Aller sur [dash.cloudflare.com/profile/api-tokens](https://dash.cloudflare.com/profile/api-tokens)
2. **Create Token** → template **"Edit Cloudflare Workers"**
3. **Continue to summary** → 
4. Ajouter une ligne → Compte → D1 → Modifier
5. Cliquer sur **Create Token**
4. **Copier le token** (affiché une seule fois)

Permissions accordées :
- Workers Scripts : Modifier
- Workers KV Storage : Modifier
- D1 : Modifier
- Account Settings : Lire
- Workers Routes : Modifier

---

### Étape 2 — Déployer le Worker (format ES Module)

Le Worker doit être déployé en **multipart/form-data** avec déclaration du format ES module (obligatoire pour le binding D1).

```powershell
$code = Get-Content "C:\chemin\vers\index.js" -Raw

$boundary = "----FormBoundary$(Get-Random)"
$metadata = '{"main_module":"index.js","bindings":[],"compatibility_date":"2024-01-01"}'

$body = @"
--$boundary
Content-Disposition: form-data; name="metadata"
Content-Type: application/json

$metadata
--$boundary
Content-Disposition: form-data; name="index.js"; filename="index.js"
Content-Type: application/javascript+module

$code
--$boundary--
"@

Invoke-RestMethod `
    -Uri "https://api.cloudflare.com/client/v4/accounts/$accountId/workers/scripts/$workerName" `
    -Method PUT `
    -Headers @{ Authorization = "Bearer $token" } `
    -ContentType "multipart/form-data; boundary=$boundary" `
    -Body $body
```

> **Important :** Ne pas utiliser `Content-Type: application/javascript` seul — Cloudflare rejette la syntaxe `export default` dans ce mode.

---

### Étape 3 — Activer le subdomain workers.dev

```powershell
Invoke-RestMethod `
    -Uri "https://api.cloudflare.com/client/v4/accounts/$accountId/workers/scripts/$workerName/subdomain" `
    -Method POST `
    -Headers @{ Authorization = "Bearer $token" } `
    -ContentType "application/json" `
    -Body '{"enabled": true}'
```

L'URL du pixel sera :
```
https://pixel-tracker.TON_SUBDOMAIN.workers.dev/?t=IDENTIFIANT
```

Le subdomain est visible dans **Workers & Pages → pixel-tracker** dans le dashboard.

---

### Étape 4 — Créer la base de données D1

```powershell
Invoke-RestMethod `
    -Uri "https://api.cloudflare.com/client/v4/accounts/$accountId/d1/database" `
    -Method POST `
    -Headers @{ Authorization = "Bearer $token" } `
    -ContentType "application/json" `
    -Body '{"name": "pixel-tracker-db"}'
```

**Noter l'`uuid` retourné** → c'est le `$dbId` pour les étapes suivantes.

---

### Étape 5 — Créer la table

```powershell
$dbId = "UUID_RETOURNÉ_ÉTAPE_4"

Invoke-RestMethod `
    -Uri "https://api.cloudflare.com/client/v4/accounts/$accountId/d1/database/$dbId/query" `
    -Method POST `
    -Headers @{ Authorization = "Bearer $token" } `
    -ContentType "application/json" `
    -Body '{"sql": "CREATE TABLE IF NOT EXISTS opens (id INTEGER PRIMARY KEY AUTOINCREMENT, token TEXT, ip TEXT, ua TEXT, country TEXT, city TEXT, time TEXT);"}'
```

---

### Étape 6 — Lier la base D1 au Worker (dashboard obligatoire)

> L'API REST ne supporte pas la gestion des bindings — cette étape se fait uniquement via le dashboard.

1. **Workers & Pages → pixel-tracker → Settings → Bindings**
2. **Ajouter une liaison → Base de données D1**
3. Renseigner :
   - **Variable name** : `DB`
   - **D1 database** : `pixel-tracker-db`
4. **Save**

> Si le binding est créé mais non pris en compte, éditer le code directement depuis le dashboard (**Edit Code**) et faire **Save and Deploy** — cela force le redéploiement avec le binding actif.

---

## Utilisation dans les emails

```html
<img src="https://pixel-tracker.TON_SUBDOMAIN.workers.dev/?t=IDENTIFIANT" width="1" height="1" />
```

Pour outlook 365 il est uniquement possible de rajouter ce code à la fin du body dans la signature
C:\Users\...\AppData\Roaming\Microsoft\Signatures

### Exemples d'identifiants

| Cas d'usage | Exemple |
|---|---|
| Mail spécifique | `?t=reunion-secu-2026-06-23` |
| Par destinataire | `?t=jean.dupont` |
| Numéro incrémental | `?t=mail-042` |

---

## Consulter les logs

### Via PowerShell

```powershell
$dbId = "UUID_RETOURNÉ_ÉTAPE_4"

$response = Invoke-RestMethod `
    -Uri "https://api.cloudflare.com/client/v4/accounts/$accountId/d1/database/$dbId/query" `
    -Method POST `
    -Headers @{ Authorization = "Bearer $token" } `
    -ContentType "application/json" `
    -Body '{"sql": "SELECT * FROM opens ORDER BY id DESC LIMIT 10;"}'

$response.result.results
```

Affichage en tableau :

```powershell
$response.result.results | Format-Table id, token, ip, city, time -AutoSize
```

### Via le dashboard Cloudflare

**Workers & Pages → D1 → pixel-tracker-db → Console**

Exemples de requêtes SQL :

```sql
-- 10 dernières ouvertures
SELECT * FROM opens ORDER BY id DESC LIMIT 10;

-- Ouvertures par token
SELECT token, COUNT(*) as nb FROM opens GROUP BY token ORDER BY nb DESC;

-- Ouvertures d'un destinataire précis
SELECT * FROM opens WHERE token = 'jean.dupont' ORDER BY time DESC;
```

---

## Redéployer le Worker (mises à jour futures)

```powershell
$code = Get-Content "C:\chemin\vers\index.js" -Raw

$boundary = "----FormBoundary$(Get-Random)"
$metadata = '{"main_module":"index.js","bindings":[],"compatibility_date":"2024-01-01"}'

$body = @"
--$boundary
Content-Disposition: form-data; name="metadata"
Content-Type: application/json

$metadata
--$boundary
Content-Disposition: form-data; name="index.js"; filename="index.js"
Content-Type: application/javascript+module

$code
--$boundary--
"@

Invoke-RestMethod `
    -Uri "https://api.cloudflare.com/client/v4/accounts/$accountId/workers/scripts/$workerName" `
    -Method PUT `
    -Headers @{ Authorization = "Bearer $token" } `
    -ContentType "multipart/form-data; boundary=$boundary" `
    -Body $body
```

> Après un redéploiement PowerShell, vérifier que le binding D1 est toujours actif dans le dashboard. Si le Worker retourne une erreur `Cannot read properties of undefined (reading 'prepare')`, le binding a été perdu → le recréer via **Settings → Bindings**.

---

## Limitations connues

| Limitation | Impact |
|---|---|
| **Apple Mail Privacy Protection** (iOS 15+) | Précharge les images → ouverture détectée même sans lecture réelle |
| **Gmail** | Proxifie les images → IP enregistrée = Google, pas le destinataire |
| **Proxy d'entreprise (Netskope)** | IP enregistrée = celle du proxy, pas de l'utilisateur individuel |
| **Clients mail bloquant les images** | Pas de tracking si l'utilisateur n'autorise pas le chargement |
| **Binding D1 perdu au redéploiement PS** | À recréer manuellement dans le dashboard après chaque redéploiement PowerShell |

---

## Références

- [Cloudflare Workers — Documentation](https://developers.cloudflare.com/workers/)
- [Cloudflare D1 — Documentation](https://developers.cloudflare.com/d1/)
- [Migrer vers ES Module Workers](https://developers.cloudflare.com/workers/reference/migrate-to-module-workers/)
- [Cloudflare API — Workers Scripts](https://developers.cloudflare.com/api/operations/worker-script-upload-worker-module)
