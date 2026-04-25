# Push container images to a private registry over Tailscale

A pattern for `docker push` from GitHub Actions to a self-hosted registry on
your tailnet, without exposing the registry to the public internet.

## Why

The obvious approach is to expose the registry through a tunnel (Cloudflare,
ngrok, etc.) and have GitHub Actions push to a public hostname. This works
until it doesn't:

- Cloudflare's Bot Fight Mode reliably blocks pushes from GitHub Actions IP
  ranges (they look like bots, because they are)
- Any public exposure widens the attack surface — even with auth, you're
  serving an HTTP endpoint to the internet
- Cloudflare's free plan caps request body size, which container image layers
  routinely exceed

A direct path over Tailscale avoids all of this: the runner joins your
tailnet for the duration of the job, pushes to the registry by its tailnet
hostname, then drops off when the job ends.

## How it works

1. `tailscale/github-action` uses an OAuth client to mint a one-time auth key
   and bring the runner up as an ephemeral, tagged node on your tailnet
2. `docker login` and `docker push` target the registry by its tailnet
   hostname — never touching the public internet
3. When the job ends the node is automatically removed from the tailnet

Step 2 has two valid shapes — see below.

## Two valid configurations: HTTP vs HTTPS

Both variants push the image over a WireGuard tunnel between the runner and
the registry. **Confidentiality and "no public exposure" are identical** —
WireGuard already encrypts and authenticates every byte. The choice is
operational:

| Variant   | Workflow file                                       | Server-side work                                  |
| --------- | --------------------------------------------------- | ------------------------------------------------- |
| **HTTP**  | `publish-to-tailscale-registry-http.yml`            | None — bind the registry to the tailnet IP        |
| **HTTPS** | `publish-to-tailscale-registry-https.yml`           | Provision a Tailscale-issued cert and renew it    |

### Pick HTTP when

- You're prototyping or rolling out to a single repo
- The registry container has no clean way to mount certs and renew them
- You're comfortable with the workflow having to set `insecure-registries`
  and restart the runner's Docker daemon (and explaining why that's safe)

### Pick HTTPS when

- You're adopting the pattern across several repos and want each workflow
  to read like a normal `docker push` — no daemon hacks, fewer steps
- You want endpoint authentication as defence in depth on top of WireGuard
- You're already running `tailscale serve` or a reverse proxy that can hold
  a cert

The HTTPS variant has one extra prerequisite (a cert on the registry host),
in exchange for a noticeably cleaner workflow.

## Prerequisites — one-time setup per tailnet

These three steps are done outside the repo and apply to both variants. Once
done, any number of repos on the same tailnet can use the same OAuth client.

### 1. Define `tag:ci` in your Tailscale ACL

In the Tailscale admin console → **Access controls → Tags**, create a new
tag named `tag:ci`. Leave the **Tag owner** field empty — only OAuth clients
holding the tag should be able to apply it, no human owner is needed.

If you prefer the JSON ACL editor, add:

```json
"tagOwners": {
  "tag:ci": []
}
```

The empty list (`[]`) is intentional and supported.

### 2. Create the OAuth client

In **Settings → OAuth clients** → "Generate OAuth client":

- **Description**: something like `<repo-name> CI`
- **Scopes**: tick `Keys → Auth Keys → Write`
- **Tags**: `tag:ci`

> The official `tailscale/github-action` README is the authority here:
> "OAuth clients used for this purpose must have the writable `auth_keys`
> scope." Older Tailscale docs sometimes refer to a `devices` scope — that
> name has been retired in the new admin console UI; what you want is
> `Auth Keys → Write` under the **Keys** section.

The admin console will show the client ID and secret once. Copy both before
closing the dialog — the secret is not retrievable afterwards.

### 3. Add the GitHub Actions secrets

In each repo that needs to push, add (Settings → Secrets and variables →
Actions):

| Secret                          | Value                                |
| ------------------------------- | ------------------------------------ |
| `TAILSCALE_OAUTH_CLIENT_ID`     | Client ID from step 2                |
| `TAILSCALE_OAUTH_CLIENT_SECRET` | Client secret from step 2            |
| `REGISTRY_USERNAME`             | Auth username for your registry      |
| `REGISTRY_PASSWORD`             | Auth password for your registry      |

The `REGISTRY_*` secrets are whatever your registry already uses (e.g. an
htpasswd entry on a self-hosted Docker registry). They are unrelated to
Tailscale.

## HTTPS-only setup — provisioning the cert

Skip this section if you're using the HTTP variant.

### Enable HTTPS in the tailnet

In the Tailscale admin console → **DNS** → **HTTPS Certificates**, enable
"HTTPS Certificates". This is tailnet-wide, free, and a one-click toggle.

### Issue and renew the cert

Tailscale issues 90-day Let's Encrypt certs for your `<host>.<tailnet>.ts.net`
hostnames. You need to issue the cert *and* automate renewal. Two viable
approaches:

**Option A — `tailscale serve` (simplest).** If your registry binds to a
local port (e.g. `127.0.0.1:5000`), point Tailscale at it:

```bash
tailscale serve --bg --https=443 http://127.0.0.1:5000
```

Tailscale terminates TLS on port 443 of the tailnet IP, manages the cert
end-to-end (issuance + renewal), and proxies to the registry. No reverse
proxy, no cron, no manual cert handling. The workflow then pushes to
`<host>.<tailnet>.ts.net` (no port).

