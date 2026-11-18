# Nest Thermostat E → Apple HomeKit

The Thermostat E has no Matter or HomeKit support. It is bridged through
Home Assistant (Container install on the UGREEN DXP8800): the **Google Nest**
integration pulls the thermostat in via Google's Smart Device Management API,
and the **HomeKit Bridge** integration exposes it to the Apple Home app.

No secrets are stored in this repo. The OAuth client secret lives only in
Home Assistant (Settings → Devices & services → ⋮ → Application credentials).

## Google side (one-time, $5)

1. Register at <https://console.nest.google.com/device-access> and pay the fee.
2. In <https://console.cloud.google.com>, create a project and enable the
   **Smart Device Management API** and **Cloud Pub/Sub API**.
3. Google Auth Platform → Branding: fill in app name, support email, home page,
   privacy policy and terms links, authorized domain, developer contact.
   Audience → External → **Publish app** (in "Testing", tokens expire after
   7 days). Do not submit for verification.
4. Clients → create an OAuth client, type **Web application**, with the
   authorized redirect URI `https://my.home-assistant.io/redirect/oauth`.
5. Pub/Sub → create a topic (e.g. `nest-events`) with a default pull
   subscription, and grant `sdm-publisher@googlegroups.com` the
   **Pub/Sub Publisher** role on the topic.
6. Device Access console → create a project with the OAuth client ID, enable
   events, and enter the topic `projects/<cloud-project-id>/topics/nest-events`.

## Home Assistant side

1. Settings → Devices & services → **+ Add integration** → `nest` → Google →
   **Google Nest**. Enter the Cloud project ID, the Device Access project ID
   (a UUID), and the OAuth client ID and secret.
2. Approve on Google's pages (Advanced → continue past the "unverified app"
   warning), link the account, then select the Pub/Sub topic and subscription.
3. Open the **HomeKit Bridge** integration → gear icon on the bridge row → add
   the **Climate** domain. The thermostat appears in the Home app with no
   re-pairing.

For a first-time HomeKit Bridge, add the integration, include only Climate, and
scan the QR code from Home Assistant's Notifications in the Home app. The
Home Assistant container must use host networking for HomeKit discovery.

## Troubleshooting

| Symptom | Cause |
|---|---|
| "Can't link to …" or "Something went wrong" on Google's page | Redirect URI missing or not yet active (wait 5 minutes), or the Cloud project ID and Device Access project ID are swapped |
| Thermostat stops updating after a week | App still in "Testing"; publish it and re-authenticate the integration |
| "No Response" in the Home app | Check the thermostat is online in the Google Home app, then that Home Assistant is running |
