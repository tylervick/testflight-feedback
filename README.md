# testflight-feedback

Turns TestFlight tester feedback into GitHub Issues: one issue per screenshot or crash submission,
in the repository of the app it came from, never the same submission twice.

Screenshots are attached, crash logs are embedded collapsed, and the tester's identity is never
read out of the App Store Connect response.

## Use it

Add one workflow to the app's repository:

```yaml
name: TestFlight feedback

on:
  schedule:
    - cron: '*/30 * * * *'
  workflow_dispatch:
    inputs:
      dry_run:
        description: List what would be created without opening issues
        type: boolean
        default: false

permissions: {}

jobs:
  import:
    uses: tylervick/testflight-feedback/.github/workflows/import.yml@<commit sha> # v1.0.0
    with:
      bundle-id: com.example.app
      dry-run: ${{ inputs.dry_run || false }}
    permissions:
      issues: write
      contents: write
    secrets:
      ASC_PRIVATE_KEY: ${{ secrets.ASC_PRIVATE_KEY }}
      ASC_KEY_ID: ${{ secrets.ASC_KEY_ID }}
      ASC_ISSUER_ID: ${{ secrets.ASC_ISSUER_ID }}
```

Pin to a commit sha rather than a tag: a tag can be moved, and a moved tag here would silently
change what runs with your credentials. Name the three secrets rather than `secrets: inherit` —
an app repository usually also holds signing certificates and profile passwords, and this needs
none of them.

It needs three secrets in the calling repository: `ASC_PRIVATE_KEY` (the `.p8` App Store Connect
key, base64 encoded), `ASC_KEY_ID`, and `ASC_ISSUER_ID`. An App Store Connect key is issued per
team, not per app, so the same three values serve every app on the team.

`permissions:` on the calling job is required — a called workflow cannot hold more than its caller
grants, and the default is read-only.

### Scheduling

GitHub drops and delays scheduled runs on shared runners, so `*/30` in practice means every few
hours. The manual dispatch covers "I know feedback just arrived". Nothing breaks if a run is
skipped or runs twice: the importer tracks what it has already opened.

## Run it locally

From a checkout of the app's repository, so `gh` resolves the right repository:

```sh
export ASC_KEY_ID=... ASC_ISSUER_ID=... ASC_KEY_PATH=/path/to/AuthKey_XXX.p8
/path/to/testflight-feedback/scripts/testflight-feedback.sh --bundle-id com.example.app --dry-run
```

`--repo owner/name` targets a repository explicitly. `--dry-run` touches nothing on GitHub.

## How it avoids duplicates

Every issue it opens carries the `testflight-feedback` label and a hidden
`<!-- testflight-feedback:<submission id> -->` marker in its body. Each run lists issues by that
label and reads the markers back.

It deliberately does not use GitHub's search: the search index lags a fresh issue by seconds to
minutes, so two runs close together would open the same submission twice. Listing by label is
served consistently.

## Tests

```sh
for t in scripts/test-*.sh; do "$t"; done
```

`curl` and `gh` are stubs on `PATH` answering from fixtures, so the tests never reach App Store
Connect or GitHub.
