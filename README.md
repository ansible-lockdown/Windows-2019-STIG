Windows Server 2019 DISA STIG
=========

Configure a Windows Server 2019 system to be DISA STIG compliant. All findings will be audited by default. Non-disruptive CAT I, CAT II, and CAT III findings will be corrected by default. ~Disruptive finding remediation can be enabled by setting `rhel7stig_disruption_high` to `yes`.~ _To be implemented_

This role is based on Windows Server 2019 DISA STIG: [Version 1, Rel 9 released on July 26, 2019](Need URL HEre).

Requirements
------------

Windows Server 2019 - Other versions are not supported.

Dependencies
------------

The following packages must be installed on the controlling host/host where ansible is executed:

- passlib (or python2-passlib, if using python2)
- python-lxml
- python-xmltodict
- python-jmespath
- pywinrm

Package 'python-xmltodict' is required if you enable the OpenSCAP tool installation and run a report. Packages python(2)-passlib and python-jmespath are required for tasks with custom filters or modules. These are all required on the controller host that executes Ansible.

Role Variables
--------------

Please see the Ansible docs for understanding [variable precedence](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable) to tailor for your needs. 

| Name                     | Default Value       | Description                   |
|--------------------------|-----------------------------------------------------|----------------------|
| `win2019stig_cat1_patch` | `yes` see defaults/main.yml](./defaults/main.yml)   | Correct CAT I findings        |
| `win2019stig_cat2_patch` | `yes`  see defaults/main.yml](./defaults/main.yml)  | Correct CAT II findings       |
| `win2019stig_cat3_patch` | `yes`  see defaults/main.yml](./defaults/main.yml)  | Correct CAT III findings      |
| `wn19_##_######`         | [see defaults/main.yml](./defaults/main.yml)        | Individual variables to enable/disable each STIG ID. |

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - role: win-2k16-stig
           when:
                - ansible_os_family == 'Windows'
                - ansible_distribution | regex_search('(Server 2019)')
