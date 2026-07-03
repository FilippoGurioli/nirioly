# Design

This file contains the design of each component exposed in the [architecture](./architecture.md) file, for the Provision Phase.

## Transactional Engine Module

A **command pattern** is used. Each command is represented by an interface with the following methods and properties:

- `name`: the name of the command in plain text (could be different from the real command launched)
- `description`: the description of what the command does
- `execute(ctx)`: method that executes the command, having injected the current context
- `rollback(ctx)`: method that reverts the changes performed in the execute method

### Concrete Commands

#### Data extraction command

- Name: data extraction
- Description: extracts `package.conf` and the `dotfiles/` directory from the binary, placing them inside the temp directory of the current OS.
- Rollback: deletes the extracted temp directory.

#### Packages provision command

- Name: packages provision
- Description: before installing anything, checks each target package's current installation status on the host and records which ones are not yet present. Then invokes the host package manager with system parameter flags (`--needed` `--noconfirm`) to synchronize and download the full package list (core packages + hardware-conditional packages).
- Rollback: uninstalls **only** the packages that were confirmed absent immediately before this run's install step and were installed by this run — never a package that was already present on the host beforehand, even if it's also part of the requested list. Removing something that predates this run risks breaking whatever already depended on it, which is a worse outcome than leaving a redundant package alone. Beyond that distinction, rollback is still best-effort: if something installed later in the same run came to depend on one of these newly-installed packages, removing it can break more than it fixes. The rollback should report packages it could not safely remove rather than force-removing them (see NFR-3).

#### Dotfiles link command

- Name: dotfiles link
- Description: invokes the linking engine (`stow --restow`) to map symlinks from the temporary store into the invoking user's home directory.
- Rollback: removes the symlinks created by this run (`stow -D`).

### Journal Registry

Takes as input a completed action (a `JournalEntry`) to record. Two things happen on `record()`:

1. The entry is pushed onto an in-memory LIFO stack, used to drive rollback if a later step in the *same run* fails.
2. The entry is appended as a line to a durable journal file on disk (`/tmp/nirioly/journal-<timestamp>.jsonl`), so the record of what completed survives a crash, even though this phase's rollback logic only acts within the current run.

### Transaction Manager

Orchestrates the launch of the commands along with their registration inside the journal. If an error occurs, it is responsible for triggering the rollback by draining the journal in reverse order.

### Hardware-conditional package selection

Rather than a generic string-based "rule" evaluated at runtime, conditional package groups are matched against hardware facts through a fixed, closed set of trigger kinds — one per hardware fact that can gate a package group: GPU vendor, form factor, battery presence, wifi adapter presence, bluetooth adapter presence, and CPU vendor. Each conditional group declares exactly one trigger. Matching is then a direct comparison against the corresponding field on the hardware facts — no string parsing, no rule-syntax validation, nothing that can be malformed at runtime:

- trigger "GPU vendor is NVIDIA" → check whether NVIDIA is present in the detected set of GPU vendors
- trigger "form factor is LAPTOP" → check whether the detected form factor equals LAPTOP
- trigger "has battery" → read the boolean fact directly
- trigger "has wifi adapter" / "has bluetooth adapter" → read the corresponding boolean fact directly
- trigger "CPU vendor is AMD" → check whether the detected CPU vendor equals AMD

The context builder walks every conditional group, keeps the ones whose trigger matches, and flattens their packages into the consolidated package list alongside the core packages.

Because the trigger kinds are a closed, finite set known up front, this can be implemented as a fixed enumeration matched exhaustively, rather than as free-form text interpreted at runtime — so an unhandled or misspelled trigger becomes something that's caught during development, not something that silently does nothing when the tool runs on someone's machine.

