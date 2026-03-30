---
name: gpc-subscription-localization
description: Bulk-localize Google Play subscription titles across all locales using the Android Publisher API. Use when you want to add or update subscription display names for multiple languages without clicking through Google Play Console manually.
---

# gpc subscription localization

Use this skill to bulk-add or bulk-update localized titles for Google Play subscriptions. The Google Play Console web UI requires clicking through each locale individually; this skill uses the Android Publisher API to do it in one batch.

## Preconditions
- Google Play service account JSON key available (typically at `~/.config/play-store-key.json`).
- Service account has "Admin" or "Release manager" access in Google Play Console (via Users & Permissions, invited by service account email).
- Python 3 available with `google-auth` and `google-api-python-client` packages.
- Subscriptions already created in Google Play Console.

## Install Dependencies

```bash
pip3 install --target /tmp/gapi google-auth google-api-python-client
```

Use `PYTHONPATH=/tmp/gapi python3` to run scripts if packages install to a non-default location.

## Supported Google Play Locales

Google Play uses BCP-47 locale codes. Common ones for subscription listings:

```
ar, be, bg, ca, cs-CZ, da-DK, de-DE, el-GR, en-AU, en-CA, en-GB,
en-IN, en-SG, en-US, en-ZA, es-419, es-ES, es-US, et, fi-FI,
fil, fr-CA, fr-FR, hi-IN, hr, hu-HU, id, it-IT, iw-IL, ja-JP,
ko-KR, lt, lv, ms, nl-NL, no-NO, pl-PL, pt-BR, pt-PT, ro, ru-RU,
sk, sl, sr, sv-SE, th, tr-TR, uk, vi, zh-CN, zh-TW
```

**Important:** Use full locale codes (e.g., `it-IT` not `it`, `pl-PL` not `pl`). The API rejects short codes with "Cannot specify listings in unsupported language".

## API Details

### Authentication

```python
from google.oauth2 import service_account
from google.auth.transport.requests import AuthorizedSession

SCOPES = ['https://www.googleapis.com/auth/androidpublisher']
creds = service_account.Credentials.from_service_account_file(
    '/path/to/play-store-key.json', scopes=SCOPES)
session = AuthorizedSession(creds)
```

### Subscription Listing Structure

Google Play subscription listings only have two fields:
- `title` (string): Display name shown to users in the purchase flow
- `languageCode` (string): BCP-47 locale code

There is NO description field in the API. Descriptions and "benefits" are only available through the Play Console web UI.

### Critical: regionsVersion Parameter

All PATCH requests to the subscriptions endpoint **must** include `regionsVersion.version=2022/02` as a query parameter. Without this, the API returns:

```
"Regions Version must be specified."
```

The Python client library does not support this parameter as a keyword argument. Use raw HTTP via `AuthorizedSession` instead.

## Workflow: Bulk-Localize Subscriptions

### 1. List Existing Subscriptions

```python
package = 'com.example.app'

# List all subscriptions
r = session.get(
    f'https://androidpublisher.googleapis.com/androidpublisher/v3/applications/{package}/subscriptions')
subs = r.json()

for sub in subs.get('subscriptions', []):
    pid = sub['productId']
    listings = sub.get('listings', [])
    existing_locales = [l['languageCode'] for l in listings]
    print(f'{pid}: {existing_locales}')
```

### 2. Add Missing Locales

```python
product_id = 'my_subscription'

# Get current subscription data
r = session.get(
    f'https://androidpublisher.googleapis.com/androidpublisher/v3/applications/{package}/subscriptions/{product_id}')
sub = r.json()

# Check existing locales
existing = [l['languageCode'] for l in sub.get('listings', [])]

# Add new listings (only for locales not already present)
new_locales = {
    'it-IT': 'My App Premium Mensile',
    'pl-PL': 'My App Premium Miesięczny',
    'tr-TR': 'My App Premium Aylık',
}

for locale, title in new_locales.items():
    if locale not in existing:
        sub['listings'].append({'title': title, 'languageCode': locale})

# Patch with updateMask=listings to only modify listings (not prices)
r = session.patch(
    f'https://androidpublisher.googleapis.com/androidpublisher/v3/applications/{package}/subscriptions/{product_id}',
    params={
        'updateMask': 'listings',
        'regionsVersion.version': '2022/02'
    },
    json=sub
)

if r.status_code == 200:
    print('OK')
else:
    print(f'Error: {r.json().get("error", {}).get("message", "")}')
```

