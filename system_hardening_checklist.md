
Not final in any way. Learning cyber security by building my own tools and scripts.

Directory Structure
homelab-hardening/
├── ansible.cfg
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       ├── linux_servers.yml
│       ├── mac_workstations.yml
│       └── docker_hosts.yml
├── roles/
│   ├── common/
│   │   ├── tasks/main.yml
│   │   └── handlers/main.yml
│   ├── ssh_hardening/
│   │   ├── tasks/main.yml
│   │   └── templates/sshd_config.j2
│   ├── firewall/
│   │   ├── tasks/main.yml
│   │   └── templates/ufw.rules.j2
│   ├── sysctl/
│   │   ├── tasks/main.yml
│   │   └── templates/sysctl-hardening.conf.j2
│   ├── auditing/
│   │   ├── tasks/main.yml
│   │   └── templates/audit.rules.j2
│   └── fail2ban/
│       ├── tasks/main.yml
│       └── templates/jail.local.j2
├── container-hardening/
│   ├── docker_daemon.yml
│   └── podman_config.yml
└── network-devices/
    ├── pfsense.yml
    ├── ubiquiti.yml
    └── mikrotik.yml
└── site.yml

Checklists

1. Authentication & Access Control
#	Setting	Command/Config Path	Recommended Value	OS Notes
1.1	Disable root SSH login	/etc/ssh/sshd_config	PermitRootLogin no	All Linux
1.2	Disable password auth	/etc/ssh/sshd_config	PasswordAuthentication no	Key-only access
1.3	Limit SSH users	/etc/ssh/sshd_config	AllowUsers ghost admin1 admin2	Whitelist approach
1.4	SSH timeout	/etc/ssh/sshd_config	ClientAliveInterval 300, ClientAliveCountMax 2	Disconnect idle
1.5	Fail2ban SSH jail	/etc/fail2ban/jail.local	enabled = true, maxretry = 5, bantime = 3600	Install package first
1.6	Sudo require password	/etc/sudoers	Defaults passwd_timeout=1	Prevents cached sudo abuse
1.7	Strong password policy	/etc/login.defs	PASS_MAX_DAYS 90, PASS_MIN_LEN 12	Ubuntu/Fedora
1.8	PAM password quality	/etc/pam.d/common-password (Ubuntu) / /etc/pam.d/system-auth (Fedora)	pam_pwquality.so minlen=12 dcredit=-1 ucredit=-1 ocredit=-1 lcredit=-1	Enforce complexity
1.9	Account lockout	/etc/pam.d/common-auth	pam_tally2.so deny=5 unlock_time=1800	Alternative: pam_faillock
1.10	macOS Gatekeeper	Terminal command	sudo spctl --master-enable	Blocks unsigned apps
1.11	macOS SIP status	Recovery mode command	SIP enabled (csrutil enable)	macOS only
1.12	macOS Firewall	defaults write /Library/Preferences/com.apple.alf globalstate -int 1	Firewall ON	Script via sudo


2. Network Security
#	Setting	Command/Config Path	Recommended Value	OS Notes
2.1	Enable firewall (iptables/ufw)	ufw enable (Ubuntu) / firewalld (Fedora)	Default deny incoming	Choose per distro
2.2	UFW default rules	ufw default deny incoming && ufw allow out	Deny in, allow out	Ubuntu
2.3	Firewalld zone config	firewall-cmd --set-default-zone=public	Restrictive default	Fedora/RHEL
2.4	macOS PF firewall	/etc/pf.conf	Custom rules with anchor	macOS only
2.5	TCP SYN cookies	/etc/sysctl.conf	net.ipv4.tcp_syncookies = 1	DDoS mitigation
2.6	IP forwarding disable	/etc/sysctl.conf	net.ipv4.ip_forward = 0	Unless routing needed
2.7	ICMP redirect ignore	/etc/sysctl.conf	net.ipv4.conf.all.accept_redirects = 0	MITM prevention
2.8	Source routing off	/etc/sysctl.conf	net.ipv4.conf.all.accept_source_route = 0	Spoofing prevention
2.9	IPv6 disable (if unused)	/etc/modprobe.d/ipv6.conf	options ipv6 disable=1	Optional
2.10	Port scan detection	sudo apt install psad or yum install psad	Install + configure	Log monitoring
Sysctl script template:

cat >> /etc/sysctl.d/99-hardening.conf << 'EOF'
net.ipv4.tcp_syncookies = 1
net.ipv4.ip_forward = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
fs.suid_dumpable = 0
EOF
sysctl -p /etc/sysctl.d/99-hardening.conf

3. File System & Permissions
#	Setting	Command/Config Path	Recommended Value	OS Notes
3.1	World-writable dirs audit	find / -xdev -perm -002 -type d	Fix/remove violations	All
3.2	Noexec on /tmp	/etc/fstab	Mount /tmp with noexec,nosuid,nodev	Critical
3.3	Remove SUID binaries	find / -xdev -perm -4000 -type f	Review/list before removing	Audit first
3.4	Remove dangerous SUID	/usr/bin/X11, /bin/su etc.	Keep essential only	Script review
3.5	Hide process info	/proc/sys/kernel/kptr_restrict	kptr_restrict = 2	Kernel info leak
3.6	Core dumps disable	/etc/security/limits.conf	* hard core 0	Prevent credential leaks
3.7	Secure umask	/etc/profile, /etc/bash.bashrc	umask 027	Restrictive default
3.8	Auditd rules	/etc/audit/rules.d/hardening.rules	Monitor auth, sudo, cron	Log all activity
3.9	Immutable critical files	chattr +i /etc/passwd /etc/shadow	Protect user db	Use cautiously
3.10	Encrypted home directories	Ubuntu: ecryptfs-utils, Fedora: LUKS setup	Encrypt sensitive data	Setup during install
4. Package & Service Management
#	Setting	Command/Config Path	Recommended Value	OS Notes
4.1	Remove unnecessary packages	apt list --installed, rpm -qa	Prune dev/tools	Manual review
4.2	Disable unused services	systemctl disable <service>	Identify non-essential	List all first
4.3	Auto-security updates	Ubuntu: unattended-upgrades, Fedora: dnf-automatic	Daily patching	Configurable schedule
4.4	GPG key verification	gpg --verify	Enforce signed packages	All
4.5	Reproducible builds flag	Compiler flags	CFLAGS="-O2 -D_FORTIFY_SOURCE=2"	For custom builds
4.6	Docker daemon hardening	/etc/docker/daemon.json	Set log-driver=json-files, limit resources	If Docker used
4.7	Snap/Flatpak restrictions	Sandboxed apps only	Prefer over deb/rpm for apps	Modern packages
4.8	Time sync enforcement	timedatectl set-ntp true	NTP synchronized	Prevent auth bypass
Service pruning script skeleton:

#!/bin/bash
# List services, filter out essentials, disable candidates
systemctl list-unit-files --type=service --state=enabled | \
grep -vE 'sshd|cron|network|docker|fail2ban|rsyslog|systemd-journald' | \
awk '{print $1}' | sed 's/.service$//' | \
while read svc; do echo "Review: $svc"; done

