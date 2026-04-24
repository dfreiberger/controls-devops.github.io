# Ansible

Ansible handles provisioning and configuration of Beckhoff IPCs, runtime targets, and supporting infrastructure, plus scheduled backups and PLC code deployment.

## Deployment repository: `controls-deployments`

A single repo holds all DevOps code: Ansible, pipeline definitions, automation scripts, and the TwinCAT CLI wrapper.

```text
controls-deployments/
├── ansible/                      # Standard Ansible structure
│   ├── group_vars/
│   ├── inventory/
│   ├── roles/
│   └── …
├── automation/
│   ├── chocolatey/
│   ├── pipelines/
│   │   ├── backup_test_environment.yml      # Scheduled backups of machines via ansible playbooks - JSON (test environment), FTP (lab machines)
│   │   ├── backup_staging_environment.yml
│   │   ├── backup_prod_environment.yml
│   │   ├── build_machine_type_a_version_1.yml   # Build, test, and optionally publish the PLC program to the Azure artifact feed
│   │   ├── build_machine_type_a_version_2.yml
│   │   ├── test_core_library.yml                # Build + run TcUnit on the library; PR validation
│   │   ├── test_machine_type_a_version_1.yml    # Build + run TcUnit on the machine; PR validation (future: VC tests)
│   │   ├── test_machine_type_a_version_2.yml
│   │   ├── deploy_machine_type_a_version_1.yml  # Deploy PLC code via Ansible (also HMI, PLM, other config)
│   │   └── deploy_machine_type_a_version_2.yml
│   ├── scripts/
│   └── twincat/
│       ├── cli/                     # .NET CLI wrapper for TwinCAT Automation Interface
│       ├── BuildTwinCATProject.ps1       # PowerShell that calls the CLI and builds the project
│       ├── New-PlcChocoPackage.ps1       # Creates a nuget chocolatey package from the PLC boot folder (`choco install <PLCProject> --version <version>`)
│       └── New-LibraryPackage.ps1        # Creates a nuget package wrapping the compiled `.library` file
└── .gitignore
```

## Scope

Currently, we use Ansible to manage the following aspects of our infrastructure:

- **Creation of OS images for Beckhoff IPCs**, which are then exported to TIB format using Acronis.
- **Installation and configuration of most software on the Beckhoff IPCs**, including:
    - Network adapter configuration
    - Driver configuration
    - SSH installation and configuration
    - PowerShell installation
    - TwinCAT 3 installation
    - TwinCAT 3 function installation and configuration, including OPC-UA
    - PLC code download
    - PLC settings backup
- **Installation and configuration of most machine and supporting infrastructure software**, including:
    - Ignition SCADA
    - PostgreSQL
    - InfluxDB
    - Grafana (including dashboard management in the future)
    - Nginx
- **Backup of robot controllers**
- **Backup of PLCs**

For certain Ansible playbooks we also create Azure DevOps pipelines that run the playbook - for example to back up robot controllers on a schedule, or to update PLC code.

## PLC deployment

The deploy pipeline runs an Ansible playbook on the Ubuntu VM that effectively executes:

```text
choco install <package name> --source <azure artifact feed>
```

on the instrument.

```yaml
- name: Register the Azure artifact feed as a chocolatey source
  win_chocolatey_source:
    name: plc-feed
    source: "{{ artifact_feed_url }}"
    source_username: "{{ artifact_feed_user }}"
    source_password: "{{ artifact_feed_token }}"
    state: present

- name: Install PLC runtime package
  win_chocolatey:
    name: "{{ plc_package }}"
    version: "{{ plc_version }}"
    source: plc-feed
    state: present
    force: true
```

The chocolatey package is a zip containing the `.nuspec` file, the boot folder in zip format, and a `chocolateyInstall.ps1` script that:

1. Puts the PLC into config mode using the ADS PowerShell library:

    ```powershell
    Restart-TwinCAT -Command Reconfig -Force -Session $session
    ```

2. Backs up the boot data file.
3. Copies the new boot folder contents to `C:\TwinCAT\3.1\Boot`.
4. Restarts the runtime.

!!! info
    There are a lot of ways to handle code deployment to the PLC. You can just copy the files to the boot folder, or you could have them available and have the HMI handle the swap out of the program. Our method is convenient because we can use chocolatey everywhere to handle the installation of PLC programs and other programs such as TwinCAT XAR itself.
