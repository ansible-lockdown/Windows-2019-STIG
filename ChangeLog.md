# Changelog

## Release 3.0.0

August 2026 Update
 - Removed update_galaxy.yml. The workflow has never run, and the Galaxy entry has not been
   updated since October 2023 at v3.0.0, so it was not the mechanism keeping Galaxy current.
 - Aligned the pipeline workflows with the Ansible-Lockdown Windows standard.
  - The devel pipeline now also runs for pull requests into benchmark* branches, and the main
    pipeline for pull requests into latest, so both branch models are covered.
  - Added workflow_dispatch so the pipelines can be run manually.
  - Added least-privilege permissions blocks. Neither job declared any, so both ran with the
    default token scope.
  - Moved actions/checkout to v7.
 - Hardened the pipeline workflows.
  - The playbook-test job now requires the pull request to originate from a branch in this
    repository. pull_request_target grants that job the repository secrets, and its steps check
    out pull request head and run it, so a pull request from a fork must never reach them.
  - Pinned every action to a released version rather than a mutable branch ref:
    actions/first-interaction to v3.1.0, arillso/action.playbook to 0.5.0,
    robertdebock/galaxy-action to 1.2.1.
  - arillso/action.playbook was referenced as @master. That repository has no master branch,
    its default is main, so the reference could not resolve and the playbook step could not
    run. Pinning to 0.5.0 also repairs that.
  - Corrected the first-interaction input names from repo-token and pr-message to repo_token
    and pr_message. The action renamed them after v1, so the welcome comment had stopped
    being posted.
  - Moved actions/checkout from the deprecated v3 to v4.

June 2024 Update
 - update WN19-SO-000140 var with title fix - Thank you @rlmass

August 2023 Update
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

## Release 2.0.0

November 2020 Update
  - Update to DISA STIG Version 2 Release 1

## Release 1.0.0

December 2020 Update
  - Initial Release
