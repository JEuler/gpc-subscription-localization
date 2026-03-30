# gpc-subscription-localization

A [Claude Code](https://claude.ai/code) skill for bulk-localizing Google Play subscription titles via the Android Publisher API.

## What it does

Adds localized subscription display names across all Google Play locales in one batch, instead of clicking through each language manually in the Play Console.

## Install

```bash
claude install-skill JEuler/gpc-subscription-localization
```

## Usage

Ask Claude Code to localize your Google Play subscriptions:

```
Localize my Google Play subscriptions for Italian, Polish, and Turkish
```

Or invoke directly:

```
/gpc-subscription-localization
```

## Prerequisites

- Google Play service account JSON key (typically `~/.config/play-store-key.json`)
- Service account with Admin or Release manager access in Play Console
- Python 3 with `google-auth` and `google-api-python-client`
- Subscriptions already created in Play Console

## Key API gotchas this skill handles

- **`regionsVersion.version=2022/02`** must be included as a query parameter on all PATCH requests
- **`updateMask=listings`** is critical to avoid accidentally overwriting prices
- **Full locale codes required** (`it-IT` not `it`, `pl-PL` not `pl`)
- **Python client library doesn't work** for this endpoint (doesn't support `regionsVersion` param), must use raw HTTP via `AuthorizedSession`
- **Listings only have `title`**, no description or benefits (those are Play Console UI only)
- **IAP (one-time products)** may return 403 requiring migration to a newer API

## License

MIT
