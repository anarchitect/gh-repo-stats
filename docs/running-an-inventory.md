# Running an inventory

A complete walkthrough for producing a repository inventory, suitable for handing to someone running it for the first time.

For a summary of the tool itself, see the [README](../README.md).

---

## Added columns

This fork appends eight columns after the existing ones, so column positions in downstream tooling are unchanged.

| Column | Source | Notes |
| --- | --- | --- |
| `Default_Branch` | GraphQL | Empty for a repository with no commits |
| `Pipeline_Count` | REST | Number of GitHub Actions workflows defined |
| `Last_Commit_Date` | GraphQL | Last commit on the default branch |
| `Last_Commit_Author` | GraphQL | Commas are replaced with spaces to protect the CSV |
| `Last_Pipeline_Status` | REST | Conclusion of the most recent run, or its status while still running |
| `Last_Pipeline_Date` | REST | When that run last updated |
| `Last_Pipeline_Ref` | REST | Branch the run executed against; not necessarily the default branch |
| `Maintainers` | REST | Direct collaborators only — see [Reading the results](#reading-the-results) |

The first three come from the existing GraphQL query at no additional API cost. The Actions and collaborator fields are only available over REST, because the GraphQL API has no schema for Actions workflow runs, and cost roughly three REST calls per repository. Because of that cost, an organization of roughly **1,500 repositories or more** may exhaust the hourly REST quota before a run finishes.

Enrichment failures abort the run rather than writing blank cells, so a `0` in `Pipeline_Count` always means "no workflows" and never "could not check". On abort the partial CSV is renamed to `*-INCOMPLETE.csv` and the exit code is non-zero.

The full column reference, including the columns inherited from upstream, is in the [README](../README.md#output).

---

## Before you start

> [!IMPORTANT]
> **Check the repository count first.** An organization of roughly **1,500 repositories or more** may exhaust the hourly REST quota of 5,000 calls before the run finishes. Raise this with whoever asked you to run the inventory before starting a large run.
>
> ```powershell
> gh api "orgs/YOUR-ORG/repos?per_page=1" --include | Select-String -Pattern "^link:"
> ```
>
> The `page=` value at the end of the `rel="last"` link is the repository count. It is also shown on the organization's home page.

**What is collected.** Metadata only: repository names and URLs, sizes, dates, branch and tag counts, issue and pull request counts, workflow counts and their most recent run status, the most recent commit date and author, and repository maintainers. **No source code is read or transmitted.** The CSV is written to your local machine and goes nowhere else.

**Access required.** Run as a **GitHub organization owner**. The added columns read repository administration settings, and an account without owner access will abort rather than produce partial data.

---

## 1. Install the prerequisites

Open **PowerShell** or **Windows Terminal**. Two tools are required: the GitHub CLI (`gh`) and `jq`, a JSON command-line processor.

```powershell
winget install --id GitHub.cli
winget install --id jqlang.jq
```

If `winget` is unavailable, see <https://cli.github.com> and <https://jqlang.org/download/>.

Close and reopen your terminal after installing, then confirm both are on your `PATH`:

```powershell
gh --version
jq --version
```

Both must succeed before continuing. If either reports "not found", the tool is not installed correctly or the terminal has not picked up the updated `PATH`.

---

## 2. Sign in to GitHub

```powershell
gh auth login
```

Follow the prompts and sign in as an organization owner.

One permission beyond the default set is required, used to count linked projects:

```powershell
gh auth refresh -h github.com -s read:project
```

This displays a one-time code and opens a browser page. Enter the code and approve the request. Confirm it applied:

```powershell
gh auth status
```

The `Token scopes` line must include `read:project`.

> [!TIP]
> If you are signed in to more than one GitHub account, make the right one active before running anything, and again in any new terminal:
>
> ```powershell
> gh auth switch --user YOUR-USERNAME
> ```

---

## 3. Install the inventory tool

```powershell
gh extension install anarchitect/gh-repo-stats --pin inventory-v1.4
```

The `--pin` flag locks you to a specific tested version, so results stay reproducible if the tool is updated later. Confirm it installed:

```powershell
gh repo-stats --help
```

---

## 4. Run the inventory

Replace `YOUR-ORG` with the organization name exactly as it appears in its GitHub URL:

```powershell
gh repo-stats --org YOUR-ORG --output CSV
```

**Expected duration: roughly 20 seconds per repository.** A 200-repository organization takes about an hour. The exact figure varies with how many issues and pull requests each repository holds, and the script deliberately paces itself to stay within GitHub's rate limits, so periods of apparent inactivity are normal and expected. Leave the terminal open until it finishes.

> [!NOTE]
> `--output` selects the output *format* (`CSV` or `Table`), not a filename. Results are always written to `<org>-all_repos-<timestamp>.csv` in the directory you ran the command from.

**Expected output** is a progress line per repository, followed by:

```
######################################################
The script has completed

Results file:[YOUR-ORG-all_repos-<timestamp>.csv]
######################################################
```

---

## 5. Collect the results

The file to keep is `YOUR-ORG-all_repos-<timestamp>.csv`.

**If the filename ends in `-INCOMPLETE.csv`**, the run stopped early and the data is partial. Keep it anyway and note the error shown in the terminal — together they identify what went wrong. Do not treat an `-INCOMPLETE` file as a full inventory.

---

## Scanning more than one organization

Pass a file of organization names instead of `--org`, one per line:

```powershell
gh repo-stats --input orgs.txt --output CSV
```

> [!WARNING]
> **The file must use Unix (LF) line endings.** With Windows (CRLF) endings the carriage return is carried into the API URL and every organization fails:
>
> ```
> parse "https://api.github.com/orgs/my-org\r/memberships/you":
> net/url: invalid control character in URL
> ```
>
> Notepad and many Windows editors default to CRLF. In VS Code, use the **CRLF/LF** selector in the status bar. From PowerShell:
>
> ```powershell
> [System.IO.File]::WriteAllText("orgs.txt", "first-org`nsecond-org`n")
> ```

Two further things to expect:

- **All organizations are written to a single CSV**, distinguished by the `Org_Name` column. The filename omits the organization name, so it begins with a hyphen — `-all_repos-<timestamp>.csv`.
- **Runtime is cumulative.** At roughly 20 seconds per repository, the total is the sum across every organization in the file. Check the combined repository count against the 1,500 guidance above before starting.

---

## Removing the tool afterwards

```powershell
gh extension remove repo-stats
```

To revoke the additional permission, visit <https://github.com/settings/applications>, select **GitHub CLI**, and adjust or revoke access.

---

## Reading the results

Two columns are easy to misread.

### How `Maintainers` is defined

`Maintainers` lists **direct collaborators** holding **admin** or **maintain** permission, separated by semicolons.

It **deliberately excludes** access inherited from organization or team membership. Only direct grants are attached to an individual repository and have to be recreated by hand in a target organization; organization-wide and team-based access is handled separately during a migration.

This means the column is **not** a list of everyone who can access a repository. A blank cell means "no individual permission grants", **not** "nobody can administer this". In a well-governed organization most repositories will legitimately be blank, because access is managed through teams.

### `0` versus blank

A `0` in `Pipeline_Count` genuinely means the repository has no workflows. The script is built to stop rather than guess: if it cannot read a repository it aborts the run instead of writing a misleading `0` or blank. Values that do appear can therefore be trusted.

An empty `Last_Pipeline_Status` means the repository has never run a workflow. An empty `Default_Branch` means the repository has no commits.

---

## Troubleshooting

| Symptom | What it means | What to do |
| --- | --- | --- |
| `jq: command not found` | `jq` is not installed or not on `PATH` | Return to [step 1](#1-install-the-prerequisites) and reopen the terminal |
| `Your token has not been granted the required scopes` | Missing `read:project` | Re-run the `gh auth refresh` command in [step 2](#2-sign-in-to-github) |
| `Error getting Membership for Org: <name>` | The organization name is wrong, or the signed-in account is not a member | Check the exact spelling in the organization's GitHub URL, and confirm the active account with `gh auth status` |
| `the token lacks administration access` | Not signed in as an organization owner | Re-run `gh auth login` as an owner |
| `API rate limit exhausted` | The hourly quota ran out mid-run | Wait for the quota to reset before retrying. If the organization has 1,500 or more repositories, raise it rather than simply re-running |
| Filename ends `-INCOMPLETE.csv` | The run stopped partway | Keep the file and the terminal error message; do not treat it as a full inventory |
| `invalid API endpoint: "C:/Program Files/Git/"` | An older version, on a Windows shell reporting `OSTYPE` as `cygwin` | Reinstall pinned to `inventory-v1.4` |
| `net/url: invalid control character in URL` | The `--input` organization file has Windows (CRLF) line endings | Save it with Unix (LF) endings — see [scanning more than one organization](#scanning-more-than-one-organization) |

---

## Why the run is paced

Three back-to-back REST calls per repository trips GitHub's secondary rate limiter long before the primary hourly quota is touched, and GitHub reports that throttling on the Actions and collaborators endpoints as `404` rather than `403` — so a throttled request is indistinguishable from a deleted repository by status code alone.

Requests are therefore spread out, retried with exponential backoff, and an exhausted primary quota is confirmed against the live `rate_limit` endpoint rather than inferred from the error text. This makes runs slower but reliable.
