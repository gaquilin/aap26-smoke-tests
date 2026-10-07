# Smoke & Regression Test Suite

Automated smoke and regression test suite for **Red Hat Ansible Automation Platform 2.6** (AAP 2.6), built with the `infra.aap_configuration` collection and Ansible Configuration as Code (CaC) principles.

The suite verifies that all core AAP 2.6 components are operational by creating, validating, and optionally cleaning up a full set of platform resources via API. An HTML summary report is generated at the end of the test suite and, optionally, sent via email.

## Tests Overview

| # | Playbook | Test | Description |
|---|---|---|---|
| 01 | `01_platform_availability.yml` | Platform Availability | Pings Gateway, Controller, Hub, and EDA endpoints; verifies authentication and controller instances |
| 02 | `02_iam_access_validation.yml` | IAM Access Validation | Validates user identity via Gateway and Controller `/me/` endpoints; extracts user details (superuser, system auditor, LDAP DN) |
| 03 | `03_push_ee_to_hub.yml` | Load & Push EE to Hub | Loads an Execution Environment image from a local `.tar` file and pushes it to Private Automation Hub using |
| 04 | `04_register_ee.yml` | Register EE on Controller | Registers the pushed EE on the Automation Controller |
| 05 | `05_create_organization.yml` | Create Organization | Creates an Organization with optional Galaxy Credentials |
| 07 | `07_project_sync.yml` | Project Synchronization | Creates a Project from a Git repository and waits for sync completion |
| 10 | `10_create_inventory.yml` | Create Inventory | Creates an Inventory with Hosts, optional Inventory Sources, and runs an Ad Hoc ping |
| 11 | `11_create_job_template.yml` | Create Job Template | Creates a Job Template with Extra Variables and Concurrent Jobs |
| 12 | `12_launch_job_template.yml` | Launch Job Template | Launches the Job Template and verifies successful execution |
| 13 | `13_create_workflow_template.yml` | Create Workflow Template | Creates a Workflow Job Template with a single node |
| 14 | `14_launch_workflow_template.yml` | Launch Workflow Template | Launches the Workflow and verifies all nodes complete successfully |
| 99 | `99_cleanup.yml` | Cleanup | Deletes all smoke test resources in reverse dependency order |

All tests can be executed sequentially via the **master playbook** (`smoke_test_master.yml`) or individually as standalone playbooks. Each test can be enabled or disabled via execution flags in `vars/main.yml`.

## Project Structure

```
aap26-smoke-tests/
├── ansible.cfg                          # Ansible configuration
├── requirements.yml                     # Collection dependencies
├── README.md                            
│
├── credentials/
│   └── credentials.yml                  # AAP 2.6 and User credentials (encrypt with ansible-vault)
│
├── vars/
│   └── main.yml                         # Configurable variables
│
├── playbooks/
│   ├── smoke_test_master.yml            # Master Playbook
│   ├── 01_platform_availability.yml
│   ├── 02_iam_access_validation.yml
│   ├── 03_push_ee_to_hub.yml
│   ├── 04_register_ee.yml
│   ├── 05_create_organization.yml
│   ├── 07_project_sync.yml
│   ├── 10_create_inventory.yml
│   ├── 11_create_job_template.yml
│   ├── 12_launch_job_template.yml
│   ├── 13_create_workflow_template.yml
│   ├── 14_launch_workflow_template.yml
│   └── 99_cleanup.yml
│
├── templates/                           # Resource definitions (one subfolder per resource type)
│   ├── organizations/                   # Organization definitions
│   ├── projects/                        # Project definitions
│   ├── inventories/                     # Inventory and Host definitions
│   ├── execution_environments/          # Execution Environment registrations on Controller
│   ├── job_templates/                   # Job Template definitions
│   ├── workflow_job_templates/          # Workflow Job Template definitions
│   ├── ee_definition/                   # Pre-built EE tar file (e.g. smoke-reg-test-ee-minimal-rhel9.tar)
│   └── reports/
│       └── smoke_test_report.html.j2    # HTML report template (Jinja2)
│
├── reports/                             # Auto-generated reports
│   └── YYYY-MM-DD/
│       └── smoke_test_report.html      
│
├── collections/                         
├── roles/                               
└── tasks/                              
```

