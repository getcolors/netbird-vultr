# netbird-vultr

Desired state for a self-hosted [NetBird](https://netbird.io) control plane
with [Authentik](https://goauthentik.io) as its identity provider, on Vultr.

- **https://netbird.bigconfig.online** — dashboard, REST API, management and
  signal gRPC, relay WebSocket, embedded IdP
- **https://authentik.bigconfig.online** — Authentik, for SSO and MFA

Built by the [`netbird`](https://github.com/getcolors/netbird) Package Skill:
OpenTofu manages the instance, its firewall and two unproxied Cloudflare `A`
records; Ansible converges Traefik, the combined `netbird-server`, the
dashboard, and Authentik with its Postgres and Redis.

## Use

```sh
direnv allow               # once, after cloning
./green build              # render .colors/netbird-vultr/
./green create --dry-run   # walk the workflow, no side effects
./green create             # converge
```

## Signing in

Sign in at https://netbird.bigconfig.online/ with the **Authentik** option, as
`claude@ululi.it` — Authentik's `akadmin`, with
`COLORS_PAR_NETBIRD_AUTHENTIK_BOOTSTRAP_PASSWORD`. That account owns the
deployment, and convergence creates it for you by performing that sign-in
itself; there is no manual step.

**Two accounts exist, and only one is the network.** `POST /api/setup` creates
`breakglass@bigconfig.online` in a *local* account, because registering an
identity provider needs an authenticated caller and that is the only way to get
the first one. A user arriving through Authentik gets a *separate* account, and
NetBird provides no way to merge them.

So the local account is **not** a way back into the network. If Authentik is
unavailable, recover by restoring from backup (`netbird-restore`), or by
registering a replacement identity provider with the local credential at
`/etc/netbird/secrets/local_pat`. Authentik's admin interface is at
https://authentik.bigconfig.online/if/admin/.

## Adding peers

```sh
netbird up --management-url https://netbird.bigconfig.online
```

The CLI uses the device-code grant against Authentik. Setup keys for unattended
peers come from the dashboard under **Setup Keys**.

## Operating

Over SSH (`ssh netbird-vultr`, an alias this deployment writes into
`~/.ssh/config`):

```sh
netbird-status              # containers, certificates, backups, ownership
netbird-backup              # take one now
netbird-restore --verify    # restore into throwaway containers and check it
```

Backups run nightly at 02:30 UTC, are encrypted with
`COLORS_PAR_NETBIRD_BACKUP_RECOVERY_KEY` before upload, and land under an
immutable timestamped key in the `netbird-backup` R2 bucket with a
`latest-known-good` pointer. Seven days of retention; pruning never removes the
newest archive.

**Keep the recovery key somewhere other than this host and this checkout.** It
is the only thing that decrypts those archives, and it is operator-supplied
precisely so that losing the server does not lose the backups.

## Deleting

```sh
COLORS_PAR_COMPUTE_PREVENT_DESTROY=false ./green delete
```

Takes a final backup first. Never edit `compute-prevent-destroy` in
`colors.yml`.

## Credentials

Nine `COLORS_PAR_*` variables in the gitignored `.envrc.private`. See the
comments in that file and
[the configuration reference](https://github.com/getcolors/netbird/blob/main/skills/package-netbird-green/references/configuration.md).
Never export `COLORS_PAR_PROFILE`.
