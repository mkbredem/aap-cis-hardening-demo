# CIS Hardening with Red Hat Ansible Automation Platform

Demo content for applying the **CIS Red Hat Enterprise Linux 9 Benchmark v2.0.0, Level 1 - Server** with Ansible Automation Platform 2.7, and keeping hosts compliant afterward.

The same sequence works in any environment:

1. **Assess** - scan with OpenSCAP to get a starting score (`cis_scan.yml`).
2. **Tailor** - record approved exceptions as variables in Git (`inventory/group_vars/`).
3. **Remediate** - run in check mode, approve, harden a canary ring, rescan, then harden the next ring (`cis_harden.yml`).
4. **Enforce** - a nightly check-mode schedule reports drift, and Event-Driven Ansible reverts SSH drift within a minute (`rulebooks/cis_drift.yml`).
5. **Prove** - before and after OpenSCAP HTML reports for auditors.

## Content source

The CIS rules come from [`RedHatOfficial.rhel9_cis_server_l1`](https://github.com/RedHatOfficial/ansible-role-rhel9-cis_server_l1), which is generated from [ComplianceAsCode](https://github.com/ComplianceAsCode/content). The OpenSCAP `cis_server_l1` profile in the RHEL `scap-security-guide` package is generated from the same rules, so the rescan score reflects exactly what the role changed. The role is pinned in `roles/requirements.yml`.

Each CIS rule is switched on or off by a variable with the rule's name, for example `sshd_disable_root_login: true`. `docs/cis_rule_map.csv` maps every CIS control number to its rule variable, for teams that track exceptions by CIS number.

## Repository layout

| Path | Purpose |
|---|---|
| `cis_scan.yml` | OpenSCAP scan, HTML report published to `report.shadowman.dev/openscap/<host>/cis-<phase>/`, score saved as a job artifact |
| `cis_harden.yml` | Applies the CIS role. Run as job type Check to report, Run to change |
| `cis_introduce_drift.yml` | Demo helper that sets `PermitRootLogin yes` |
| `inventory/hosts.yml` | Demo hosts in two rollout rings: `cis_canary` and `cis_wave1` |
| `inventory/group_vars/cis_rhel9.yml` | Approved exceptions and organization values, each with reason and change ticket |
| `rulebooks/cis_drift.yml` | Event-Driven Ansible rulebook: Auditbeat file change on `/etc/ssh/sshd_config` starts the enforcement workflow |
| `roles/requirements.yml`, `collections/requirements.yml` | Installed by the AAP project sync |
| `docs/cis_rule_map.csv` | CIS control number to rule variable map (generated from ComplianceAsCode `products/rhel9/controls/cis_rhel9.yml`) |

## AAP objects (organization "mbredeme - Default")

| Object | Type | Uses |
|---|---|---|
| mbredeme - CIS Hardening | Project | this repository |
| mbredeme - CIS Demo | Inventory, with source "mbredeme - CIS Demo (Git)" | `inventory/hosts.yml` |
| mbredeme - CIS Scan (OpenSCAP) | Job template | `cis_scan.yml` |
| mbredeme - CIS Check (Check Mode) | Job template, job type Check, diff on | `cis_harden.yml` |
| mbredeme - CIS Remediate | Job template, diff on | `cis_harden.yml` |
| mbredeme - CIS Demo Introduce Drift | Job template | `cis_introduce_drift.yml` |
| mbredeme - CIS Harden RHEL | Workflow | scan, check, approve, canary, rescan, approve, wave 1, scan |
| mbredeme - CIS Enforce Drift | Workflow | remediate and rescan one host |
| mbredeme - CIS Nightly Drift Check | Schedule | Check job template, daily 2:00 AM ET |
| mbredeme - CIS Drift Enforcement | Rulebook activation | `rulebooks/cis_drift.yml` |

## Running one CIS section: 5.1 Configure SSH Server

The role tags every task with its rule name. To apply only CIS section 5.1, set Job Tags to:

```
configure_custom_crypto_policy_cis,disable_host_auth,file_groupowner_sshd_config,file_groupownership_sshd_private_key,file_groupownership_sshd_pub_key,file_owner_sshd_config,file_ownership_sshd_private_key,file_ownership_sshd_pub_key,file_permissions_sshd_config,file_permissions_sshd_private_key,file_permissions_sshd_pub_key,sshd_disable_empty_passwords,sshd_disable_rhosts,sshd_disable_root_login,sshd_do_not_permit_user_env,sshd_enable_pam,sshd_enable_warning_banner_net,sshd_set_idle_timeout,sshd_set_keepalive,sshd_set_login_grace_time,sshd_set_loglevel_verbose,sshd_set_max_auth_tries,sshd_set_max_sessions,sshd_set_maxstartups
```

## Warnings

- CIS Level 1 changes SSH, PAM, sudo, firewall, audit and mount settings. Run check mode first and harden a canary ring before any other hosts.
- Some rules change kernel or boot settings that only take effect after a reboot.
- Red Hat does not provide an automated way to revert these changes. Take a snapshot first on hosts that matter.
