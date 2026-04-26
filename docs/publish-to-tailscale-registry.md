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

## Versioning — CalVer tags

Both workflow templates push two tags on every build:

- `latest` — always points to the most recent build; used by running containers
- `YYYY.MM.DD-<sha7>` — e.g. `2026.04.26-a1b2c3d`; immutable, for auditability

The date is taken from the **commit timestamp** (not the run date) so the tag is
stable if the job is re-run days later:

```bash
commit_date="$(TZ=UTC git show -s --format=%cd --date=format-local:%Y.%m.%d "$GITHUB_SHA")"
echo "tag=${commit_date}-$(echo "$GITHUB_SHA" | cut -c1-7)" >> "$GITHUB_OUTPUT"
```

CalVer suits this pattern well. Images are built on every merge to `main`, not
on every feature completion — there is no semantic version to increment. The
date tells you roughly when it was built; the SHA tells you exactly what's in
it.

## Auto-redeploy with Portainer

If your containers are managed by a Portainer instance on the same tailnet, you
can trigger a pull-and-redeploy immediately after the push, giving you a full
CI → image → running container loop in a single workflow run.

### The 127.0.0.1:5000 pull trick

**Do not** use the public Cloudflare-fronted hostname for automated pulls.
Cloudflare's bot fight mode (and Access policies) block Docker daemon `pull`
requests, returning HTML instead of a registry manifest. The Docker daemon
reports this as:

```
error unmarshalling content: invalid character '<' looking for beginning of value
```

Instead, pull via `127.0.0.1:5000`. The registry container publishes port 5000
on all interfaces of the host, and `127.0.0.0/8` is already in the Docker
daemon's default `insecure-registries` CIDR — no `daemon.json` change or Docker
restart is needed. This works even though CI pushed using the Tailscale hostname;
it is the same registry backend.

Do **not** use `pullImage: true` in the Portainer stack update when Cloudflare
fronts your registry. Portainer passes that flag to `docker compose pull`, which
hits the Cloudflare-gated public URL and fails. Split it into two steps instead:

1. Explicit pull via Portainer's Docker API with credentials in `X-Registry-Auth`
2. Stack update with `pullImage: false` — image already on the host

### Required secrets

Add `PORTAINER_ACCESS_KEY` to the repo's Actions secrets alongside the existing
four. Generate a Portainer long-lived API key in **Account settings → Access
tokens**.

### Redeploy step

Add this after the `docker/build-push-action` step:

```yaml
- name: Redeploy on Portainer
  env:
    PORTAINER_ACCESS_KEY: ${{ secrets.PORTAINER_ACCESS_KEY }}
    REGISTRY_USERNAME: ${{ secrets.REGISTRY_USERNAME }}
    REGISTRY_PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
  run: |
    python3 << 'PYEOF'
    import base64, json, urllib.request, os

    PORTAINER = "http://nas:9000"   # Tailscale hostname of the Portainer host
    STACK_ID = 0                    # replace with your Portainer stack ID
    ENDPOINT_ID = 1                 # replace with your Portainer endpoint ID

    key = os.environ["PORTAINER_ACCESS_KEY"]
    headers = {"X-API-Key": key, "Content-Type": "application/json"}

    # Pull via 127.0.0.1:5000 — already in the daemon's insecure CIDR,
    # bypasses Cloudflare entirely.
    reg_auth = base64.b64encode(json.dumps({
        "username": os.environ["REGISTRY_USERNAME"],
        "password": os.environ["REGISTRY_PASSWORD"],
        "serveraddress": "127.0.0.1:5000",
    }).encode()).decode()
    pull_req = urllib.request.Request(
        f"{PORTAINER}/api/endpoints/{ENDPOINT_ID}/docker/images/create"
        "?fromImage=127.0.0.1:5000/your-image-name&tag=latest",
        data=b"",
        headers={"X-API-Key": key, "X-Registry-Auth": reg_auth},
        method="POST",
    )
    with urllib.request.urlopen(pull_req, timeout=120) as resp:
        resp.read()
    print("Image pulled via 127.0.0.1:5000")

    # Fetch env vars stored in Portainer (source of truth for secrets)
    req = urllib.request.Request(
        f"{PORTAINER}/api/stacks/{STACK_ID}",
        headers={"X-API-Key": key},
    )
    with urllib.request.urlopen(req, timeout=30) as resp:
        stack = json.load(resp)

    with open("docker-compose.yml") as f:
        compose = f.read()

    # Image is already on the host — no pull needed during stack update
    payload = {"stackFileContent": compose, "env": stack["Env"], "pullImage": False}
    req = urllib.request.Request(
        f"{PORTAINER}/api/stacks/{STACK_ID}?endpointId={ENDPOINT_ID}",
        data=json.dumps(payload).encode(),
        headers=headers,
        method="PUT",
    )
    with urllib.request.urlopen(req, timeout=120) as resp:
        result = json.load(resp)
        print(f"Redeployed stack (status={result['Status']})")
    PYEOF
```

The `docker-compose.yml` read in this step should be the canonical stack
definition committed to the repo. Portainer's stored env vars (fetched via the
GET) are passed back in the PUT so they are not lost; secrets live in Portainer,
not in the workflow.

Use `127.0.0.1:5000/your-image-name:latest` as the image reference in that
`docker-compose.yml` for the same reason — it's the address the Docker daemon
on the host can always reach without Cloudflare in the path.

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
