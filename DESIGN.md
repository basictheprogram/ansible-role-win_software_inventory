# Design Document: ansible-role-win_software_inventory

## Overview

This document is the authoritative spec for the `win_software_inventory`
Ansible role. It collects the list of installed software (name, version,
publisher, install date) from Windows endpoints via the registry Uninstall
keys, and writes one structured JSON file per host into a local git working
copy on the Ansible control node. A single, run-once task then commits and
pushes any changes to GitLab, giving a version-controlled history of
software changes across the fleet over time.

Code that disagrees with this document is wrong. Flag the discrepancy and
ask before changing either side.

---

## Goals

* Enumerate installed software per host, in a form comparable across runs.
* Store results in a git repo so changes over time are visible via normal
  `git log`/`git diff` — no NetBox or other new system of record required.
* Avoid concurrent-write races when the role runs across many hosts in a
  single play.
* Be reusable, idempotent, and easy to integrate into existing playbooks.

---

## Collected Facts

| Field | Source | Notes |
|---|---|---|
| Name | `DisplayName` | Registry Uninstall key value |
| Version | `DisplayVersion` | Registry Uninstall key value; may be absent for some entries |
| Publisher | `Publisher` | Registry Uninstall key value; may be absent |
| Install Date | `InstallDate`, or a Chocolatey fallback — see below | Raw `YYYYMMDD` string when known, else empty |
| Install Date Source | derived | `"registry"`, `"chocolatey"`, or `""` — see below |

Source keys, all queried via PowerShell/registry (no WMI `Win32_Product` —
notoriously slow and triggers MSI reconfiguration as a side effect):

* `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*` — native 64-bit apps
* `HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*` — 32-bit apps on 64-bit OS
* `HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*` — per-user installs (relevant for
  Windows 10/11 desktop targets; typically empty on servers)

Entries with no `DisplayName` (patches, updates, and other non-application
registry noise) are excluded.

### Install date fallback

`InstallDate` is an optional, installer-set registry value — MSI-based
installers usually populate it (from the `ARPINSTALLDATE` property); many
non-MSI installers (NSIS, Inno Setup, custom bootstrapper EXEs, hand-registered
Uninstall entries) simply never write it. This is expected, source-side data
quality, not a bug — every tool that reads these registry keys hits the same
gap.

Since the goal is all workstations running Chocolatey, entries with a blank
registry `InstallDate` get one fallback attempt before giving up:

1. **Registry `InstallDate`**, when present → `install_date_source: "registry"`.
2. **Chocolatey**, when the target host has `win_software_inventory_chocolatey_lib_path`
   (`C:\ProgramData\chocolatey\lib`) and a package there fuzzy-matches this
   entry's `DisplayName` → `install_date_source: "chocolatey"`. Chocolatey
   has no explicit install-date field; the matched package's `.nupkg` file
   `LastWriteTime` is used as a proxy for "when this version was installed"
   (a reasonable semantic match, since `version` itself always reflects the
   currently-installed version too, not a historical record).
3. Otherwise, blank → `install_date_source: ""`.

**Matching is fuzzy and best-effort, by design.** Chocolatey package IDs
(e.g. `anita-terminal`) rarely match a registry `DisplayName` exactly (e.g.
`AniTa Terminal`). Both sides are normalized (lowercased, non-alphanumerics
stripped) and matched on substring containment either direction. This will
occasionally miss a real match or, less likely, match the wrong package —
`install_date_source` exists specifically so a Chocolatey-derived guess is
never silently indistinguishable from an authoritative registry value.
Consumers of this data that care about date accuracy should filter or
weight by `install_date_source`.

The Chocolatey lookup only runs when the lib directory exists on that host —
most servers won't have it and pay no extra cost; most workstations are
expected to (see Settled Decisions).

---

## Role Structure

```
ansible-role-win_software_inventory/
├── defaults/
│   └── main.yml              # User-overridable defaults
├── vars/
│   └── main.yml              # Internal implementation constants (not user-overridable)
├── tasks/
│   ├── main.yml              # Primary task entry point
│   └── preflight.yml         # Fail-fast assertions
├── handlers/
│   └── main.yml              # Empty — no service restarts needed
├── meta/
│   ├── main.yml               # Galaxy metadata
│   └── argument_specs.yml     # Role argument documentation
├── requirements.yml           # ansible.windows collection dependency
├── DESIGN.md                  # This file
├── CLAUDE.md                  # AI assistant project notes
└── README.md
```

