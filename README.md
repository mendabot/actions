# @mendabot/actions

[![CI](https://img.shields.io/github/actions/workflow/status/mendabot/actions/build.yml?branch=main&logo=github&label=CI)](https://github.com/mendabot/actions/actions/workflows/build.yml)
[![license](https://img.shields.io/github/license/mendabot/actions)](./LICENSE)

Reusable GitHub Actions and workflows used across Mendabot projects.

```
setup/                      composite action — pnpm + Node + install
.github/workflows/release.yml   reusable workflow — semantic-release
```

## Pinning

Reference everything here by **commit SHA**, not by branch or tag. A tag is a movable
pointer, so `@main` or `@v1` lets whoever controls this repository change what runs inside
your job after you have reviewed it. A SHA cannot be repointed.

Keep the version in a trailing comment — Dependabot reads it, and updates both the hash and
the comment together.

```yaml
uses: mendabot/actions/setup@<sha> # v1.2.3
```

## `setup`

Installs pnpm, installs Node with the pnpm store cached, and runs `pnpm install --frozen-lockfile`.

```yaml
- name: Setup
  uses: mendabot/actions/setup@<sha>
```

| Input     | Default | Description                           |
| --------- | ------- | ------------------------------------- |
| `install` | `true`  | Set to `false` to skip `pnpm install` |

Node is pinned to 24 because it bundles npm 11.x. Trusted publishing needs npm >= 11.5.1 and
staged publishing needs >= 11.15.0, so lowering it silently breaks publishing rather than failing
at setup.

The registry is pinned to npmjs.org under the `@mendabot` scope. Setting `registry-url` at all is
what makes setup-node write an `.npmrc` — drop it and `NODE_AUTH_TOKEN` is ignored, so publishing
fails on authentication even when the secret is set correctly.

## `release`

Runs semantic-release with a signed release commit.

```yaml
name: Release new version

on:
  workflow_dispatch:

jobs:
  release:
    uses: mendabot/actions/.github/workflows/release.yml@<sha>
    secrets:
      signing-key: ${{ secrets.MENDABOT_SIGNING_KEY }}
      git-token: ${{ secrets.MENDABOT_WRITE_ACCESS_GIT_TOKEN }}
```

Takes no inputs. Node 24, the `Mendabot` git identity, `pnpm run build` and `pnpm run release`
are all fixed — every caller is a Mendabot repo, so the calling repo must define `build` and
`release` scripts.

| Secret        | Description                                                     |
| ------------- | --------------------------------------------------------------- |
| `signing-key` | Passphrase-less SSH private key used to sign the release commit  |
| `git-token`   | Token with write access — see the caveat below                   |

### `git-token` must not be the default `GITHUB_TOKEN`

GitHub suppresses workflow runs for events raised by the built-in `GITHUB_TOKEN`. If you pass it
here, the GitHub Release is still created but nothing downstream fires — a publish workflow
listening on `release: published` will never run, with no error to explain why. Use a PAT or a
GitHub App token.

### Signing

The key is written to `~/.ssh/signing_key` and wired up with `gpg.format ssh`, so commits are
signed without gpg-agent or a passphrase prompt. Generate one with:

```sh
ssh-keygen -t ed25519 -C "mendabot@pm.me" -f ./mendabot-signing -N ""
```

The private half becomes the `signing-key` secret. The public half goes on the GitHub account under
SSH and GPG keys as a **Signing Key** — not an Authentication Key. For the Verified badge, that
account also needs the committer email confirmed on it.

Only commits are signed. `tag.gpgsign` is deliberately not set: semantic-release creates its tag
with a bare `git tag <name>`, and signing promotes that to an annotated tag, which demands a
message and fails the release.

## Notes

`setup` puts pnpm before Node because setup-node's `cache: pnpm` locates the store by running
`pnpm store path`. Reversed, it fails with `Unable to locate executable file: pnpm`.

The release workflow inlines those same steps instead of calling `./setup`. Inside a reusable
workflow, a relative action path resolves against the *caller's* checkout rather than this
repository, so `./setup` would look for a directory the calling repo does not have.

## License

[MIT](./LICENSE)
