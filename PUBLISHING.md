# Publishing win95

How to release a new version of `asilva.win95` to the [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=asilva.win95) and [Open VSX](https://open-vsx.org/extension/asilva/win95).

## How it works

Pushing a `v*` tag runs [`.github/workflows/publish.yml`](.github/workflows/publish.yml). The workflow:

1. Checks that the tag matches the `version` in `package.json`.
2. Logs in to Azure with a Microsoft Entra ID managed identity (OIDC, no stored token).
3. Packages the extension once (`npm run package` → `win95.vsix`).
4. Publishes that same VSIX to the Marketplace (`npm run publish:marketplace`) and to Open VSX (`npm run publish:openvsx`).

Both publish steps use `--skip-duplicate`, so re-running a release is safe.

## Cut a release

1. Add an entry for the new version to [`CHANGELOG.md`](CHANGELOG.md) and commit it. It shows on the Changelog tab of both listings. `npm version` needs a clean working tree.
2. Bump the version, commit and tag:
   ```sh
   npm version patch -m "release v%s"   # or minor / major
   ```
   This updates `package.json` and `package-lock.json`, commits them, and creates the tag `vX.Y.Z`.
3. Push the commit and the tag:
   ```sh
   git push --follow-tags
   ```
4. Follow the run:
   ```sh
   gh run watch
   ```
5. Confirm the published versions. The Marketplace verifies each upload, which can take a few minutes.
   ```sh
   npx vsce show asilva.win95
   curl -s https://open-vsx.org/api/asilva/win95 | jq .version
   ```

### Publish again without a new tag

If a run failed after the tag was pushed (for example, an expired Open VSX token), fix the cause and run the workflow from `master`:

```sh
gh workflow run publish.yml --ref master
```

Versions that already exist are skipped.

## Test the package locally

```sh
npm ci
npm run package
npx vsce ls
```

The package should contain only `package.json`, `README.md`, `CHANGELOG.md`, `LICENSE`, `icon.png` and `themes/win95-color-theme.json` (about 9 KB). Everything else is excluded by [`.vscodeignore`](.vscodeignore). To try it in VS Code:

```sh
code --install-extension win95.vsix
```

## Update the screenshot

1. Take a screenshot of VS Code with the win95 theme and a sample project. Make sure nothing personal shows (paths, terminal history, account names).
2. Save it with a **new** filename (`example-v4.png`, `example-v5.png`, …). Reusing a name can leave the old image cached on GitHub and on the listings.
3. Update the image path and alt text in `README.md`. SVG images are not allowed on the Marketplace.
4. Push to `master` **before** releasing. The listings load README images from `github.com/arxdsilva/win95/raw/HEAD/...`.

## Authentication setup (reference)

This is already configured. Use it when something breaks or has to be recreated. IDs are kept out of this file on purpose. Find them in the Azure portal and in the GitHub environment secrets.

### VS Code Marketplace (Microsoft Entra ID)

- **Azure:** resource group `win95` → user-assigned managed identity.
- **Federated credential** on that identity:
  - Issuer: `https://token.actions.githubusercontent.com`
  - Subject (immutable format): `repo:arxdsilva@<owner_id>/win95@<repo_id>:environment:marketplace-publish`
  - Audience: `api://AzureADTokenExchange`
- **GitHub OIDC:** the repository uses immutable subject claims. Check with:
  ```sh
  gh api repos/arxdsilva/win95/actions/oidc/customization/sub
  # expected: "use_immutable_subject": true
  ```
- **GitHub environment:** `marketplace-publish`, with the secrets `AZURE_CLIENT_ID` and `AZURE_TENANT_ID` (from the identity's Properties page).
- **Marketplace publisher `asilva`:** the identity is a **Contributor** under Members. The workflow step *Show Marketplace identity id* prints the ID that has to be added there. The Client ID and Tenant ID are not accepted.

### Open VSX

- **Account:** sign in at [open-vsx.org](https://open-vsx.org) with GitHub. The Eclipse Foundation Publisher Agreement must be signed (profile page).
- **Namespace:** `asilva`.
- **Token:** create it under Settings → Access Tokens and store it as the `OVSX_PAT` secret in the `marketplace-publish` environment:
  ```sh
  gh secret set OVSX_PAT --env marketplace-publish --repo arxdsilva/win95
  ```

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `AADSTS700213: No matching federated identity record found` | The subject GitHub sends doesn't match the Azure federated credential. Compare the *subject claim* in the log with the credential. Check the immutable-subject setting and the environment name. |
| `Access Denied: <id> needs the following permission(s) on the resource /asilva/win95` | That ID is not a Contributor on the Marketplace publisher. Add it under Members. |
| `Tag vX.Y.Z does not match package.json version` | The tag was created by hand. Delete it and use `npm version`. |
| Broken image on a listing | The image wasn't on `master` when the version was published. Push it, then release a new patch version. |
| Open VSX `401` or unknown publisher error | `OVSX_PAT` expired or was revoked, or the Publisher Agreement lapsed. Create a new token and update the secret. |
| `npm warn deprecated prebuild-install` | Harmless. It comes from `keytar`, a dependency of `@vscode/vsce`. Nothing to fix in this repository. |

## Fallback: manual upload

If the workflow can't run, build the package and upload it by hand:

```sh
npm ci
npm run package   # creates win95.vsix
```

- **Marketplace:** [marketplace.visualstudio.com/manage/publishers/asilva](https://marketplace.visualstudio.com/manage/publishers/asilva) → **⋯** on the win95 row → **Update** → upload `win95.vsix`.
- **Open VSX:** [open-vsx.org/user-settings/extensions](https://open-vsx.org/user-settings/extensions) → **Publish** → upload `win95.vsix`.