No `templates/` directory — the JSON output is rendered directly from a
structured variable via the `to_nice_json` filter (see "Output rendering"
below), not a hand-written Jinja2 template. A variable-length package list
is exactly the case where a hand-rolled template loop (commas between
items, last-item handling) is fragile; `to_nice_json` sidesteps that
entirely.

---

## Task Design

### `tasks/preflight.yml`

Included first, before any host interaction. Asserts:

* Ansible version is >= 2.20.
* Target host `ansible_facts['os_family']` is `Windows`.
* `win_software_inventory_output_dir` is defined and non-empty.
* `win_software_inventory_git_repo` is defined and non-empty when
  `win_software_inventory_git_commit` is true (nothing to push to otherwise).

### `tasks/main.yml`

1. **Include preflight** — fail fast before touching any host.
2. **Sync the git working copy** (`run_once: true`, `delegate_to: localhost`) —
   clone `win_software_inventory_git_repo` to
   `win_software_inventory_output_dir` if absent, otherwise pull latest. Only
   when `win_software_inventory_git_commit` is true. Uses
   `ansible.builtin.git`, which has full clone/pull support (unlike commit/push
   — see below).
3. **Gather installed software** from all three registry Uninstall hives and
   apply the Chocolatey install-date fallback, in a single
   `ansible.windows.win_shell` task (`changed_when: false`). Registry paths
   and field-renaming both happen inside the PowerShell script, not via
   Jinja2 — see "Why PowerShell does the renaming and looping, not Jinja2"
   below. One WinRM round trip per host instead of three.
4. **Parse and dedupe/sort** the single JSON result into
   `win_software_inventory_packages` (`set_fact` x2): dedupe on exact-match
   entries (covers the same package appearing identically in multiple
   hives), sort by name so unrelated reordering doesn't show up as diff
   noise on unrelated runs.
5. **Ensure the output directory exists** in the git working copy
   (`delegate_to: localhost`) — needed unconditionally, not just when
   `win_software_inventory_git_commit` is true, since the directory may not
   exist yet if git sync (step 2) was skipped.
6. **Write the JSON file** — `ansible.builtin.copy` with
   `content: "{{ win_software_inventory_packages | to_nice_json }}"`, directly
   to `{{ win_software_inventory_output_dir }}/{{ inventory_hostname }}.json`
   (`delegate_to: localhost`) — no `inventory/` subfolder; whatever path
   `win_software_inventory_output_dir` is set to is exactly where the file
   lands. (Also matches `win_hw_inventory`'s convention, which has no
   subfolder either.)
7. **Commit and push** (`run_once: true`, `delegate_to: localhost`, only when
   `win_software_inventory_git_commit` is true) — `git add -A`, then commit
   only if there are staged changes (`git diff --cached --quiet`, `changed_when`
   on the commit task itself), then push. `git add`/`git commit` have no
   `ansible.builtin`/`community.general` module equivalent, so these use
   `ansible.builtin.command`/`ansible.builtin.shell` with a trailing
   `# noqa: command-instead-of-module` comment, matching the convention
   already used for control-node git push tasks elsewhere in this repo
   (e.g. `realtime.dns_servers`). Same for the `git push`.

All registry query tasks use `changed_when: false` — they are read-only.

### Why PowerShell does the renaming and looping, not Jinja2

Two real bugs were found and fixed while building this role, both worth
recording so they aren't reintroduced:

* **Jinja2 has no Python-style list/dict comprehension syntax.** An earlier
  draft tried `{{ [ {'name': i.DisplayName, ...} for i in raw_entries ] }}`
  in a `set_fact` to rename fields — this is a `TemplateSyntaxError` at
  render time, not just a style choice to avoid. Field renaming
  (`DisplayName` → `name`, etc.) is done inside the PowerShell script instead,
  where it's natural syntax.
