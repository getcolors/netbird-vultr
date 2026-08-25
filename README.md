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

Two accounts exist, deliberately.

**`claude@ululi.it` through Authentik** is the owner. On a first converge it
does not exist yet — NetBird only imports an external user after it has
authenticated once — so convergence reports a pending manual step:

1. Open https://netbird.bigconfig.online/ and choose **Authentik**.
2. Sign in as `claude@ululi.it`.
3. Run `./green create` again.

The second run approves that user, promotes it to owner and asserts the role.

**`breakglass@bigconfig.online`** is the break-glass administrator on NetBird's
embedded IdP, with `COLORS_PAR_NETBIRD_BOOTSTRAP_PASSWORD`. It is kept on
purpose: it is the only way in when Authentik is unavailable, which is exactly
when an Authentik-only account is no use. After ownership transfers it is an
administrator rather than the owner — promotion transfers ownership rather than
adding a second owner.

Authentik's own admin is `akadmin` with
`COLORS_PAR_NETBIRD_AUTHENTIK_BOOTSTRAP_PASSWORD`, at
https://authentik.bigconfig.online/if/admin/. Its bootstrap token is revoked
once the blueprint has applied.

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
