# Lakewatch-related skills

Databricks Genie Code skills that help migrate security detections into **Databricks Lakewatch** by translating legacy query languages into Databricks SQL against Lakewatch OCSF gold (and silver) tables.

This repository packages the skills locally and deploys them to a Databricks workspace with Terraform.

## Contents

| Path | Description |
|------|-------------|
| [`lakewatch-spl-migration/`](lakewatch-spl-migration/) | Translates Splunk SPL (raw sourcetypes and CIM datamodels) into Lakewatch SQL with mapping notes. |
| [`lakewatch-kql-migration/`](lakewatch-kql-migration/) | Translates Microsoft Sentinel / Defender KQL into Lakewatch SQL with mapping notes. |

Each skill directory follows the standard Assistant skill layout:

```text
<skill-name>/
  SKILL.md              # Skill instructions and workflow
  references/           # Table mappings, operator guides, OCSF notes
  evals/
    evals.json          # Evaluation cases for the skill
```

### `lakewatch-spl-migration`

Use when porting Splunk searches, dashboards, saved searches, or Enterprise Security correlations to Lakewatch.

| Reference | Purpose |
|-----------|---------|
| `references/sourcetype-mapping.md` | Splunk sourcetype → Lakewatch table routing |
| `references/cim-to-ocsf.md` | CIM datamodel → OCSF gold class mapping |
| `references/spl-operators.md` | SPL operator → Spark SQL equivalents |

### `lakewatch-kql-migration`

Use when porting Microsoft Sentinel or Defender Advanced Hunting queries to Lakewatch.

| Reference | Purpose |
|-----------|---------|
| `references/sentinel-table-mapping.md` | Sentinel log tables → Lakewatch tables |
| `references/defender-table-mapping.md` | Defender Advanced Hunting tables → Lakewatch tables |
| `references/ocsf-gold-tables.md` | OCSF gold layer overview |
| `references/kql-operators.md` | KQL operator → Spark SQL equivalents |

## Installation

You can install skills using Terraform or just upload them to the workspace as per [documentation](https://docs.databricks.com/aws/en/genie-code/skills)

### Deploy with Terraform

Prerequisits:

- [Terraform](https://developer.hashicorp.com/terraform/install) >= 1.0
- [Databricks CLI](https://docs.databricks.com/dev-tools/cli/index.html) configured, **or** environment variables / provider settings the [Databricks Terraform provider](https://registry.terraform.io/providers/databricks/databricks/latest/docs) accepts (for example `DATABRICKS_HOST` and `DATABRICKS_TOKEN`)
- Permission to create workspace directories and files at the install path (see [Install target](#install-target))

Authenticate to your workspace if you have not already:

```bash
databricks auth login --host https://<workspace-host>
```

After this is done, from this directory:

```bash
terraform init
terraform plan
terraform apply
```

Terraform discovers every **immediate subdirectory** that contains a `SKILL.md` file (`lakewatch-spl-migration`, `lakewatch-kql-migration`) and uploads all files under those folders to the workspace.

#### Install target

By default, [`main.tf`](main.tf) sets `global = true`, so skills are installed for all workspace users at:

```text
/Workspace/.assistant/skills/<skill-name>/...
```

To install only for the authenticated user instead, change the module block in `main.tf`:

```hcl
module "assistant_skills" {
  source = "./modules/install-assistant-skills"

  skills_source_dir = local.skills_dir
  global            = false
  username          = var.databricks_username
}
```

User-scoped installs land under `/Users/<user>/.assistant/skills/...`. Optionally set `databricks_username` when the provider identity differs from the target user's workspace home folder:

```bash
terraform apply -var='databricks_username=user@example.com'
```

#### Verify deployment

After `apply`, check Terraform outputs (from the module):

- `workspace_skills_root` — workspace path of the skills root
- `skill_names` — installed skill directory names
- `workspace_file_paths` — full paths of uploaded files

In the workspace UI, confirm files exist under **Workspace** → `.assistant` → `skills` (global install) or under your user folder (user-scoped install).

#### Update or remove skills

Re-run `terraform apply` after editing local skill files; changed files are updated in the workspace. To remove the deployment:

```bash
terraform destroy
```

## Using the skills

Once deployed, open **Databricks Assistant** in a workspace where Lakewatch is available and ask it to migrate SPL or KQL queries. The assistant loads skills from `.assistant/skills/` automatically when they match the request (for example, "convert this Sentinel KQL to Lakewatch SQL").

You can also invoke the skill explicitly in supported clients with `/lakewatch-spl-migration` or `/lakewatch-kql-migration` where slash commands are supported.
