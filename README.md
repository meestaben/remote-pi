# remote-pi Ansible Playbooks

This repo bootstraps and maintains a Raspberry Pi 4 that serves as a WireGuard gateway plus general services (CCTV storage, etc.). Everything is driven through Ansible so the box can be rebuilt quickly without remembering ad‑hoc steps.

## Prerequisites

- Ansible 12.x on your control machine.
- Install the pinned Galaxy collections locally:
  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```
- Collections install into `./collections` (configured via `ansible.cfg`); rerun the command whenever `requirements.yml` changes.
- The playbook configures Docker’s official apt repository (`download.docker.com`) to install `docker-ce`, `docker-compose-plugin`, etc., so outbound HTTPS to that domain must be allowed.
- SSH access to the Pi defined in `inventory.ini`.
- WireGuard key pairs for the server and each peer.

## Secret management

1. Create (or edit) the vaulted group vars file:
   ```bash
   ansible-vault create group_vars/all/vault.yml
   # or: ansible-vault edit group_vars/all/vault.yml
   ```
2. Store secrets there using the exact variable names referenced in `group_vars/all/main.yml`:
   ```yaml
   wireguard_server_private_key: "..."
   wireguard_server_public_key: "..."
   wireguard_peer_profiles:
     - name: home-user
       private_key: "..."
       public_key:  "..."
       address: 10.6.0.2/32
       allowed_ips: [10.6.0.2/32]
       client_allowed_ips: ["0.0.0.0/0", "::/0"]
       dns: ["1.1.1.1", "1.0.0.1"]
       persistent_keepalive: 25
   ftp_users_env: "camera|SuperSecret!|/home"
   pihole_admin_password: "SetMe!"
   ```
3. Keep the vault password in `.vaultpw` (ignored) or supply it interactively with `--ask-vault-pass`.

## Generating WireGuard keys

On any machine with `wireguard-tools`:
```bash
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee home_user_private.key | wg pubkey > home_user_public.key
```
Paste the base64 strings into the vaulted file and remove the temporary files once recorded.

## Running the playbooks

Ensure you've created inventory.ini and edited the hostname and IP to suit your requirements.

Full bootstrap (locale, packages, firewall, WireGuard, etc.):
```bash
ansible-playbook -i inventory.ini playbooks/main.yml --vault-password-file=.vaultpw
```
Targeted reruns using tags:
- Initial setup only: `--tags initial_setup`
- Firewall rules only: `--tags firewall`
- Docker/Compose stack only: `--tags docker`
- WireGuard config only: `--tags wireguard`
- All “app” layers (firewall + WireGuard): `--tags apps`

## Docker apps

- Compose files live under `/opt/docker/<service>/docker-compose.yml` (rendered from `templates/docker/...`).
- FTP stack (`delfer/alpine-ftp-server`) stores uploads under `/srv/cctv/ftp`. Define `ftp_users_env` in the vault using the image’s syntax: `user|password|/home/user;user2|password2|/home/user2`. Ports default to 21 + 30000-30009; adjust `ftp_*` vars in `group_vars/all/main.yml` if needed.
- Firewall rules automatically allow the FTP control port and passive range listed in those variables for any source (loopback included), so LAN clients can reach them without extra config.
- FTP configuration lives on the host at `/opt/docker/ftp/config/vsftpd.conf`. Edit that file (or override `ftp_vsftpd_extra_options`) and rerun `ansible-playbook ... --tags docker` to push updates.
- Filebrowser (`filebrowser/filebrowser`) serves `/srv/cctv/filebrowser` on port 8080. Log in with the default `admin/admin` the first time, then create proper users from the UI.
- Filebrowser configuration resides at `/opt/docker/filebrowser/config/filebrowser.json`; adjust it (port, branding, etc.) and rerun the docker tag to restart the container. Database files stay in `/opt/docker/filebrowser/database`.
- Pi-hole (`pihole/pihole`) listens on TCP/UDP 53 (DNS) and exposes its admin UI on `pihole_web_port` (default 8081). Config/state live under `/opt/docker/pihole/config` (`etc-pihole` + optional `dnsmasq.d`). The playbook populates upstream DNS servers with Cloudflare’s family-safe endpoints (1.1.1.3/1.0.0.3) by default; override `pihole_dns_servers` / `pihole_dns_listening_mode` as needed. Pi-hole runs as `pihole_uid`/`pihole_gid` (defaults 1000); adjust those vars if you need a different host user and rerun the docker tag so ownership is fixed. DHCP mode is disabled entirely.
- Each compose stack uses Docker's default bridge network; if you add additional custom networks make sure their subnets are listed in `firewall_docker_subnets` and rerun `ansible-playbook ... --tags firewall`.
- Docker bridge networks are permitted outbound via `firewall_docker_subnets` (defaults `172.17.0.0/16`, `172.18.0.0/16`). Add any additional container subnets there and rerun `ansible-playbook ... --tags firewall` if your stacks use other bridges.
- Docker Engine/Buildx/Compose all come from Docker’s official apt repo (see `docker_packages` in `group_vars/all/main.yml`). Update that list or the repo settings there if you need edge/testing builds.
- Run `ansible-playbook ... --tags docker` after changing Docker-related vars to rebuild the stack. Use `docker compose -f /opt/docker/<service>/docker-compose.yml logs` on the Pi for troubleshooting.

## Post-deploy checks

- Verify firewall: `sudo nft list ruleset` and ensure `chain input` shows `policy drop` plus the expected SSH/WireGuard/FTP/Filebrowser/Pi-hole accept rules.
- Confirm WireGuard is up: `sudo systemctl status wg-quick@wg0` and `sudo wg show`.
- Grab client profiles from `/etc/wireguard/peers/*.conf` (SCP or `ansible -m fetch`); use `qrencode` if you want QR codes for mobile clients.
- Confirm Docker apps: `sudo systemctl status docker`, `docker ps`, and test FTP/filebrowser ports from a LAN host.

Keep `group_vars/all/main.yml` for non-secret defaults (interfaces, ports, etc.) and adjust as your LAN changes. Everything sensitive belongs in `group_vars/all/vault.yml`.
