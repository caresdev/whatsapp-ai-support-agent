# Common issues & fixes

Traps that are easy to hit while following the setup guides, written as symptom first.

## DNS & TLS

### Browser shows a certificate error, but `curl` from the server is fine

The browser and your shell are resolving the name to different IPs. Adding
a subdomain under an existing `*` wildcard is the usual cause: resolvers
that cached the wildcard answer keep serving it until its TTL expires.
Browsers make it worse by bypassing the system resolver — Chrome and
Firefox default to DNS-over-HTTPS.

```bash
dig +short n8n.<your-domain>              # system resolver
dig +short @1.1.1.1 n8n.<your-domain>     # what the browser likely uses
```

Different answers confirm it. Either wait out the TTL (second column of
`dig @1.1.1.1 n8n.<your-domain> A`), or turn off secure DNS in the browser
so it uses the system resolver.

### Chrome says "Dangerous site" after DNS is already correct

A stale Google Safe Browsing verdict. Wildcards often point at registrar
parking pages, which get flagged; the verdict survives the DNS fix and
clears only when Google recrawls.

Confirm the server is fine before doing anything else:

```bash
echo | openssl s_client -connect n8n.<your-domain>:443 \
  -servername n8n.<your-domain> 2>&1 | grep "Verify return code"
```

`0 (ok)` plus a certificate matching your hostname means the connection is
authentic and the verdict is out of date. To clear it, request a review in
Google Search Console (Security Issues), which needs the domain verified
by DNS TXT. Another browser works meanwhile — blocklist caches are
per-browser. Don't disable Safe Browsing globally.

### Let's Encrypt: "too many failed authorizations"

The stack started before DNS resolved. Traefik retried, and Let's Encrypt
allows only **5 failed validations per hostname per hour**. `acme.json`
stays empty.

The counter can't be cleared — wait out the hour, then verify DNS against
the domain's own nameservers before restarting (step 1 of
[`setup/n8n.md`](setup/n8n.md)). For repeated testing use
[Let's Encrypt staging](https://letsencrypt.org/docs/staging-environment/),
which has far higher limits.

## Docker & Compose

### n8n asks me to create an owner account, but I already had one

n8n is pointed at a different volume. Compose prefixes volume names with
the project (directory) name, so `n8n_data` becomes `<project>_n8n_data`.
An install that declared volumes `external: true` used the bare name.
Swapping one compose file for the other silently starts n8n empty, and the
warning is easy to miss:

```
volume "n8n_n8n_data" already exists but was not created by Docker Compose
```

**Don't claim the owner account** — that writes to the empty volume. Stop
the stack, check `docker volume ls` against
`docker compose config | grep -A2 "^volumes:"`, then copy the data across:

```bash
docker volume create <project>_n8n_data
docker run --rm -v n8n_data:/from -v <project>_n8n_data:/to \
  alpine sh -c "cd /from && cp -a . /to/"
```

The original volume is untouched, so this is reversible. Check that
`/home/node/.n8n/config` came across — it holds the encryption key,
without which saved credentials can't be decrypted.

## n8n

### My workflows are gone

You're probably on **n8n Cloud** (`app.n8n.cloud`), not your own instance.
The two UIs are nearly identical, and Cloud will take you through signup
onto a paid plan for an instance you never use.

Check the address bar — your workflows are at `https://n8n.<your-domain>`.
Treat an empty workflow list as a wrong-instance symptom before assuming
data loss.

## VPS access & firewall

### UFW is enabled but the port is still reachable

Docker writes its own iptables rules into the `DOCKER-USER` chain, which
is evaluated before UFW's. A container published to `0.0.0.0` is reachable
regardless of UFW.

Don't publish it: bind to loopback (`127.0.0.1:5678:5678`, as n8n does) or
publish nothing and let other containers reach it over the Compose
network. UFW protects host services, not containers.

### Pasting into the provider's web console mangles the command

Browser consoles truncate or wrap long pasted lines — an SSH public key is
enough to trigger it. The symptom is `syntax error near unexpected token
'newline'`, or a command running against only part of its arguments.

Fetch on the server instead of pasting:

```bash
curl -fsSL https://github.com/<your-github-username>.keys \
  >> /root/.ssh/authorized_keys
```

Then switch to a real SSH client. If a paste already failed, check whether
it partly succeeded before re-running it.
