# AAP 2.6 Smoke Regression Test Suite

Automated smoke and regression test suite for **Red Hat Ansible Automation Platform 2.6**, built with the `infra.aap_configuration` collection and Ansible Configuration as Code (CaC) principles.

The suite verifies that all core AAP 2.6 components are operational by creating, validating, and optionally cleaning up a full set of platform resources via API.

---

## Tests Overview

| Playbook | Test | Description |
|---|---|---|
| `01_platform_availability.yml` | Platform Availability and Login | Pings Gateway, Controller, Hub, and EDA endpoints; verifies authentication and instances |
| `02_build_custom_ee.yml` | Build Custom Execution Environment | Builds a custom EE image locally with `ansible-builder` and Podman |
| `03_push_ee_to_hub.yml` | Push EE to Private Automation Hub | Tags and pushes the EE image to the internal Private Automation Hub registry |
| `04_register_ee.yml` | Register EE on Controller | Registers the pushed EE on the Automation Controller |
| `05_create_organization.yml` | Create Organization | Creates an Organization with Galaxy Credentials |
| `08_create_credential_type.yml` | Create Credential Type | Creates a custom Credential Type with input/injector configuration |
| `06_create_credentials.yml` | Create Credentials | Creates all credentials (SCM, custom, machine, etc.) from template files |
| `07_project_sync.yml` | Project Synchronization | Creates a Project from a Git repository and waits for sync completion |
| `10_create_inventory.yml` | Create Inventory | Creates an Inventory with Hosts, Inventory Sources, and runs an Ad Hoc ping |
| `11_create_job_template.yml` | Create Job Template | Creates a Job Template with Extra Variables and Concurrent Jobs |
| `12_launch_job_template.yml` | Launch Job Template | Launches the Job Template and verifies successful execution |
| `13_create_workflow_template.yml` | Create Workflow Template | Creates a Workflow Job Template with a single node |
| `14_launch_workflow_template.yml` | Launch Workflow Template | Launches the Workflow and verifies all nodes complete successfully |
| `99_cleanup.yml` | Cleanup | Deletes all smoke test resources in reverse dependency order |

All tests can be executed sequentially via the **master playbook** (`smoke_test_master.yml`) or individually as standalone playbooks.

---

## Project Structure

```
aap-smoke-regression-test/
├── ansible.cfg                     # Ansible configuration (Galaxy servers, inventory plugins)
├── requirements.yml                # Ansible collection dependencies
├── README.md                       # This file
│
├── credentials/
│   └── credentials.yml             # AAP 2.6 connection credentials (encrypt with ansible-vault)
│
├── inventories/
│   └── localhost.yml               # Localhost inventory for local execution
│
├── playbooks/
│   ├── smoke_test_master.yml       # Master orchestrator (runs all tests sequentially)
│   ├── 01_platform_availability.yml
│   ├── 02_build_custom_ee.yml
│   ├── ...                         # Individual test playbooks
│   ├── 14_launch_workflow_template.yml
│   └── 99_cleanup.yml              # Resource cleanup (child-to-parent deletion)
│
├── templates/                      # Resource definitions (one subfolder per resource type)
│   ├── organizations/              # Organization definitions
│   ├── credential_types/           # Custom Credential Type definitions
│   ├── credentials/                # All credentials (SCM, custom, machine, etc.)
│   ├── projects/                   # Project definitions
│   ├── inventories/                # Inventory, Inventory Source, and Host definitions
│   ├── execution_environments/     # Execution Environment registrations
│   ├── job_templates/              # Job Template definitions
│   ├── workflow_job_templates/     # Workflow Job Template definitions
│   └── ee_definition/              # ansible-builder EE build files
│
├── reports/                        # Test reports (auto-generated per execution date)
│   └── YYYY-MM-DD/                 # Date-based subfolder
│       ├── 00_data_processing_report.md
│       ├── 01_platform_availability_report.md
│       ├── ...
│       └── 99_cleanup_report.md
│
├── collections/                    # Installed Ansible collections
├── roles/                          # Custom roles (currently empty)
├── vars/                           # Additional variables (currently empty)
└── tasks/                          # Shared task files (currently empty)
```

---

## Setup

### 1. Prerequisites

- **Ansible Core** 2.16+ installed on the control node
- **Python** 3.9+
- **Podman** and **ansible-builder** (only for the EE build test)
- Network access to the AAP 2.6 Gateway URL

### 2. Install Collections

```bash
cd aap-smoke-regression-test
ansible-galaxy collection install -r requirements.yml -p collections/
```

### 3. Configure Credentials

