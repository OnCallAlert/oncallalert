# OnCallAlert

Self-hosted on-call scheduling, escalation and alerting, with a companion Android app.

This repo hosts the **public Android builds (APK)**, the **docs** and the **issue tracker** for support, bugs and feature requests.
The server ships as a Docker image. Full documentation: **https://oncallalert.dev/docs.html**

- Website: https://oncallalert.dev
- Pricing and license keys: https://oncallalert.dev/pricing.html
- Contact: contact@oncallalert.dev

---

## Android app

The app connects to your own OnCallAlert server. With it you get push alerts and can acknowledge or resolve alerts from your phone.
It needs a server on **v1.0.0 or later** and a plan that includes the mobile app.

### Install the APK

The Google Play release is coming. Until then, install from this repo:

1. Open [Releases](https://github.com/OnCallAlert/oncallalert/releases/latest) on your phone and download `oncallalert-<version>.apk`.
2. When Android asks, allow **Install unknown apps** for your browser or file manager.
3. Open the downloaded file and tap **Install**.
4. Pair the app with your account. In the OnCallAlert web app, open **Profile → Connect Mobile App**, then scan the QR code with the app.
   If you can't scan, tap **Can't scan? Enter code manually** and type the server URL and pairing token shown on that page.

The app needs Google Play services, because push alerts come through Firebase Cloud Messaging.

To update, install the newer APK over the old one. Your settings are kept.

### Verify the download

Each release has a `SHA256SUMS` file. Check that the APK matches it:

```sh
sha256sum -c SHA256SUMS
```

Every APK is signed with the same release key. Its certificate SHA-256 is:

```
c9:67:c7:e9:b4:0c:9b:c5:50:f5:04:37:24:36:b9:33:54:bb:dc:ce:36:4c:0d:60:42:d3:a6:63:ed:8e:63:d8
```

To check it yourself (Android SDK build-tools): `apksigner verify --print-certs oncallalert-<version>.apk`.

### Alerts during Do Not Disturb

In the app, **Settings → Alarm → Bypass Do Not Disturb** (on by default) makes alerts ring as a full-screen call, even during Do Not Disturb or on a locked screen.
For this to work, allow **Notifications** and **Full-screen notifications** for OnCallAlert (Android settings → Apps → OnCallAlert).

---

## Server quickstart

OnCallAlert runs as one container with a PostgreSQL database (16 or newer, with the `pgcrypto` extension; the server tries to enable it on first start).

### 1. Start the server

Create an empty database, then run the image. Replace the placeholder values with your own.

```sh
docker run -d \
  --name oncallalert \
  --restart unless-stopped \
  -p 8080:8080 \
  -e ONCALLALERT_DB_URL="postgres://user:password@your-postgres:5432/oncallalert?sslmode=require" \
  -e ONCALLALERT_PUBLIC_URL="https://oncall.yourcompany.com" \
  -e ONCALLALERT_DATA_ENCRYPTION_KEY="a-long-random-secret" \
  oncallalert/oncallalert:latest
```

The image is [`oncallalert/oncallalert`](https://hub.docker.com/r/oncallalert/oncallalert) on Docker Hub. `latest` follows the newest release, and you can pin a version tag instead.
The server listens on `0.0.0.0:8080` inside the container and creates its database tables on start.
Keep `ONCALLALERT_DATA_ENCRYPTION_KEY` the same on every restart and upgrade. If it changes, secrets stored in the database can no longer be read.

### 2. Create the first administrator

There is no sign-up page. Create the first account inside the running container:

```sh
docker exec -it oncallalert oncallalert add-user \
  --user admin --email you@example.com --admin
```

Without `--pass`, the command asks for a password, which is why `-it` is needed. Leave out `--admin` to create a normal user.

### 3. Sign in with GitHub or Google (optional)

To let users sign in with GitHub or Google, create an OAuth app with that provider and enter its client ID and secret on the admin settings page.
After that you can turn off password login entirely.

### 4. Sign in and activate

Open your public URL and sign in as the administrator. Then the **Activate OnCallAlert** screen asks for your license key (`oca_live_...`, from the [pricing page](https://oncallalert.dev/pricing.html)).
Paste it and submit. A confirmation screen shows who the license is for, the plan and the expiry date.

- Until it is activated, the server is locked. Only login, activation, static files and the health check work, and alert intake is closed.
- The server checks the license once a day, and when you first enter a key. What it sends is listed in the [Privacy Policy](https://oncallalert.dev/privacy.html). Grace periods and offline behaviour are in the [Terms](https://oncallalert.dev/terms.html).
- Admins can see, refresh or change the key at **Admin → License** (`/admin/license`).

## Configuration

Options are environment variables that start with `ONCALLALERT_` (the command-line flag in upper case, with `-` replaced by `_`). The older `GOALERT_` prefix also works, and it wins if both are set.
The license key is not an environment variable; you enter it in the web UI.

| Variable | Required | Description |
|---|---|---|
| `ONCALLALERT_DB_URL` | Yes | PostgreSQL connection string (`postgres://...`). |
| `ONCALLALERT_PUBLIC_URL` | Recommended | URL that users and services can reach the server at. Used for links, sign-in callbacks and license checks. |
| `ONCALLALERT_DATA_ENCRYPTION_KEY` | Recommended | Any string. Encrypts sensitive values in the database. Keep it identical across restarts and upgrades. |
| `ONCALLALERT_LISTEN` | No | Address and port to listen on (the image sets `0.0.0.0:8080`). |

## Sending alerts

Create an integration key on a service, then point your monitoring tool at it. Grafana, Prometheus Alertmanager and Site24x7 webhooks work directly.
Anything else can use the generic API: use a key of type **Generic API** as the token.

```sh
curl -X POST https://oncall.yourcompany.com/api/v2/generic/incoming \
  -H "Authorization: Bearer your-integration-key" \
  -H "Content-Type: application/json" \
  -d '{
    "summary": "Production Redis Out of Memory",
    "details": "Replica node memory saturation exceeded 95%",
    "dedup": "redis-memory"
  }'
```

`dedup` is optional and groups repeat alerts.

## Infrastructure as code (Terraform, OpenTofu, Pulumi)

Manage escalation policies, rotations, schedules, overrides, services and integration keys as code with the
[OnCallAlert provider](https://registry.terraform.io/providers/OnCallAlert/oncallalert/latest)
([source](https://github.com/OnCallAlert/terraform-provider-oncallalert)). It talks only to your own server and is included with every license.

1. You need a server newer than v1.0.0 with an active license.
2. As an admin, open your profile → **API Tokens** → **Create token**, scope **Admin**. Copy the token; it is shown once.
3. Configure the provider:

```hcl
terraform {
  required_providers {
    oncallalert = {
      source = "OnCallAlert/oncallalert"
    }
  }
}

provider "oncallalert" {
  url   = "https://oncall.yourcompany.com" # or ONCALLALERT_URL
  token = var.oncallalert_token            # or ONCALLALERT_TOKEN
}
```

- **OpenTofu:** the same configuration works; the provider is also on the [OpenTofu registry](https://search.opentofu.org/provider/oncallalert/oncallalert).
- **Pulumi:** `pulumi package add terraform-provider OnCallAlert/oncallalert`, then set `oncallalert:url` and `oncallalert:token` (as a secret).
- Existing objects can be brought under code with `terraform import`. Resource list and examples: [provider docs](https://registry.terraform.io/providers/OnCallAlert/oncallalert/latest/docs).
- Step-by-step guide: https://oncallalert.dev/docs.html#iac

## Moving from PagerDuty, Opsgenie or Jira Service Management

Open **Admin → Import** (`/admin/import`) as an admin. Enter a read-only API key from your old tool, check the preview, and import.
Users, contact methods, teams, schedules, escalation policies and services come across. Your old tool is only read; nothing changes there.
Step-by-step guide: https://oncallalert.dev/docs.html#migrate

---

## Support

- **Bugs and feature requests:** [open an issue](https://github.com/OnCallAlert/oncallalert/issues/new). Include your server version, Android version and app version, and what you expected to happen.
- **Security issues:** do not open a public issue. Email contact@oncallalert.dev.
- **Billing and licenses:** contact@oncallalert.dev.

Never post license keys, passwords, API keys or integration keys in an issue.
