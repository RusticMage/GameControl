# Game Control

**Distributed Game Hosting & Stateful Workload Migration Platform**

Game Control is a distributed platform designed to separate **persistent game-world state** from the **computer currently running the game server**.

In many co-op games, one player's computer acts as the multiplayer host. When that player shuts down their computer, the world becomes unavailable to everyone else.

Game Control explores a different architecture:


                    ┌──────────────────────┐
                    │     Game Control     │
                    │         Cloud        │
                    │                      │
                    │ World State          │
                    │ Versioning           │
                    │ Host Leases          │
                    │ Synchronization      │
                    │ Metadata             │
                    └──────────┬───────────┘
                               │
                        Latest World
                               │
              ┌────────────────┴────────────────┐
              │                                 │
              ▼                                 ▼
      ┌───────────────┐                 ┌───────────────┐
      │   Player A    │                 │   Player B    │
      │               │                 │               │
      │ Game Server   │                 │ Game Server   │
      │               │                 │               │
      └───────┬───────┘                 └───────┬───────┘
              │                                 │
           Players                           Players


The cloud maintains persistent world state while the actual game-server workload runs on an authorized player's computer.

When the active host leaves, another eligible player can acquire the host lease, synchronize the latest state, and continue the session.



# The Core Problem

Traditional peer-hosted multiplayer often couples two separate concerns:


Game World
     +
Game Server
     +
Host Computer


This creates a dependency:


             Player A
                │
          Game Server
                │
             World
                │
        ┌───────┴───────┐
        ▼               ▼
    Player B         Player C


If A's computer disappears, the game world effectively disappears with it.

Game Control separates these concerns.


                 Persistent State
                       │
                       ▼
                 Game Control
                     Cloud
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Player A     Player B     Player C
       Server       Server       Server
       workload     workload     workload


Only one player runs the active game-server workload at a time.

The persistent state survives independently of that player's machine.



# Core Principle


ONE WORLD
    │
    ▼
ONE ACTIVE HOST
    │
    ▼
MULTIPLE PLAYERS


Game Control does not attempt to make multiple hosts simultaneously modify the same persistent world.

Instead, ownership of the active game-server workload can move between authorized machines.


OLD HOST
    │
    ▼
SAVE
    │
    ▼
SYNCHRONIZE
    │
    ▼
VERSION COMMIT
    │
    ▼
HOST LEASE RELEASE
    │
    ▼
NEW HOST
    │
    ▼
GAME SERVER



# Host Migration

Suppose Player A is hosting but has poor connectivity while Player B has a better connection.

The active host can be transferred:


A is hosting
     │
     ▼
A requests migration
     │
     ▼
B accepts
     │
     ▼
Synchronize latest world
     │
     ▼
Commit latest version
     │
     ▼
Release A's host lease
     │
     ▼
B acquires host lease
     │
     ▼
B starts game server
     │
     ▼
Players reconnect


A short interruption is expected.

The objective is safe, predictable migration rather than zero downtime.



# Distributed Host Lease

A world has exactly one active host.

The control service maintains a lease containing information such as:


World ID
Host ID
Lease ID
Expiration
Last Heartbeat
World Version


The host periodically sends heartbeats.

If the host fails or becomes unreachable, its lease eventually expires.

Another eligible player can then acquire hosting.

This prevents two independent machines from simultaneously believing that they own the world.



# Host Selection

Multiple players may be willing to host.

A server-side queue can determine who receives the next opportunity.


Current Host = D

Waiting:
    A
    B
    C

D releases lease
       │
       ▼
A receives opportunity
       │
       ├── unavailable
       ▼
B receives opportunity
       │
       ├── unavailable
       ▼
C receives opportunity

This is a distributed coordination problem rather than a shared-memory mutual exclusion problem.




# Persistent World Versioning

Every successful synchronization produces a new world version.


v100 → A
v101 → A
v102 → B
v103 → B
v104 → C


Previous versions can be retained for:

* rollback
* recovery
* debugging
* corruption detection
* migration recovery

The world therefore behaves more like versioned state than a single mutable file.



