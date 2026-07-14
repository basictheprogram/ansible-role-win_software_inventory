# ansible-role-win_software_inventory

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Ansible Galaxy](https://img.shields.io/badge/galaxy-realtime.win__software__inventory-blue.svg)](https://galaxy.ansible.com/ui/standalone/roles/realtime/win_software_inventory/)

Collects the list of installed software (name, version, publisher, install
date) from Windows endpoints via the registry Uninstall keys, and writes a
structured JSON file per host into a local git working copy on the Ansible
control node. Commits and pushes once per play run, giving a version-controlled
history of software changes across the fleet over time via normal `git log`/
`git diff` — no NetBox or other system of record required (see `DESIGN.md`
for why NetBox was ruled out).

## Requirements

* Ansible core >= 2.20
* `ansible.windows` collection >= 1.11 (declared in `requirements.yml`)
* WinRM configured and reachable on all managed hosts
* A pre-existing git repository (and control-node credentials/SSH agent
  able to push to it) for `win_software_inventory_git_repo`

## Supported Platforms

| OS | Versions |
|---|---|
| Windows | 10, 11 |
| Windows Server | 2016, 2019, 2022, 2025 |

> **Note:** Windows 10 reached end of life in October 2025 but is kept as a
> supported target deliberately — there are still EOL machines in the fleet
> this role needs to inventory.

## Role Variables

### `defaults/main.yml` — user-overridable

| Variable | Default | Description |
|---|---|---|
| `win_software_inventory_output_dir` | `/opt/ansible-inventory/win-software` | Path on the control node for the git working copy. Cloned automatically if absent. |
| `win_software_inventory_git_repo` | `""` | Git URL to clone/pull/push. Required (non-empty) when `win_software_inventory_git_commit` is true. |
| `win_software_inventory_git_branch` | `main` | Branch checked out, committed to, and pushed. |
| `win_software_inventory_git_commit` | `true` | Perform git clone/pull/commit/push. Set `false` to only write local files. |
| `win_software_inventory_git_commit_message` | `Update Windows software inventory (<UTC timestamp>)` | Commit message for the single per-run commit. |

### `vars/main.yml` — internal, not user-overridable

These are loaded automatically and should not be overridden in playbooks.

| Variable | Value | Description |
|---|---|---|
| `win_software_inventory_ps_encoding` | `UTF8` | PowerShell output encoding for `win_shell` commands. |
| `win_software_inventory_registry_paths` | 3 Uninstall key paths (HKLM 64-bit, HKLM WOW6432Node, HKCU) | Registry paths queried for installed-software entries. |
| `win_software_inventory_chocolatey_lib_path` | `C:\ProgramData\chocolatey\lib` | Chocolatey's package library directory. Used as an `install_date` fallback when present — see below. |

## Collected Data

Each host produces one sorted JSON file, named `<inventory_hostname>.json`,
under `{{ win_software_inventory_output_dir }}/inventory/`:

```json
[
  {
    "name": "7-Zip 23.01",
    "version": "23.01",
    "publisher": "Igor Pavlov",
    "install_date": "20250114",
    "install_date_source": "registry"
  },
  {
    "name": "AniTa Terminal",
    "version": "12.2.0.1",
    "publisher": "April System Design",
    "install_date": "20260528",
    "install_date_source": "chocolatey"
  }
]
```

| Field | Source |
|---|---|
| `name` | Registry `DisplayName` |
| `version` | Registry `DisplayVersion` (may be empty) |
| `publisher` | Registry `Publisher` (may be empty) |
| `install_date` | Registry `InstallDate` (`YYYYMMDD`), else a Chocolatey-derived date, else empty |
| `install_date_source` | `"registry"`, `"chocolatey"`, or `""` — see below |

### Install date fallback

`InstallDate` is optional and many non-MSI installers never set it — not a
bug, just source-side data quality every registry-based inventory tool
hits. When it's blank, the role tries one fallback: if
`win_software_inventory_chocolatey_lib_path` exists on the host (most
workstations are expected to run Chocolatey; most servers won't), each
installed Chocolatey package's `.nupkg` file `LastWriteTime` is used as a
proxy install date for any registry entry whose name fuzzy-matches that
package's ID. The match is best-effort (normalized substring containment,
since Chocolatey IDs like `anita-terminal` rarely match a `DisplayName`
like `AniTa Terminal` exactly) — `install_date_source` is included so a
heuristic Chocolatey-derived date is never mistaken for the authoritative
registry value. See `DESIGN.md`'s "Install date fallback" section for the
full rationale.

Entries with no `DisplayName` (patches, updates, other registry noise) are
excluded. `Win32_Product` (WMI) is deliberately not used — it enumerates via
MSI reconfiguration as a side effect.

## Task Flow

1. **Preflight** — fail-fast assertions run before any host changes:
   - Ansible version is >= 2.20.
   - Target host is Windows (`ansible_facts['os_family'] == 'Windows'`).
   - `win_software_inventory_output_dir` is defined and non-empty.
   - `win_software_inventory_git_repo` is set when `win_software_inventory_git_commit` is true.
2. **Sync git working copy** — clone/pull `win_software_inventory_git_repo`
   into `win_software_inventory_output_dir` (`run_once`, `delegate_to: localhost`).
   Skipped when `win_software_inventory_git_commit` is false.
3. **Registry query + Chocolatey fallback** — a single `ansible.windows.win_shell`
   task queries all three Uninstall key paths and applies the Chocolatey
   install-date fallback, all in one PowerShell script (`changed_when: false`,
   read-only).
4. **Parse, dedupe, sort** — parse the JSON result, dedupe exact duplicates
   (the same entry appearing identically in more than one hive), sort by name.
5. **Write JSON** — `ansible.builtin.copy` with `content: … | to_nice_json`
   to `{{ win_software_inventory_output_dir }}/inventory/{{ inventory_hostname }}.json`
   (`delegate_to: localhost`).
6. **Commit + push** — exactly once per play run, after every host's file
   is written (`run_once`, `delegate_to: localhost`): `git add -A`, commit
   only if there are staged changes, then `git push`. Skipped when
   `win_software_inventory_git_commit` is false. See `DESIGN.md` for why
   this is race-free even across many hosts in one play.

## Example Playbook

```yaml
- name: Collect Windows software inventory
  hosts: windows_hosts
  gather_facts: true
  roles:
    - role: realtime.win_software_inventory
      vars:
        win_software_inventory_git_repo: "git@gitlab.real-time.com:ansible-roles/win-software-inventory-data.git"
        win_software_inventory_output_dir: "/opt/ansible-inventory/win-software"
```

## Out of Scope

* WinRM configuration or bootstrapping
* NetBox integration of any kind (see `DESIGN.md`, Settled Decisions)
* Per-run history files — git commit history on the overwritten
  current-state file is the history
* Automatic retry/rebase on `git push` failure (e.g. a concurrent push from
  outside this play) — the run fails loudly instead
* CSV or other output formats — JSON only

## License

MIT

## Author

Bob Tanner, Real Time Enterprises, Inc.
