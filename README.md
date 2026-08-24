# @the-heimdall/sample-js-2

A small package used to prove out a release pipeline: **pushing a git tag publishes to
npm automatically**, with no credential stored anywhere.

## Usage

```js
import { greet } from '@the-heimdall/sample-js-2';

greet();        // "Hello, world!"
greet('Sai');   // "Hello, Sai!"
```

## Releasing

Nothing is published from a laptop. A release happens by pushing a version tag:

```bash
npm version 1.0.0-rc1 -m "Release v%s"
git push origin main --follow-tags
```

`npm version` bumps `package.json`, commits it, and creates a matching tag. The tag
push triggers `.github/workflows/release.yml`, which verifies the tag matches
`package.json`, then lints, builds, and publishes.

### Version channels

| Version | npm dist-tag | Who gets it |
|---|---|---|
| `1.0.0-rc1-beta.1` | `beta` | Only `npm install @the-heimdall/sample-js-2@beta` |
| `1.0.0-rc1` | `latest` | Everyone |
| `1.0.0` | `latest` | Everyone |

A version containing `-beta.` publishes under the `beta` dist-tag, so it is fully
installable but never becomes the default install. Note the rule is specifically
`-beta.` and not "contains a hyphen" — `1.0.0-rc1` has a hyphen but is a real release.

## The two workflows

| File | Fires when | What it does | Publishes? |
|---|---|---|---|
| `.github/workflows/ci.yml` | Every pull request | install → lint → build | No |
| `.github/workflows/release.yml` | Push of a `v*` tag | verify tag → install → lint → build → publish | Yes |

## How authentication works

There is **no npm token** — not in this repo, not in GitHub secrets, not on any laptop.

Publishing uses **npm Trusted Publishing (OIDC)**. GitHub mints a short-lived identity
token scoped to this exact repository and workflow, and npm verifies it against the
trusted publisher registered on the package. Nothing long-lived exists to leak, and
published versions carry build provenance automatically.

> **Do not commit an `.npmrc` with an `_authToken` line for registry.npmjs.org.**
> A project-level `.npmrc` overrides your user-level `~/.npmrc`, which silently breaks
> `npm login` for everyone working in the repo. The resulting failure reports as
> `404 Not Found` rather than `401`, because npm hides the existence of scoped packages
> you can't prove access to.
>
> Committing scope lines for a *private* registry is fine and normal — it is
> specifically an auth token for your publish target that causes the conflict.

## Working locally

```bash
npm install
npm run lint
npm run build
npm pack --dry-run   # shows exactly what would be published
```

## One-time setup

- The package must exist on npm before a trusted publisher can be attached to it, so
  the very first version is published by hand. Every release after that is automated.
- Register the publisher with:
  ```bash
  npm trust github @the-heimdall/sample-js-2 \
    --file release.yml \
    --repo the-heimdall/sample-js-2 \
    --allow-publish
  ```
- Check it with `npm trust list @the-heimdall/sample-js-2`.