## Prerequisites

### Ansible Version and AAP 2.6 Resources

- **Ansible Core** 2.16+ installed on the control node
- **Python** 3.9+
- **Podman** installed on the control node (required for loading and pushing the EE image)
- Network access to the AAP 2.6 Gateway URL
- **Pre-existing credentials on AAP 2.6**: the credentials and credential types referenced by the test resources (SCM credential, Galaxy credentials, Job Template credentials) must already exist on the target AAP environment

### Ansible Collections

The following Ansible Collections are required:

| Collection |
|---|
| `infra.aap_configuration` |
| `ansible.controller` | 
| `containers.podman` | 
| `community.general` |
| `ansible.platform`|
| `ansible.hub` |
| `ansible.eda` |
| `ansible.utils` |

To install the Collections, follow these steps:

1. Provide a valid token in the `ansible.cfg` file to download Collections from the Red Hat Public Automation Hub.
2. Install the Collections:

```bash
cd aap26-smoke-tests
ansible-galaxy collection install -r requirements.yml -p collections/
```

### 3. Configure the Credential file

To run the tests suite, two sets of credentials are required:

1. A Service Account, preferably with Admin permissions, to ensure the proper execution of the tests in playbooks `01` and `03` through `99`.
2. An Account cread on an external identity provider to ensure the proper execution of the tests run by playbook `02`.

Provide `credentials/credentials.yml` with the required AAP 2.6 and User Credentials:

```yaml
---
aap_hostname: "<AAP_HOSTNAME>"
aap_username: "<AAP_USERNAME>"
aap_password: "<AAP_PASSWORD>"
aap_validate_certs: false

ah_hostname: "{{ aap_hostname }}"
ah_username: "{{ aap_username }}"
ah_validate_certs: false

controller_hostname: "{{ aap_hostname }}"
controller_username: "{{ aap_username }}"
controller_password: "{{ aap_password }}"
controller_validate_certs: "{{ aap_validate_certs }}"

smoke_iam_username: "<IAM_USERNAME>"
smoke_iam_password: "<IAM_PASSWORD>"
```

Encrypt with Ansible Vault:

```bash
ansible-vault encrypt credentials/credentials.yml
```

### 4. Execution Environment (EE) Tar File

The EE test flow (i.e playbook `03_push_ee_to_hub.yml`) expects a **pre-built Execution Environment image** saved as a `.tar` file in the `templates/ee_definition/` directory.

**Current default configuration:**

| Setting | Value |
|---|---|
| Tar file path | `templates/ee_definition/smoke-reg-test-ee-minimal-rhel9.tar` |
| Image name on Hub | `smoke-reg-test-ee-minimal-rhel9` |
| Image tag | `latest` |

The tar file can be created from any existing EE image using:

```bash
podman save -o templates/ee_definition/<my-custom-ee>.tar <my-ee-image>:tag
```

**If you use a different EE file**, update the following three variables in `vars/main.yml`:

```yaml
smoke_ee_tar_path: "{{ playbook_dir }}/../templates/ee_definition/smoke-reg-test-ee-minimal-rhel9.tar"
smoke_ee_image_name: "smoke-reg-test-ee-minimal-rhel9"
smoke_ee_image_tag: "latest"
```

### Configure Variables

All configurable settings are in `vars/main.yml`:

#### Inventory Hosts

```yaml
smoke_inventory_hosts:
  - name: "<hostname>"
    ansible_host: "<ansible_host>"
    ansible_connection: "<ansible_connection>"
```

#### Project SCM

```yaml
smoke_project_scm_url: "https://<scm_url>"
smoke_project_scm_branch: "<scm_branch>"
```

#### Credential References (must exist on AAP)

Configure the following credentials and resources:

1. SCM Credentials for Project Synchronization (mandatory)
2. A list of Organization Galaxy Credentials (optional)
3. Job Template Playbook (mandatory)
4. A list of Credentials required by the Job Template (optional)

