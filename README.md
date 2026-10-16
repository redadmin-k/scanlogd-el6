# scanlogd for Enterprise Linux 6

Unofficial RPM packaging of [Openwall scanlogd](https://www.openwall.com/scanlogd/) 2.2.8 for **Red Hat Enterprise Linux 6** and **CentOS Linux 6**, using a traditional **SysV init** script and `chkconfig` instead of systemd.

This repository is an independent packaging project and is **not an official Openwall release**. The EL6 build and runtime status must be established independently; successful tests on EL7 or EL9 do not establish EL6 compatibility.

> **End-of-life warning:** CentOS Linux 6 stopped receiving updates on **November 30, 2020**. RHEL 6 passed its standard maintenance period on the same date, and its standard Extended Life Cycle Support (ELS) ended on **June 30, 2024**. Any specialized support arrangement must be checked with Red Hat. This repository is intended for compatibility testing, migration, or explicitly managed legacy environments. **Building an RPM does not make an unsupported operating system secure.**

## Overview

scanlogd detects certain TCP port scans and writes a record through syslog. On Linux, its native packet input uses a TCP raw socket and does not require libpcap. The daemon **does not block traffic by itself**. Optional Fail2Ban integration can interpret scanlogd's messages and block source IP addresses through iptables.

It does not replace a default-deny firewall, a WAF, an IDS/IPS, operating-system security updates, or continuous monitoring.

## Packaging changes compared with upstream

The Openwall source includes a SysV init script with Owl-specific startup logic. This EL6 adaptation uses a CentOS/RHEL 6-compatible SysV init script:

- Installs `/etc/rc.d/init.d/scanlogd` via the RPM `%{_initrddir}` macro.
- Registers the service with `chkconfig` and leaves it **disabled by default**.
- Uses legacy RPM install, erase, and upgrade scriptlets instead of systemd macros.
- Builds the upstream Linux raw-socket implementation, without libpcap.
- Preserves scanlogd's internal startup privilege drop to the dedicated `scanlogd` account.
- Installs a root-owned, empty chroot directory at `/var/lib/scanlogd/empty`; the path compiled into `params.h` must match it.
- Includes the upstream manual, documentation, and license.

**EL6 compatibility notes:** RHEL/CentOS 6 uses older RPM, GCC, glibc, init scripts, and user-space tools. The EL6 SPEC deliberately avoids `%autosetup`, `%make_build`, `%{build_ldflags}`, `systemd-rpm-macros`, and `%systemd_*` scriptlets. The Linux `make` target and the startup sequence still require actual EL6 testing.

Do **not** start scanlogd as the unprivileged `scanlogd` user in the init script: it needs to open its raw socket first, after which it drops privileges internally.

## Repository layout

```text
scanlogd-el6/
├── README.md
├── SPECS/
│   └── scanlogd.spec
└── SOURCES/
    └── scanlogd.init
```

`RPMS/`, `SRPMS/`, and `Logs/` or a local `build/` directory can hold generated build artifacts. Download the original `scanlogd-2.2.8.tar.gz` from Openwall rather than silently modifying or republishing it. The source RPM must contain all files referenced by its SPEC.

## Build an EL6 RPM with Mock

An EL8/EL9 build host can be used, **if** its Mock release still supports initializing an EL6 buildroot and you have a working EL6 config with archived repositories. Modern Mock versions and EL6-era Yum/RPM components may require additional compatibility work; a successful EL9-host build is not guaranteed.

### 1. Install build tools on the host

On an AlmaLinux 9 build host with appropriate repositories configured:

```bash
dnf install -y mock mock-core-configs rpm-build curl git
```

Run Mock as root or as an appropriately configured member of the `mock` group. Inspect available EL6 configurations:

```bash
ls /etc/mock/*6*x86_64*.cfg
```

Some older Mock installations provided `epel-6-x86_64.cfg`; current installations may not. **`epel-6-x86_64` is only an example**, not a guaranteed configuration. Use the actual name of your EL6 config throughout the commands below.

### 2. Configure archived EL6 repositories

CentOS Linux 6 and EPEL 6 are archived. The original `mirrorlist=` and metalink settings may fail. Example archive roots for **CentOS 6.10 x86_64** are:

```text
https://vault.centos.org/6.10/os/x86_64/
https://vault.centos.org/6.10/updates/x86_64/
https://vault.centos.org/6.10/extras/x86_64/
https://archives.fedoraproject.org/pub/archive/epel/6/x86_64/
```

These are archive *locations*, not a complete Mock configuration. Verify current availability, Yum metadata, signatures, and repository layout before using them. EL6's old TLS libraries may have trouble contacting modern HTTPS endpoints directly. Prefer fetching content through a maintained build host, a vetted local mirror, or a validated proxy rather than weakening security controls on the legacy system. Preserve RPM GPG verification; do **not** set `gpgcheck=0` merely to make a build work.

For RHEL 6, use repository content that your organization is authorized to access rather than assuming CentOS archives are equivalent to entitled Red Hat packages.

Initialize the selected EL6 buildroot:

```bash
mock -r epel-6-x86_64 --init
```

If this fails, resolve the repository, signature, and compatibility problems before trying to build the package.

### 3. Get the packaging files and upstream source

```bash
git clone https://github.com/redadmin-k/scanlogd-el6.git
cd scanlogd-el6
mkdir -p build/SOURCES build/SRPMS build/RPMS
curl -fL https://download.openwall.net/pub/projects/scanlogd/scanlogd-2.2.8.tar.gz \
  -o build/SOURCES/scanlogd-2.2.8.tar.gz
cp SOURCES/scanlogd.init build/SOURCES/scanlogd.init
```

The `cp` command assumes the repository layout shown above. Adapt it if your paths differ. Verify the inputs:

```bash
grep -E '^(Name|Version|Release|Source[0-9]*|BuildRequires|Requires):' SPECS/scanlogd.spec
ls -lh build/SOURCES/
sha256sum build/SOURCES/scanlogd-2.2.8.tar.gz
```

For reference, the published scanlogd 2.2.8 source archive has SHA-256:

```text
0bec45ecbcc8a9a3599cb38e21dcacf639ca3b33bb6973b20261315d065ea158
```

Openwall also publishes a detached signature. Verify the archive with an independently trusted Openwall signing key before distribution:

<https://download.openwall.net/pub/projects/scanlogd/scanlogd-2.2.8.tar.gz.sign>

### 4. Verify RPM macros and build requirements

The provided EL6 SPEC uses the older interfaces:

```spec
%setup -q
make %{?_smp_mflags} linux CFLAGS="-Wall %{optflags}"
%{_initrddir}
```

After initializing the buildroot, inspect its RPM macro values:

```bash
mock -r epel-6-x86_64 --chroot 'rpm --eval "%{?dist} %{_initrddir}"'
```

A conventional EL6 build should resolve `%{?dist}` to `.el6` and `%{_initrddir}` to an EL6 init-script directory such as `/etc/rc.d/init.d`. Confirm that the SPEC uses `chkconfig`, `initscripts`, `shadow-utils`, `gcc`, and `make` as required by the packaging design.

### 5. Build the EL6 source RPM

```bash
mock -r epel-6-x86_64 \
  --resultdir "$PWD/build/SRPMS" \
  --buildsrpm \
  --spec "$PWD/SPECS/scanlogd.spec" \
  --sources "$PWD/build/SOURCES"
```

For `Version: 2.2.8` and `Release: 1%{?dist}`, the expected source RPM name is:

```text
build/SRPMS/scanlogd-2.2.8-1.el6.src.rpm
```

Verify the actual output before continuing; the filename depends on the SPEC and the buildroot's distribution macros.

### 6. Build the binary RPM

```bash
mock -r epel-6-x86_64 \
  --resultdir "$PWD/build/RPMS" \
  --rebuild "$PWD/build/SRPMS/scanlogd-2.2.8-1.el6.src.rpm"
```

Inspect the results:

```bash
ls -lh build/RPMS/
rpm -qpi build/RPMS/scanlogd-*.x86_64.rpm
rpm -qpl build/RPMS/scanlogd-*.x86_64.rpm
```

Read `build/RPMS/build.log` and `build/RPMS/root.log` if the binary build fails. An SRPM build alone does **not** demonstrate that scanlogd compiles or runs on EL6.

## Install and test on an EL6 host

Use a separate, authorized RHEL 6 or CentOS 6 test machine with appropriate network and raw-socket permissions. Do not conduct first-time tests on a critical legacy production host.

On the EL6 test machine, after transferring the built RPM:

```bash
yum localinstall -y ./scanlogd-2.2.8-1.el6.x86_64.rpm
chkconfig --list scanlogd
service scanlogd start
service scanlogd status
ps -ef | grep '[s]canlogd'
```

The package registers the init service but **does not enable it at boot**. After a successful test, enable it if desired:

```bash
chkconfig scanlogd on
chkconfig --list scanlogd
```

Check logging through syslog/rsyslog:

```bash
tail -f /var/log/messages
grep 'scanlogd\[' /var/log/messages | tail -20
```

On a typical EL6 installation, rsyslog writes applicable syslog messages to `/var/log/messages`. Confirm your actual rsyslog configuration rather than assuming that path is always correct.

To restart or stop scanlogd:

```bash
service scanlogd restart
service scanlogd stop
```

**Do not use `systemctl` or `journalctl` on a standard CentOS 6 installation:** its service management is based on SysV init rather than systemd.

## Reproduce TCP port-scan detection

Only scan systems you own or are explicitly authorized to test. Use a separate external scanner, and make sure its IP address is not needed for remote administration. If Fail2Ban is active, the scanner may be blocked automatically.

From the external scanner:

```bash
sudo nmap -n -Pn -sS -T4 --max-retries 0 \
  -p 22,25,53,80,110,143,443,465,587,993,995 <authorized-test-host>
```

On the EL6 target:

```bash
tail -f /var/log/messages | grep --line-buffered 'scanlogd\['
```

Illustrative syslog record (**not an EL6 test result**):

```text
Oct 16 10:41:17 server scanlogd[12345]: 203.0.113.35:39864 to 192.0.2.50 ports 110, 143, 80, 22, 25, 53, ...
```

scanlogd uses port-count, weighted-threshold, and timing rules; a short or narrow probe may not trigger a record. Visibility depends on the network namespace, capture path, and any upstream or host packet filtering. If no event appears, check the daemon, raw-socket access, syslog, packet arrival, and firewall behavior. A controlled packet capture can help confirm that SYN packets reached the test interface.

## Optional: Fail2Ban with iptables

The scanlogd RPM does **not** block IP addresses. This example assumes a separately installed, functional **EL6-compatible** version of Fail2Ban and a working `/var/log/messages` syslog file.

Older Fail2Ban releases differ in their supported configuration directories and options. For widest compatibility, use `/etc/fail2ban/jail.local`, not `jail.d/`, and use `service`, not `systemctl`. Verify the installed version first:

```bash
fail2ban-client --version
ls /etc/fail2ban/action.d/iptables-allports.conf
```

Create `/etc/fail2ban/filter.d/scanlogd.conf`:

```ini
[Definition]
failregex = ^.*scanlogd\[\d+\]:\s+<HOST>(?::\d+)?\s+to\s+.*\s+ports\b
ignoreregex =
```

Add the following section to `/etc/fail2ban/jail.local` (create the file if it does not exist; keep any existing configuration):

```ini
[scanlogd]
enabled = true
filter = scanlogd
logpath = /var/log/messages
maxretry = 1
findtime = 60
bantime = 600
action = iptables-allports[name=scanlogd]
```

The example bans after one scanlogd event for ten minutes. This is for a **controlled test**, not a recommended production policy. Set appropriate `ignoreip` exclusions for administrative and monitoring hosts before enabling automatic bans. The action name and available parameters depend on the installed Fail2Ban release.

Test the filter and restart the service:

```bash
fail2ban-regex /var/log/messages /etc/fail2ban/filter.d/scanlogd.conf
service fail2ban restart
fail2ban-client status
fail2ban-client status scanlogd
```

Check both Fail2Ban state and the actual iptables rules after a permitted scan:

```bash
fail2ban-client status scanlogd
iptables -S | grep -E 'scanlogd|fail2ban'
iptables -nvL --line-numbers
```

Depending on the action shipped with the older Fail2Ban package, the chain may be named `fail2ban-scanlogd` rather than `f2b-scanlogd`. Verify the relevant `INPUT` jump, target rule, and packet counters. A listed banned IP alone does not prove that packets were dropped.

### Security limitations

- **IP spoofing:** scanlogd does not require a completed TCP connection. A forged source address could cause an innocent IP to be banned. Avoid permanent bans based only on scanlogd events.
- **Incomplete coverage:** slow, distributed, or narrowly targeted scans can evade detection.
- **Network visibility:** upstream filtering, capture location, and network namespaces can affect which packets scanlogd observes.
- **False positives:** automatic all-port blocking can interrupt legitimate clients or administrators.
- **Operational cost:** monitor log volume, CPU, memory, and firewall rule counts.
- **Legacy platform risk:** neither scanlogd nor Fail2Ban compensates for unpatched EL6 kernel or user-space vulnerabilities.

## Verification status

As of **2026-10-16**, EL6-specific Mock build and runtime validation have **not been documented** for this repository. Replace these entries only after collecting evidence on an EL6 build and test system.

| Test | EL6 status |
| --- | --- |
| Mock source RPM build | Not yet documented |
| Mock binary RPM build | Not yet documented |
| SysV init service startup | Not yet documented |
| `chkconfig` service registration | Not yet documented |
| TCP raw-socket operation | Not yet documented |
| External TCP scan detection | Not yet documented |
| Fail2Ban automatic ban | Not yet documented |
| iptables packet drop | Not yet documented |
| SELinux enforcing mode | Not yet documented |
| LXC or other container operation | Not yet documented |

Tests performed on AlmaLinux 9 establish **EL9 behavior only** and must not be represented as EL6 results.

## Troubleshooting

- `No such config file`: create or install a usable EL6 Mock config and use its actual name with `-r`.
- EL6 Mock initialization fails: check archive URLs, repository metadata, GPG keys, and compatibility between modern Mock and old Yum/RPM.
- Yum cannot connect over HTTPS: check legacy TLS/certificate capabilities and use a trusted build-host mirror or proxy; do not simply disable signature verification.
- `Bad file: scanlogd.init`: ensure `SOURCES/scanlogd.init` was copied to `build/SOURCES` and matches `Source1` in the SPEC.
- Unknown RPM macro or BuildRequires: check that the SPEC is the EL6 version and that the correct EL6 repositories are enabled.
- RPM builds but init service fails: inspect `service scanlogd status`, `/var/log/messages`, executable location, raw-socket permissions, and chroot ownership.
- No scan records: check packet arrival, scanlogd threshold, rsyslog output, and network filtering.
- Scan records but no Fail2Ban action: test with `fail2ban-regex`, examine the jail and action configuration, and inspect `/var/log/fail2ban.log` if configured.
- Fail2Ban reports a ban but no DROP is observed: inspect iptables chain jumps and packet counters.
- SELinux denial: inspect AVC audit messages and address the underlying policy issue rather than disabling SELinux without investigation.

## Upstream source and license

- Openwall project: <https://www.openwall.com/scanlogd/>
- Upstream source repository: <https://github.com/openwall/scanlogd>
- Openwall source archive: <https://download.openwall.net/pub/projects/scanlogd/>
- Upstream license: <https://github.com/openwall/scanlogd/blob/main/LICENSE>
- CentOS 6 lifecycle: <https://www.centos.org/distro-faq/>
- RHEL 6 lifecycle: <https://access.redhat.com/articles/4665701>

scanlogd is distributed under a simplified BSD-like license. Its upstream SPEC calls this `BSD-compatible`, which is not necessarily identical to standard SPDX `BSD-2-Clause` or `BSD-3-Clause` terms. Preserve the original LICENSE and review the exact wording before publishing license metadata.

This project provides **unofficial EL6 RPM packaging** and does not claim endorsement by Openwall, Red Hat, or CentOS.
