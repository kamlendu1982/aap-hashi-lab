# Branch: aap-terraform-ansible-playbook

This branch runs Terraform **locally inside the AAP execution environment**
(no Terraform Cloud remote apply) and configures EC2 with the Terraform
`ansible_playbook` resource from the [ansible/ansible provider](https://registry.terraform.io/providers/ansible/ansible/latest).

## How it works

1. AAP prep play builds the Terraform bundle under `/var/terraform/plans/...`
   (TF files, inventory template, private key tfvars, `install_nginx-rhel.yml`).
2. AAP runs `terraform init` and `terraform apply` **inside the EE**.
3. During apply on the **same EE container**:
   - AWS EC2 resources are created
   - `ansible_host` / `ansible_group` register inventory in Terraform state
   - `local_file` writes `inventory.ini` and `ansible_private_key.pem`
   - `ansible_playbook` runs `install_nginx-rhel.yml` against that inventory

AAP does **not** run a separate nginx play. Configuration happens inside
`terraform apply` via `resource "ansible_playbook"`.

Terraform state is stored locally as `terraform.tfstate` in the working
directory (default local backend).

## Prerequisites

### Execution environment

Rebuild and use the project EE (`execution-environment/execution-environment.yml`):

- `terraform` CLI (already installed)
- `ansible-core` / `ansible-playbook` (added in EE build)
- `openssh-clients` (SSH from ansible_playbook to EC2)

### AAP Job Template (apply)

- Playbook: `playbooks/terraform_apply_plan_and_configure.yml`
- Project branch: `aap-terraform-ansible-playbook`
- **Amazon AWS** credential attached (injects `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`)
- Survey/extra vars (same as before):
  - `aws_region`, `aws_name_tag`, `aws_instance_size`, `aws_instance_count`
  - `aws_instance_public_key`
  - `aws_instance_private_key` (encrypted) — passed into Terraform for `ansible_playbook` SSH
- **No Terraform Cloud token or workspace required** on this branch

### Not required on this branch

- Terraform Cloud remote apply
- Self-hosted TFC agent VM
- `tf_hostname`, `tf_token`, `tf_org`, `tf_workspace` job vars

## Destroy

Playbook: `playbooks/terraform_destroy_plan.yml`

Runs `terraform destroy` in the same working directory. Requires an existing
`terraform.tfstate` from a prior apply on the **same execution node path**
(`/var/terraform/plans/<working_dir_name>/`).

## Compare with other branches

| | `main` / original | `aap-terraform-aap` | This branch |
|--|-------------------|---------------------|-------------|
| TF apply runs on | TFC remote | TFC remote | **AAP EE** |
| Nginx runs on | — | AAP EE | **AAP EE via ansible_playbook** |
| TFC required | Yes | Yes | **No** |
| TFC agent VM | No | No | **No** |

## Verify

After a successful job:

```bash
curl http://<ec2-public-ip>
```

Check Terraform output in job log or run `terraform output` in the working dir.