Edit `credentials/credentials.yml` with your AAP 2.6 Gateway connection details:

```yaml
aap_hostname: "your-aap-gateway.example.com"
aap_username: "admin"
aap_password: "your-password"
aap_validate_certs: false
```

Encrypt with Ansible Vault:

```bash
ansible-vault encrypt credentials/credentials.yml
```

### 4. Customize Templates

Add or modify resource definitions in the `templates/` subfolders. Each YAML file uses the standard `controller_*` root key expected by the `infra.aap_configuration` collection:

```yaml
# templates/credentials/my_new_credential.yml
---
controller_credentials:
  - name: "My New Credential"
    organization: "My_Organization"
    credential_type: "Machine"
    inputs:
      username: "{{ my_username }}"
      password: "{{ my_password }}"
    state: present
```

New files are automatically discovered and processed by the playbooks.

---

## Usage

### Run the Full Test Suite

Ensure all sensitive variables (`scm_username`, `scm_password`, etc.) are defined in `credentials/credentials.yml` and the file is encrypted with `ansible-vault encrypt credentials/credentials.yml`.

```bash
ansible-playbook playbooks/smoke_test_master.yml -e @credentials/credentials.yml --ask-vault-pass
```

### Run a Single Test

Each playbook can be executed independently:

```bash
ansible-playbook playbooks/05_create_organization.yml -e @credentials/credentials.yml --ask-vault-pass
```

### Run Cleanup

Delete all resources created by the smoke tests:

```bash
ansible-playbook playbooks/99_cleanup.yml -e @credentials/credentials.yml --ask-vault-pass
```

---

## How It Works

### Master Playbook Flow

The `smoke_test_master.yml` orchestrates the full test suite:

1. **Data Processing** -- Scans `templates/` subfolders, loads all YAML files, and maps each resource to the correct variable expected by the `infra.aap_configuration` roles.

2. **OAuth Token** -- Obtains an OAuth token from the AAP 2.6 Gateway API for authenticated operations.

3. **Sequential Test Execution** -- Imports each test playbook in dependency order. Variables set as host facts in the Data Processing phase persist across all imported playbooks.

### Dual-Mode Execution

Each test playbook supports two execution modes:

- **From Master** -- Template data is pre-loaded by the Data Processing phase; the OAuth token is already available as a host fact.
- **Standalone** -- The playbook loads its own templates from the corresponding `templates/` subfolder and creates its own OAuth token.

### Template Loading

Templates are loaded using `lookup('template', ...)` which evaluates Jinja2 expressions (e.g., `{{ scm_username }}`) before YAML parsing. This allows credentials and dynamic values to be injected at runtime via extra vars.

---

## Expected Results

A successful full run produces:

- **All tests PASSED** with exit code `0`
- **Reports** generated in `reports/YYYY-MM-DD/` as Markdown files
- **Resources created** on AAP 2.6: Organization, Credential Types, Credentials, Project, Inventory, Execution Environment, Job Template, Workflow Template
- **Job and Workflow executions** completed successfully

Each report contains:
- Test name and execution date
- Pass/Fail status with verification details
- Resource type and configuration summary
- Verification table confirming the resource exists on AAP

### Sample Report Output

```markdown
# Create Organization

**Date:** 2026-09-25T07:34:21Z
**Result:** PASSED

## Resources Created

### Organization: Smoke_Test_Organization

| Field | Value |
|---|---|
| **Resource Type** | Organization |
| **Name** | Smoke_Test_Organization |
| **Description** | Organization created by AAP 2.6 Smoke Regression Test |
| **Galaxy Credentials** | Automation Hub RH Certified Repository, ... |

## Verification

| Organization | Found on AAP |
|---|---|
| Smoke_Test_Organization | Yes (count: 1) |
```

---

## Collection Compatibility

This project is built for `infra.aap_configuration` **v4.10.0+** which uses the `aap_*` variable naming convention for some roles. Key variable mappings:

| Role | Loop Variable | Auth Variable |
|---|---|---|
| `controller_organizations` | `aap_organizations` | `aap_token` |
| `controller_credentials` | `controller_credentials` | `aap_token` |
| `controller_credential_types` | `controller_credential_types` | `aap_token` |
| `controller_projects` | `controller_projects` | `aap_token` |
| `controller_inventories` | `controller_inventories` | `aap_token` |
| `controller_hosts` | `controller_hosts` | `aap_token` |
| `controller_job_templates` | `controller_templates` | `aap_token` |
| `controller_workflow_job_templates` | `controller_workflows` | `aap_token` |
| `controller_execution_environments` | `controller_execution_environments` | `aap_token` |