### 3. Verify

```python
r = session.get(
    f'https://androidpublisher.googleapis.com/androidpublisher/v3/applications/{package}/subscriptions/{product_id}')
sub = r.json()
for l in sorted(sub.get('listings', []), key=lambda x: x['languageCode']):
    print(f'  {l["languageCode"]}: {l["title"]}')
```

## Workflow: Bulk-Localize All Subscriptions in an App

```python
# 1. List all subscriptions
r = session.get(
    f'https://androidpublisher.googleapis.com/androidpublisher/v3/applications/{package}/subscriptions')
subs = r.json().get('subscriptions', [])

# 2. Define localized titles per subscription
titles = {
    'my_app_monthly': {
        'it-IT': 'My App Premium Mensile',
        'pl-PL': 'My App Premium Miesięczny',
    },
    'my_app_yearly': {
        'it-IT': 'My App Premium Annuale',
        'pl-PL': 'My App Premium Roczny',
    },
}

# 3. Patch each subscription
for sub in subs:
    pid = sub['productId']
    if pid not in titles:
        continue

    existing = [l['languageCode'] for l in sub.get('listings', [])]
    added = []

    for locale, title in titles[pid].items():
        if locale not in existing:
            sub['listings'].append({'title': title, 'languageCode': locale})
            added.append(locale)

    if not added:
        print(f'{pid}: all locales exist')
        continue

    r = session.patch(
        f'https://androidpublisher.googleapis.com/androidpublisher/v3/applications/{package}/subscriptions/{pid}',
        params={
            'updateMask': 'listings',
            'regionsVersion.version': '2022/02'
        },
        json=sub
    )

    if r.status_code == 200:
        print(f'{pid}: added {added}')
    else:
        print(f'{pid}: ERROR - {r.json().get("error", {}).get("message", "")[:80]}')
```

## In-App Products (One-Time Purchases)

The older `inappproducts` API may return a 403 "Please migrate to the new publishing API" error for some apps. If so, IAP localization must be done manually through the Play Console web UI.

If the API is accessible, IAP listings use a different format (dict keyed by locale, not array):

```python
# IAP listings are a dict: {"en-US": {"title": "...", "description": "..."}, ...}
r = session.get(
    f'https://androidpublisher.googleapis.com/androidpublisher/v3/applications/{package}/inappproducts/{sku}')
iap = r.json()
listings = iap.get('listings', {})
```

## Agent Behavior

- Always list existing subscriptions and their locales first to understand current state.
- Only add locales that are missing; the API replaces the entire listings array on PATCH.
- **Always use `updateMask=listings`** to avoid accidentally modifying prices or regional configs.
- **Always include `regionsVersion.version=2022/02`** in query params.
- Use full locale codes (`it-IT`, not `it`).
- Use `AuthorizedSession` for raw HTTP, not the Python client library (which doesn't support `regionsVersion`).
- If a PATCH fails, check the full error message for the specific locale or field that's invalid.
- After bulk creation, always verify by listing the subscription again.
- When the user provides translated titles, use them. When they provide a single title, use it for all locales.

## Notes
- Google Play subscription listings only have a `title` field. No description, no benefits via API.
- Benefits and descriptions are Play Console UI-only features.
- The `updateMask=listings` parameter is critical: without it, the PATCH would attempt to update ALL fields including pricing, which requires additional regional config data.
- Service account key location is typically `~/.config/play-store-key.json` but may vary.
- The `regionsVersion` requirement was introduced in the 2022 API update for regional pricing support.