```yaml
smoke_project_scm_credential: "<your_scm_credential>"
smoke_org_galaxy_credentials:
  - "<your_org_galaxy_credential>"
smoke_jt_playbook: "<your_playbook>.yml"
smoke_jt_credentials: []
```

#### Execution Flags

Each test, and therefore each playbook, can be individually enabled (`true`) or disabled (`false`):

```yaml
run_platform_availability: true
run_iam_access_validation: true
run_push_ee_to_hub: true
run_register_ee: true
run_create_organization: true
run_project_sync: true
run_create_inventory: true
run_create_job_template: true
run_launch_job_template: true
run_create_workflow_template: true
run_launch_workflow_template: true
run_cleanup: true
```

#### Email Report

```yaml
smoke_report_email_enabled: true
smoke_report_smtp_host: "<your_smtp_host>"
smoke_report_smtp_port: <your_smtp_port>
smoke_report_email_from: "<your_email@example.com"
smoke_report_email_to:
  - "<your_recipient@example.com>"
smoke_report_email_subject: "AAP 2.6 Smoke & Regression Test Report"
```

### Customize Templates

Add or modify resource definitions in the `templates/` subfolders. Each YAML file uses the standard `controller_*` root key expected by the `infra.aap_configuration` collection. Files support Jinja2 templating and are automatically discovered by the playbooks.

## Automation Execution

### Run the Full Test Suite

```bash
ansible-playbook playbooks/smoke_test_master.yml -e @credentials/credentials.yml --ask-vault-pass
```

### Run a Single Test

Each playbook can be executed independently:

```bash
ansible-playbook playbooks/05_create_organization.yml -e @credentials/credentials.yml --ask-vault-pass
```

### Run with Selective Tests

Disable specific tests via extra vars:

```bash
ansible-playbook playbooks/smoke_test_master.yml \
  -e @credentials/credentials.yml \
  -e run_push_ee_to_hub=false \
  -e run_register_ee=false \
  --ask-vault-pass
```

### Run Cleanup Only

```bash
ansible-playbook playbooks/99_cleanup.yml -e @credentials/credentials.yml --ask-vault-pass
```

---

## Automation Overview

### Master Playbook Flow

The `smoke_test_master.yml` orchestrates the full test suite:

1. **Data Processing**: scans `templates/` subfolders, loads all YAML files via Jinja2 rendering, and maps each resource to the correct variable expected by the `infra.aap_configuration` roles.

2. **Sequential Test Execution**: imports each test playbook in dependency order. Variables set as host facts in the Data Processing phase persist across all imported playbooks. Each playbook that needs API access creates its own OAuth token at the start and deletes it at the end.

3. **Report Generation**: generates HTML summary report (`reports/YYYY-MM-DD/smoke_test_report.html`) with the tests details.

4. **Email Delivery**: optionally sends the HTML report via email.

### OAuth Token Lifecycle

Each playbook that requires an OAuth token manages its own token lifecycle:

- **Creates** a token at the start via `POST /api/controller/v2/tokens/`
- **Deletes** the token at the end via `DELETE` on the token URL
- No shared/long-lived tokens: each token exists only for the duration of a single playbook

Playbooks that use only basic authentication (01, 02, 03, 12, 14) do not create tokens.

### Dual-Mode Execution

Each test playbook supports two execution modes:

- **From Master** -- Template data is pre-loaded by the Data Processing phase; each playbook creates its own short-lived OAuth token.
- **Standalone** -- The playbook loads its own templates from the corresponding `templates/` subfolder and creates its own OAuth token. No dependency on the master playbook.

### Test Result Accumulation

Each playbook registers its result via `set_fact` into the `smoke_test_results` dictionary, which persists across all imported playbooks. The final report play reads this dictionary to render the HTML report.

## Expected Results

A successful full run produces:

- **All tests PASSED**
- **HTML summary report** generated at `reports/YYYY-MM-DD/smoke_test_report.html`
- **Email notification** sent (if enabled) with the HTML report as body
- **Resources created** on AAP 2.6: Organization, Project, Inventory, Execution Environment, Job Template, Workflow Template
- **Job and Workflow executions** completed successfully
- **All resources cleaned up** (if cleanup is enabled)