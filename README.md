RHEL 9 STIG IPv4 Hardening (Manual & Automated with Ansible)
============================================================

📌 Overview
-----------

This project demonstrates how I manually validated and remediated IPv4-related STIG controls on RHEL 9, then automated enforcement using Ansible. I used DISA STIG benchmark Version 2, Release 3 (Benchmark Date: 2025-01-30) to guide compliance hardening and implemented persistent changes via sysctl. The final result is a repeatable, scalable security baseline applied to multiple remote RHEL hosts.

* * * * *

Technologies Used
--------------------

-   Red Hat Enterprise Linux 9 & Rocky (Ansible control node)

-   Ansible

-   Stig Viewer

-   Jinja

-   DISA STIG v2r3 (2025-01-30)


IPv4 STIGs Applied
---------------------

| STIG ID | sysctl Setting | Compliant Value | Description |
| --- | --- | --- | --- |
| RHEL-09-253010 | `net.ipv4.tcp_syncookies` | `1` | Enables TCP syncookies to protect against SYN floods |
| RHEL-09-253015 | `net.ipv4.conf.all.accept_redirects` | `0` | Disables acceptance of ICMP redirects |
| RHEL-09-253020 | `net.ipv4.conf.all.accept_source_route` | `0` | Disables source-routed packet forwarding |
| RHEL-09-253025 | `net.ipv4.conf.all.log_martians` | `1` | Enables logging of suspicious packets (martians) |
| RHEL-09-253030 | `net.ipv4.conf.default.log_martians` | `1` | Enables martian logging on default interfaces |
| RHEL-09-253035 | `net.ipv4.conf.all.rp_filter` | `1` | Enables reverse path filtering on all interfaces |
| RHEL-09-253040 | `net.ipv4.conf.default.accept_redirects` | `0` | Disables ICMP redirect acceptance on default interfaces |
| RHEL-09-253045 | `net.ipv4.conf.default.accept_source_route` | `0` | Disables source-routing on default interfaces |
| RHEL-09-253050 | `net.ipv4.conf.default.rp_filter` | `1` | Enables reverse path filtering by default |
| RHEL-09-253055 | `net.ipv4.icmp_echo_ignore_broadcasts` | `1` | Ignores ICMP echo requests to broadcast addresses |
| RHEL-09-253060 | `net.ipv4.icmp_ignore_bogus_error_responses` | `1` | Ignores bogus ICMP error messages |
| RHEL-09-253065 | `net.ipv4.conf.all.send_redirects` | `0` | Disables sending of ICMP redirects (all interfaces) |
| RHEL-09-253070 | `net.ipv4.conf.default.send_redirects` | `0` | Disables ICMP redirects on default interfaces |
| RHEL-09-253075 | `net.ipv4.conf.all.forwarding` | `0` | Disables IP forwarding unless acting as a router |


* * * * *

Phase 1: Manual STIG Application & Validation
------------------------------------------------

To validate compliance manually, I ran the following:

```
sudo /usr/lib/systemd/systemd-sysctl --cat-config | grep net.ipv4
```

This revealed several non-compliant values on my local system, including:

-   `net.ipv4.conf.default.send_redirects = 1` (expected: 0)

-   `net.ipv4.conf.all.send_redirects = 1` (expected: 0)

-   `net.ipv4.icmp_ignore_bogus_error_responses = 0` (expected: 1)

### Manual Remediation

Before applying the changes, you can view this pie chart to see the state of compliance: [Assessment results summary before remediation](/screenshots/before_remediation.png)

I created the following file to apply persistent sysctl changes:

```
sudo vim /etc/sysctl.d/stig-ipv4.conf
```

See full contents of file here: [System configurations](/screenshots/sysctl_config_file.png)

Example contents:

```
# RHEL-09-253010: Enable TCP syncookies
net.ipv4.tcp_syncookies = 1

# RHEL-09-253065: Disable sending of ICMP redirects
net.ipv4.conf.all.send_redirects = 0

# RHEL-09-253075: Disable IPv4 forwarding unless acting as a router
net.ipv4.conf.all.forwarding = 0
```

To apply the settings:

```
sudo sysctl --system
```

* * * * *

Phase 2: STIG Automation with Ansible
----------------------------------------

I wanted to create a seperate file to store the variables. I like to keep my playbooks clean and readable.

