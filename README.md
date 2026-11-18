# blog-postfix-relay-docker

Companion code for the blog post
[Set Up Your Own Postfix SMTP Relay with Docker Compose](https://webuildyourcloud.com/set-up-your-own-postfix-smtp-relay-with-docker-compose/).

It builds the setup from the post. It creates an Ubuntu 24.04 VM, installs Docker CE with the Compose plugin, and runs
`mwader/postfix-relay` with the post's `docker-compose.yml`:
- SASL authentication, using a `passwd_file` hashed with `mkpasswd -m sha-512` from the image.
- STARTTLS, with `smtpd_tls_auth_only=yes`.
- OpenDKIM signing.
- Relaying only for authenticated clients (`permit_sasl_authenticated,reject`).

## What's in here

| Path | What it does |
| --- | --- |
| `terraform/` | One Proxmox VE VM from the Ubuntu 24.04 cloud image (bpg/proxmox provider). Outputs an Ansible inventory. |
| `ansible/roles/docker_engine` | Docker CE, containerd and the Compose v2 plugin from download.docker.com. |
| `ansible/roles/postfix_relay` | The post, step by step. It creates the `traefik-proxy` network and the `opendkim_keys`/`var_log` volumes, writes the SASL password file and a self-signed certificate in `tls/` (or copies your own), templates `docker-compose.yml`, starts the relay, and prints the DKIM TXT record you need to publish. |
| `ansible/site.yml` | Runs both roles against the `relay` group. |

## Requirements

- Terraform >= 1.6 and a Proxmox VE API token (only if you want Terraform to create the VM).
- Ansible >= 2.16 (ansible-core) and the collections in `ansible/requirements.yml`.
- SSH access with sudo to the machine. The Terraform cloud-init user is `ubuntu`.
- Outbound port 25, or a smarthost (see `postfix_relay_extra_env`). Many providers block port 25.

## Quick start

```bash
# 1. Machine (optional; any Ubuntu 24.04 host works)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in endpoint, token, node, IP, SSH key
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. Configuration
cd ../ansible
ansible-galaxy collection install -r requirements.yml
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml   # set vault_postfix_relay_myuser_password
ansible-playbook site.yml --ask-vault-pass
```

Without Terraform, copy `ansible/inventory.example.yml` to `ansible/inventory.yml` and put your host in the `relay` group.

Publish the DKIM record that the last task prints, and an SPF record (`v=spf1 ip4:YOUR_SERVER_IP ~all`). Then test from
a client, as the post does:

```bash
swaks --server smtp.example.com --port 25 --tls \
  --auth PLAIN --auth-user myuser --auth-password mypassword \
  --from alerts@smtp.example.com --to recipient@domain.com
```

## Variables

| Variable | Default | Meaning |
| --- | --- | --- |
| `postfix_relay_myhostname` | `smtp.example.com` | `POSTFIX_myhostname`, and the CN of the self-signed certificate |
| `postfix_relay_dkim_domains` | `smtp.example.com` | `OPENDKIM_DOMAINS`, the sender domains that get signed (space-separated) |
| `postfix_relay_sasl_users` | `myuser` | List of `{name, password}` allowed to relay. Passwords come from `vault.yml` |
| `postfix_relay_timezone` | `Europe/Amsterdam` | `TZ` |
| `postfix_relay_rsyslog_remote_host` | `syslog.example.com` | `RSYSLOG_REMOTE_HOST`; empty leaves the remote syslog lines out |
| `postfix_relay_rsyslog_remote_port` | `514` | `RSYSLOG_REMOTE_PORT` |
| `postfix_relay_tls_self_signed` | `true` | Create a self-signed certificate. Set it to `false` and set `postfix_relay_tls_cert_src`/`postfix_relay_tls_key_src` to use your own |
| `postfix_relay_extra_env` | `[]` | Extra environment entries, e.g. `POSTFIX_relayhost=[smarthost.example.com]:587` |
| `postfix_relay_dir` | `/opt/postfix-relay` | Project directory on the host |
| `postfix_relay_network` | `traefik-proxy` | External Docker network the relay joins |
| `postfix_relay_smtp_port` | `25` | Published host port |

The SASL hash uses a salt derived from the user name and password. The password file only changes, and the relay only
restarts, when a password changes, so a second run reports `changed=0`.

## Tested

`site.yml` was run against a fresh Ubuntu 24.04 systemd container (Docker 29, Compose v2). A mail sink was set through
`postfix_relay_extra_env` (`POSTFIX_relayhost`). The second run reported `changed=0`. The post's `swaks` checks passed:
- `235 2.7.0 Authentication successful` and `250 2.0.0 Ok: queued` with `--tls --auth`.
- `554 5.7.1 ... Access denied` without authentication.
- No AUTH offered before STARTTLS.
- The mail arrived with `DKIM-Signature: ... d=smtp.example.com`.

ansible-lint (production profile), yamllint, `terraform validate` and tflint are clean. The Terraform code has not been
applied against a real Proxmox cluster.

## License

MIT