`ContextBuilder` walks the manifest's `conditionalGroups`, keeps the ones where `matches()` returns true, and flattens their `packages` into `consolidatedPackageArray` alongside `corePackages`.

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
        -List~JournalEntry~ historyStack
        -Path journalFilePath
        +record(JournalEntry entry) Result
        +getHistory() List~JournalEntry~
        -appendToDisk(JournalEntry entry) Result
    }

    class JournalEntry {
        +String commandName
        +DateTime timestamp
        +ExecutionOutcome outcome
    }

    class RuntimeContext {
        +Path baseTemporaryDirectory
        +Path homeDirectory
        +List~String~ consolidatedPackageArray
        +String administrativePassword
        +SystemHardwareFacts hardware
        +TelemetryHub telemetry
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

    class PackagesProvisionCommand {
        -List~String~ newlyInstalledPackages
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

    JournalRegistry "1" o-- "many" JournalEntry : tracks completed actions in LIFO stack

    Transaction <|.. DataExtractionCommand : realizes
    Transaction <|.. PackagesProvisionCommand : realizes
    Transaction <|.. DotfilesLinkCommand : realizes

    DataExtractionCommand ..> RuntimeContext : consumes configuration
    PackagesProvisionCommand ..> RuntimeContext : consumes configuration
    DotfilesLinkCommand ..> RuntimeContext : consumes configuration
```

## System Introspection Module

This module is responsible for providing information gathered from the bare metal. It should be extensible. It is the provider of the hardware facts consumed by the transaction engine's package-selection step.

```mermaid
classDiagram
    class ConfigurationLoader {
        +readEmbeddedManifest() PackageManifest
    }

    class HardwareDetector {
        +interrogateHost() SystemHardwareFacts
        -detectGpuVendors() Set~GpuVendor~
        -detectChassisType() FormFactor
        -detectBatteryPresence() Boolean
        -detectWifiAdapter() Boolean
        -detectBluetoothAdapter() Boolean
        -detectCpuVendor() CpuVendor
    }

    class SystemHardwareFacts {
        +Set~GpuVendor~ activeGpuVendors
        +FormFactor machineFormFactor
        +Boolean hasBattery
        +Boolean hasWifiAdapter
        +Boolean hasBluetoothAdapter
        +CpuVendor cpuVendor
    }

    class ContextBuilder {
        -ConfigurationLoader configLoader
        -PasswordResolver passwordResolver
        -HardwareDetector sysDetector
        +buildContext() RuntimeContext
        -selectConditionalPackages(PackageManifest manifest, SystemHardwareFacts facts) List~String~
    }

    class RuntimeContext {
        +Path baseTemporaryDirectory
        +Path homeDirectory
        +List~String~ consolidatedPackageArray
        +String administrativePassword
        +SystemHardwareFacts hardware
        +TelemetryHub telemetry
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

    class CpuVendor {
        <<enumeration>>
        INTEL
        AMD
        OTHER
    }
```

## Configuration Module

This module loads the embedded configuration and provides it to the context builder, and takes care of resolving the administrative password from either `.env` or an environment variable.

```mermaid
classDiagram
    class ConfigurationLoader {
        +readEmbeddedManifest() PackageManifest
    }

    class PasswordResolver {
        +resolvePassword() String
        -checkLocalDotEnvFile() Optional~String~
        -checkSystemEnvironment() Optional~String~
    }

    class ConditionalPackageGroup {
        +String groupName
        +PackageTrigger trigger
        +List~String~ packageList
    }

    class PackageTrigger {
        <<enumeration>>
        GPU_VENDOR
        FORM_FACTOR
        HAS_BATTERY
        HAS_WIFI_ADAPTER
        HAS_BLUETOOTH_ADAPTER
        CPU_VENDOR
    }

    class PackageManifest {
        +List~String~ corePackages
        +List~ConditionalPackageGroup~ conditionalGroups
    }

    class ContextBuilder {
        -ConfigurationLoader configLoader
        -PasswordResolver passwordResolver
        -HardwareDetector sysDetector
        +buildContext() RuntimeContext
        -selectConditionalPackages(PackageManifest manifest, SystemHardwareFacts facts) List~String~
    }

    class RuntimeContext {
        +Path baseTemporaryDirectory
        +Path homeDirectory
        +List~String~ consolidatedPackageArray
        +String administrativePassword
        +SystemHardwareFacts hardware
        +TelemetryHub telemetry
    }

    ContextBuilder --> ConfigurationLoader : utilizes
    ContextBuilder --> PasswordResolver : utilizes
    ContextBuilder --> HardwareDetector : utilizes
    ConfigurationLoader ..> PackageManifest : parses
    PackageManifest "1" o-- "many" ConditionalPackageGroup : organizes
    ConditionalPackageGroup --> PackageTrigger : matched against
    ContextBuilder ..> RuntimeContext : compiles matching arrays
```

`PackageTrigger` is shown here as a plain enumeration for diagram purposes; a real implementation would typically attach data to some of its variants (which GPU vendor, which CPU vendor) rather than matching by a string comparison — see the description under "Hardware-conditional package selection" above for the full behavior.

## System Execution Module

This module serves as the execution arm that communicates directly with the bare operating system. It isolates low-level subshell execution details and prevents messy subshell output from cluttering the master terminal console.

### Core Components

#### Command Runner

**Responsibility**: Spawns native system processes, executes commands inside the underlying operating system environment, and handles task execution status outcomes.

#### Stream Hijacker

**Responsibility**: Attaches directly to the subshell's standard output (stdout) and standard error (stderr) streams to intercept data chunks in real time, exposing these streams to downstream observers.

## Telemetry and Presentation Module

This module handles real-time visual progress reporting to the user and manages underlying raw diagnostic data persistence.

### Core Components

#### File Sink

**Responsibility**: Generates timestamped text files inside persistent temporary storage (`/tmp/nirioly/`) to write out the unfiltered, raw text streams passed by the execution modules. This is separate from the Journal Registry's own journal file — the File Sink captures raw subshell noise, while the journal file captures structured, completed-transaction records for rollback purposes.

#### Terminal Interface

**Responsibility**: Consumes progress ticks and execution metrics to render user-facing updates, leveraging a clean loading progress bar and contextual emojis.

```mermaid
classDiagram
    class CommandRunner {
        +runCommand(String program, List~String~ args) ExecutionStream
    }

    class ExecutionStream {
        +readNextChunk() Optional~StreamChunk~
        +waitForCompletion() ExecutionOutcome
    }

    class StreamChunk {
        +ChunkSource source
        +String textContent
    }

    class ChunkSource {
        <<enumeration>>
        STDOUT
        STDERR
    }

    class ExecutionOutcome {
        +Int exitCode
        +Boolean isSuccess
    }

    class TelemetryHub {
        -FileSink logSink
        -TerminalInterface view
        +monitorStream(ExecutionStream stream, String commandName) ExecutionOutcome
    }

    class FileSink {
        -Path sessionLogFile
        +writeLine(String line) void
    }

    class TerminalInterface {
        +initializeProgressBar(String label) void
        +tickProgress(Int percentage) void
        +printStatusMessage(String message, String emoji) void
    }

    class Transaction {
        <<interface>>
        +execute(RuntimeContext ctx) Result
        +rollback(RuntimeContext ctx) Result
    }

    class ConcreteCommand {
        +execute(RuntimeContext ctx) Result
    }

    Transaction <|.. ConcreteCommand
    ConcreteCommand ..> RuntimeContext : accesses telemetry & facts
    ConcreteCommand ..> CommandRunner : invokes
    RuntimeContext --> TelemetryHub : holds reference to

    CommandRunner ..> ExecutionStream : spawns
    ExecutionStream ..> StreamChunk : yields
    ExecutionStream ..> ExecutionOutcome : terminates with
    StreamChunk ..> ChunkSource : classifies
    TelemetryHub --> FileSink : utilizes
    TelemetryHub --> TerminalInterface : utilizes
    TelemetryHub ..> ExecutionStream : consumes
```
