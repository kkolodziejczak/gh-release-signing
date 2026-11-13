# gh-release-signing

Signs Windows release files on a GitHub-hosted runner with a Certum SimplySign cloud certificate.
It has three parts:

- a reusable workflow, `.github/workflows/sign.yml`, that a repository's release workflow calls;
- the runner script `scripts/SimplySign-Ci.ps1`;
- a GitHub CLI extension, `gh release-signing approve`, that the owner runs locally to approve
  the signing job and type the phone code.

Repositories keep no copy of any of it.

## How a signed release runs

1. The calling workflow builds the unsigned files and uploads them as an artifact.
2. `sign.yml` waits for the `code-signing` environment approval. Only the owner can give it.
3. The job installs the pinned SimplySign Desktop, opens its login window, and waits for the
   owner's one-time phone code. The code arrives in an environment variable named for the run
   attempt, `SIMPLYSIGN_CODE_<run id>_<attempt>`: the "mailbox".
4. `gh release-signing approve` asks for the approval and the code, and puts the code in the
   mailbox. The job masks the code and deletes the mailbox as soon as it reads it. Then it logs
   in, signs every matching file with an RFC 3161 Certum timestamp, checks each signature against
   the certificate thumbprint, and uploads the signed files as a new artifact.
5. At the end the job removes any leftover mailboxes and stops SimplySign.

Nothing stored in GitHub can sign on its own: every signing needs a fresh phone code that works
once, for about 30 seconds. Only collaborators, or tokens with Environments read on the calling
repository, can read the mailbox through the API. In workflows, only jobs past the owner's
approval get the environment's secrets. Watching a public run shows when a code is sent, but not
the code.

## Use it in a repository

```yaml
jobs:
  build:
    # ... builds the files and uploads them as the artifact "unsigned-release"

  sign:
    needs: build
    uses: kkolodziejczak/gh-release-signing/.github/workflows/sign.yml@main
    secrets: inherit    # without it the job sees no secrets, not even its environment's
    permissions:
      contents: read
      id-token: write   # lets sign.yml find its own commit and check out its scripts
    with:
      artifact: unsigned-release
      files: "*.exe"            # default; searched in every folder of the artifact
      signed-artifact: signed-release
      retention-days: 7

  release:
    needs: sign
    # downloads "signed-release"; ${{ needs.sign.outputs.thumbprint }} is the signing certificate
```

Sign executables before packing them into a zip or installer, and compute hashes after signing.

### One-time setup per repository

1. Create the `code-signing` environment. Add the owner as required reviewer, and allow
   deployments only from the release tags (for example `v*.*.*`) and the default branch.
2. Add two environment secrets:
   - `CERTUM_EMAIL`: the SimplySign account e-mail.
   - `MAILBOX_TOKEN`: a fine-grained token with Environments read and write on that repository.
     One token can cover several repositories; add the repository to the token's access list.

### One-time setup per computer

```powershell
gh extension install kkolodziejczak/gh-release-signing
```

On Windows, gh runs the extension through Git Bash, which starts Windows PowerShell. Update it
with `gh extension upgrade release-signing`.

## Sign a release

Push the release tag, then run this in the repository's folder and keep the window open:

```powershell
gh release-signing approve -Tag vX.Y.Z
```

It shows the run's event, ref, commit and actor and asks you to approve. With `-Tag`, it refuses a
run that is not the push of that tag. About a minute later it beeps and asks for the SimplySign
phone code, shown as you type it to catch typos. If the runner does not pick the code up within
30 seconds, it withdraws the code and asks for a fresh one.

If the code expired or the login failed, run `gh run rerun <run-id> --failed`, then
`gh release-signing approve -RunId <run-id> -Tag vX.Y.Z`. Pass `-Workflow <file>` when the calling
workflow is not `release.yml`.

With a fine-grained `gh` token, the extension needs Actions read, Contents read, Deployments read
and write, and Environments read and write on the repository. Without Deployments write it tells
you to approve on the run page and still asks for the code.

## Maintenance

- **Yearly certificate renewal:** update `SIGNING_CERT_THUMBPRINT` in `sign.yml`, then run the
  self-test. Every repository picks it up from `main`.
- **Expired mailbox token:** replace the `MAILBOX_TOKEN` secret in each repository.
- **Self-test:** `gh workflow run self-test.yml -R kkolodziejczak/gh-release-signing`, then
  `gh release-signing approve -Workflow self-test.yml` in this repository's folder. It signs and
  runs a throwaway executable through `sign.yml`. This repository needs its own `code-signing`
  environment for it.
- Signing jobs in one repository run one at a time. A run waiting for approval holds that queue,
  so reject runs you will not sign. Approve signing, or rerun failed jobs, before the calling
  workflow's artifacts expire.
- Typing into SimplySign's login window depends on the runner's desktop. If that becomes
  unreliable, consider a self-hosted runner.
