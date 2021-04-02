Windows Server 2019 DISA STIG
=========
![Build Status](https://img.shields.io/github/workflow/status/ansible-lockdown/Windows-2019-STIG/CommunityToDevel?label=Devel%20Build%20Status&style=plastic)
![Build Status](https://img.shields.io/github/workflow/status/ansible-lockdown/Windows-2019-STIG/DevelToMain?label=Main%20Build%20Status&style=plastic)
![Release](https://img.shields.io/github/v/release/ansible-lockdown/Windows-2019-STIG?style=plastic)

Configure a Windows Server 2019 system to be DISA STIG compliant. All findings will be audited by default. Non-disruptive CAT I, CAT II, and CAT III findings will be corrected by default.

This role is based on Windows Server 2019 DISA STIG: [Version 2, Rel 1 released on November 13, 2020](https://dl.dod.cyber.mil/wp-content/uploads/stigs/zip/U_MS_Windows_Server_2019_V2R1_STIG.zip).

Caution(s)
-------
This role **will make changes to the system** that could break things. This is not an auditing tool but rather a remediation tool to be used after an audit has been conducted.

This role was developed against a clean install of the Operating System. If you are implimenting to an existing system please review this role for any site specific changes that are needed.

To use release version please point to main branch
Based on [Windows Server 2019 DISA STIG](https://dl.dod.cyber.mil/wp-content/uploads/stigs/zip/U_MS_Windows_Server_2019_V2R1_STIG.zip).

Documentation
-------------
[Getting Started](https://www.lockdownenterprise.com/docs/getting-started-with-lockdown)<br>
[Customizing Roles](https://www.lockdownenterprise.com/docs/customizing-lockdown-enterprise)<br>
[Per-Host Configuration](https://www.lockdownenterprise.com/docs/per-host-lockdown-enterprise-configuration)<br>
[Getting the Most Out of the Role](https://www.lockdownenterprise.com/docs/get-the-most-out-of-lockdown-enterprise)<br>
[Wiki](https://github.com/ansible-lockdown/Windows-2019-STIG/wiki)<br>
[Repo GitHub Page](https://ansible-lockdown.github.io/Windows-2019-STIG/)<br>

Requirements
------------
**General:**
- Basic knowledge of Ansible, below are some links to the Ansible documentation to help get started if you are unfamiliar with Ansible
  - [Main Ansible documentation page](https://docs.ansible.com)
  - [Ansible Getting Started](https://docs.ansible.com/ansible/latest/user_guide/intro_getting_started.html)
  - [Tower User Guide](https://docs.ansible.com/ansible-tower/latest/html/userguide/index.html)
  - [Ansible Community Info](https://docs.ansible.com/ansible/latest/community/index.html)
- Functioning Ansible and/or Tower Installed, configured, and running. This includes all of the base Ansible/Tower configurations, needed packages installed, and infrastructure setup. 
- Please read through the tasks in this role to gain an understanding of what each control is doing. Some of the tasks are disruptive and can have unintended consiquences in a live production system. Also familiarize yourself with the variables in the defaults/main.yml file or the [Main Variables Wiki Page](https://github.com/ansible-lockdown/Windows-2019-STIG/wiki/Main-Variables).

**Technical Dependencies:**
- Running Ansible/Tower setup (this role is tested against Ansible version 2.9.1 and newer)

The following packages must be installed on the controlling host/host where ansible is executed:

- passlib (or python2-passlib, if using python2)
- python-lxml
- python-xmltodict
- python-jmespath
- pywinrm

Package 'python-xmltodict' is required if you enable the OpenSCAP tool installation and run a report. Packages python(2)-passlib and python-jmespath are required for tasks with custom filters or modules. These are all required on the controller host that executes Ansible.

Role Variables
--------------
This role is designed that the end user should not have to edit the tasks themselves. All customizing should be done via the defaults/main.yml file or with extra vars within the project, job, workflow, etc. These variables can be found [here](https://github.com/ansible-lockdown/Windows-2019-STIG/wiki/Main-Variables) in the Main Variables Wiki page. All variables are listed there along with descriptions.

Branches
--------
- **devel** - This is the default branch and the working development branch. Community pull requests will pull into this branch
- **main** - This is the release branch
- **reports** - This is a protected branch for our scoring reports, no code should ever go here
- **gh-pages** - This is the github pages branch
- **all other branches** - Individual community member branches

Community Contribution
----------------------

We encourage you (the community) to contribute to this role. Please read the rules below.

- Your work is done in your own individual branch. Make sure to Signed-off and GPG sign all commits you intend to merge.
- All community Pull Requests are pulled into the devel branch
- Pull Requests into devel will confirm your commits have a GPG signature, Signed-off, and a functional test before being approved
- Once your changes are merged and a more detailed review is complete, an authorized member will merge your changes into the main branch for a new release
