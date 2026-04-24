# Azure DevOps

Azure DevOps hosts the git repositories, build/test/release pipelines, and the artifact feed that ties library, machine, and deployment repos together.

## Build, Test & Release Pipelines

A single pipeline covers build, test, and release for a PLC or library project. It is driven by parameters so that one pipeline file handles all the project variants.

```d2
direction: down

test_build: "Test build" {
  a: "Clone library + machine repos"
  b: "Resolve TwinCAT libraries"
  c: "Build runtime package"
  a -> b -> c
}

variant_test: "variant = UnitTest" {
  shape: oval
}
variant_test -> test_build.c

run_tests: "Run tests" {
  a: "Deploy to test PLC"
  b: "Collect TcUnit results"
  a -> b
}

release_build: "Release build" {
  a: "Clone library + machine repos"
  b: "Resolve TwinCAT libraries"
  c: "Build runtime package"
  a -> b -> c
}

variant_release: "variant = VariantB" {
  shape: oval
}
variant_release -> release_build.c

publish: "Publish" {
  a: "Push package to artifact feed"
}

test_build -> run_tests
run_tests -> release_build
release_build -> publish
```

### Stages

1. **Build for test**
    - Runs a PowerShell script `BuildTwinCATProject.ps1` that replicates the manual process of building a TwinCAT project.
    - Uses the project variant specified for testing (e.g. `UnitTest`).
    - Runs on a Windows 10 VM with the Azure agent installed and a full TwinCAT XAE development environment.
2. **Test**
    - Deploys the build-for-test output onto a target Beckhoff IPC.
    - Expects a **TcUnit unit test result file** to be generated.
    - Virtual commissioning (VC) integration tests are currently handled by separate pipelines; we may fold them into this pipeline in the future.
3. **Build for release**
    - Same as build-for-test, except the project variant used for the machine is specified.
