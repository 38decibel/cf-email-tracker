# Pixel Email Tracker — Cloudflare Worker + D1

> Deploying an pixel email tracking, using Cloudflare Workers and a D1 database — serverless, hosting-free, and maintenance-free.

# Legal & Privacy Notice

This methods provides technical capabilities for generating and managing email tracking pixels. It is intended solely for lawful and transparent use.

Users are solely responsible for ensuring that their use of this software complies with all applicable laws and regulations, including but not limited to privacy, electronic communications, and data protection legislation (such as the GDPR, the ePrivacy Directive, and any applicable national implementations).

In particular, users sending emails to recipients located in France should be aware of the recommendations published by the French Data Protection Authority (CNIL) regarding the use of email tracking pixels. Depending on the purpose of the tracking (e.g. marketing, audience measurement, profiling), prior consent from recipients may be required, while certain strictly necessary technical uses may be exempt under specific conditions.

This project does not encourage or endorse the use of tracking technologies in violation of applicable laws. By using this methods, you acknowledge that you are responsible for determining the legal basis for your processing activities, providing appropriate information to recipients, obtaining any required consent, and respecting recipients' rights.

For more information, please refer to the CNIL recommendation on email tracking pixels:
[https://www.cnil.fr/fr/recommandation-pixel-suivi-courriels](https://www.cnil.fr/fr/recommandation-pixel-suivi-courriels)

# Overview

A tracking pixel is a transparent 1×1 GIF image embedded in an HTML email. When the recipient opens the email, their mail client loads the image, causing the Cloudflare Worker to record the event (token, IP address, user agent, timestamp, country, city) into a persistent D1 database.

**Benefits of this approach:**

* No server to maintain
* Free (100,000 requests/day and 5 GB of D1 storage on the Cloudflare Free plan)
* Deployed on the Cloudflare Edge (fast and reliable)
* Persistent logs that can be queried using SQL

# Prerequisites

* A Cloudflare account (free at [https://cloudflare.com](https://cloudflare.com))
* PowerShell (Windows)

# Files

## `index.js` — Worker Code

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
        'Content-Type': 'image/gif',
        'Cache-Control': 'no-store',
      },
    })
  }
}
```

## `deploy.ps1` — PowerShell Deployment Script

Keep this script and update it whenever you need to redeploy.

```powershell
$token      = "YOUR_API_TOKEN"
$accountId  = "YOUR_ACCOUNT_ID"
$workerName = "pixel-tracker"
$dbId       = "YOUR_DATABASE_UUID"
```

# Deployment Procedure (First Installation)

## Step 1 — Create a Cloudflare API Token

1. Go to [https://dash.cloudflare.com/profile/api-tokens](https://dash.cloudflare.com/profile/api-tokens)
2. Click **Create Token**
3. Choose the **"Edit Cloudflare Workers"** template
4. Click **Continue to Summary**
5. Add a permission:

   * Account → D1 → Edit
6. Click **Create Token**
7. **Copy the token** (it is displayed only once).

Granted permissions:

* Workers Scripts: Edit
* Workers KV Storage: Edit
* D1: Edit
* Account Settings: Read
* Workers Routes: Edit

## Step 2 — Deploy the Worker (ES Module Format)

The Worker must be deployed using **multipart/form-data** while explicitly declaring the ES Module format (required for D1 bindings).

```powershell
$code = Get-Content "C:\path\to\index.js" -Raw

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

> **Important:** Do **not** use `Content-Type: application/javascript` by itself. Cloudflare will reject the `export default` syntax in that mode.

## Step 3 — Enable the workers.dev Subdomain

```powershell
Invoke-RestMethod `
    -Uri "https://api.cloudflare.com/client/v4/accounts/$accountId/workers/scripts/$workerName/subdomain" `
    -Method POST `
    -Headers @{ Authorization = "Bearer $token" } `
    -ContentType "application/json" `
    -Body '{"enabled": true}'
```

Your tracking pixel URL will be:

```
https://pixel-tracker.YOUR_SUBDOMAIN.workers.dev/?t=IDENTIFIER
```

The subdomain can be found under **Workers & Pages → pixel-tracker** in the Cloudflare dashboard.

## Step 4 — Create the D1 Database

```powershell
Invoke-RestMethod `
    -Uri "https://api.cloudflare.com/client/v4/accounts/$accountId/d1/database" `
    -Method POST `
    -Headers @{ Authorization = "Bearer $token" } `
    -ContentType "application/json" `
    -Body '{"name": "pixel-tracker-db"}'
```

