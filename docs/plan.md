# Project Requirements

It follows the list of requirements for my own distro.

## Core Architecture

- **Language**: Rust
- **Bootstrapping**: Runs on Arch ISO
- **Input**: Config file (flexible) OR lock file (reproducible)
- **Output**: Generates lock file for near-reproducibility
- **Extensions**: Fork-based, compile-time only

## User Experience

- **Dry-run mode**: Preview changes before applying
- **Progress reporting**: Real-time status updates
- **Config validation**: Validate config before execution

## Reliability & Safety

- **Rollback capabilities**: Automatic recovery on failure
- **Hash verification**: Verify package integrity
- **Arch snapshot support**: Frozen repository states for reproducibility

## Advanced Features

- **Multiple machine profiles**: Desktop, laptop, server, etc.

# Implementation phases

## Phase 1: MVP

- Config file parsing (TOML/YAML)
- Basic package installation via pacman
- Lockfile generation
- Dry-run mode
- Progress reporting

## Phase 2: Safety & Reproducibility

- Hash verification
- Rollback mechanisms
- Config validation
- Arch snapshot support

## Phase 3: Advanced Features

- Multiple machine profiles
- Compile-time extension system
- Enhanced error handling
