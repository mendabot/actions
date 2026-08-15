# @mendabot/actions

[![CI](https://img.shields.io/github/actions/workflow/status/mendabot/actions/build.yml?branch=main&logo=github&label=CI)](https://github.com/mendabot/actions/actions/workflows/build.yml)
[![license](https://img.shields.io/github/license/mendabot/actions)](./LICENSE)

Reusable GitHub Actions and workflows used across Mendabot projects.

```
setup/                         composite action — pnpm + Node + install
.github/workflows/release.yml  reusable workflow — semantic-release
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

## License

[MIT](./LICENSE)
