# n8n and Cloudflare Tunnel

The supported public-ingress pattern keeps n8n itself private and exposes only
production webhook paths through a locally managed Cloudflare Tunnel.

## Steady-State Boundary

- n8n remains bound to `127.0.0.1:5678`.
- Public VPS ports `80`, `443`, and `5678` remain closed.
- Only `/webhook/*` reaches n8n through the tunnel.
- `/webhook-test/*`, `/webhook-waiting/*`, `/rest/*`, the editor, and sign-in
  routes remain private unless temporarily enabled for a specific setup step.
- Workflows with side effects use webhook-level authentication.

For n8n role variables and private access options, see the
[n8n role guide](../templates/ansible/roles/n8n/README.md). For all Cloudflare
role variables and WAF guidance, see the
[cloudflared role guide](../templates/ansible/roles/cloudflared/README.md).

## Prepare n8n Privately

Set the following in ignored `templates/ansible/vars/local.yml`:

```yaml
install_n8n: true
n8n_enable_service: false
```

Run Ansible once so it creates the service files. On the VPS, replace the
placeholders in `/etc/n8n/.env` with strong values for
`N8N_ENCRYPTION_KEY` and `N8N_USER_MANAGEMENT_JWT_SECRET`. Then set
`n8n_enable_service: true`, rerun Ansible, and verify private access through an
SSH tunnel:

```sh
ssh -N -L 5678:127.0.0.1:5678 <vps-ssh-target>
```

Open `http://127.0.0.1:5678` locally.

## Create the Tunnel

Prerequisites:

- The parent domain already uses Cloudflare nameservers.
- A public hostname such as `n8n.example.com` has been chosen.
- `cloudflared` is installed on the controller machine.

```sh
brew install cloudflared
cloudflared tunnel login
cloudflared tunnel create hermes-n8n
cloudflared tunnel route dns hermes-n8n n8n.example.com
cloudflared tunnel list
```

The login creates an account-scoped `cert.pem`; keep it on the controller. The
create command writes a tunnel-specific credentials JSON. Copy only that JSON
to the VPS path configured by `cloudflared_credentials_file`. Never paste
either file into chat or commit it.

Before logging in, deny agent access to these controller-local files where your
agent configuration supports exclusions:

```text
~/.cloudflared/cert.pem
~/.cloudflared/*.json
```

## Stage the Rollout

Configure the tunnel in ignored `templates/ansible/vars/local.yml` and keep the
service disabled for the first Ansible pass:

```yaml
install_cloudflared: true
cloudflared_enable_service: false
cloudflared_install_method: apt_repo
cloudflared_tunnel_uuid: "<tunnel-uuid>"
cloudflared_hostname: "n8n.example.com"
cloudflared_hello_world_enabled: true
cloudflared_n8n_webhook_enabled: false
cloudflared_n8n_webhook_test_enabled: false
cloudflared_n8n_webhook_waiting_enabled: false
cloudflared_n8n_oauth_callback_enabled: false
```

After Ansible creates `/etc/cloudflared` and the service group, install the
tunnel JSON on the VPS with owner `root`, group `cloudflared`, and mode `0640`.
Enable the service with the hello-world route first. Once that works, switch to
the production webhook route:

```yaml
cloudflared_enable_service: true
cloudflared_hello_world_enabled: false
cloudflared_n8n_webhook_enabled: true
cloudflared_n8n_webhook_test_enabled: false
cloudflared_n8n_webhook_waiting_enabled: false
cloudflared_n8n_oauth_callback_enabled: false
```

Configure `WEBHOOK_URL`, `N8N_PROXY_HOPS`, and `N8N_EDITOR_BASE_URL` for the
public hostname. Leave `N8N_SECURE_COOKIE=false` while the editor is reached
through local HTTP over SSH or Tailscale.

## Validate

```sh
systemctl is-active cloudflared
sudo hermes-vps status
cloudflared tunnel ingress rule https://n8n.example.com/webhook/abc
cloudflared tunnel ingress rule https://n8n.example.com/webhook-test/abc
cloudflared tunnel ingress rule https://n8n.example.com/rest/login
```

Steady state should send the first path to n8n and return the catch-all `404`
for the others. Temporary onboarding routes must be removed after use.

Online SQLite backups should be age-encrypted and include `/etc/n8n` plus the
relevant tunnel configuration. Enable retention pruning only after the off-box
copy has been checksum-verified. See [Operations and Recovery](operations.md).
