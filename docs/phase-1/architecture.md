# Architecture

The project will have 5 core components:

## Configuration Module

**Responsibility**: Ingests the baseline instructions and processes parameters required for runtime.

**Core Components**:

- **Asset Loader**: Bakes the static package configuration strings and the entire dotfiles folder payload directly into the binary compile unit.

- **Secret Manager**: Queries the environment state or reads local variables to resolve sensitive parameters like primary administrative passwords securely.

- **Outputs**: Generates a global Runtime Context data structure used by downstream execution modules.

## System Introspection Module

**Responsibility**: Interrogates the active host hardware to make real-time setup adjustments.

**Core Components**:

- **Hardware Detector**: Examines the host system metadata at runtime to identify the active GPU vendor (AMD, Intel, or NVIDIA).

- **Outputs**: Generates a System Profile indicating the exact graphical driver targets that must be merged into the installation stream.

## Transactional Engine Module

**Responsibility**: The core coordinator of system transformations and lifecycle safety.

**Core Components**:

- **Journal Stack**: An append-only memory registry that logs successfully executed operations. It behaves as a Last-In, First-Out (LIFO) memory structure.

- **Transaction Interface**: A strict structure requirement where any operation must natively possess two methods: a forward execution instruction and a corresponding reverse rollback instruction.

- **Concrete Actions**: Individual implementation blocks satisfying the core transaction structure:
  - **Add User Command**: Manages system account initialization and permissions updates.
  - **Package Provision Command**: Installs target system dependencies.
  - **Dotfile Link Command**: Mounts user configurations to target pathways.

## System Execution Module

**Responsibility**: The execution arm that communicates directly with the bare operating system.

**Core Components**:

- **Command Runner**: Spawns native shell processes (useradd, pacman, stow).

- **Stream Hijacker**: Intercepts low-level subshell standard output and error channels in real time, keeping the master console empty.

## Telemetry & Presentation Module

**Responsibility**: Handles reporting to the user and underlying system logging.

**Core Components**:

- **File Sink**: Generates persistent, timestamped text files inside temporary storage (/tmp/) to capture the unfiltered output of every executed subshell command.

- **Terminal Interface**: Intercepts status changes emitted by the transactional engine, translating them into status emojis and a fluid progress bar.

## Diagrams

### Module Structure

```mermaid
graph TD
    %% Define Styles & Palette
    classDef config fill:#2d3748,stroke:#4a5568,stroke-width:2px,color:#fff;
    classDef engine fill:#1a365d,stroke:#2b6cb0,stroke-width:2px,color:#fff;
    classDef runner fill:#2c5282,stroke:#4299e1,stroke-width:2px,color:#fff;
    classDef output fill:#234e52,stroke:#319795,stroke-width:2px,color:#fff;

    %% Module Structure
    subgraph M_Config [Configuration Module]
        A[Application Entrypoint] --> B[Configuration Loader]
        B -->|Embeds static assets| C[Package Blueprint & Dotfiles Payload]
        B -->|Resolves credentials| D[Administrative Secrets]
    end

    subgraph M_Sys [System Introspection Module]
        E[Hardware Detector] -->|Queries GPU Vendor Metadata| F[System Profile]
    end

    subgraph M_Tx [Transaction Module]
        G[Transaction Manager] <-->|Coordinates LIFO Stack| H[Journal Registry]
        G -->|Triggers Next Operation| I[Transaction Interface]
        G -.->|Triggers Rollback Operation| I
        I -->|Implementation| J[Add User Command]
        I -->|Implementation| K[Package Provision Command]
        I -->|Implementation| L[Dotfile Link Command]
    end

    subgraph M_Exec [System Execution Module]
        M[Command Runner] -->|Invokes Operating System Hooks| N[OS Subshell Processes]
    end

    subgraph M_UI [Telemetry & Presentation Module]
        O[Telemetry Hub] -->|Aggregates Streams| P[Persistent File Log]
        O -->|Renders Real-Time UI| Q[Terminal Interface Progress & Emojis]
    end

    %% Lifecycle Logic & Data Flow Connections
    B & F -->|Assembles Master Execution Plan| G
    I -->|Directs Subshell Actions| M
    
    %% Output telemetry streams out passively
    M -->|Pipes Raw Streams & Progress Ticks| O
    
    %% Success/Failure Pivots return to the engine core
    M -->|Returns Execution Outcome| I
    I -->|Success/Failure| G
    H -.->|Drains Log in Reverse Order| G

    %% Apply Classes
    class A,B,C,D config;
    class E,F config;
    class G,H,I,J,K,L engine;
    class M,N runner;
    class O,P,Q output;
```

### Core Flow Diagram

```mermaid
flowchart TD
    A[Submit Transaction to Queue] --> B(Execute Forward Component)
    B --> C{Success?}
    C -->|Yes| D[Commit to Journal]
    C -->|No| E[Trigger Rollback State]
    D --> F[Process Next Transaction]
    E --> G[Drain Journal Backward]
```
