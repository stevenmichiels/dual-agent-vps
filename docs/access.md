# Private Access

The steady-state access model is ordinary OpenSSH over the Tailscale network.
It is not public SSH and, by default, it is not Tailscale SSH.

## Bootstrap to Private Access

1. Restrict initial public SSH to the controller's current `/32` in both
   Terraform and Ansible.
2. Create and validate the non-root admin account before disabling root SSH.
3. Join the VPS to Tailscale.
4. Keep Tailscale connected but disable its SSH interception unless that
   separate access model was deliberately chosen:

   ```sh
   sudo tailscale set --ssh=false
   ```

5. Prove ordinary OpenSSH works through the VPS Tailscale IP or MagicDNS name.
6. Remove the public bootstrap CIDR from UFW and the Hetzner firewall.

Never reopen public SSH merely to support a new client device.

## Termius

Termius should connect through Tailscale using normal SSH:

- Install Tailscale on the Termius device and join the same tailnet.
- Use the VPS Tailscale IP or MagicDNS name.
- Use port `22` and the configured non-root admin user.
- Authenticate with a private key whose public key is listed in ignored
  `templates/ansible/vars/local.yml`.
- Keep password authentication disabled.

If Termius hangs at `Authenticating`, verify whether Tailscale SSH is
intercepting port `22`. Keep Tailscale networking enabled, but run
`sudo tailscale set --ssh=false` when the intended model is OpenSSH over
Tailscale.

## Device-Specific Keys

For a phone or tablet, create a separate Ed25519 key inside Termius:

1. Copy only its public key.
2. Add it as a separate `admin_authorized_keys` entry in ignored local vars.
3. Give the entry a stable label such as `termius-phone`.
4. Rerun Ansible from the trusted controller.
5. Test login over the Tailscale address.

Record the public-key fingerprint and verify presence without printing every
authorized key:

```sh
ssh-keygen -l -f <(printf '%s\n' '<public-key>')
grep -F '<public-key-body>' ~/.ssh/authorized_keys >/dev/null && echo present
```

Multiple authorized keys are normal when the controller and client devices use
separate, independently revocable identities. Never copy the controller's
private key, runtime env files, Terraform state, or backups into Termius.

## Troubleshooting Boundaries

- A working `ssh -T` Git test does not prove GitHub CLI authentication.
- Do not create a replacement key merely because a hostname or client config is
  wrong; verify the target IP, user, and effective SSH mode first.
- Do not remove `known_hosts` entries without confirming the host was rebuilt or
  the key legitimately changed.
- Prefer direct `ssh <user>@<tailscale-host>` tests before changing firewall or
  authentication policy.
- Keep GUI access, optional service ports, and SSH on the Tailscale boundary
  unless a separate exposure change is reviewed explicitly.
