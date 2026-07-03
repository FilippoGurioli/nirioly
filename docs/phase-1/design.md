# Design

This file will contain all designs of each component exposed in the [architecture](./architecture.md) file.

## Transactional Engine Module

A **command pattern** will be used. Each command will be represented by an interface with the following methods and properties:

- `name`: the name of the command in plain text (could be different from the real command launched)
- `description`: the description of what does the command do
- `execute(ctx)`: method that executes the command, having injected the current context
- `rollback(ctx)`: method that reverts the changes performed in the execute method

### Concrete Commands

#### Data extraction command

- Name: data extraction
- Description: extracts `packages.conf` file and `dotfiles/` directory from the binary, placing them inside the temp directory of the current OS.

#### Add user command

- Name: add user
- Description: invokes system hooks to create the account, generate the home directory structure, and append security rules to the host's privilege configuration file (`sudoers`).

#### Packages provision command

- Name: packages provision
- Description: Invokes the host package manager with system parameters flags (`--needed` `--noconfirm`) to synchronize and download the package list

#### Dotfiles link command

- Name: dotfiles link
- Description: Invokes the linking engine (`stow --restow`) to map symlinks from the temporary store into the active environment.

### Journal Registry

It takes as input an action that will be tracked in the journal.

### Transaction Manager

It orchestrates the launch of the commands along with their registration inside the journal. If ever an error occurs, it is responsible of triggering the rollback.

### Class Diagram

```mermaid
classDiagram
    %% Core Orchestration Entities
    class TransactionManager {
        -List~Transaction~ transactionQueue
        -JournalRegistry journal
        -RuntimeContext context
        +executePlan() Result
        -handleRollback() void
    }

    class JournalRegistry {
        -List~Transaction~ historyStack
        +record(Transaction cmd) void
        +getHistory() List~Transaction~
    }

    class RuntimeContext {
        +String administrativePassword
        +String targetUsername
        +Path baseTemporaryDirectory
        +List~String~ customPackages
        +SystemHardwareFacts hardware
    }

    %% The Behavioral Interface Contract
    class Transaction {
        <<interface>>
        +String name
        +String description
        +execute(RuntimeContext ctx) Result
        +rollback(RuntimeContext ctx) Result
    }

    %% Concrete Task Implementations
    class DataExtractionCommand {
        +execute(RuntimeContext ctx) Result
        +rollback(RuntimeContext ctx) Result
    }

    class AddUserCommand {
        +execute(RuntimeContext ctx) Result
        +rollback(RuntimeContext ctx) Result
    }

    class PackagesProvisionCommand {
        +execute(RuntimeContext ctx) Result
        +rollback(RuntimeContext ctx) Result
    }

    class DotfilesLinkCommand {
        +execute(RuntimeContext ctx) Result
        +rollback(RuntimeContext ctx) Result
    }

    %% Structural Relationships & Dependencies
    TransactionManager "1" o-- "many" Transaction : hosts execution queue
    TransactionManager "1" --> "1" JournalRegistry : records execution events into
    TransactionManager ..> RuntimeContext : passes down configuration
    
    JournalRegistry "1" o-- "many" Transaction : tracks completed actions in LIFO stack
    
    Transaction <|.. DataExtractionCommand : realizes
    Transaction <|.. AddUserCommand : realizes
    Transaction <|.. PackagesProvisionCommand : realizes
    Transaction <|.. DotfilesLinkCommand : realizes
    
    DataExtractionCommand ..> RuntimeContext : consumes configuration
    AddUserCommand ..> RuntimeContext : consumes configuration
    PackagesProvisionCommand ..> RuntimeContext : consumes configuration
    DotfilesLinkCommand ..> RuntimeContext : consumes configuration
```

## System Introspection Module

This module is responsible of providing information grasped from the bare metal. It should be able to be extensible as much as wanted. It will be the provider of the context passed to the transaction engine.

```mermaid
classDiagram
    class ConfigurationLoader {
        +readEmbeddedPackages() List~String~
        +resolveAdministrativePassword() String
    }

    class HardwareDetector {
        +interrogateHost() SystemHardwareFacts
        -detectGpuVendors() Set~GpuVendor~
        -detectChassisType() FormFactor
        -detectStorageType() DriveType
    }

    class SystemHardwareFacts {
        +Set~GpuVendor~ activeGpuVendors
        +FormFactor machineFormFactor
        +DriveType primaryDriveType
        +Boolean hasBattery
    }

    class ContextBuilder {
        -ConfigurationLoader configLoader
        -HardwareDetector sysDetector
        +buildContext() RuntimeContext
    }

    class RuntimeContext {
        +String administrativePassword
        +String targetUsername
        +Path baseTemporaryDirectory
        +List~String~ customPackages
        +SystemHardwareFacts hardware
    }

    ContextBuilder --> ConfigurationLoader : utilizes
    ContextBuilder --> HardwareDetector : utilizes
    HardwareDetector ..> SystemHardwareFacts : assembles
    ContextBuilder ..> RuntimeContext : constructs

    class GpuVendor {
        <<enumeration>>
        AMD
        INTEL
        NVIDIA
        GENERIC
    }

    class FormFactor {
        <<enumeration>>
        LAPTOP
        DESKTOP
        VIRTUAL_MACHINE
    }

    class DriveType {
        <<enumeration>>
        SOLID_STATE
        ROTATIONAL
    }

```

## Configuration Module

This module is responsible of loading configuration file and provide its information to the context builder. It should also take care of lading sensible information from either `.env` or envvars.

```mermaid
classDiagram
    class ConfigurationLoader {
        +readEmbeddedManifest() PackageManifest
    }

    class SecretsResolver {
        +resolveAdministrativePassword() String
        -checkLocalDotEnvFile() Optional~String~
        -checkSystemEnvironment() Optional~String~
    }

    class PackageProfile {
        +String profileName
        +String ruleCondition
        +List~String~ packageList
    }

    class PackageManifest {
        +List~String~ corePackages
        +List~PackageProfile~ structuralProfiles
    }

    class ContextBuilder {
        -ConfigurationLoader configLoader
        -SecretsResolver secretsResolver
        -HardwareDetector sysDetector
        +buildContext() RuntimeContext
        -evaluateCondition(String rule, SystemHardwareFacts facts) Boolean
    }

    class RuntimeContext {
        +String administrativePassword
        +String targetUsername
        +Path baseTemporaryDirectory
        +List~String~ consolidatedPackageArray
    }

    ContextBuilder --> ConfigurationLoader : utilizes
    ContextBuilder --> SecretsResolver : utilizes
    ContextBuilder --> HardwareDetector : utilizes
    ConfigurationLoader ..> PackageManifest : parses
    PackageManifest "1" o-- "many" PackageProfile : organizes
    ContextBuilder ..> RuntimeContext : compiles matching arrays
```