4. **Publish to artifact feed**
    - Publishes either a **library package** (`.nupkg` containing the `.library` file) or a **runtime package** (`.nupkg` following the chocolatey specification, containing the runtime files).
    - The `.library` nuget file is currently not consumed by downstream pipelines. The plan is to switch to [twinpack](https://github.com/Zeugwerk/Twinpack) at some point - at which point this package would replace the TwinCAT library repository folder.

### Example pipeline file

We keep one top-level pipeline per PLC project. Each one just fixes a couple of project-specific values and extends a shared template.

```yaml
name: $(Date:yyyyMMdd).$(Rev:r)-${{ parameters.variant }}

trigger: none

parameters:
  - name: variant
    displayName: Project variant
    type: string
    default: VariantA
    values: [VariantA, VariantB, UnitTest, Simulation]

  - name: runTests
    displayName: Run unit tests
    type: boolean
    default: true

  - name: publishArtifact
    displayName: Publish to artifact feed
    type: boolean
    default: false

resources:
  repositories:
    - repository: machineRepo
      type: git
      name: controls-machine-type-a
      ref: main
    - repository: libraryRepo
      type: git
      name: controls-library
      ref: main

extends:
  template: templates/plc-build.yml
  parameters:
    projectName: MachineTypeAVersion1
    solution: projects/v1/MachineTypeAVersion1/MachineTypeAVersion1.sln
    plcTarget: TwinCAT RT (x64)
    variant: ${{ parameters.variant }}
    testVariant: UnitTest
    packageKind: Runtime
    runTests: ${{ parameters.runTests }}
    publishArtifact: ${{ parameters.publishArtifact }}
    libraryPaths:
      - $(Pipeline.Workspace)/libraryRepo/repository
      - $(Pipeline.Workspace)/machineRepo/repository
```

### `build_plc_package.yml` - build step

`build_plc_package.yml` contains all of the steps to build a project. The snippet below calls the universal `BuildTwinCATProject.ps1` script, which uses Automation Interface to build the project and output either a `.library` file or the Boot folder.

```yaml
- task: PowerShell@2
  displayName: Build TwinCAT project
  condition: ${{ eq(parameters.runTests, true) }}
  inputs:
    pwsh: true
    filePath: $(Pipeline.Workspace)/deployments/automation/twincat/BuildTwinCATProject.ps1
    arguments:
      -Solution "$(Pipeline.Workspace)/machineRepo/${{ parameters.solution }}"
      -Project "${{ parameters.projectName }}"
      -Output "$(Build.ArtifactStagingDirectory)"
      -Variant "${{ parameters.testVariant }}"
      -Platform "${{ parameters.plcTarget }}"
      -LibraryPaths ${{ parameters.libraryPaths }}
- publish: $(Build.ArtifactStagingDirectory)
  artifact: plc-build-output
```

### `build_plc_package.yml` - packaging

The same file packages the build output as either a chocolatey package or a nuget library file.

```yaml
- task: PowerShell@2
  displayName: Build runtime chocolatey package
  condition: eq('${{ parameters.packageKind }}', 'Runtime')
  inputs:
    pwsh: true
    filePath: $(Pipeline.Workspace)/deployments/automation/twincat/New-PlcChocoPackage.ps1
    arguments:
      -Id "${{ parameters.projectName }}-${{ parameters.variant }}"
      -Version "$(version)"
      -SourceZip "$(Build.ArtifactStagingDirectory)/${{ parameters.projectName }}.zip"
      -Output "$(Build.ArtifactStagingDirectory)"

- task: PowerShell@2
  displayName: Build library nuget package
  condition: eq('${{ parameters.packageKind }}', 'Library')
  inputs:
    pwsh: true
    filePath: $(Pipeline.Workspace)/deployments/automation/twincat/New-LibraryPackage.ps1
    arguments:
      -Id "${{ parameters.projectName }}"
      -Version "$(version)"
      -SourceLibrary "$(Build.ArtifactStagingDirectory)/${{ parameters.projectName }}.library"
      -Output "$(Build.ArtifactStagingDirectory)"
```

## Deploy pipeline

The deploy pipeline uses an Ubuntu VM running Ansible to deploy a runtime (boot) executable package onto a target Beckhoff IPC. It uses the chocolatey format.

See [Ansible → PLC deployment](../ansible/index.md#plc-deployment) for the Ansible task details.

## Agents

We use on-premise agents for various parts of the DevOps infrastructure.

| Agent              | Type            | Purpose                                                                                                               |
| ------------------ | --------------- | --------------------------------------------------------------------------------------------------------------------- |
| `controls-build-agent-1` | Windows VM    | XAE installed + Azure agent. Handles all pipelines that require TwinCAT XAE. We build here but don't run using XAR.   |
| `test-plc-1`             | Beckhoff IPC  | Fully licensed TwinCAT PLC + Azure agent. Receives pre-built PLC artifacts, runs TcUnit, collects unit test results.  |
| `ansible-agent-1`        | Ubuntu VM     | Handles all on-premise Ansible tasks. |


## BuildTwinCATProject.ps1

This is a powershell script that calls into a separate CLI that I created that wraps around Automation Interface. Although I wrote mine from scratch, borrowing heavily from online examples and resources, you should be able to use an agent to quickly re-create the same functionality.

> Hey Claude, create a .NET 10 CLI using https://www.nuget.org/packages/Beckhoff.TwinCAT.TCatSysManager/ and the DTE to provide functions for building a TwinCAT 3 project. Refer to online documentation from Beckhoff, ensure that you implement a COM message filter, and write unit tests that check each feature we develop against a real TwinCAT 3 XAE project. Refer to Infosys for each feature that you create before creating it. Features required include:
>
>  - Load TwinCAT 3 solution
>  - Build solution
>  - Traverse PLC projects
>  - Build a single PLC project
>  - Read and modify PLC project version
>  - Run Static Analysis
>  - Run Check All Objects
>  - Add library repository
>  - Install library
>  - Update library version
>  - Save as library
