# CLAUDE.md

## Repository

`netbird-vultr` is desired state only: no source code. It configures the
[`netbird`](https://github.com/getcolors/netbird) Package Skill to run a
self-hosted NetBird control plane with Authentik on one Vultr instance, serving
`netbird.bigconfig.online` and `authentik.bigconfig.online`.

`colors.yml` is the only file to edit. Everything else is either generated
(`.colors/`), secret (`.envrc.private`), or a copy of the installed skill.

## The launcher is a copy, not a symlink

Root `./green` is a copy of `.agents/skills/package-netbird-green/green`.
`npx skills update -p` rewrites the payload and leaves the root file alone, so
the project keeps running the old pin while `skills-lock.json` claims the new
one. After any update:

```sh
npx skills update -p
cp .agents/skills/package-netbird-green/green green
```

## Two accounts, on purpose

`netbird-owner-email` (`claude@ululi.it`) is the Authentik-backed owner.
`netbird-bootstrap-email` (`breakglass@bigconfig.online`) is a break-glass
administrator on NetBird's embedded IdP. Validation refuses them being equal:
one address for both would make the recovery path depend on the identity
provider it exists to survive.

A first converge cannot create the owner — NetBird imports an external user
only after it has authenticated once — so it reports a pending manual step and
exits zero. Sign in once through Authentik, converge again, and the transfer
completes. Do not "fix" this by deleting the break-glass account, which is what
the upstream article does.

## Do not

- Read or print `.envrc.private`.
- Edit, read as source, or commit `.colors/`.
- Export `COLORS_PAR_PROFILE`; the package refuses to run when it is set.
- Weaken `compute-prevent-destroy` in committed desired state.
- Run a real `create` or `delete` without explicit authorization.
- Regenerate anything under `/etc/netbird/secrets` on the host. Those are
  create-once: a new store encryption key orphans the peer database and a new
  Authentik `SECRET_KEY` invalidates every session, both while the stack still
  looks healthy.

## Backups

Encrypted on the host with `COLORS_PAR_NETBIRD_BACKUP_RECOVERY_KEY` before
upload. That key is operator-supplied and exists nowhere else — if it is lost,
every archive is unrecoverable. It is deliberately not generated on the server,
because a key generated there would be lost with the server it protects.

## Git

Work on the current branch. Do not commit or push unless explicitly authorized.
The `.gitignore` is default-deny for dotfiles: check `git ls-files` rather than
inferring what is tracked from the working tree.
