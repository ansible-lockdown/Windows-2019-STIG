# Changes to WIN19STIG

## Based on STIG v3.9.0 - ALD Windows Alignment Updates

### Breaking changes

- BREAKING: **the variable prefixes are standardized on `win19stig_`, and 45 names change.** Measured
  against the previous public release: 37 `wn19stig_<name>` become `win19stig_<name>`, 4
  `win2019stig_<name>` become `win19stig_<name>`, the three category switches
  `win2019stig_cat<N>_patch` become `win19stig_cat<N>_controls`, and
  `wn19stig_machineaccountpsswd_max_age` becomes `win19stig_machineaccountpassword_max_age`. An
  override left under an old name is no longer read and is silently ignored. Rule toggles are
  unchanged and keep the `wn19_<control id>` form.
- BREAKING: **four variables are removed.** `win19stig_cloud_based_system`, along with the cloud
  detection it gated, which could not distinguish Azure from on-premises Hyper-V.
  `win2019stig_min_ansible_version`, now `min_ansible_version` in `meta/main.yml` and raised from
  `2.10.1` to `2.16.1`. And the rule toggles `wn19_00_000290` and `wn19_cc_000451`, whose controls
  DISA retired and which appear nowhere in the V3R9 benchmark.

### Feature removals gated on the feature existing

