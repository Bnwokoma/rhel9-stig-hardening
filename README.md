RHEL 9 STIG IPv4 Hardening (Manual & Automated with Ansible)
============================================================

📌 Overview
-----------

This project demonstrates how I manually validated and remediated IPv4-related STIG controls on RHEL 9, then automated enforcement using Ansible. I used DISA STIG benchmark Version 2, Release 3 (Benchmark Date: 2025-01-30) to guide compliance hardening and implemented persistent changes via sysctl. The final result is a repeatable, scalable security baseline applied to multipleremote RHEL hosts.

* * * * *

Technologies Used
--------------------

-   Red Hat Enterprise Linux 9 & Rocky (Ansible control node)

-   Ansible

-   Stig Viewer

-   DISA STIG v2r3 (2025-01-30)

-   `sysctl` + `/etc/sysctl.d` for runtime and persistent tuning


* * * * *

Phase 1: Manual STIG Application & Validation
------------------------------------------------

To validate compliance manually, I ran the following:

```
sudo /usr/lib/systemd/systemd-sysctl --cat-config | grep net.ipv4
```

This revealed several non-compliant values on my local system, including:

-   `net.ipv4.conf.default.send_redirects = 1` (expected: `0`)

-   `net.ipv4.conf.all.send_redirects = 1` (expected: `0`)

-   `net.ipv4.icmp_ignore_bogus_error_responses = 0` (expected: `1`)

### Manual Remediation

[Before applying the changes, you can view this pie chart to see the state of compliance:](/screenshots/before_remediation)

I created the following file to apply persistent sysctl changes:

```
sudo vim /etc/sysctl.d/stig-ipv4.conf
```

[See full contents of file here:](/screenshots/sysctl_config_file)

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

[Confirmed Compliance:](/screenshots/compliance) 
```sudo /usr/lib/systemd/systemd-sysctl --cat-config | grep net.ipv4
```


* * * * *

🤖 Phase 2: STIG Automation with Ansible
----------------------------------------

### 📁 group_vars/rhel.yml

I wanted to create a seperate file to store the variables. I like to clean my playbooks clean and readable.
```
stig_ipv4_settings:
  - { name: "net.ipv4.tcp_syncookies", value: "1" }
  - { name: "net.ipv4.conf.all.accept_redirects", value: "0" }
  - { name: "net.ipv4.conf.all.accept_source_route", value: "0" }
  - { name: "net.ipv4.conf.default.log_martians", value: "1" }
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

### 📜 playbooks/ipv4_hardening.yml

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
ansible-playbook -i inventory playbooks/ipv4_hardening.yml
```

[Playbook Results:] (/screenshots/running_pb)


[Pie chart after remediation:] (/screenshots/after_remediation)
* * * * *

### Verifying Compliance on Remote Hosts

After applying STIG IPv4 hardening, I created a separate Ansible playbook to verify compliance across all remote hosts. This playbook:

- Checks the current sysctl values for each STIG-related setting

- Compares them to expected DISA STIG values

- Generates a readable Markdown compliance report on each host using a Jinja2 template


#### The playbooks:


- [ipv4_hardening.yml](playbooks/ipv4_hardening.yml) – Applies all IPv4 STIG sysctl settings
- [verify_ipv4_stigs.yml](playbooks/verify_ipv4_stigs.yml) – Verifies compliance
- [jinja2 template] (playbooks/templates/ipv4_stig_report.j2)


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
