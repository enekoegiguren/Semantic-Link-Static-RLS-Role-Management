# Email notifications — template

This project expects a notebook named `Email notifications`, `%run` before
`RLS_Management_Functions`, that provides email-sending via Microsoft
Graph. It's only needed if you pass `alert_to=` to `run_with_audit()` — if
you leave `alert_to=None`, alerts are printed to the notebook log instead
and this dependency isn't required at all.

**Do not commit this file with real tenant/client credentials to a public
repo.** Store `CLIENT_SECRET` in a Key Vault or Fabric secret store, not
inline in a notebook cell, even privately.

```python
# ── Required by RLS_Management_Functions.ipynb's send_alert() ─────────────
import requests

TENANT_ID     = "<your Entra tenant ID>"
CLIENT_ID     = "<your app registration client ID>"
CLIENT_SECRET = "<your app registration client secret — from Key Vault, not hardcoded>"
SENDER_EMAIL  = "<the mailbox that will appear as the sender>"

def get_graph_token():
    """
    Client-credentials flow against Microsoft Graph. Requires the app
    registration to have Mail.Send application permission, admin-consented.
    """
    url = f"https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token"
    resp = requests.post(url, data={
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET,
        "scope": "https://graph.microsoft.com/.default",
        "grant_type": "client_credentials",
    })
    resp.raise_for_status()
    return resp.json()["access_token"]


def send_email(subject, html_body, to, access_token):
    """
    to: a single address or a list of addresses.
    Called by RLS_Management_Functions.ipynb's send_alert() with a token
    already resolved via get_graph_token() — one token per run, reused
    across every alert in that run.
    """
    if isinstance(to, str):
        to = [to]

    url = f"https://graph.microsoft.com/v1.0/users/{SENDER_EMAIL}/sendMail"
    payload = {
        "message": {
            "subject": subject,
            "body": {"contentType": "HTML", "content": html_body},
            "toRecipients": [{"emailAddress": {"address": addr}} for addr in to],
        }
    }
    resp = requests.post(
        url,
        headers={"Authorization": f"Bearer {access_token}"},
        json=payload,
    )
    resp.raise_for_status()
```