# Atomic Synchronization

A new version does not become active until synchronization succeeds completely.


Current = v104

        B
        │
        ▼
 Upload v105
        │
        ▼
Upload chunks
        │
        ▼
Verify hashes
        │
        ▼
Validate manifest
        │
        ▼
Commit v105


If synchronization fails:


Active = v104


The incomplete version cannot replace the last valid state.



# Incremental Synchronization

Large worlds do not necessarily need to be transferred in their entirety.

Persistent data can be divided into chunks.


World v10

Chunk A
Chunk B
Chunk C
Chunk D


After gameplay:


Chunk A → unchanged
Chunk B → changed
Chunk C → unchanged
Chunk D → changed


Only B and D need to be transferred.

Content hashes can identify identical chunks and provide the foundation for content-addressable storage and deduplication.



# Compression

Chunks may be compressed before transfer.

Game Control can use a streaming compression algorithm such as **Zstandard (zstd)**.

Metadata can include:

Original Size
Compressed Size
Compression Algorithm
Content Hash


If compression does not provide meaningful savings, the system can retain the uncompressed representation.



# Integrity and Recovery

Game Control uses multiple validation layers.

### Transport Integrity

Cryptographic hashes such as SHA-256 can verify transferred data.

### Manifest Validation

The system can verify:

* required chunks exist
* chunk hashes match
* sizes are correct
* metadata is internally consistent

### Game-Specific Validation

Optional game adapters can perform additional validation when the save format is understood.

The core platform does not require access to a game's proprietary engine.



# Game Adapters

Game Control is designed around a generic synchronization layer rather than being hard-coded to one game.

A game adapter can define:


Game
 │
 ├── Save location
 ├── Server executable
 ├── Startup command
 ├── Shutdown procedure
 ├── Save detection
 └── Optional validation


This allows the core distributed system to remain independent of individual games.

Potential future adapters could support different games and different server architectures.



# Game Software Distribution

A future version of Game Control may also distribute the software required to run a particular game-server workload.

For example:


Game Control Cloud
       │
       ├── World Version
       ├── Server Version
       ├── Configuration
       └── Required Assets
               │
               ▼
          Player Machine
               │
               ▼
          Game Server


This would allow the platform to manage not only persistent state but also the version of the server workload executing that state.

This capability is intentionally separate from the core world-synchronization system.



# Network Policy

Players can control how Game Control uses different network connections.

Example:


                 Download    Upload    Host
Wi-Fi               YES        YES       YES
Ethernet            YES        YES       YES
Mobile/Metered      ASK        NO        NO


Hosting and synchronization are treated as separate permissions.

A player may therefore allow downloads while preventing large uploads over a metered connection.



# Architecture

A possible architecture is:


┌───────────────────────────────────────────────┐
│                 GAME CONTROL                  │
├───────────────────────────────────────────────┤
│                                               │
│  Player Agent                                 │
│  ├── Host Manager                             │
│  ├── Game Server Manager                      │
│  ├── Sync Manager                             │
│  ├── Local World Manager                      │
│  ├── Network Policy                           │
│  └── Game Adapter                             │
│                                               │
│                    │                          │
│                    ▼                          │
│              Control API                      │
│                                               │
│  ├── Authentication                           │
│  ├── World Management                         │
│  ├── Host Lease Management                    │
│  ├── Host Queue                               │
│  ├── Version Management                       │
│  └── Migration Coordination                   │
│                                               │
│                    │                          │
│                    ▼                          │
│             Persistent Storage                │
│                                               │
│  ├── World Manifests                          │
│  ├── World Versions                           │
│  ├── World Chunks                             │
│  └── Server Artifacts                         │
│                                               │
└───────────────────────────────────────────────┘


The architecture is expected to evolve during development.


# Engineering Domains

Although Game Control is primarily motivated by multiplayer gaming, the underlying engineering problems belong to several broader areas:

### Distributed Systems

* distributed leases
* failure detection
* host coordination
* consistency
* fault recovery
* concurrency

### Cloud Computing