* **`ansible-lint`'s static parser shell-quote-balances free-form
  `win_shell`/`command` strings.** An earlier draft built the PowerShell
  `$regPaths` array via a Jinja `{% for %}...{% endfor %}` block emitting
  quoted array elements, directly inside the `win_shell` block scalar. The
  rendered PowerShell was valid, but `ansible-lint` failed with
  `parser-error: failed at splitting arguments, either an unbalanced jinja2
  block or quotes` — its parser gets confused by Jinja control-flow tags
  mixed with quote characters inside a free-form module string. Fixed by
  passing the registry paths as a single inline expression instead:
  `'{{ win_software_inventory_registry_paths | to_json }}' | ConvertFrom-Json`.
  No `{% %}` tags inside the script string at all now. If a future change
  needs to template a list into a `win_shell`/`command` string, prefer
  `to_json` + `ConvertFrom-Json` over a Jinja `{% for %}` block for this
  reason.

### Why `run_once` is safe for the git steps here (not a race)

Ansible executes a play's tasks in lockstep across the batch: task *N* runs
on every host in the batch (subject to `forks`/`serial`) before task *N+1*
begins on any host. Because the file-write task (step 6) is *before* the
commit/push task (step 7) in `tasks/main.yml`, every host's
`delegate_to: localhost` file write has already completed by the time the
`run_once: true` commit/push task executes — there is exactly one commit/push
per play run, after every host's file has landed, with no concurrent git
operations. This avoids the multi-host git-push race that a naive
per-host commit/push would hit.

---

## Variables

### `defaults/main.yml` — user-overridable

```yaml
win_software_inventory_output_dir: "/opt/ansible-inventory/win-software"
win_software_inventory_git_repo: ""   # required when win_software_inventory_git_commit is true
win_software_inventory_git_branch: "main"
win_software_inventory_git_commit: true
win_software_inventory_git_commit_message: "Update Windows software inventory ({{ now(utc=true, fmt='%Y-%m-%d %H:%M') }} UTC)"
```

### `vars/main.yml` — internal, not user-overridable

```yaml
win_software_inventory_ps_encoding: "UTF8"
win_software_inventory_registry_paths:
  - 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*'
  - 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*'
  - 'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*'
```

---

## Output Format

One JSON file per host, named `<inventory_hostname>.json`, written directly
to `{{ win_software_inventory_output_dir }}` — no `inventory/` subfolder (an
earlier draft had one; dropped for a simpler mental model — whatever path you
set is exactly where the files land — and to match `win_hw_inventory`'s
convention). Sorted by package name so that a single new/removed/updated
package produces a minimal, readable diff in git history.

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

---

## Compatibility

| Requirement | Version |
|---|---|
| Ansible core | >= 2.20 |
| `ansible.windows` collection | >= 1.11 (declared in `requirements.yml`) |
| Target OS | Windows 10, Windows 11, Windows Server 2016 / 2019 / 2022 / 2025 |
| WinRM | Must be configured on all managed hosts |

> **Windows 10 note:** reached end of life October 2025. Kept as a
> supported target deliberately — there are still EOL machines in the
> fleet this role needs to inventory. Revisit if that stops being true.

---

## Settled Decisions

These decisions are closed. Do not reopen without a strong reason; document
the new decision here if one is made.

* **Storage target: git, not NetBox.** NetBox has no native structured
  model for installed-software data. Journal entries are free text
  (unindexed, unqueryable beyond "attached to this object"); JSON custom
  fields can't be filtered on nested keys via the REST or GraphQL API
  (confirmed against NetBox's own filtering docs and a maintainer
  discussion). Getting real fleet-wide queries ("who has packageA vX")
  out of NetBox would require a custom plugin with proper relational
  models — real, ongoing-maintenance development work, not something this
  role should take on. Git is what the team already uses and trusts for
  this class of data (see `realtime.dns_servers`), and `git log`/`git diff`
  plus `jq`/`grep` across tracked files covers the "history of changes over
  time" requirement directly.
* **Write-then-commit split, not per-host git push.** The role never
  performs a git operation `delegate_to: localhost` per-host outside a
  `run_once: true` gate. Every host writes its own file (step 6); exactly
  one commit/push happens per play run (step 7), after all files are
  written. This was chosen specifically to avoid concurrent
  `git push` races when the role targets many hosts in one play — a naive
  per-host commit/push was considered and rejected.
