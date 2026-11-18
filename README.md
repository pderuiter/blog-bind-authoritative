# blog-bind-authoritative

Companion code for the blog post
[How to Setup BIND as an Authoritative DNS Server](https://blog.bsdserver.nl/how-to-setup-bind-as-an-authoritative-dns-server/).

It builds the post's setup: two authoritative-only BIND 9 servers, a master (`ns1`,
`type primary`) serving `example.com` and `1.168.192.in-addr.arpa`, and a slave (`ns2`,
`type secondary`) that transfers `example.com` from it. Recursion is off, zone transfers are
limited to the slave, responses are rate limited and the server version is hidden. Logging
goes to `/var/log/named/` as in section 11 of the post.

## What is in here

| Path | Purpose |
|------|---------|
| `terraform/` | Two Ubuntu 24.04 VMs on Proxmox VE (bpg/proxmox provider, cloud-init), outputs an Ansible inventory |
| `ansible/site.yml` | One playbook for the whole setup |
| `ansible/roles/bind` | Installs BIND and writes `named.conf` with includes for options and zones |
| `ansible/roles/bind_auth` | Options, zone definitions, zone files and logging from the post |

## Requirements

- Terraform >= 1.6 and a Proxmox VE API token (only if you want Terraform to create the VMs)
- ansible-core >= 2.15 (tested with 2.20.1)
- Target hosts: Ubuntu 24.04 (tested, BIND 9.18) or RHEL/Rocky 9 (paths and package names are handled; not lab-tested)

## Quick start

```bash
# 1. Machines (optional; skip if you already have two hosts)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. BIND
cd ../ansible
cp inventory.example.yml inventory.yml         # if you skipped Terraform
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml

# 3. Test (section 9 of the post)
dig @10.0.0.1 www.example.com A
dig @10.0.0.2 www.example.com A                 # answered by the slave
dig @10.0.0.1 example.com AXFR                  # "Transfer failed." from anywhere but ns2
```

Hosts in the `dns_primary` group become masters, hosts in `dns_secondary` become slaves.

## Variables

Set in `ansible/group_vars/all/vars.yml` (defaults in `roles/bind_auth/defaults/main.yml`):

| Variable | Default | Meaning |
|----------|---------|---------|
| `bind_auth_primary_address` | `10.0.0.1` | Master address, used in the slave's `primaries` and the `ns1` A record |
| `bind_auth_secondary_addresses` | `["10.0.0.2"]` | Slaves allowed to transfer and notified on change |
| `bind_auth_zones` | `example.com`, `1.168.192.in-addr.arpa` | Zones on the master (name, file, template, serial) |
| `bind_auth_secondary_zones` | `["example.com"]` | Zones the slave transfers |
| `bind_auth_rrl_responses_per_second` / `bind_auth_rrl_window` | `15` / `5` | Response rate limiting |
| `bind_auth_logging` | `true` | File logging to `bind_auth_log_dir` (`/var/log/named`) |

To change a zone, edit its template under `roles/bind_auth/templates/zones/`, bump its
`serial` and run the playbook again; the master restarts and notifies the slave.

This setup has no secrets, so `group_vars/all/vault.yml.example` is empty.

## Terraform variables

See `terraform/variables.tf` and `terraform/terraform.tfvars.example`. The VMs are defined in
the `vms` map (name, IP in CIDR notation, Ansible groups, optional cores/memory/disk).

## Tested

`site.yml` was run against fresh Ubuntu 24.04 systemd containers (BIND 9.18.39) with the
post's `dig` checks afterwards; the second run reports `changed=0`. ansible-lint (production
profile), yamllint, `terraform validate` and tflint are clean. The Terraform code has not
been applied against a real Proxmox cluster.

## License

MIT