* persistent object storage
* cloud APIs
* workload coordination
* versioned state
* infrastructure

### Distributed Storage

* chunking
* content addressing
* deduplication
* compression
* manifests
* atomic commits
* rollback

### Networking

* client-server communication
* host discovery
* connectivity
* bandwidth management
* reconnection

### Systems Engineering

* process management
* workload lifecycle
* crash recovery
* monitoring
* logging

Gaming provides the initial application and demonstration environment for these technologies.


# Example Lifecycle

## 1. World Creation


Create World
     │
     ▼
Version v1


## 2. Host A


A acquires lease
     │
     ▼
A downloads latest version
     │
     ▼
A starts game server


## 3. Players Join


A → HOST
B → CLIENT
C → CLIENT


## 4. Host A Stops


A stops server
     │
     ▼
A saves world
     │
     ▼
A synchronizes changes
     │
     ▼
Version v2 committed
     │
     ▼
A releases lease


## 5. Host B


B acquires lease
     │
     ▼
B downloads v2
     │
     ▼
B starts game server
     │
     ▼
A + C reconnect


The world continues without requiring A's computer to remain online.



# Project Goals

Game Control is primarily an engineering and learning project focused on:

* distributed systems
* cloud computing
* distributed storage
* networking
* state synchronization
* distributed leases
* heartbeats
* failure detection
* workload migration
* versioning
* hashing
* chunking
* compression
* atomic commits
* fault tolerance
* rollback and recovery
* process management
* observability



# Non-Goals

Game Control is not intended to:

* replace dedicated servers for every game
* modify proprietary game engines
* allow multiple hosts to concurrently modify one world
* guarantee zero-downtime migration
* solve every game's save-format compatibility problem
* function as a general-purpose game marketplace

Game-specific functionality depends on the architecture and licensing of each supported game.



# Security Considerations

A production deployment would need to address:

* authentication
* authorization
* world ownership
* host authorization
* lease security
* encrypted transport
* malicious clients
* malicious world uploads
* corrupted data
* replay attacks
* secret management
* rate limiting
* denial-of-service protection
* secure server artifact distribution



# Project Status

🚧 **Work in Progress**

Game Control is currently being developed as a learning and research project.

The architecture, APIs, storage model, synchronization protocol, and implementation may change substantially as the project evolves.


# Roadmap

* [ ] Basic world creation
* [ ] Local world versioning
* [ ] World manifests
* [ ] Chunk hashing
* [ ] Incremental synchronization
* [ ] Persistent cloud storage
* [ ] Host leases
* [ ] Heartbeats
* [ ] Host queue
* [ ] Host migration
* [ ] Network policy
* [ ] Compression
* [ ] Authentication
* [ ] Recovery and rollback
* [ ] Game adapters
* [ ] Server artifact management
* [ ] Automated testing
* [ ] Observability
* [ ] Containerized deployment
* [ ] Multi-game support



# Why Game Control?

The project started from a simple question:

> Why should a persistent multiplayer world permanently depend on the computer of the player who happens to host it?

Game Control explores whether **persistent state and compute ownership can be separated**.

Instead of:


WORLD + SERVER
       │
       ▼
PLAYER A


the system separates them:


             PERSISTENT STATE
                    │
                    ▼
               GAME CONTROL
                  CLOUD
                    │
                    ▼
          ┌─────────┴─────────┐
          │                   │
      PLAYER A            PLAYER B
      SERVER              SERVER


The game-server workload can move between authorized players while the persistent world remains independent of the machine currently executing it.



# License

This repository is currently **source-available / all rights reserved**.

The source is publicly visible for learning, research, and evaluation. No broad open-source license is granted unless explicitly stated in the repository.

Forking may be possible through GitHub's platform functionality, but a GitHub fork does not by itself grant rights to commercially use, redistribute, or create derivative products from the software. GitHub's Terms permit public repositories to be viewed and forked; additional copyright permissions come from the project's license.

Commercial licensing and future distribution terms may be established separately as the project evolves.

**Copyright © 2026 Arfat Zafar. All rights reserved.**