**Save the returned `uuid`**—this becomes `$dbId` for the following steps.

## Step 5 — Create the Table

```powershell
$dbId = "UUID_RETURNED_IN_STEP_4"

Invoke-RestMethod `
    -Uri "https://api.cloudflare.com/client/v4/accounts/$accountId/d1/database/$dbId/query" `
    -Method POST `
    -Headers @{ Authorization = "Bearer $token" } `
    -ContentType "application/json" `
    -Body '{"sql": "CREATE TABLE IF NOT EXISTS opens (id INTEGER PRIMARY KEY AUTOINCREMENT, token TEXT, ip TEXT, ua TEXT, country TEXT, city TEXT, time TEXT);"}'
```

## Step 6 — Bind the D1 Database to the Worker (Dashboard Required)

> The REST API does **not** support managing bindings. This step must be completed through the Cloudflare dashboard.

1. **Workers & Pages → pixel-tracker → Settings → Bindings**
2. **Add Binding → D1 Database**
3. Configure:

   * **Variable name:** `DB`
   * **D1 database:** `pixel-tracker-db`
4. Click **Save**

> If the binding exists but is not being applied, edit the Worker directly in the dashboard (**Edit Code**) and click **Save and Deploy**. This forces a redeployment with the binding attached.

# Using the Tracking Pixel in Emails

```html
<img src="https://pixel-tracker.YOUR_SUBDOMAIN.workers.dev/?t=IDENTIFIER" width="1" height="1" />
```

For Outlook 365, it is only possible to append this HTML to the end of the email body by editing the signature files located in:

```
C:\Users\...\AppData\Roaming\Microsoft\Signatures
```

## Example Identifiers

| Use Case       | Example                          |
| -------------- | -------------------------------- |
| Specific email | `?t=security-meeting-2026-06-23` |
| Per recipient  | `?t=john.smith`                  |
| Incremental ID | `?t=mail-042`                    |

# Viewing the Logs

## Using PowerShell

```powershell
$dbId = "UUID_RETURNED_IN_STEP_4"

$response = Invoke-RestMethod `
    -Uri "https://api.cloudflare.com/client/v4/accounts/$accountId/d1/database/$dbId/query" `
    -Method POST `
    -Headers @{ Authorization = "Bearer $token" } `
    -ContentType "application/json" `
    -Body '{"sql": "SELECT * FROM opens ORDER BY id DESC LIMIT 10;"}'

$response.result.results
```

Display as a table:

```powershell
$response.result.results | Format-Table id, token, ip, city, time -AutoSize
```

## Using the Cloudflare Dashboard

**Workers & Pages → D1 → pixel-tracker-db → Console**

Example SQL queries:

```sql
-- Last 10 opens
SELECT * FROM opens ORDER BY id DESC LIMIT 10;

-- Opens grouped by token
SELECT token, COUNT(*) AS nb
FROM opens
GROUP BY token
ORDER BY nb DESC;

-- Opens for a specific recipient
SELECT *
FROM opens
WHERE token = 'john.smith'
ORDER BY time DESC;
```

---

# Redeploying the Worker (Future Updates)

```powershell
$code = Get-Content "C:\path\to\index.js" -Raw

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

> After redeploying via PowerShell, verify that the D1 binding is still present in the dashboard. If the Worker returns the error `Cannot read properties of undefined (reading 'prepare')`, the binding has been lost and must be recreated under **Settings → Bindings**.

---

# Known Limitations

| Limitation                                        | Impact                                                                                             |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| **Apple Mail Privacy Protection** (iOS 15+)       | Images are preloaded, resulting in false-positive open events.                                     |
| **Gmail**                                         | Images are proxied through Google, so the recorded IP belongs to Google rather than the recipient. |
| **Corporate proxies (e.g., Netskope)**            | The recorded IP is the proxy's IP instead of the individual user's.                                |
| **Email clients blocking remote images**          | No tracking occurs unless the recipient allows image loading.                                      |
| **D1 binding lost after PowerShell redeployment** | The binding must be recreated manually in the dashboard after each PowerShell deployment.          |

# References

* Cloudflare Workers Documentation
* Cloudflare D1 Documentation
* Migrating to ES Module Workers
* Cloudflare API — Worker Script Upload
