# Universe email templates

Authentik renders its user-facing email from its own Django templates. Everything
in this directory overrides those, so that recovery, verification and
notification emails are Universe-branded instead of Authentik-branded.

## Why these live in the repo

Until now the templates existed **only as files on the production host**
(`/data/authentik/templates/email/`). A database dump/restore does not carry files,
so the Railway → Coolify migration silently lost them and every email reverted to
the default Authentik look — the same class of loss that took the SMTP settings
with it. This directory is the source of truth; the host is a deployment target.

## Contents

| file | used by |
|---|---|
| `base.html` | every other template extends this — logo, card, colours, footer |
| `password_reset.html` / `.txt` | the "Reset your Universe password" stage |
| `account_confirmation.html` / `.txt` | the "Universe Email Verification" stage |
| `event_notification.html` / `.txt` | notification transports (2 are configured) |
| `email_otp.html` / `.txt` | email-MFA authenticator stage (currently unused) |
| `setup.html` / `.txt` | setup/invite email and `manage.py test_email` |

## Two rules that must hold

1. **Keep `.html` and `.txt` in sync.** They are separate templates and a client may
   show either. Every `.txt` file carried its own copy of the wording *and* its own
   `Powered by goauthentik.io.` footer — fixing only the HTML leaves plain-text
   clients looking unbranded.

2. **Never put brand-critical artwork in an image that assumes a background colour.**
   Mail clients invert *colours* but never *images*. That is why the header uses a
   coloured mark (survives any background) plus the wordmark as **text** (text
   inverts with the card, so contrast always holds). A white-on-transparent logo
   image is invisible the moment a client flips the card to light.

## Why the markup looks the way it does

Written for clients that discard in-head CSS (the Gmail app does):

- backgrounds sit on `<td>` with **both** `bgcolor="…"` and
  `style="background-color:…"` — never on `<table>` alone, and never the
  `background:` shorthand;
- the card's text colour is set **once** on the container `<td>` and inherited, so
  content templates need no per-element colour;
- button colours are **inlined** (the class styling in `base.html` is only a
  progressive enhancement for clients that honour it);
- the `<style>` block is retained but nothing depends on it.

## Known client behaviour

Inverting dark-mode clients (Gmail app) flip our dark card to light and the light
text to dark — so it stays readable — but they also flip the brand blue button to
orange, and they do **not** invert the logo mark. The design is built to be legible
in both states rather than to look identical in both.

## Applying changes

**Preferred — bake into the image (`Dockerfile.server`):**

```dockerfile
COPY --chown=1000:1000 custom-templates/email/ /templates/email/
```

**Current production deployment — host bind mount.** The live instance mounts
`/data/authentik/templates → /templates`, so copy the files there. Ownership and
mode matter: the container runs as uid 1000, and Authentik logs
`Custom template file is not readable, check permissions` if it cannot read them.

```bash
scp custom-templates/email/* root@<identity-host>:/data/authentik/templates/email/
ssh root@<identity-host> 'chown -R 1000:1000 /data/authentik/templates/email && chmod 644 /data/authentik/templates/email/*'
```

Templates are read on each render, so **no restart is required**.

## Verifying a change

`manage.py test_email` renders `email/setup.html` through the real path (the
recipient is positional, not `--to`):

```bash
docker exec <server-container> python manage.py test_email you@bawes.net
```

The body is stored on the resulting event, so it can be checked without a mailbox:

```sql
SELECT to_char(created AT TIME ZONE 'UTC','HH24:MI:SS') || ' | ' || (context #>> '{message}')
     || ' | bgcolor=' || ((length(context #>> '{body}') - length(replace(context #>> '{body}','bgcolor=',''))) / 8)::text
     || ' | akfooter=' || (position('goauthentik' in (context #>> '{body}')) > 0)::text
FROM authentik_events_event WHERE action='email_sent' ORDER BY created DESC LIMIT 5;
```

## Rollback

Delete the files from the host (or drop the Dockerfile `COPY`) and Authentik falls
back to its stock templates immediately — no restart, no data change:

```bash
ssh root@<identity-host> 'rm -rf /data/authentik/templates/email'
```

## Brand assets

The logo mark and flow background are served from `/media/custom/`, baked into the
server image from `custom-assets/` by `Dockerfile.server`. The email references them
by absolute URL, so `auth.bawes.net` must stay reachable for images to load.