[📁 group_vars/rhel.yml](/group_vars/rhel.yml)

```
stig_ipv4_settings:
  - { name: "net.ipv4.tcp_syncookies", value: "1" }                      
  - { name: "net.ipv4.conf.all.accept_redirects", value: "0" }          
  - { name: "net.ipv4.conf.all.accept_source_route", value: "0" }
  - { name: "net.ipv4.conf.default.log_martians", value: "1" }
  - { name: "net.ipv4.conf.all.log_martians", value: "1"}
  - { name: "net.ipv4.conf.all.rp_filter", value: "1" }
  - { name: "net.ipv4.conf.default.rp_filter", value: "1" }
  - { name: "net.ipv4.conf.default.accept_redirects", value: "0" }
  - { name: "net.ipv4.conf.default.accept_source_route", value: "0" }
  - { name: "net.ipv4.icmp_echo_ignore_broadcasts", value: "1" }
  - { name: "net.ipv4.icmp_ignore_bogus_error_responses", value: "1" }
  - { name: "net.ipv4.conf.all.send_redirects", value: "0" }
  - { name: "net.ipv4.conf.default.send_redirects", value: "0" }
  - { name: "net.ipv4.conf.all.forwarding", value: "0" }

```

### 📜 [playbooks/ipv4_hardening.yml](/playbooks/ipv4_hardening.yml)

```
---
- name: Enforce RHEL 9 STIG IPv4 settings
  hosts: rhel
  become: true

  tasks:
    - name: Apply IPv4 STIG sysctl settings
      ansible.posix.sysctl:
        name: "{{ item.name }}"
        value: "{{ item.value }}"
        state: present
        reload: true
      loop: "{{ stig_ipv4_settings }}"

    - name: Persist settings to /etc/sysctl.d/stig-ipv4.conf
      copy:
        dest: /etc/sysctl.d/stig-ipv4.conf
        content: |
          {% for setting in stig_ipv4_settings %}
          {{ setting.name }} = {{ setting.value }}
          {% endfor %}
        owner: root
        group: root
        mode: '0644'

    - name: Reload sysctl settings
      command: sysctl --system
```

### Running the Playbook

```
ansible-playbook playbooks/ipv4_hardening.yml
```

[Playbook Results](/screenshots/running_pb.png)


[Assessment results summary after remediation](/screenshots/after_remediation.png)
* * * * *

### Verifying Compliance on Remote Hosts
[Results of playbook](screenshots/verify_pb.png)

After applying STIG IPv4 hardening, I created a separate Ansible playbook to verify compliance across all remote hosts. This playbook:

- Checks the current sysctl values for each STIG-related setting

- Compares them to expected DISA STIG values

- Generates a readable Markdown compliance report on each host using a Jinja2 template

I ran the playbook twice, once showing what would happen if a remote host is not compliant and vice versa.
- [Screenshot with a non-compliant STIG:](/screenshots/verify_compliance_red.png)

- [Screenshot with 100% compliant STIGs:](/screenshots/verify_compliance_green.png)

#### The playbooks:


- [ipv4_hardening.yml](playbooks/ipv4_hardening.yml) – Applies all IPv4 STIG sysctl settings
- [verify_ipv4_stigs.yml](playbooks/verify_ipv4_stigs.yml) – Verifies compliance
- [jinja2 template](playbooks/templates/ipv4_stig_report.j2)


```
IPv4 STIG Compliance Report - {{ ansible_date_time.date }}

Host: {{ inventory_hostname }}
====================================================================

{% for result in sysctl_check.results %}
STIG: {{ result.item.name }}
Expected: {{ result.item.value }}
Actual:   {{ result.stdout }}
Status:   {% if result.stdout == result.item.value %}✅ COMPLIANT{% else %}❌ NON-COMPLIANT{% endif %}
--------------------------------------------------------------------
{% endfor %}


```


Summary
---------

-   Validated 15 IPv4 STIGs manually, confirmed 13 non-compliant

-   Remediated settings manually, then automated using Ansible

-   Used `sysctl --system` to enforce and verify changes

-   Created a repeatable automation approach for multi-host enforcement

This project demonstrates real-world Linux hardening for enterprise environments and showcases both manual security auditing and Infrastructure-as-Code principles.
