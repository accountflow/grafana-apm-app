# Accountflow fork of nais/grafana-apm-app

This is a temporary fork of [nais/grafana-apm-app](https://github.com/nais/grafana-apm-app)
(Apache-2.0). Accountflow self-hosts the plugin in Grafana 13 (installed by
`iac` from a pinned GitHub release zip). It exists for one reason: the
"Play replay" button crashes in the nais release, and the fix is a dependency
swap nais has not shipped.

## Why

The plugin depends on `@grafana/rrweb-player@2.0.0-grafana.2`, Grafana's fork of
rrweb-player and the only version on the `grafana` dist-tag. That published build
is broken: its Svelte `onMount` bodies are empty and it never calls
`new Replayer(...)` (39 bare `return;` stubs, although the shipped source map still
shows the real code). The player therefore never mounts its Controller, and
`ReplayPlayer.tsx` crashes on the first seek:

```
TypeError: Cannot read properties of undefined (reading 'goto') at ReplayPlayer.tsx:58
```

The Jest tests mock the player, so the stub build was never caught upstream.

Upstream `rrweb-player@2.1.4` (npm) is intact: three `onMount` bodies and one
`Replayer` construction. Its constructor options, props (`events`, `autoPlay`,
`showController`, `skipInactive`, `speedOption`, `width`, `height`) and handles
(`goto`, `pause`, `$destroy`) match what `src/pages/tabs/frontend/replay/ReplayPlayer.tsx`
uses. The rrweb `EventType`, `IncrementalSource` and `NodeType` enums are identical
between `@grafana/rrweb-types@2.0.0-grafana.2` (what the `@nais/apm` recorder emits)
and `@rrweb/types@2.1.4`.

Not yet verified: playback of a real recorded session on the deployed Grafana. The
header comment in `ReplayPlayer.tsx` still says player and recorder must come from the
same Grafana fork release; that comment is deliberately left untouched to keep the
patch minimal.

## The patch

One line in `package.json`, resolved through a pnpm alias so the imports
(`@grafana/rrweb-player`, `@grafana/rrweb-player/dist/style.css`), the Jest mock and
the TypeScript types stay as they are:

```diff
-    "@grafana/rrweb-player": "2.0.0-grafana.2",
+    "@grafana/rrweb-player": "npm:rrweb-player@2.1.4",
```

plus the resulting `pnpm-lock.yaml` refresh. Everything else on this branch is
fork plumbing, not plugin changes:

- `package.json` `version` is `0.24.0-accountflow.1` so the built `plugin.json`
  version distinguishes the fork build (iac keys its install marker on it).
- `.github/workflows/ci.yml` also runs on pushes and PRs to `accountflow`.
- `.github/workflows/release-fork.yml` builds and publishes the fork release (below).
- This file.

Nothing else in the plugin is changed. No other dependency bumps, no refactors.

## Branches

- `main` is a pristine mirror of `upstream/main`. Never commit to it, never
  force-push it.
- `accountflow` is the fork's default branch and carries our commits on top of
  `main`.

Workflows disabled on the fork: `release-please.yml` (would open Release PRs on our
`main` and mint tags colliding with upstream), `cp-update.yml` (needs a PAT secret
we do not have), `bundle-stats.yml` (noise). `ci.yml` and `is-compatible.yml`
stay on.

## Resync with upstream

```sh
git fetch upstream
git checkout accountflow
git rebase upstream/main          # replay our few commits on the new upstream
git push --force-with-lease origin accountflow
git checkout main && git merge --ff-only upstream/main && git push origin main
```

If nais changes the `@grafana/rrweb-player` line themselves, the rebase conflicts
on exactly that line, which is the signal to read "Exit criterion" below.

## Cut a release

Releases are unsigned (there is no `GRAFANA_ACCESS_POLICY_TOKEN` on the fork).
Accountflow's Grafana loads the plugin via `allow_loading_unsigned_plugins`, and
iac's init container strips `MANIFEST.txt` when present.

1. On `accountflow`, set `package.json` `version` to `<upstream-version>-accountflow.<n>`
   (bump `<n>` for another build of the same upstream version) and commit.
2. Tag and push: the tag must be `nais-apm-app-v<that version>`, because
   `release-fork.yml` checks the tag against `package.json`.

   ```sh
   git tag nais-apm-app-v0.24.0-accountflow.2
   git push origin nais-apm-app-v0.24.0-accountflow.2
   ```

3. The tag push triggers `release-fork.yml`: frontend build, Go backend build
   (mage `buildAll`), zip, `.zip.sha1`, `plugin-validator -analyzer=metadatavalid`,
   then `gh release create` with both assets. Re-running it on the same tag
   replaces the assets. It can also be started by hand:
   `gh workflow run release-fork.yml -R accountflow/grafana-apm-app -f tag=<tag>`.
4. Hand the asset URL and sha1 to iac (`monitoring/k8s/grafana.ts`).

The stock `release.yml` (grafana/plugin-actions/build-plugin) cannot be used: it has
an unconditional check requiring the tag to be exactly `v<version>`, which rejects
the `nais-apm-app-v…` tag shape.

## Published release

| | |
|---|---|
| Tag | `nais-apm-app-v0.24.0-accountflow.1` |
| Plugin version | `0.24.0-accountflow.1` |
| Zip | https://github.com/accountflow/grafana-apm-app/releases/download/nais-apm-app-v0.24.0-accountflow.1/nais-apm-app-0.24.0-accountflow.1.zip |
| sha1 | `c89c2a74316057135725d7fb6222285ac101e384` |

## Prepared release

`package.json` prepares `0.24.0-accountflow.3` with Go 1.26.10, including the
updated certificate distrust filter for eMudhra. After its PR lands on
`accountflow`, publish the fresh immutable tag
`nais-apm-app-v0.24.0-accountflow.3` using the procedure above. Version
`0.24.0-accountflow.2` is already published; do not rebuild or replace its assets.
The new release zip and checksum do not exist until the workflow succeeds. Then
update iac's `monitoring/k8s/grafana.ts` with the new version and the published
`.zip.sha1`, and verify the infrastructure change after applying it.

## Exit criterion

The fork is done when upstream ships a working player: either a
`@grafana/rrweb-player` build whose `onMount` bodies are present, or nais switching
the dependency away from the stub build. At that point iac's
`NAIS_APM_PLUGIN_URL` and Renovate rule point back at nais releases, and this
repository is archived.