**Option B — Reverse proxy with `tailscale cert`.** If you already run Caddy,
Traefik, or similar, fetch the cert manually and let the proxy serve it:

```bash
tailscale cert <host>.<tailnet>.ts.net
```

Mount the resulting `.crt` and `.key` into the proxy container, point it at
the registry, and set up a cron/systemd timer to re-run `tailscale cert`
plus reload the proxy every ~60 days. Renewal is the operational gotcha —
a 90-day cert that silently expires will turn the workflow red overnight.

Option A is the right default unless you have a reason to want a separate
proxy in front of the registry.

## Per-repo workflow

Drop the `publish` job from the appropriate workflow file into your existing
CI workflow, or copy the file as a standalone workflow.

| You picked | Copy this file                                      |
| ---------- | --------------------------------------------------- |
| HTTP       | `github/workflows/publish-to-tailscale-registry-http.yml`  |
| HTTPS      | `github/workflows/publish-to-tailscale-registry-https.yml` |

You will need to change two things in either file:

1. The registry hostname — must resolve to a node on your tailnet, not a
   public DNS name. HTTP variant: `nas:5000` style. HTTPS variant:
   `nas.<tailnet>.ts.net` (or `:5000` if you didn't use `tailscale serve`'s
   default port 443).
2. The image name (e.g. `your-image-name:latest`).

The job is gated to only run on `push` to `main` and to depend on existing
lint/typecheck/test jobs. Adjust both to match the workflow you're dropping
it into.

## Why plain HTTP is also safe (HTTP variant)

Standing instinct says "no plain HTTP for a registry". That instinct is
right when the traffic crosses the public internet, but the threat model
here is different:

- Traffic between the GitHub Actions runner and the registry travels over a
  WireGuard tunnel — encrypted and authenticated end-to-end by Tailscale
- The registry is bound to a tailnet IP, not a public one — there is no
  public path to it at all
- TLS would add a second layer of encryption on top of an already-encrypted
  tunnel, plus the operational cost of provisioning, rotating, and trusting
  a certificate

The Docker daemon will refuse a plain-HTTP push by default; the
`insecure-registries` setting tells it to allow this for your specific
hostname only. It does not weaken security globally.

The HTTPS variant exists because that operational cost is worth paying once
you adopt the pattern across multiple repos — workflows get shorter, the
explanation tax goes away, and you gain endpoint authentication as defence
in depth. It is *not* required for confidentiality.

## Gotchas

### `oauth-secret`, not `oauth-client-secret`

Applies to both variants. The `tailscale/github-action` input is named
`oauth-secret`. If you write `oauth-client-secret` (a natural mistake —
that's what you'd guess from "OAuth client secret"), GitHub Actions only
emits a warning, not an error. The action then receives the client ID with
no secret and bails out with:

> Please provide either an auth key, OAuth secret and tags, or federated
> identity client ID and audience with tags.

The valid input names per the action's runtime warning are:

```
['authkey', 'oauth-client-id', 'audience', 'oauth-secret', 'tags',
 'version', 'args', 'tailscaled-args', 'hostname', 'timeout',
 'retry', 'use-cache', 'statedir', 'sha256sum', 'ping']
```

### Overwriting `/etc/docker/daemon.json` is fine on `ubuntu-latest`

HTTP variant only. The workflow writes a fresh `daemon.json` rather than
merging with any existing one. This is safe on GitHub-hosted `ubuntu-latest`
runners because they are ephemeral and don't ship a meaningful
`/etc/docker/daemon.json`. If you adopt this on a self-hosted runner that
already has daemon settings, switch to merging the JSON instead.

### `docker daemon` restart is required

HTTP variant only. Changing `insecure-registries` only takes effect after
`systemctl restart docker`. The workflow does this; don't skip it.

### Cert renewal is the silent killer

HTTPS variant only. Tailscale-issued certs are 90-day Let's Encrypt certs.
If you go with Option B (manual `tailscale cert` + reverse proxy), the
renewal cron is what keeps the workflow working. Set it up the same day you
issue the cert, not "later". `tailscale serve` (Option A) handles renewal
for you and is the recommended default for this reason.

### Eventual consistency on tailnet enrolment

Applies to both variants. New tailnet nodes take a few seconds to propagate
to peers. If the very first step after Tailscale comes up is a push, you
may occasionally see connection failures. The action's `ping:` input can be
used to wait for specific peers before continuing — see the
[`tailscale/github-action` README](https://github.com/tailscale/github-action#eventual-consistency).
In practice the gap between Tailscale starting and `docker push` running
is large enough that this hasn't been a problem.

## References

- [`tailscale/github-action`](https://github.com/tailscale/github-action) —
  authoritative on inputs, scopes, and federation
- [Tailscale: GitHub Actions integration](https://tailscale.com/docs/integrations/github/github-action)
- [Tailscale: Trust credentials and scopes](https://tailscale.com/kb/1623/trust-credentials)
- [Tailscale: Tags](https://tailscale.com/kb/1068/tags)
- [Tailscale: HTTPS for tailnet services](https://tailscale.com/kb/1153/enabling-https)
- [Tailscale: `tailscale serve`](https://tailscale.com/kb/1242/tailscale-serve)
