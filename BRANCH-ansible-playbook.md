# Branch: aap-terraform-ansible-playbook

This branch runs post-provision configuration with the Terraform
`ansible_playbook` resource from the [ansible/ansible provider](https://registry.terraform.io/providers/ansible/ansible/latest).

## How it works

1. AAP prepares the Terraform bundle (TF files, inventory template, private key tfvars, `install_nginx-rhel.yml`).
2. AAP uploads the bundle to Terraform Cloud and triggers apply.
3. During apply on the **Terraform runner**:
   - AWS EC2 resources are created
   - `ansible_host` / `ansible_group` register inventory in Terraform state
   - `local_file` writes `inventory.ini` and `ansible_private_key.pem`
   - `ansible_playbook` runs `install_nginx-rhel.yml` against that inventory

AAP does **not** run the nginx play on this branch. Configuration happens inside Terraform apply.

## Prerequisites (required)

### Self-hosted Terraform Cloud agent

`ansible_playbook` executes `ansible-playbook` on the Terraform runner. **Default Terraform Cloud remote runners do not have Ansible.**

1. Install a [Terraform Cloud agent](https://developer.hashicorp.com/terraform/cloud-docs/agents) on a VM you control.
2. On that VM install:
   - `terraform` (agent handles this)
   - `ansible-core` / `ansible-playbook`
3. Create an agent pool in Terraform Cloud.
4. Assign your workspace to that agent pool (Execution mode: Agent).

### AAP Job Template

- Playbook: `playbooks/terraform_apply_plan_and_configure.yml`
- Project branch: `aap-terraform-ansible-playbook`
- Survey/extra var: `aws_instance_private_key` (encrypted) matching `aws_instance_public_key`
- Machine Credential optional if using Credential Input Source → `aws_instance_private_key`

### Terraform workspace

- Agent pool assigned (not default remote)
- AWS credentials configured as today
- Providers: `hashicorp/aws`, `ansible/ansible`, `hashicorp/local`

## Compare with `aap-terraform-aap` branch

| | aap-terraform-aap (hybrid) | This branch |
|--|---------------------------|-------------|
| Nginx runs on | AAP EE | TFC agent VM |
| TF resource | `ansible_host` / `ansible_group` | + `ansible_playbook` |
| TFC runner | Default remote OK | Self-hosted agent required |

## Verify

After a successful apply:

```bash
curl http://<ec2-public-ip>
```

Check Terraform outputs: `instance_ip_addr`, `ansible_playbook_stdout`.
