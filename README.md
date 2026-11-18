# Nest Thermostat E → Apple HomeKit

The Thermostat E has no Matter or HomeKit support, so it needs a bridge. This
runs [Homebridge](https://homebridge.io) with the
[`homebridge-google-nest-sdm`](https://github.com/potmat/homebridge-google-nest-sdm)
plugin, which uses Google's official Smart Device Management (SDM) API.

## Requirements

- An always-on Linux machine (Raspberry Pi, NAS, mini PC) with Docker, on the
  same network as your Apple home hub. Docker Desktop on macOS/Windows will not
  work: host networking there does not pass mDNS.
- The Google account that owns the thermostat. If the thermostat is still on a
  legacy Nest account, migrate it to a Google account first.
- $5 one-time Google Device Access registration fee.

## 1. Google setup (about 20 minutes)

Follow Google's guide: <https://developers.google.com/nest/device-access/get-started>.
Collect these values as you go:

| Value | Where it comes from |
|---|---|
| `clientId`, `clientSecret` | Google Cloud Console → OAuth client (type: Web application, redirect URI `https://www.google.com`) |
| `gcpProjectId` | The Google Cloud project you created for the OAuth client |
| `projectId` | Device Access Console → your project |
| `subscriptionId` | Pub/Sub pull subscription on the Device Access project's topic |
| `refreshToken` | The authorize step below |

Also enable the **Smart Device Management API** and **Cloud Pub/Sub API** in
the Google Cloud project, and set the OAuth consent screen to **In production**
(in "Testing" the refresh token expires after 7 days).

**Authorize with this URL, not the one in Google's guide.** It adds the
`pubsub` scope, without which temperature updates never reach HomeKit:

```
https://nestservices.google.com/partnerconnections/PROJECT_ID/auth?redirect_uri=https://www.google.com&access_type=offline&prompt=consent&client_id=CLIENT_ID&response_type=code&scope=https://www.googleapis.com/auth/sdm.service+https://www.googleapis.com/auth/pubsub
```

Tick the thermostat, approve, copy the `code=` value from the redirect URL,
then exchange it for a refresh token:

```sh
curl -L -X POST 'https://www.googleapis.com/oauth2/v4/token' \
  -d client_id=CLIENT_ID -d client_secret=CLIENT_SECRET \
  -d code=AUTH_CODE -d grant_type=authorization_code \
  -d redirect_uri=https://www.google.com
```

## 2. Start Homebridge

On the UGREEN DXP8800 (UGOS Pro):

1. App Center → install **Docker**.
2. Files → create the folder `docker/homebridge` on a shared volume and upload
   `docker-compose.yml` into it.
3. Docker → Project → Create → name it `homebridge`, select that folder, and
   use the existing compose file → Deploy.

Or over SSH (Control Panel → Terminal → enable SSH), from that folder:

```sh
sudo docker compose up -d
```

Open `http://<nas-ip>:8581` and create the admin user.

## 3. Install and configure the plugin

1. Plugins → search `homebridge-google-nest-sdm` → Install.
2. Enter the values from step 1 in the plugin settings (same fields as
   [`nest-platform.example.json`](nest-platform.example.json)).
3. Restart Homebridge. The log should list the thermostat.

## 4. Pair with HomeKit

Home app → Add Accessory → scan the QR code on the Homebridge status page.
The thermostat appears as a normal HomeKit thermostat (mode, target and current
temperature).

## Notes

- `homebridge/` holds your OAuth secrets and HomeKit pairing keys. It is
  gitignored; back it up privately and never commit it.
- Eco mode is exposed by the plugin as a separate switch.
- If the thermostat shows "No Response", check it is online in the Google Home
  app first.
