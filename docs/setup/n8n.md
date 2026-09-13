# n8n Setup

n8n is the **runtime** for this project: it receives the WhatsApp webhook,
runs the agent, and talks to Google Sheets. It runs in Docker on a VPS
behind Traefik, which terminates TLS and renews certificates on its own.

The compose file is [`infra/docker-compose.yml`](../../infra/docker-compose.yml).

## What runs on the box

| Service | Exposed | Purpose |
|---|---|---|
| `traefik` | `:80`, `:443` | TLS termination, Let's Encrypt, routes to n8n |
| `n8n` | `127.0.0.1:5678` | Workflow runtime |
| `qdrant` | nothing | Vector store, Phase 4, commented out |

n8n publishes to loopback only. The internet reaches it through Traefik or not at all.

## Why Traefik

Traefik reads its routing from container labels and has an ACME client
built in, so the certificate is requested, stored, and renewed by the same
process that serves it. The alternative — Nginx plus Certbot — needs a
renewal timer and a reload hook wired up separately.

## Before you start

- A VPS with Docker and the Compose plugin, and root or sudo access.
- A public hostname pointing at it — your own domain, or the free one
  your provider assigns the server. See below.
- A plain Docker host. If your provider offers a one-click n8n image,
  skip it — this compose file is the whole stack and expects to own
  ports 80 and 443.

### Do I need my own domain?

No — but you do need a *hostname*. Let's Encrypt won't issue certificates
for bare IP addresses and Meta requires an HTTPS webhook, so serving on
`https://<your-vps-ip>` isn't an option.

| Option | Cost | Trade-off |
|---|---|---|
| Your own domain | Registration fee | Survives a server rebuild |
| Provider-assigned hostname | Free | Tied to the server ID |

The provider hostname works the same way — Traefik issues and renews a
certificate for it normally. The catch is that it encodes the server ID,
so rebuilding or migrating the VPS changes the name, and `DOMAIN_NAME`,
the certificate, and the webhook URL registered with Meta all have to move
with it at once.

Self-hosted n8n does not supply a hostname of its own. (n8n Cloud does,
but that's the paid hosted product — not this setup.)

Using the provider hostname? You can skip step 1, since it already
resolves — but still run its check against the full
`n8n.<provider-hostname>`, because not every provider resolves subdomains
of it.

## 1. Point DNS at the VPS first

Do this before starting the stack. Traefik requests a certificate as
soon as a request arrives for the hostname, and Let's Encrypt allows only
5 failed validations per hostname per hour. Starting early, against a
name that doesn't resolve yet, burns that budget.

Create an `A` record:

| Host | Type | Points to |
|---|---|---|
| `n8n` | `A` | `<your-vps-ip>` |

If the domain already has a `*` wildcard record, leave it alone — a
specific record wins over a wildcard. Resolvers that already cached the
wildcard answer will keep serving it until its TTL expires, so the new
name can look wrong from some networks for a while.

**Check** — ask the domain's own nameservers, which bypass caching:

```bash
dig +short NS <your-domain>
dig +short @<one-of-those-nameservers> n8n.<your-domain> A
```

That must return your VPS IP before you continue.

## 2. Configure the environment

Copy `.env.example` to `.env` next to the compose file and set:

| Variable | Meaning |
|---|---|
| `DOMAIN_NAME` | Your domain, e.g. `example.com` |
| `SUBDOMAIN` | `n8n` |
| `SSL_EMAIL` | Where Let's Encrypt sends expiry warnings |
| `GENERIC_TIMEZONE` | Timezone for schedule nodes, e.g. `America/Sao_Paulo` |

`N8N_HOST` and `WEBHOOK_URL` are derived from `SUBDOMAIN` and
`DOMAIN_NAME` inside the compose file — don't set them separately.

Use an address you'll still control after a server rebuild. Tying it to a
provider-generated hostname means losing the warnings along with the box.

## 3. Start the stack

```bash
docker compose up -d
docker compose ps
```

**Check** — `n8n` reaches `healthy` within about a minute.

## 4. Verify TLS

```bash
curl -sSI https://n8n.<your-domain>/ | head -1
curl -sS -o /dev/null -w "%{redirect_url}\n" http://n8n.<your-domain>/
```

**Check** — the first returns `HTTP/2 200`, the second redirects to
`https://`. A certificate error here usually means DNS, not TLS: confirm
step 1 still resolves correctly from the machine you're testing from.

## 5. Claim the owner account

n8n has **no basic-auth setting** — it was removed in n8n 1.x. Access is
an owner account, and **the first visitor creates it**. Open
`https://n8n.<your-domain>` and set it up immediately, before the host is
left running unattended.

## 6. Close the firewall

```bash
ufw allow 22/tcp && ufw allow 80/tcp && ufw allow 443/tcp
ufw --force enable
```

Do this **last**, and keep `22/tcp` in the list — enabling UFW without it
will end your SSH session.

Note that UFW does **not** filter ports published by Docker. UFW's rules
sit in the `INPUT` chain, but a published container port is
destination-NAT'd and then filtered in `FORWARD`, through Docker's own
chain — so it never reaches UFW's rules at all. A container published to
`0.0.0.0` stays reachable whatever UFW says. That is why n8n is bound to
`127.0.0.1` in the compose file rather than left to the firewall.

## Maintenance

Run these from the directory holding the compose file.

```bash
docker compose ps                    # health
docker compose logs -f n8n           # follow logs
docker compose logs --tail=50 traefik
docker compose restart n8n
```

**Certificates renew themselves.** Traefik renews about 30 days before
expiry and stores the result in its volume — nothing to schedule. To check:

```bash
echo | openssl s_client -connect n8n.<your-domain>:443 \
  -servername n8n.<your-domain> 2>/dev/null | openssl x509 -noout -dates
```

**Updating.** Both images are pinned, so updating is a deliberate edit:
back up first, bump the tag, then `docker compose pull && docker compose up -d`.
Read n8n's release notes before crossing a major version.

**Backing up.** Workflows, credentials, the encryption key, and execution
history all live in one volume:

```bash
docker run --rm -v <project>_n8n_data:/d -v "$PWD":/backup \
  alpine tar czf /backup/n8n-$(date +%F).tar.gz -C /d .
```

Restore into a stopped stack:

```bash
docker compose down
docker run --rm -v <project>_n8n_data:/d -v "$PWD":/backup \
  alpine sh -c "rm -rf /d/* && tar xzf /backup/n8n-<date>.tar.gz -C /d"
docker compose up -d
```

`<project>` is the Compose project name — the directory name by default.
Confirm with `docker volume ls`.

That archive holds the credential encryption key, so treat it as a secret:
it decrypts everything else in the file. Note that execution history is
pruned after 7 days by `EXECUTIONS_DATA_MAX_AGE`, so backups are not a
substitute for it.

## Phase 4: Qdrant

Uncomment the `qdrant` service and the `qdrant_data` volume, then
`docker compose up -d`. It publishes no ports; n8n reaches it at
`http://qdrant:6333` over the Compose network. See
[`docs/setup/qdrant.md`](qdrant.md).

## Final check

```bash
curl -sS https://n8n.<your-domain>/healthz
```

A `200` with the right certificate, and an owner account you can log into,
means this step is done. Next:
[WhatsApp Business Cloud API](whatsapp.md).
