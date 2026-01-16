# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an Ansible infrastructure-as-code repository that bootstraps and maintains a Raspberry Pi 4 as a WireGuard VPN gateway with additional services (FTP server for CCTV storage, file browser, Pi-hole DNS filtering). The entire configuration is idempotent and can rebuild the Pi without manual intervention.

## Common Commands

### Setup
```bash
# Install Ansible Galaxy collections (required before first run)
ansible-galaxy collection install -r requirements.yml

# Create inventory file (copy from example and edit)
cp inventory.ini.example inventory.ini
```

### Running Playbooks
```bash
# Full bootstrap (all tasks)
ansible-playbook -i inventory.ini playbooks/main.yml --vault-password-file=.vaultpw

# With vault password prompt
ansible-playbook -i inventory.ini playbooks/main.yml --ask-vault-pass

# Targeted runs using tags
ansible-playbook -i inventory.ini playbooks/main.yml --vault-password-file=.vaultpw --tags initial_setup
ansible-playbook -i inventory.ini playbooks/main.yml --vault-password-file=.vaultpw --tags firewall
ansible-playbook -i inventory.ini playbooks/main.yml --vault-password-file=.vaultpw --tags docker
ansible-playbook -i inventory.ini playbooks/main.yml --vault-password-file=.vaultpw --tags wireguard
ansible-playbook -i inventory.ini playbooks/main.yml --vault-password-file=.vaultpw --tags apps
```

### Secret Management
```bash
# Create vaulted secrets file
ansible-vault create group_vars/all/vault.yml

# Edit existing vaulted file
ansible-vault edit group_vars/all/vault.yml

# Generate WireGuard keys (on any machine with wireguard-tools)
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee peer_private.key | wg pubkey > peer_public.key
```

## Architecture

### Task Organization
The playbook follows a modular structure:

- **playbooks/main.yml**: Main orchestration playbook that includes task files with proper tagging
- **tasks/initial_setup.yml**: System initialization (locale, packages, sensors)
- **tasks/firewall.yml**: nftables-based firewall configuration
- **tasks/docker.yml**: Docker installation and all containerized services (FTP, Filebrowser, Pi-hole)
- **tasks/wireguard.yml**: WireGuard VPN server setup with routing

### Variable Hierarchy
- **group_vars/all/main.yml**: Non-secret defaults (ports, paths, Docker images)
- **group_vars/all/vault.yml**: Encrypted secrets (WireGuard keys, passwords)
- **ansible.cfg**: Configures collections path (`./collections`) and SSH settings

### Docker Services
All Docker services use docker-compose-plugin (`docker compose v2`), not the legacy `docker-compose` binary. Compose files are templated from `templates/docker/<service>/docker-compose.yml.j2` and deployed to `/opt/docker/<service>/` on the target.

Services managed:
1. **FTP** (delfer/alpine-ftp-server): CCTV camera uploads to `/srv/cctv/ftp`
2. **Filebrowser**: Web UI for browsing CCTV footage on port 8080
3. **Pi-hole**: DNS filtering and ad-blocking on ports 53 (DNS) and 8082 (web UI)

### Firewall Strategy
Uses nftables (not iptables) with a deny-all input policy. The template at `templates/nftables.conf.j2` is rendered with variables from `group_vars/all/main.yml`:
- Allows SSH, WireGuard, FTP (control + passive range), Filebrowser, Pi-hole ports
- Permits Docker bridge networks for container outbound traffic
- Enables forwarding for WireGuard VPN routing

### WireGuard Configuration
- Server config template: `templates/wg0.conf.j2` → `/etc/wireguard/wg0.conf`
- Per-peer client configs: `templates/wireguard-peer.conf.j2` → `/etc/wireguard/peers/<name>.conf`
- Peer profiles defined in `vault.yml` with keys, addresses, allowed IPs
- Uses iptables rules in PostUp/PostDown for NAT masquerading (despite nftables for host firewall)

### Templating Pattern
All configuration files are Jinja2 templates in `templates/` that reference variables from `group_vars/all/main.yml` or `group_vars/all/vault.yml`. This includes:
- Docker Compose files
- WireGuard configs
- nftables ruleset
- FTP vsftpd.conf
- Filebrowser settings

## Key Variables

### Required Secrets (in vault.yml)
- `wireguard_server_private_key` / `wireguard_server_public_key`
- `wireguard_peer_profiles`: List of peer configurations with keys, addresses, allowed IPs
- `ftp_users_env`: FTP users in format `user|password|/home;user2|password2|/home2`
- `pihole_admin_password`: Pi-hole web UI password

### Commonly Modified (in main.yml)
- `wireguard_listen_port`: Default 51975
- `wireguard_endpoint_host`: Public hostname for WireGuard server
- `ftp_control_port`, `ftp_pasv_start`, `ftp_pasv_end`: FTP port configuration
- `filebrowser_port`: Default 8080
- `pihole_web_port`: Default 8082
- `docker_packages`: List of Docker packages from official repo
- `firewall_docker_subnets`: Docker bridge networks to allow outbound

## Development Notes

- The playbook expects Ansible 12.x and requires `ansible.posix` and `community.docker` collections installed locally
- Collections install to `./collections` (not system-wide) per `ansible.cfg`
- All tasks use `become: true` at the play level (see playbooks/main.yml:3)
- Docker Compose operations use `community.docker.docker_compose_v2` module, not shell commands
- Tags allow selective runs: `initial_setup`, `firewall`, `docker`, `wireguard`, `apps` (apps = firewall + docker + wireguard)
- The firewall reload task (tasks/firewall.yml:26-34) uses shell commands to delete and recreate the nftables table to avoid state conflicts