* **No `inventory/` subfolder — files write directly to
  `win_software_inventory_output_dir`.** An earlier draft treated
  `win_software_inventory_output_dir` as the git working-copy *root* and
  wrote per-host files under an `inventory/` subfolder inside it. That
  surprised a real user: setting
  `win_software_inventory_output_dir: "{{ playbook_dir }}/reports/software"`
  in `host_vars` produced files at `.../reports/software/inventory/<host>.json`,
  not the `.../reports/software/<host>.json` the variable name implied.
  Reversed so whatever path you set is exactly where files land — also
  now matches `win_hw_inventory`'s convention (no subfolder there either).
* **Registry over WMI `Win32_Product`.** `Win32_Product` enumerates via MSI
  reconfiguration as a side effect (slow, and can trigger repair actions).
  The registry Uninstall keys are the standard low-risk source and match
  what most software-inventory tooling does.
* **Chocolatey as an install-date fallback, gated on presence, fuzzy-matched,
  and source-tagged.** Motivated by the fleet's stated direction (all
  workstations expected to run Chocolatey) and by real data seen in
  practice (`AniTa Terminal`, Chocolatey-installed, no registry
  `InstallDate`). Only queried when
  `win_software_inventory_chocolatey_lib_path` exists on the target host —
  no cost on hosts without Chocolatey. Matching is fuzzy (normalized
  substring containment) because Chocolatey package IDs don't reliably
  match registry `DisplayName` strings; `install_date_source` exists so
  this heuristic is never confused with the authoritative registry value.
  See "Install date fallback" above.
* **Output rendering: `to_nice_json` filter via `copy: content:`, not a
  Jinja2 template file.** A variable-length list is the case where
  hand-written comma/last-item template logic is fragile. Filtering to a
  structured variable first and letting `to_nice_json` render it removes
  that failure mode entirely — see `win_hw_inventory`'s
  `templates/inventory.json.j2` for the pattern this deliberately avoids
  (safe there only because its field set is fixed/scalar).
* **Sort by package name before writing.** Minimizes diff noise — a
  reordering with no real change should never show up as a git diff.
* **Windows 10 stays a supported target** despite October 2025 EOL — see
  Compatibility section above.
* **Variable prefix:** all role variables (defaults, internal vars, and
  registered task variables) use the `win_software_inventory_` prefix, to
  satisfy the `.ansible-lint` `loop_var_prefix` rule and avoid namespace
  collisions with other roles. (Note: `win_hw_inventory`'s defaults use a
  mismatched `win_inventory_` prefix rather than `win_hw_inventory_`; this
  role does not repeat that inconsistency.)

---

## Out of Scope

* WinRM configuration or bootstrapping.
* NetBox integration of any kind (see Settled Decisions).
* Per-run history files (e.g. one file per host per run-date) — git commit
  history on the overwritten current-state file *is* the history; no
  separate versioning scheme is layered on top.
* Merge-conflict handling on the git push beyond a plain `git push`
  failing loudly. If the upstream branch has moved (e.g. a manual
  concurrent push, or two separate playbook runs racing each other outside
  a single play), the push fails and the run fails — no automatic
  rebase/retry logic. Flagged as a real gap, not silently handled; see
  `TODO.md`.
* CSV or other output formats — JSON only.

---

## Open Questions

Leave a `# TODO(open-q):` comment in any task that touches one of these.

1. Should `win_software_inventory_git_repo` support SSH deploy keys,
   HTTPS+token auth, or both? Current implementation assumes whatever
   `ansible.builtin.git` and the ambient control-node git config /
   SSH agent already handle — no explicit credential wiring in this role
   yet.
2. Retry/backoff on `git push` failure (e.g. another process pushed to the
   same branch between this role's pull and its push)? Not implemented —
   see Out of Scope.
3. Per-user installs (`HKCU`) are only meaningful for the user context
   WinRM connects as. Is that acceptable, or does this need to enumerate
   all local user profiles' hives?

---

## Next Steps

1. ~~Review and approve this design document~~ ✓
2. Scaffold the role directory structure
3. Implement `tasks/main.yml` registry queries + git sync/commit/push
4. Implement `tasks/preflight.yml`
5. Lint pass — `pre-commit run --all-files` clean
6. Molecule scenario — scaffold `molecule/default/` with WinRM target and a
   self-contained local git fixture (see Step 10 guidance: the
   `delegate_to: localhost` commit/push path may not be fully
   fixture-testable — confirm scope with the user before assuming full
   coverage)
7. Pytest test suite — `molecule/default/tests/` covering output file
   content and (where feasible) the git commit