- FIXED: **eight feature removal controls aborted the play on a host that does not ship the feature.**
  `win_feature` raises "The role, role service, or feature name is not valid" for a name the OS does
  not know, which ends the run on a control whose requirement is already trivially met. The affected
  controls are `WN19-00-000320` (Fax), `-000330` (Web-Ftp-Server), `-000340` (PNRP), `-000350`
  (Simple-TCPIP), `-000360` (Telnet-Client), `-000370` (TFTP-Client), `-000380` (FS-SMB1) and
  `-000410` (PowerShell-V2). `prelim.yml` now enumerates the valid feature names once with
  `Get-WindowsFeature` and each control is gated on membership of that list, replacing the
  `prelim_windows_installation_type != 'Server Core'` test carried by two of them, which covered
  only one of the reasons a feature can be absent. Reported by the community as
  [ansible-lockdown/Windows-2019-STIG#57](https://github.com/ansible-lockdown/Windows-2019-STIG/issues/57).

### check_mode safety

- FIXED: **`WN19-00-000150` modified the host during `--check`.** `check_mode: false` makes a task
  run in a check pass, so the remediation step ran `icacls ... /grant` against `C:\`,
  `C:\Program Files` and `C:\Windows` on a run the operator had asked to change nothing. The
  annotation is removed so check mode skips it. Nothing downstream reads the register except the
  task's own `changed_when`, which is not evaluated when a task is skipped, so no guard was needed.
  It was reachable only with `win19stig_auto_remediate_permissions: true`, not on a default run.

### Connection severing controls guarded - carried across from the Windows Fleet

Benchmark: **Windows Server 2019 STIG v3.9.0**.

A Windows Fleet role was run against a live domain joined host in September 2026 and lost the
host twice to controls that remove a local account's network access. Both have direct equivalents
here, and neither was guarded. Neither is reachable by `ansible-lint`, `yamllint` or
`--syntax-check`; both only bite at runtime, against a domain joined member server.

**These guards are ported and statically verified. They have not been run against a Windows Server 2019
host.** Each was host proven on the fleet test host.

- FIXED: **WN19-MS-000020 would sever the control connection.** It writes
  `LocalAccountTokenFilterPolicy=0`, filtering the privileged token of local accounts on network
  logon, so a run connected over WinRM or psrp as a local administrator loses the host as it
  applies. It presents as `the specified credentials were rejected by the server` rather than a
  dropped connection, because the port keeps answering and the service keeps running.

- FIXED: **WN19-MS-000080 would sever both transports at once.** Its member server branch denies
  network logon to `Guests`, `Enterprise Admins`, `Domain Admins`, `Local account` and
  `Local account and member of Administrators group`. That removes the network logon right from
  every local account on the host, including the one the run is connected as. Unlike WN19-MS-000020
  it takes SSH with it, because Windows OpenSSH password authentication uses the same logon path.
  On the fleet test host, after the equivalent control applied, every remote transport failed - WinRM
  password as both a local and a domain admin principal, SSH password, SSH public key, and WinRM
  certificate authentication - while 5985, 5986 and 22 all still accepted TCP. The console was the
  only way back.

  Note the deny list also names the two privileged domain groups, so **a Domain Admin does not
  survive it either**. The principal that does is a domain account that is neither a Domain Admin
  nor an Enterprise Admin, and that is a member of the local Administrators group on the target.

  Both controls now carry `not win_skip_for_test` (this role's default remains `true`) and are
  listed with the other connection severing controls in `defaults/main/main.yml`. Both are **also**
  skipped automatically when the run connects as a local account, whatever `win_skip_for_test` is
  set to, from two new facts in `prelim.yml`. The operator is warned once per control and each
  warning is counted through `warning_facts.yml`.

  The facts are deliberately separate. `prelim_control_account_is_local` is about the principal and
  applies on any transport, and guards WN19-MS-000080. `prelim_local_account_network_logon` adds the
  transport test and guards WN19-MS-000020. Collapsing them would either leave WN19-MS-000080 able to
  sever an SSH run, or stop WN19-MS-000020 applying over SSH where it is safe.

  The guard sits on the **member server branch only**. The non-domain branch of WN19-MS-000080, which
  sets the right to `Guests` alone and cannot sever anything, is untouched and still applies.

- NOTE: the local account detection builds its backslash from a YAML single quoted variable rather
  than a Jinja literal. In a Jinja expression neither a single nor a doubled backslash literal
  matches one backslash in this position - the first is a syntax error, the second tests for two -
  so a `DOMAIN\user` principal went undetected. The YAML form takes the character literally and
  never reaches Jinja's string parser.

- `README.md`: added the `Compliance facts` section, which the Windows Fleet already carried and this role did not. It documents where the facts
  file is written, that Windows collects no local facts by default so `fact_path` must be passed
  explicitly, and that the result appears as `ansible_compliance_facts` rather than under
  `ansible_local`. The Windows Fleet now carries an identical section.

- CHANGED: the compliance facts file is now JSON. `templates/compliance_facts.ps1.j2` is replaced by
  `templates/compliance_facts.json.j2`, and the role writes
  `C:\ProgramData\ansible\facts.d\compliance_facts.json` instead of `...\compliance_facts.ps1`.
  This aligns the ALD Windows roles on a single facts format. The
  file is now parsed rather than executed to produce the fact.

  **The fact name and its keys are unchanged.** `ansible.windows.setup` builds the key from the
  file's `BaseName`, so it is still `ansible_compliance_facts`, and the five existing keys plus the
  conditional `Cat_N_tag_run` entries keep their names and meanings. Anything already reading this
  fact continues to work.

  Every value goes through `| to_json`, so the CAT toggles are now real JSON booleans rather than
  PowerShell `$true` / `$false`. The conditional tag entries use a leading comma, because a trailing
  comma after `cat_3_hardening_enabled` would emit invalid JSON on the default untagged run.

- ADDED: a task removing a superseded `compliance_facts.ps1` before the new file is written. Both
  extensions are collected and both produce the same fact key, so on a host hardened by an earlier
  release the stale file would otherwise be a second source for `ansible_compliance_facts` with no
  guaranteed precedence.

- ADDED: a `managed_by` key. The PowerShell file opened with a "managed by ansible" comment banner
  and JSON cannot hold comments, so that provenance is recorded as a field instead of being lost. Its
  value is derived from `file_managed_by_ansible`, so the wording stays in one place and the variable
  remains in use.

- `win_template` now sets `newline_sequence: "\r\n"`, so the file is written with CRLF line endings.
  **Not executed.** No Windows inventory was available, so the file is never actually collected by
  `ansible.windows.setup` here. The reasoning about fact naming comes from reading the module source.
  The rendered output was validated offline against all eight CAT tag combinations.

- `README.md`: the `Community Contribution` section still described the old open-contribution model -
  "We encourage you (the community) to contribute to this role" and "All community Pull Requests are
  pulled into the devel branch". That contradicted both `CONTRIBUTING.md`, which states pull requests
  come from approved contributors, and this README's own `Contributing` section a few screens above
  it. Replaced with the wording the Windows Fleet already carried: pull requests from
  approved contributors, issues welcome from everyone, and a pointer to `CONTRIBUTING.md` for
  onboarding and the commit signing requirements. The Windows Fleet now carries an identical
  section.

- `README.md`: removed four controller-side dependencies the role does not use. It declared
  `passlib`, `python-lxml`, `python-xmltodict` and `python-jmespath`, and a paragraph describing an
  OpenSCAP tool installation. This role calls only `ansible.windows`, `community.windows` and
  `ansible.builtin` modules: there is no `password_hash` filter, no `xml` module, no `json_query`
  filter and no OpenSCAP task anywhere in it. `pywinrm` is retained and now carries the note, since
  it is the controller-side connection library. The only XML in the role is PowerShell
  `Get-AppLockerPolicy -Effective -XML` running on the target, which implies nothing about
  controller packages.
- `README.md`: repaired a corrupted Technical Dependencies entry. The Ansible requirement line had
  been deleted leaving its tail fused to the line above, which read "Windows 2019 - Other versions
  are not supported newer)". The OS line is restored and the Ansible requirement re-added, stating
  the floor the role asserts in `tasks/main.yml`. The `Local Testing` block claiming
  `ansible-core 2.17.x` is removed: it disagreed with `meta/main.yml`, which declares `2.16.1`.

- `README.md`: the `Release Tag` and `Closed Issues` badge URLs used a redundant double ampersand
  (`&&color=success`). Corrected to a single `&`, matching the Windows Fleet. Every role now carries an identical badge block.

### Account policy scope on domain joined hosts

- ADDED: **the 13 secedit backed `[System Access]` controls are skipped on a domain joined host.**
  On such a host the Default Domain Policy owns that section and overwrites it at every policy
  refresh, so a local write reverts within roughly two hours while the run still reports success.
  An audit taken straight afterwards therefore reports a compliance that does not last. A new
  `discovered_account_policy_is_domain_scoped` fact gates the account lockout trio and the password
  policy controls, and the operator is warned once with a pointer to set them in the Default Domain
  Policy instead.

  This is the failure class behind the `The key 'LockoutDuration' in section 'System Access' is not
  a valid key` reports from domain joined hosts. Not reachable by `ansible-lint`, `yamllint` or
  `--syntax-check`: the controls are individually valid and only fail against a joined host.

- ADDED: `prelim.yml` gathers the `windows_domain` fact subset explicitly when
  `ansible_facts['windows_domain_member']` is not already defined, rather than assuming a prior
  gather supplied it. `discovered_domain_joined` is seeded `false` in `vars/main.yml`, so a host
  whose membership cannot be determined still applies the controls rather than silently skipping
  them.

- REMOVED: `tasks/cat2_cloud_lockout_order.yml`, the conditional import that loaded it, and the
  `win19stig_cloud_based_system` and `win19stig_cloud_vendors` variables. The three account lockout
  controls were implemented twice, here and in that file, selected by a cloud detection fact.
  Both copies had already converged on the same keys in the same order once the account
  lockout ordering above was corrected, so the duplicate carried nothing the standard path
  did not.
  Cloud detection was also unreliable by construction: Azure and on-prem Hyper-V both report
  `Microsoft Corporation` with model `Virtual Machine`, so the two could not be told apart by the
  facts the detection used. Any inventory setting either variable can drop it; neither is read any
  more.


- `README.md`: added a `Domain Members` section documenting that account policy is domain scoped,
  which controls are skipped on a domain joined host, and the instruction to set them in the
  Default Domain Policy instead.

### Repository hygiene

- FIXED: **Tofu Destroy did not run when `ENABLE_DEBUG` was unset.** The teardown step was gated on
  `env.ENABLE_DEBUG == 'false'`, which is false for an unset or empty variable, so the Azure test
  instance was left running. Now gated on `!= 'true'`, which tears down by default and keeps the
  instance only when debugging is explicitly requested.
- FIXED: **the IAC_BRANCH test was a shell syntax error when the variable was unset.**
  `if [ ${{ vars.IAC_BRANCH }} != '' ]` expands to `if [ != '' ]` with no value. Replaced with
  `if [ -n "${{ vars.IAC_BRANCH }}" ]`.
- FIXED: **the debug step echoed an undefined variable.** `$benchmark_type` is never set; the
  environment carries `TF_VAR_benchmark_type`. Corrected, so `DEBUG - Show IaC files` reports the
  benchmark type instead of an empty string.
- ADDED: explicit least-privilege `permissions:` blocks on both jobs, rather than inheriting the
  default token scope, and `workflow_dispatch` so either pipeline can be run manually.
- ADDED: `issue_message` to the pinned `actions/first-interaction@v3.1.0` step. The v3.1.0 runtime
  calls `getInput` for it with `required: true` even though its own `action.yml` does not mark it
  required, so the welcome job failed without it.
- `README.md`: rewrote the `Pipeline Testing` section. The previous text described an audit-on-devel
  pipeline this role does not run, and claimed collections are resolved from the requirements file,
  which no pipeline step does. It now records the ansible-core floor, the Azure target and its
  teardown, the branches each pipeline gates on, and the fork restriction on the job holding the
  cloud credentials. The two pipeline-status badges are dropped, matching the rest of the Windows
  Fleet. `Local Testing` is unchanged and still accurate.

### Account lockout ordering corrected

- FIXED: **the standalone account lockout path could not complete.** `tasks/Cat2/WN19-AC-xxxxxx.yml`
  applied `WN19-AC-000030` (`ResetLockoutCount`) before `WN19-AC-000010` (`LockoutDuration`).
  Setting `LockoutBadCount` causes Windows to default both of the others to 10, and Windows enforces
  `ResetLockoutCount <= LockoutDuration`, so writing a reset counter of 15 against a duration still
  at 10 is rejected. The order is now `000020` -> `000010` -> `000030`, matching the order the
  Windows Server 2022 and 2025 roles already use and the order this role's own
  `cat2_cloud_lockout_order.yml` used. Only the cloud path was correct before this change, so a
  standalone host took the broken ordering.

  Not reachable by `ansible-lint`, `yamllint` or `--syntax-check`: the tasks are individually valid
  and only the sequence is wrong, which bites at runtime against a real host.

  This reorder is statically verified against the constraint documented on the 2022 and 2025 roles,
  where `Duration=10/Reset=15 REJECTED` and `Duration=15/Reset=15 ACCEPTED` were observed on a live
  host. It has not been re-run against a Windows Server 2019 host.

- The unresolved comment above `WN19-AC-000010`, asking whether a custom failure message should be
  raised for `The key 'LockoutDuration' in section 'System Access' is not a valid key`, is removed.
  The ordering above is what prevents that error.

### Legal banner title corrected

- FIXED: **`WN19-SO-000140` wrote the entire consent banner into the dialog box title.**
  `win19stig_legalnoticecaption` defaulted to `{{ win19stig_legalnoticetext }}`, so the
  `LegalNoticeCaption` registry value received the full multi-paragraph banner rather than a title.
  The benchmark requires `DoD Notice and Consent Banner`, `US Department of Defense Warning
  Statement`, or an organization-defined equivalent; the banner text itself belongs to
  `WN19-SO-000130` and `LegalNoticeText`. The default is now the literal title. Raised by `rlmass`
  as a community contribution.

### Remediation defects fixed - Windows 10 parity sweep

Found while auditing the Windows Fleet; these were present here too.

- FIXED: three `AUDIT`-labelled tasks wrote to the registry. `--tags audit` is a real selection
  mechanism, so they handed registry writes to an operator who asked for a read-only pass. The two
  `WN19-CC-000110` tasks and `WN19-CC-000320` are now `PATCH`. Tasks that only emit a warning or
  import `warning_facts.yml` were left as `AUDIT`, because they write nothing.
- FIXED: the `WN19-00-000090` TPM check carried three defects. Its warning fired on the *compliant*
  TPM state, because the conditions tested for the healthy values being present rather than absent.
  Every assertion read `stderr_lines`, where `wmic` writes nothing but its "No Instance(s)"
  diagnostic, so they could essentially never match and the inversion was masked. And the
  SpecVersion clause read `'SpecVersion=2' or 'SpecVersion=1.2' in ...`, where the bare non-empty
  string on the left of the `or` is always truthy, making that disjunct unconditionally true. The
  probe now uses `Get-CimInstance` rather than `wmic`, which is deprecated and is a
  Feature-on-Demand that can be absent, leaving the register with no `stdout`/`stderr` keys at all.
  Adopted from the Windows Fleet, where it had already been corrected.
- FIXED: `.gitignore` ignored `.github/` wholesale, which meant every new workflow file had to be
  force-added and any that was not would be silently untracked. Replaced with the narrow
  runtime-checkout patterns the rest of the Windows fleet uses, which carry a comment warning
  against the wholesale form.


Benchmark alignment to Windows Server 2019 STIG V3R9 (benchmark date 01 Jul 2026), regenerated
from `U_MS_Windows_Server_2019_STIG_V3R9_Manual-xccdf.xml`. V3R8 -> V3R9 is updates-only: 282 rules
in both releases, 0 added, 0 removed, 0 severity changes, 0 title changes, 2 rule revisions.

- CHANGED: `WN19-00-000100` retagged `SV-205849r991589_rule` -> `SV-205849r1210247_rule`, and its
  CCI moved from `CCI-000366` to `CCI-003376`. The NIST tag follows the CCI and is now
  `NIST800-53R4_SA-22`, from the DISA CCI list mapping of `CCI-003376` to NIST SP 800-53 Revision 4
  `SA-22 a`. The SRG is unchanged and no check or fix logic changed
- CHANGED: `WN19-CC-000280` retagged `SV-205797r1186384_rule` -> `SV-205797r1210306_rule`. Rule
  revision only; the SRG, CCI, check and fix are unchanged
- `README.md` banner now points at the V3R9 benchmark and its download URL

- replaced `CONTRIBUTING.rst` with `CONTRIBUTING.md`, carrying the current Ansible-Lockdown
  contributing guide. The Windows Fleet now ships a byte-identical file
- `README.md`: added a Contributing section pointing at `CONTRIBUTING.md`, normalized the social
  badge to the `X URL` form on `x.com`, and pointed the Discord link at
  `https://www.lockdownenterprise.com/discord`
- `README.md`: aligned the shared heading text and the benchmark banner format with the rest of the
  Windows fleet. Role-specific sections are unchanged
- `defaults/main.yml` moved to `defaults/main/main.yml`, matching the directory layout the Linux
  roles have adopted. Ansible loads role defaults from the directory, so no task or template change
  was needed and no variable name or value changed. Verified by loading the defaults tree through
  Ansible's role-defaults loader
- CHANGED: `benchmark_version` is now expressed numerically, `v3r8` becomes `v3.9.0`. This is the
  value `templates/compliance_facts.json.j2` renders into the local fact `Benchmark_release`, so that
  fact changes from `STIG-v3r8` to `STIG-v3.9.0` on the next run

## 2026 August - Contributing guide and README refresh

- `README.md`: removed the decorative emoji from headings, and switched the social badge from
  `twitter.com` to `x.com`

## Based on STIG v3.8.0 - August 2026

The 2026 August modernization, applied to the V3R8 benchmark line.

The role structure, variables, workflows and CI gate in this release were adopted wholesale from
the V3R9 line, so every breaking change documented under `## Release 4.0.0` below now applies to
V3R8 consumers as well. Read that section before upgrading. The benchmark content itself remains
V3R8; only the two rules DISA changed between V3R8 and V3R9 were retargeted.

Benchmark alignment, every identifier regenerated from
`U_MS_Windows_Server_2019_STIG_V3R8_Manual-xccdf.xml` (Release 8, benchmark date 01 Apr 2026,
282 rules) rather than inherited from the V3R9 line:
 - CHANGED: `WN19-00-000100` retagged to `SV-205849r991589_rule` and `CCI-000366`. V3R9 moved this
   rule to Sunset status, renumbered it and swapped the CCI to `CCI-003376`. The role's audit block
   already asserts `CurrentBuildNumber >= 17763` and `ReleaseId >= 1809`, which is the V3R8 wording
   literally, so no logic changed.
 - CHANGED: `WN19-CC-000280` retagged to `SV-205797r1186384_rule`. Rule number only; V3R9 changed
   nothing in the check or fix text.
 - ADDED: the rev-5 CCI that V3R8 lists alongside the legacy CCI for thirteen controls, which had
   been carrying only the legacy value: `CCI-004066` on `WN19-00-000020`, `WN19-00-000210`,
   `WN19-AC-000050`, `WN19-AC-000060`, `WN19-AC-000070`, `WN19-AC-000080` and `WN19-00-000050`;
   `CCI-004062` on `WN19-AC-000090` and `WN19-SO-000300`; `CCI-003980` on `WN19-CC-000420` and
   `WN19-CC-000430`; `CCI-003627` on `WN19-00-000190`; `CCI-004923` on `WN19-00-000440`.
 - FIXED: `WN19-DC-000401` was tagged `SRG-OS-000327-GPOS-00127`. V3R8 assigns it
   `SRG-OS-000080-GPOS-00048`. Both SRGs exist in V3R8 on other rules, so this was the wrong one
   selected rather than an unknown identifier.
 - CHANGED: `benchmark_version` is now `v3r8`, and the README banner points at the V3R8 benchmark.

Coverage against V3R8 is exact both ways: 282 benchmark rules, 282 implemented, none missing and
none implemented that V3R8 does not list.

Changelog history restored:
 - The `## April 28 2026` and `## Release 5.0.0` sections below are specific to this branch and
   were absent from the V3R9 line. They are restored here rather than lost.
 - Note that two different releases carry the heading `## Release 4.0.0`. The section immediately
   below is the V3R9 line's August 2026 modernization. This branch previously used that same
   heading for a much older entry, preserved here in full so it is not lost:
   `- Update tasks, tags and controls based on STIG 2 Version 8 Release.`
 - The `## Release 4.0.0` section below states that `WN19-00-000290` was withdrawn in V3R9. It was
   withdrawn in V3R8; the V3R8 Revision History records it under "Removed unnecessary requirement",
   and the control is absent from both the V3R8 and V3R9 benchmarks. The outcome in the role is
   correct either way, only the attribution was wrong.

## Based on STIG v3.9.0 - Release 4.0.0 - August 2026

 - ADDED: `skip_os_check`, defaulting to false. Setting it true bypasses the OS version and
   family assert for hosts the check misidentifies.
 - ADDED: `create_benchmark_facts` and `ansible_facts_path`. The role now writes
   `compliance_facts.ps1` under `C:\ProgramData\ansible\facts.d`, recording the benchmark
   release, the run date and which CAT levels were enabled. Windows has no default local-facts
   directory, so it is collected only when `ansible.windows.setup` is given a matching
   `fact_path`; it then surfaces as `ansible_compliance_facts`.
 - ADDED: `company_title` and `file_managed_by_ansible` in `vars/main.yml`, the provenance
   strings the rest of the fleet uses to head managed files.
 - BREAKING: `win19stig_min_ansible_version` is renamed to `min_ansible_version` and moved from
   `defaults/main.yml` to `vars/main.yml`, matching the rest of the fleet. Rename it if you
   override it; the old name is no longer read.
 - FIXED: every operational switch used in a `when:` or a template is now filtered through
   `| bool`. Supplied as an extra var (`-e skip_reboot=false`) a switch arrives as the string
   "false", which is truthy, so a negated conditional evaluated to the opposite of what was
   asked and a non-negated one failed outright on ansible-core 2.19+.
 - CHANGED: `ChangeLog.md` renamed to `CHANGELOG.md` to match the rest of the fleet.
 - FIXED: WN19-MS-000140 wrote `RequirePlatformSecurityFeatures`, which belongs to
   WN19-CC-000110. MS-000140 set it to 1 in CAT I and CC-000110 then set it from
   `win19stig_dma_protection` (default 3) in CAT II, so on every run of a domain member both
   tasks reported changed and the tunable was silently overridden. The control's check text
   requires only `LsaCfgFlags`, so the colliding value has been removed from its loop.
 - BREAKING: `win19stig_machineaccountpsswd_max_age` is renamed to
   `win19stig_machineaccountpassword_max_age`. Rename it if you override it.
 - FIXED: `meta/main.yml` gave the Galaxy platform version as the integer `2019` instead of
   a string, which ansible-lint rejected as a schema error.
 - FIXED: cloud detection in `tasks/prelim.yml` classified almost every host as cloud-based. The
   condition was `not virtualization_type == 'VMware' or (system_vendor == 'Microsoft Corporation'
   and virtualization_type in ['Hyper-V', 'hvm', 'kvm'])`. Any value in that list already satisfies
   `!= 'VMware'`, so the right-hand side is a strict subset of the left and the whole expression
   reduces to `virtualization_type != 'VMware'`. Checked over every combination of realistic fact
   values: there is no input for which the `system_vendor` test changes the outcome. The practical
   effect was that bare metal (`NA`), VirtualBox, Xen, on-prem Hyper-V and local KVM were all marked
   cloud-based; only a literal `VMware` was not. Confirmed against a local QEMU/KVM Windows Server
   2022 guest, which reports `system_vendor` `QEMU` and `virtualization_type` `kvm` and evaluated
   true under the old condition.
    - the fact selects which order the three AC lockout controls are applied in - WN19-AC-000010,
      WN19-AC-000020 and WN19-AC-000030 - so a wrong classification means the wrong ordering, and
      those controls fail if applied out of order. The two paths are mutually exclusive on the same
      boolean, so there was no double-apply or coverage gap, and `defaults/main.yml` already
      defaults the variable to false, so no host could hit an undefined variable.
    - replaced with a two-item AND: `system_vendor` must be in the new `win19stig_cloud_vendors`
      list and `virtualization_type` must be one of `Hyper-V`, `kvm`, `xen`. `system_vendor` is the
      discriminator, not the type: `ansible.windows` derives `virtualization_type` from the
      Manufacturer, so Amazon EC2, Google and DigitalOcean all report `kvm` - and so does a local
      QEMU box. Only the vendor separates them.
    - dropped `hvm` from the list. `ansible.windows` never emits it on Windows; it is a Linux-only
      value, and the comment above the task cited a Linux fact-collection source as its reference.
      Added `xen`, which older AWS instances report through model `HVM domU`, together with the
      matching `Xen` vendor so that entry can actually fire.
    - `win19stig_cloud_vendors` is a variable so an unlisted cloud can be added without editing the
      task. Scaleway and Nutanix also map to `kvm` and are deliberately omitted. Azure and on-prem
      Hyper-V both report `Microsoft Corporation` with model `Virtual Machine`, so they cannot be
      told apart by these facts; set `win19stig_cloud_based_system` explicitly where that matters.
 - FIXED: two play-aborting defects reachable through supported toggle settings, neither visible to
   ansible-lint or yamllint. Both were found by an adversarial review pass, which is the point worth
   recording: clean linters are not a correct role.
  - `tasks/prelim.yml`: `Get Drive Letters` is gated on `wn19_00_000240 or wn19_au_000060 or
    wn19_00_000390 or wn19_00_000400`, but the two SecGuide lookups that expand
    `prelim_drive_letters.stdout_lines` were gated on a wider set that also included
    `wn19_cc_000020` and `wn19_ms_000020`. Enabling only one of those two left the register holding a
    skipped-task dict and the run died with `object of type 'dict' has no attribute 'stdout_lines'`.
    The wider gate was also wrong on its own terms: nothing consumes those two registers for
    `WN19-CC-000020` or `WN19-MS-000020` - the only readers are `WN19-00-000390` and
    `WN19-00-000400` - so both lookups are now gated on exactly those two controls, which the
    register's own gate already covers. This also stops two `win_find` sweeps of every fixed drive
    running for controls that never look at the result.
  - `tasks/Cat2/WN19-UC-xxxxxx.yml`: `WN19-UC-000010` loops over `prelim_hku_loaded_list` but was
    gated only on `wn19_uc_000010`. That fact is set inside a block gated on
    `win19stig_hku_based_controls`, a documented default whose only consumer is this control, so
    setting it false gave `'prelim_hku_loaded_list' is undefined`. The toggle is now part of the
    task's `when:`.
  - Verified by reproducing both aborts and then confirming the fixed form skips instead. Worth
    knowing for anyone reviewing this class of bug: a task-level `when:` evaluating false does prevent
    the loop expression from being templated, so gating the consumer is a sufficient fix. That was
    tested rather than assumed, because the per-item `when:` behavior is often described the other
    way round.
 - Corrected four CCI tags that named a CCI V3R9 does not list for that rule, each checked against
   `U_MS_Windows_Server_2019_STIG_V3R9_Manual-xccdf.xml`: `WN19-00-000100` `CCI-000366` to
   `CCI-003376`; `WN19-AC-000040` `CCI-000200` to `CCI-004061`; `WN19-DC-000020` drops the stale
   `CCI-001941`, keeping `CCI-001942`; `WN19-DC-000310` drops `CCI-000765` and `CCI-000766`, keeping
   `CCI-000767`, `CCI-000768` and `CCI-001948`. Benchmark coverage is unchanged at 282/282.
  - Still open: 13 controls carry only the legacy CCI where V3R9 adds a rev-5 CCI (mostly
    `CCI-004066`, `CCI-004062`, `CCI-003980`), and at least one SRG tag does not match its V3R9 Group
    title (`WN19-DC-000401`). Those are additive changes and are deliberately not bundled here.
 - Fixed a malformed task name in `tasks/prelim.yml` that ended in a stray `"`, and gave it the
   `PRELIM | ` prefix the rest of the file uses. Five more malformed names remain, all invisible to
   both linters because the rule that would catch them only inspects names whose first character is
   not a quote.
  - Pinned to checker `2.8.3`, `ansible-lint==26.8.0` and `yamllint==1.38.0`. There is no
    `.pre-commit-config.yaml` in this role for those pins to agree with; 26.4.0 and 26.8.0 were both
    measured and return an identical finding set, so the pin is a stability choice.
  - `--strict` is load-bearing here: every finding this role produces is WARN severity, so without
    it a regression would exit 0 and pass silently.
  - Verified green by running the workflow's exact command locally, and verified non-vacuous by
    injecting a bad register name, which the gate rejected with exit 1.
 - Cleared the lint backlog: ansible-lint 22 findings to 3, yamllint 8 to 0, measured identically on
   26.4.0 and 26.8.0.
  - `community.windows.win_audit_policy_system` in WN19-AU-000200 replaced with
    `ansible.windows.win_audit_policy_system`. `ansible-doc` reports the former deprecated with
    removal scheduled for `community.windows` 4.0.0, so the task would have broken on that major.
    `community.windows` remains a dependency for the 22 `win_security_policy` uses, which carries no
    deprecation notice.
  - `meta/main.yml`: `collections` and `dependencies` were nested inside `galaxy_info`, where the
    Galaxy schema allows neither. Both moved to the top level. Worth knowing that `schema[meta]` stops
    processing the file at the first error, so this was masking a second one.
  - WN19-PK-000010: four registers renamed from `discovered_pk_000010_root_N_Check` to `_check`,
    clearing `var-naming[pattern]`. Twelve references in total, four definitions and eight uses,
    including the four chained in the combined condition.
  - WN19-00-000020: six task names carried `{{ win19stig_pass_age_administrator }}` mid-name, which
    `name[template]` rejects because a template must end the name. They now use the XCCDF title
    verbatim, "must be changed at least every 60 days", which matches the variable's own default and
    the convention every other task in the role already follows - no other task interpolates a
    variable into its name.
  - Nine comments reindented to match the content they precede, clearing all eight
    `yaml[comments-indentation]` findings.
 - Comment coherence pass over `defaults/main.yml` and `vars/main.yml`:
  - `defaults/main.yml`: the EventLog Application comment named `win19stig_app_maxsize`, which does
    not exist; the variable it documents is `win19stig_application_event_log_max_size`. The EventLog
    Security comment embedded its value (`: 196608`) where its two siblings give the name only, so
    the value could drift from the line below it. All three now read alike.
  - `vars/main.yml`: three comments corrected. Two were grammatically broken, one describing
    `change_requires_reboot` and one describing `win19stig_default_expected_permissions`, each
    carrying a doubled verb that made the sentence unparseable. The reboot comment now states what
    actually happens: the `Change_requires_reboot` handler sets it when notified and `tasks/post.yml`
    either reboots or warns depending on `skip_reboot`.
  - Checked and deliberately left alone: the `# CAT 1/2/3 rules` section headers and the
    `# WINRM CONTROL` / `# WINRM CONTROL END` bracketing markers, which are consistent visual
    structure rather than prose; and the 275 rule toggles with no comment, which are self-documenting
    through names that match their control IDs.
 - Removed the `gpo` galaxy tag. The role no longer creates GPOs. `grouppolicy` is kept, since the
   role still configures policy settings through the registry.
 - FIXED: WN19-00-000420 and WN19-00-000430 aborted the play on any host without the FTP feature.
   Both set a fact with `stdout_lines | regex_search('Installed')` and then gated three tasks on
   `'Installed' in <fact>`. `regex_search` returns the matched text when it matches and `None` when
   it does not, so on a host without `Web-Ftp-Service` - the common case - the fact was `None` and
   the `when:` failed with `argument of type 'NoneType' is not a container or iterable`. Both facts
   now hold a real boolean, from a substring test against `.stdout` defaulted to an empty string, and
   the five consuming `when:` clauses test the boolean directly. These are AUDIT-only controls that
   should warn, never halt. The bug was invisible on an FTP-enabled test host, because that is the
   one case that worked.
 - Replaced the remaining `regex_search` filter used as a truth test in `tasks/prelim.yml` with a
   plain substring test, which returns a boolean rather than relying on `when:` treating `None` as
   falsy, and dropped a capture group that captured nothing. Behavior is unchanged: `Primary domain
   controller` and `Backup domain controller` match, and the other four documented
   `windows_domain_role` values do not. `regex_search` no longer appears anywhere in the role.
 - All three literal string checks in the role now read the same way, as `'<literal>' in <value>`:
   the two FTP audits, the domain controller role, and the release year in the OS gate below. None of
   the three patterns contains a regex metacharacter, so no regex engine is involved in any of them.
 - Reworked the OS version and family check to test the installation type alongside the release
   year, replacing a regular expression over the whole distribution caption:
  - `ansible_facts['os_installation_type']` must be `Server` or `Server Core`, which rejects
    client SKUs. This is load-bearing rather than cosmetic: Windows 10 reports build 10.0.17763
    for the 1809 release, the same build as Server 2019, and the `Windows 10 Enterprise 2019
    LTSC` caption even carries the year, so neither the build nor the year can tell client from
    server on its own. `Server Core` is accepted deliberately - the role supports Core
    installations and `win19stig_increase_scheduling_priority_users` branches on exactly that
    value, so asserting `Server` alone would have rejected every Core host before any control ran.
  - The release year is matched as a substring of the caption rather than by regular expression,
    which tolerates the edition suffixes (`Datacenter`, `Standard`, `Standard Evaluation`)
    without enumerating them. A build-number range was considered and rejected: the retired
    Server semi-annual-channel releases (1903 through 20H2, builds 18362 to 19042) sit between
    2019's 10.0.17763 and 2022's 10.0.20348 and would have been admitted by any such range.
  - `distribution` is gathered only for an administrative connection, so it is defaulted rather
    than left to raise an undefined error that would take the failure message down with it.
  - Messages now report the full `distribution_version` rather than
    `distribution_major_version`, which on Windows is only `10`, plus the detected installation
    type.
 - Removed the `win_reg_stat` and `set_fact` pair that read `InstallationType` from
   `HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion`. `ansible_facts['os_installation_type']`
   is populated from that same registry value and belongs to the `distribution` fact subset the
   role already gathers, so the read was a second round-trip for a value already in hand.
   `prelim_windows_installation_type` is kept under its own name, now aliasing the fact, because
   `defaults/main.yml` consumes it by name. The fact gather is also now conditioned on
   `os_installation_type` as well as `distribution`, so a prior partial gather cannot skip it and
   leave the assert reading an undefined fact.
 - Restructured the task files to the standard Ansible-Lockdown layout. tasks/Cat1, tasks/Cat2
   and tasks/Cat3 each hold one file per control ID band, named
   WN19-AU-xxxxxx.yml and so on, with a main.yml in each directory listing its imports. The
   single 5600 line CAT 2 file is now ten files, the largest of which is 1265 lines.
  - The ansible_hardening directory has gone. It existed only to separate remediation from GPO
    creation, which was removed earlier in this release, and no role in the fleet has a
    directory between tasks and the CAT directories. prelim and post now sit at tasks root.
  - BREAKING: the CAT group tags are now CAT1, CAT2 and CAT3 with high, medium and low
    aliases, matching the fleet. The previous cat1controls, cat2controls and cat3controls no
    longer select anything. The per control CAT1, CAT2 and CAT3 tags were already present on
    every task, so tag based selection by severity is unchanged.
  - Ordering is preserved exactly. WN19-AC-000020, WN19-AC-000030 and WN19-AC-000010 must run
    in that order on non-cloud systems, and the cloud variant must load before them; both are
    intact. ansible-playbook --list-tasks produces an identical 531 task sequence before and
    after the split.
 - Reindented meta/main to two spaces and corrected the roles list indentation in site.yml.
 - .yamllint now ignores the ansible-lint collection cache, which is not repository content.
 - Single-entry when and tags are now written inline rather than as one-item lists, matching
   the fleet. 50 occurrences, verified to produce an identical task list.
 - Reindented every task file to two spaces, matching the rest of the Ansible-Lockdown roles
   and the repo's own .yamllint. The role previously indented four and six spaces against a
   configuration demanding two, which was the source of all 1120 YAML Lint findings. That count
   is now zero.
   The reindent was done with a comment-preserving YAML round trip, and every file was checked
   by parsing it before and after and comparing the result. All nine are byte-for-byte
   equivalent in meaning, and ansible-playbook --list-tasks produces an identical task list.
 - Standardized where the manual-task warning identifier lives. vars: warn_control_id now sits
   on the task that imports warning_facts, not on the enclosing block, matching the rest of the
   Ansible-Lockdown roles. 92 controls moved; 3 were already correct.
 - The warning identifier now carries the severity, for example 'HIGH | WN19-00-000010' rather
   than 'WN19-00-000010'. The end of run warning summary shows severity without a lookup,
   matching the fleet format.
 - Fixed WN19-DC-000401, a manual task that never reached the warning summary. It carried a
   warn_control_id but had no warning_facts import to consume it, so the control was never
   counted. The import has been added.
 - Rebranded the parent company from Tyto Athene to Quantum Sky in LICENSE, meta/main and the
   banner template. The banner carried the name in upper case, which a case-sensitive search
   for the old name misses.
 - Removed the emoji from the README headings and converted the unicode arrows in the NIST
   reference table to ASCII. The README is now plain ASCII, matching the public mirror.
 - BREAKING: standardized the role behavior variables and security tunables on the win19stig_
   prefix. If you override any security tunable, rename it or your setting will be ignored. See the Role Variables section
   of the README for the mapping.
  - wn19stig_<name> becomes win19stig_<name>, 39 variables.
  - win2019stig_<name> becomes win19stig_<name>, 5 variables. This also makes the README
    correct: it documented win19stig_disruption_high, which until now did not exist, so anyone
    following it had disruptive remediation silently disabled.
  - Rule toggles are unchanged and keep the wn19_<control id> form, for example wn19_au_000010.
 - Task registers now use the fleet's discovered_ prefix, keeping the control ID, for example
   discovered_00_000100_currentbuildnumber. Discovery facts use prelim_ without a role prefix.
 - Fixed WN19-00-000150, which referenced win19stig_auto_remediate_permissions in four when
   clauses without the variable being defined anywhere. The task errored on every run. The
   variable is now declared in defaults and defaults to false.
 - Fixed WN19-00-000100, whose OS Build task read the CurrentBuild registry value but asserted
   against the previous task's CurrentBuildNumber register, so the value it read was never
   checked.
 - WN19-AC-000080 now honours win19stig_passwordcomplexityvalue instead of hardcoding 1. The
   removed GPO path used the variable; the remediation path had been ignoring it.
 - Removed a dead duplicate of win19stig_cloud_based_system from vars, and a prelim task that
   registered a value nothing read while claiming to detect TPM availability when it actually
   queried the operating system product type.
 - Removed the GPO creation path. The role is now remediation only.
  - Deleted tasks/gpo_creation (11 files), tasks/domain_creation, templates/stig_templates and
    templates/windows_templates.
  - Deleted the two GPO pipeline workflows and their README badges, the Custom-Built STIG GPO
    Creation section, and the Notes section, whose entire content explained ADMX and ADML
    limitations in Group Policy.
  - Removed the playbook-style selector and the assert that kept the two paths mutually
    exclusive, along with the fifteen GPO variables in defaults and the nine GPO name variables
    in vars.
  - Control coverage is unchanged at 282 of 282. The GPO path implemented a 200 control subset
    of what the remediation path already covers, so nothing was lost with it. The public mirror
    has never carried a GPO path, so this also brings the two into line.
  - Two of the removed variables were already inoperative: win19stig_backup_custom_gpos and
    win19stig_backup_location_style were documented in defaults and in the README, including a
    backup path, but no task ever read them and no backup was ever taken.
 - Updated tasks, tags and controls based on STIG Version 3 Release 9, benchmark date
   01 July 2026. Coverage is 282 of 282 rules.
 - Added Controls
  - Added Control WN19-AU-000581, audit file system failures
  - Added Control WN19-AU-000582, audit file system successes
  - Added Control WN19-AU-000583, audit handle manipulation failures
  - Added Control WN19-AU-000584, audit handle manipulation successes
  - Added Control WN19-AU-000585, audit registry failures
  - Added Control WN19-AU-000586, audit registry successes
  - Added Control WN19-AU-000587, audit sensitive privilege use successes
  - Added Control WN19-AU-000588, audit sensitive privilege use failures
 - Removed Controls
  - Removed Control WN19-00-000290, withdrawn in V3R9
 - Note that WN19-AU-000587 and WN19-AU-000588 configure the same Sensitive Privilege Use
   subcategory as the existing WN19-AU-000300 and WN19-AU-000310. Both pairs are present in
   V3R9 as separate rule IDs, so both are implemented.
 - Updated the README benchmark banner and win19stig_gpo_hardening_version to V3R9. The GPO
   names built from that variable were still being created as V3R3.
 - Aligned all rule identifiers with STIG V3R9. 56 SV-IDs were carrying superseded revision
   numbers and were bumped to their V3R9 values.
  - WN19-UR-000080 was tagged SV-205756r958726_rule, which belongs to WN19-UR-000090. Its
    own V-205755 was already correct. Corrected to SV-205755r958726_rule.
  - WN19-SO-000370 in the GPO path was tagged V-205871, which belongs to WN19-CC-000320.
    Corrected to V-205923, matching the remediation path and SV-205923r991589_rule.
  - The consolidated GPO task covering WN19-AU-000190 and WN19-AU-000200 was missing the
    WN19-AU-000200 tag, so selecting that control by tag skipped the GPO work. Tag added.
 - Corrected rule identifiers and one severity against STIG V3R9.
  - WN19-00-000150 was tagged SV-253274r1016661_rule and V-253274, neither of which exists
    in V3R9. Corrected to SV-205735r958702_rule and V-205735.
  - WN19-MS-000070 was tagged SV-205672r958472_rule, which belongs to WN19-MS-000080. Its
    own V-205671 was already correct. Corrected to SV-205671r1137691_rule.
  - WN19-00-000250 is CAT1 in V3R9 but was tagged CAT2 and named MEDIUM. Retagged and
    renamed. The task was already in the CAT1 task file.
 - Fixed a runtime failure in WN19-00-000150. Its when: clause referenced WN19_00_000150 and
   WN19_SO_000240 in uppercase. Ansible variable names are case sensitive and the toggles are
   defined lowercase, so both were undefined and the task errored on every run.
 - Fixed a mislabelled subtask inside the WN19-00-000110 block, which was named
   WN19-00-000120. That is a separate control covering host-based intrusion detection and is
   implemented elsewhere. Its register was also renamed from win19_00_000120_av_sftw_status
   to discovered_00_000110_av_sftw_status, correcting both the control number and the prefix
   typo.
 - Completed the ansible_facts migration. Seven more reads of windows_domain_role,
   virtualization_type and system_vendor in prelim and cat2 were still using the bare magic
   variable form.
 - Raised min_ansible_version to 2.16.1 in defaults, meta and the assert in tasks/main, which
   must agree.
 - Prepared the role for current ansible-core templating.
  - default('') becomes default('', true) at 160 sites. default('') substitutes only when a
    name is undefined, not when it is None, which is the case current core surfaces. All but
    one are the GPO guid expressions; the other reads a registry value that can be null.
  - The distribution and OS family facts in tasks/main are now read through ansible_facts,
    matching the rest of the Ansible-Lockdown roles.
 - Excluded the binary PolicyDefinitions.zip from ansible-lint, which tried to decode it as
   UTF-8 and warned on every run.
 - Fixed WN19-00-000150, whose two PowerShell blocks could not be parsed by ansible-core. The
   free form module argument is run through Ansible's argument splitter, which treats the
   backslash in "C:\" as escaping the closing quote and reports unbalanced quotes. The whole of
   the CAT 2 task file failed to load as a result, so none of its controls could run. Both blocks
   now use the documented cmd option instead of the free form parameter. The role passes
   ansible-playbook --syntax-check again.
 - Modernized the pipeline workflows against the current Ansible-Lockdown standard.
  - Added least-privilege permissions blocks. Neither job declared any, so both ran with the
    default token scope. The welcome job now takes issues and pull-requests write; the job
    holding the cloud credentials takes id-token write with contents and pull-requests read.
  - Added workflow_dispatch so the pipelines can be run manually.
  - Moved actions/checkout from v4 to v7.
  - Aligned the devel branch filter to benchmark*, matching the rest of the fleet.
 - Hardened the pipeline workflows.
  - The Azure build jobs now require the pull request to originate from a branch in this
    repository. pull_request_target exposes repository secrets to the job, and the job checks
    out pull request head and executes it, so a fork pull request must never reach it.
  - Pinned actions/first-interaction to v3.1.0 rather than the mutable main ref, and
    corrected its input names from repo-token and pr-message to repo_token and pr_message.
    The action renamed them after v1, so the welcome comment had stopped being posted.
  - The IAC_BRANCH test is now quoted. Unquoted, an unset variable made the test a bash syntax
    error rather than falling through to the default.
  - Tofu Destroy now runs unless ENABLE_DEBUG is exactly true. Previously it required the value
    to be exactly false, so an unset variable would have left the Azure system running.
  - The debug step printed an unset benchmark_type; it now prints TF_VAR_benchmark_type.
 - Pipeline triggers corrected for this repository's branch layout.
  - Devel and GPO Devel validation now run on pull requests targeting benchmark_* branches.
  - Main and GPO Main validation now run on pull requests targeting the latest branch.
 - Removed update_galaxy.yml, which triggered on a main branch that does not exist here.
 - Added benchmark and benchmark_version metadata to defaults/main, so the benchmark revision this
   role targets is recorded in the role itself.
 - Removed parseable from .ansible-lint. Current ansible-lint rejects the whole config file
   for that key, so linting could not run against this role at all. The rest of the fleet
   dropped it already.
 - Pinned collections/requirements.yml to explicit collection versions.
 - Removed community.general from collections/requirements.yml and meta/main, as no task
   in this role calls it.
 - Aligned meta/main author and company with the Ansible-Lockdown convention and removed a
   duplicate windows2019 galaxy tag.
 - Corrected the LICENSE copyright year and the company name spelling.
 - Corrected a warning message in WN19-SO-000030 that told users to edit
   win19stig_admin_username. No such variable exists; the variable is
   win19stig_newadministratorname.
 - Removed the orphan toggle for WN19-CC-000451 from defaults. It gated no task and
   WN19-CC-000451 is not present in V3R9.
 - Aligned .gitignore with the rest of the Ansible-Lockdown roles. This adds the secret and
   key file block, the Lockdown artifact patterns and the ansible-lint cache, none of which
   were present. Added Windows entries, sensitive_info.json as written by the pipeline, and
   the templates/GPOBackups directory the role writes when backing up to the Ansible host.

### Also carried in this release, previously unreleased

These two sections were numbered 5.0.0 and 4.0.0 in the private role but no public release was
ever cut for them, so their content reaches consumers for the first time here.

#### V3R3 / V3R2 (previously 5.0.0)

February 2025
 - Update tasks, tags and controls based on STIG 3 Version 3 Release.
 - Added Controls
  - Added Control WN19-DC-000391
  - Added Control WN19-DC-000401

December 2024
 - Update tasks, tags and controls based on STIG 3 Version 2 Release.
 - Task restructure and GPO Creation Feature enhancement
 - Ansible Linting and typo fixes

#### V2R8 (previously 4.0.0)

 - Update tasks, tags and controls based on STIG 2 Version 8 Release.

## Based on STIG v3.8.0 - April 28 2026
- Started change to changelog headers from a release version to a date. 
- Updates for Version 3 Release 8 updates. 
  - Updates to tasks, tags, and controls based on STIG Version 3 Release 8
  - Added controls 
    - WN19-AU-000581
    - WN19-AU-000582
    - WN19-AU-000583
    - WN19-AU-000584
    - WN19-AU-000585
    - WN19-AU-000586
    - WN19-AU-000587
    - WN19-AU-000588
  - Removed controls
   - WN19-00-000290

## Based on STIG v3.3.0 - Release 5.0.0 - February 2025

 - Update tasks, tags and controls based on STIG 3 Version 3 Release.
 - Added Controls
  - Added Control WN19-DC-000391
  - Added Control WN19-DC-000401

December 2024
 - Update tasks, tags and controls based on STIG 3 Version 2 Release.
 - Task restructure and GPO Creation Feature enhancement
 - Ansible Linting and typo fixes

## Based on STIG v2.5.0 - Release 3.0.0 - August 2023

  - Updated Workflows To Centralized Repo and renamed them to better run across all repos.
  - Removed Templates & PR Template from repo and adjusted to Org level.
  - Updated Readme Layout to add new pipeline badges.
  - Fixed WN16 References in defaults/main.
  - Cat2_Cloud moved from tasks/main and renamed to cat2_cloud_lockout_order and in cat2.yml workflow.
  - Updated Tags in tasks/main.

June / July 2023 Update
  - Updated tags on controls based on Version 2 Release 7
  - Updated win_skip_for_test var to true
  - Updated Readme
  - Updated Changelog
  - Added Version 2 Release 6 changes to this update.
    - Removed Control WN19-AU-000150
    - Removed Control WN19-DC-000270
  - Added Version 2 Release 7 changes to this update.
  - Added Warning To Control WN19-PK-000010
  - Added Controls
    - Added Control WN19-AU-000100
    - Added Control WN19-AU-000110
    - Added Control WN19-SO-000070
  - Rule IDs updated due to changes in the content management system.
  - Updated all tags to the proper format.
  - Major updates to all controls
    - Updated Misc Controls to have additional data in warning messages.
    - Fixed Misc Controls Registry entry errors.
    - Updated controls to allow for additional variables to be used

April 2023 Update
  - Updated the testing pipelines
  - Updated Controls For Cloud Systems
  - Updated skip variable for testing controls

March 2023 Update
  - Updated yamllint
  - Updated Ansible Lint
  - Updated Workflows
  - Fixed Linting
  - Added Skip for WinRm breaking changes for Ansible.
  - Updated Full Module Names
  - Removed All Asserts
  - Added AUDIT | PATCH to names lines.
  - Removed Variable For WN19-AC-000040 the DoD has decided 24 is the appropriate value.

January 2023 Release
  - Added Changelog.md
  - Updated Readme
  - Added Version 2 Release 3 changes during this update.
  - Added Version 2 Release 4 changes during this update.
  - Added Version 2 Release 5 changes during this update.
  - Added Warning Count Summary to the End Of Playbook.

## Release 2.1.0

April 2021 Update
  - Fixed issue #12
  - Updated README.md
  - Updated CONTRIBUTING.rst

## Based on STIG v2.1.0 - Release 2.0.0 - November 2020

  - Update to DISA STIG Version 2 Release 1

## Release 1.0.0

December 2020 Update
  - Initial Release
