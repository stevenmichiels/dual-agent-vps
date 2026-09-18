# Remote Desktop

The default workbench is headless. A desktop adds packages, memory pressure,
and local session surface, so enable it only for a concrete GUI requirement.

## Enable the Profile

The supported opt-in profile installs XFCE and NoMachine. NoMachine is
proprietary software that is free for personal use, so treat enabling it as an
explicit supply-chain decision.

After OpenSSH over Tailscale is stable, set the following in ignored
`templates/ansible/vars/local.yml`:

```yaml
install_remote_desktop: true
install_nomachine: true
```

Rerun Ansible. The role:

- installs the official Linux amd64 DEB and verifies the vendor MD5;
- disables NoMachine UPnP/NAT-PMP port mapping;
- disables NoMachine's automatic firewall changes;
- removes installer-created public UFW rules;
- requires NX private-key authentication;
- writes `/home/<admin_user>/.nx/config/authorized.crt` from
  `admin_authorized_keys`;
- allows TCP/UDP port `4000` only from the Tailscale CIDR; and
- starts virtual desktops with `/etc/X11/Xsession startxfce4`.

Do not open public VNC, RDP, NoMachine, or X11 ports.

## Connect From macOS

Create a connection in the NoMachine client with:

```text
Protocol: NX
Host: <vps-tailscale-ip-or-magicdns-name>
Port: 4000
Username: <admin_user>
Authentication: Private key
Private key: ~/.ssh/<key-listed-in-admin_authorized_keys>
```

Use ignored `templates/ansible/inventory.ini` and
`templates/ansible/vars/local.yml` as the source of truth for the host, user,
and key path. Do not copy those private values into this repository.

If NoMachine cannot detect a display, allow it to create a new virtual display;
that is expected on a headless VPS. Do not sign in to NoMachine Network/cloud
for this host—connect directly through its Tailscale IP or MagicDNS name.

## Resource Guidance

- Minimum for light desktop use: 2 vCPU and 4 GB RAM.
- Preferred for browsers, IDEs, and agents: 4 vCPU and 8 GB RAM or more.
- Check `free -h`, `swapon --show`, and Docker memory pressure before adding
  browser-heavy workloads.
- Recheck UFW and the Hetzner firewall after enabling the profile.

## Validate

On the VPS:

```sh
dpkg-query -W xfce4 nomachine
grep -E '^(DefaultDesktopCommand|AcceptedAuthenticationMethods)' \
  /usr/NX/etc/node.cfg /usr/NX/etc/server.cfg
sudo ss -ltnp | grep ':4000 '
```

Then connect over Tailscale and confirm no public GUI rule exists in UFW or the
Hetzner firewall.
