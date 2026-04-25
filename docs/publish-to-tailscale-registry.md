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
2. The Docker daemon on the runner is told to treat the registry hostname as
   an "insecure registry" — fine, because WireGuard handles transport security
3. `docker login` and `docker push` target the registry by its tailnet
   hostname, never touching the public internet
4. When the job ends the node is automatically removed from the tailnet

## Prerequisites — one-time setup per tailnet

These three steps are done outside the repo. Once done, any number of repos
on the same tailnet can use the same OAuth client.

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

## Per-repo workflow

Drop the `publish` job from
`github/workflows/publish-to-tailscale-registry.yml` into your existing CI
workflow file, or copy it as a standalone workflow. You will need to change
two things:

1. The registry hostname (e.g. `nas:5000`) — must match a node on your
   tailnet, not a public DNS name
2. The image name (e.g. `your-image-name:latest`)

The job is gated to only run on `push` to `main` and to depend on the
existing lint/typecheck/test jobs. Adjust both to match the workflow you're
dropping it into.

## Why plain HTTP is safe here

Standing instinct says "no plain HTTP for a registry". That instinct is
right when the traffic crosses the public internet, but the threat model
here is different:

- Traffic between the GitHub Actions runner and the registry travels over a
  WireGuard tunnel — encrypted and authenticated end-to-end by Tailscale
- The registry is bound to a tailnet IP, not a public one — there is no
  public path to it at all
- Pushing through TLS would add an extra layer of encryption on top of an
  already-encrypted tunnel, plus the operational cost of provisioning,
  rotating, and trusting a certificate the runner has never seen

The Docker daemon will refuse a plain-HTTP push by default; the
`insecure-registries` setting tells it to allow this for your specific
hostname only. It does not weaken security globally.

## Gotchas

### `oauth-secret`, not `oauth-client-secret`

The `tailscale/github-action` input is named `oauth-secret`. If you write
`oauth-client-secret` (a natural mistake — that's what you'd guess from
"OAuth client secret"), GitHub Actions only emits a warning, not an error.
The action then receives the client ID with no secret and bails out with:

> Please provide either an auth key, OAuth secret and tags, or federated
> identity client ID and audience with tags.

The valid input names per the action's runtime warning are:

```
['authkey', 'oauth-client-id', 'audience', 'oauth-secret', 'tags',
 'version', 'args', 'tailscaled-args', 'hostname', 'timeout',
 'retry', 'use-cache', 'statedir', 'sha256sum', 'ping']
```

### Overwriting `/etc/docker/daemon.json` is fine on `ubuntu-latest`

The workflow writes a fresh `daemon.json` rather than merging with any
existing one. This is safe on GitHub-hosted `ubuntu-latest` runners because
they are ephemeral and don't ship a meaningful `/etc/docker/daemon.json`.
If you adopt this on a self-hosted runner that already has daemon settings,
switch to merging the JSON instead.

### `docker daemon` restart is required

Changing `insecure-registries` only takes effect after `systemctl restart
docker`. The workflow does this; don't skip it.

### Eventual consistency on tailnet enrolment

New tailnet nodes take a few seconds to propagate to peers. If the very
first step after Tailscale comes up is a push, you may occasionally see
connection failures. The action's `ping:` input can be used to wait for
specific peers before continuing — see the
[`tailscale/github-action` README](https://github.com/tailscale/github-action#eventual-consistency).
In practice the gap between Tailscale starting and `docker push` running
is large enough that this hasn't been a problem.

## References

- [`tailscale/github-action`](https://github.com/tailscale/github-action) —
  authoritative on inputs, scopes, and federation
- [Tailscale: GitHub Actions integration](https://tailscale.com/docs/integrations/github/github-action)
- [Tailscale: Trust credentials and scopes](https://tailscale.com/kb/1623/trust-credentials)
- [Tailscale: Tags](https://tailscale.com/kb/1068/tags)
