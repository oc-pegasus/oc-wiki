# Privacy Rules

This is a public knowledge base. Follow these rules strictly.

## Hard Rules (Non-negotiable)

- Never publish real personal identifiers (names, Discord IDs, handles, emails, phone numbers)
- Never publish secrets (tokens, keys, passwords, internal URLs, credentials)
- Never publish customer/internal/private project data

## Allowed Content

- General methods, lessons learned, templates, anonymized examples
- Redacted snippets only

## Redaction Standard

Replace sensitive values with placeholders:

- `<@USER_ID>` / `OWNER_USER_ID`
- `<ROLE_NAME>` / `MAIN_BOT_ID`
- `<INTERNAL_URL>`
- `<API_KEY>`

## Release Check

Before every commit/push:

1. Scan for numeric IDs and mentions
2. Confirm all examples are anonymized
3. If uncertain, do not publish
