# Organism

## A Distributed Personal and Collective System for Replacing Operating Systems

### Draft Specification v0.3

## Executive overview

Organism is a distributed operating system replacement for people and groups whose computational life is not tied to a single machine. Devices (phones, laptops, servers, wearables) are organs: they join, sleep, fail, and are replaced while the organism—the persistent identity and its data and agents—continues. The design rejects Unix-style processes, threads, and hierarchical files as kernel primitives; instead, everything useful is built from actors that communicate by message passing and hold capabilities, not ambient permissions.

Execution is hypervisor-grounded: on each organ, an Organism hypervisor hosts many slim actor kernels, one per actor. Isolation is kernel-to-kernel; cooperation across actors is always serialized messaging through capability membranes, including on a single device. Foreign or third-party code is minimized where possible and never trusted as a substitute for that isolation. Persistent state lives in memory tissue (immutable, content-addressed object graphs with tiers from hot “active” material to archival storage), coordinated so actors do not reinvent persistence ad hoc.

The document proceeds from purpose and principles to system model (organisms, collectives, organs, actors), messaging, supervision, and memory, placement and migration under policy, identity and trust, interfaces and networking, the split trusted runtime (hypervisor, per-actor kernels, membranes—not a monolithic “Organism kernel”), a conceptual API surface, failure assumptions, and non-goals. Readers who want the architectural spine first can skim sections 1–3, 8, and 11–13; readers implementing isolation and routing should weigh sections 3.4–4, 7, and 11 especially.

---

This revision merges the v0.1 system model with the v0.2 execution architecture. It preserves the full conceptual scope of v0.1 while making explicit how that scope is realized: actors do not share a single global kernel address space; instead, each organ runs an Organism hypervisor that hosts many isolated **actor kernels—one very slim, small kernel per actor**. All interaction across actor boundaries is serialized messaging mediated by capability membranes. The story from identity and collective structure down to messaging and healing should read as one continuous design, not two parallel documents.

---

# 1. Purpose

Organism is a distributed system designed to replace operating systems. It is centered on persistent computational continuity rather than individual machines.

The system treats a person’s or group’s collection of devices as a unified computational organism composed of communicating actors distributed across specialized organs.

Organism is:

- actor-native
- distributed-first
- identity-centric
- capability-secured
- persistent by default
- self-healing
- device-agnostic
- collective-aware
- hypervisor-grounded (each organ’s runtime is rooted in **per-actor** isolation: one slim kernel per actor, not a shared multiprogrammed kernel heap visible to untrusted code)

Organism is not derived from:

- Unix
- Windows
- POSIX
- process/thread models
- hierarchical filesystems

---

# 2. Foundational Principles

## 2.1 The Organism Is Primary

The system instance is the organism itself, not any individual device.

Devices are temporary organs participating in the organism.

The organism persists despite:

- device replacement
- network partition
- power loss
- hardware failure
- mobility

---

## 2.2 Actors Are the Kernel Primitive

Actors are the fundamental computational entity.

All system behavior emerges from actors.

There are no kernel processes or threads exposed to actors.

Actors:

- communicate exclusively through messaging
- possess isolated state
- are independently schedulable
- may persist beyond device lifetime
- may migrate between organs
- may replicate
- may be supervised

---

## 2.3 Messaging Is Fundamental

All interaction occurs through asynchronous message passing.

There is no distinction between:

- local communication
- IPC
- RPC
- network communication

Differences are expressed only in:

- latency
- trust
- consistency
- availability

---

## 2.4 Persistence Is Intrinsic

Persistence is an organism capability: durable continuity of identity, mailboxes, causal history, and memory tissue is provided by the trusted runtime and tissues (hypervisor-local services, distributed protocols, capability-gated recall—not a single Unix kernel and not “whatever actor code bolts on”). Actors participate through policy and primitives; they do not own the substrate that makes “survive partition” meaningful.

Actors may survive:

- restart
- migration
- replication
- disconnection
- organ failure

Organism has no separate application layer above actors: coordination of storage, replication, and continuity semantics stays kernel- and tissue-shaped in the Organism sense—implemented by the split trusted base (Section 11)—while actors declare persistence policy and call durable primitives. They do not reimplement the organism’s storage, replication, and continuity story as an ad hoc stack outside that substrate.

### What “intrinsic” excludes

There is no supported model where only actor-local hacks (private databases, hidden files, opaque cloud buckets) define whether the organism is still the same organism after hardware churn. Such stores may exist as materializations of tissue or as foreign systems imported behind membranes, but the authority and lineage of durable state—what counts as committed, replicated, and recoverable—are organism rules. Otherwise continuity would be anecdotal: different actors with different ad hoc persistence would not compose into one persistent subject (Sections 3.1 and 8.1).

### Policy and mechanism

Mechanism (checkpoints, journals, routing of replicas, tier promotion, migration handoff) lives in the trusted runtime and memory tissues (Sections 3.4, 6, 7, and 11). Policy is declared per actor: `PersistencePolicy` and related fields (Section 3.5) state how aggressively to checkpoint, which organs or trust tiers may hold replicas, how much loss is acceptable under partition, and which operations require synchronous durable commit. The split matches “you choose your continuity contract” with “the organism enforces it uniformly.”

### Modes of survival

- Restart—an actor kernel or process-like incarnation ends; durable actors reload state from tissue and journal under supervision (Section 5). Ephemeral actors (Section 3.5) may not survive restart by design.
- Migration—execution moves to another organ’s slim actor kernel; ActorId, mailbox routing, capabilities, and supervision links are preserved across a membrane handoff (Section 7.3). Durable state need not move byte-for-byte with the actor if content-addressed replicas already exist where policy places them.
- Replication—multiple incarnations or shard copies reduce the odds that organ or disk loss erases policy-critical state (Section 3.5, Replicated Actor). Replication choices interact with trust: replicas must sit on organs attested to the right tier (Section 8.2).
- Disconnection—organs lose contact for arbitrary time. Mailbox and tissue semantics (Section 4) define what queues, what drops, and what merges when links return; persistence means effects are accounted for, not that every message is instantaneously global.
- Organ failure—a node is gone or untrusted. The organism continues on other organs; durable actors are restarted or migrated under placement and supervision; capabilities may be revalidated at new entry membranes (Sections 7 and 13).

Intrinsic persistence is failure-shaped: it is defined precisely through these survivable events and the failure model (Section 13), not through the fiction of always-on hardware.

### Relation to memory tissue

Durable truth lives in immutable, content-addressed, capability-protected object graphs and their tiers (Section 6). Actor heaps remain in their kernels; only commits accepted by tissue under capability rules become part of the shared durable lineage. Working memory may hold speculative state that disappears on crash; promotion to persistent tiers is the boundary where intrinsic persistence engages.

---

## 2.5 Capability Security

Authority is granted exclusively through capabilities.

There are:

- no global namespaces
- no ambient authority
- no user/group permission model

Capabilities are:

- delegable
- revocable
- composable
- cryptographically verifiable

### Who grants?

No actor “has” authority in the abstract; it holds only what has been issued along a verifiable chain. The question “who grants?” therefore means: who may lawfully mint or extend the first links, and who may delegate onward?

Organism root and bootstrap. The cryptographic root identity of the organism (Sections 3.1 and 8.1)—typically keys and ceremony controlled by the owner or founding collective—is the ultimate issuer for bootstrapping: initial capabilities that attach organs, stand up policy actors, admit trusted runtime components, and spawn the first ordinary actors. This is not day-to-day “root login”; it is lineage: a small, auditable set of operations that create the capability graph from cold start or recovery. Design-wise, everything else should be derivable under policy from that root (or from explicit collective ratification) so there is no shadow grant table in a hidden daemon.

Trusted runtime components. The Organism hypervisor, membrane services, memory-tissue handlers, and related privileged code do not grant authority because they are “bigger than” actors in a moral sense—they grant only where policy and cryptography say they may: for example issuing the capability bundle that accompanies `spawn_actor`, attaching a new actor kernel’s mailbox to the routing fabric, or minting narrow device or tissue handles that the spawn policy authorizes. Their power is mechanical execution of issuable events recorded against the root’s rules, not a second ambient permission universe. If the runtime could widen rights without proof, capability security would be hollow; implementations must treat that as TCB surface to minimize and attest (Sections 3.4.2, 8.2, and 11).

Actors. Any actor that holds a capability may delegate a subset (normally an attenuation) to another actor if the capability’s shape and policy allow it—Section 8.3 develops chains, restriction, time limits, and composition. Ordinary cooperation is grant along a path, not shared memory and not “same user account.”

Collectives and federated policy. In collective organisms (Section 3.2), who may grant what can itself be constrained by agreements, votes, or capability-shaped treaties between roots. The issuer is still explicit—often a policy actor or joint ceremony—not an invisible global admin.

Membranes verify; they do not invent. Membranes (Section 3.4.4) check proofs on privileged sends; they consume delegation evidence rather than granting new authority out of band. Invention of authority stays at issuers the root and policy recognize.

In short: granting is always an issuance event someone can name—bootstrap from the organism root, mechanically policy-bound issuance by the trusted runtime at spawn or service entry, or delegation from an already-authorized holder. Verification is everywhere else. Together that replaces user/group tables and ambient “I’m on the box” privilege with an auditable token graph.

---

## 2.6 Isolation by Construction

Organism’s security and coherence rest on a strict isolation contract that applies everywhere execution occurs, not only across the network.

The organism rejects shared computational shortcuts that collapse trust boundaries:

- There is no shared mutable memory between actors as a means of cooperation.
- There is no ambient authority beyond what capabilities explicitly grant.
- There is no exchange of raw pointers or opaque handles that imply direct access to another actor’s heap or kernel-internal structures.
- Actors cannot directly observe or mutate kernel memory; the kernel remains outside their addressability.
- Cross-actor and cross-kernel collaboration proceeds only through defined messaging paths, with payloads that the receiver can interpret without trusting the sender’s memory layout.

This principle is realized on each organ by a stack that Section 3.4 develops in architectural detail: hardware, Organism hypervisor, and—per actor—a dedicated slim actor kernel that hosts only that actor’s runtime. Actors never execute “inside” a monolithic kernel shared with other actors in the Unix sense; pairwise isolation is kernel-to-kernel: distinct actors mean distinct kernels, each minimal in trusted code and private address space, that the hypervisor isolates from one another.

Foreign code—code that was not authored by the organism’s owner or was obtained from outside the organism’s direct lineage—is a fact of life in open systems. Organism’s posture is not to trust foreign source after compilation, but to minimize unnecessary foreign surface area and to enforce isolation regardless of provenance. Static and dynamic analysis may inform policy (what an actor may do, which membranes it may use, which organs may host it), yet the hard boundary remains: compiled artifacts run in isolated kernels, under capability rules, with no escape hatch through shared memory with peers.

---

# 3. System Model

---

# 3.1 Organism

An Organism is a persistent distributed computational identity.

An organism consists of:

- actors
- memory tissues
- organs
- capabilities
- routing structures
- supervision structures

Each organism possesses:

- a cryptographic root identity
- a continuity history
- causal lineage metadata

The organism’s abstract model is unchanged by the v0.3 execution substrate: identity, memory tissues, and routing remain conceptual. What changes is the mandatory shape of implementation on each organ: that substrate must be able to enforce isolation, capabilities, and serialized cross-boundary communication without relying on a trusted shared heap between actors.

---

# 3.2 Collective Organisms

Organisms may represent:

- individuals
- families
- teams
- research groups
- organizations
- communities

Organisms may contain sub-organisms and participate in larger federated organisms.

Examples:

```text
Personal Organism
  participates in:
      Family Organism
      Research Organism
      Open Source Organism
```

Actors and capabilities may selectively participate across organism boundaries.

---

# 3.3 Organs

An Organ is a participating hardware node.

Examples:

- phone
- laptop
- home server
- wearable
- vehicle
- cloud node

Organs contribute:

- computation
- storage
- sensors
- interfaces
- network access
- energy

Organs are not authoritative system roots.

Organs may:

- join
- depart
- sleep
- partition
- fail

without destroying the organism.

On each organ, the Organism hypervisor is the local root of the execution stack. The organ’s operating-system services, device access, and energy policy feed into scheduling and placement, but the semantic authorities of the organism—capabilities, identity continuity, causal ordering of durable effects—are not “owned” by the organ; they are upheld by the distributed organism and enforced locally at membranes and kernels.

---

# 3.4 Hypervisor, Actor Kernels, and Capability Membranes

This section is the architectural bridge between “devices as organs” and “actors as the unit of behavior.” It states how v0.2’s direction is absorbed into the v0.1 story without collapsing the distributed organism into a single-machine OS.

## 3.4.1 Layered Stack

On every organ, the intended structure is:

```text
hardware
  ↓
Organism hypervisor
  ↓
one slim actor kernel per actor (many per organ)
  ↓
the single actor hosted by that kernel
```

Actors do not execute inside one shared global kernel that also hosts their peers’ heaps. **Each actor has its own very small actor kernel:** a thin trusted runtime whose job is scheduling that one actor, wiring its mailbox to membranes, and coordinating persistence and supervision hooks—not a shared multiprogramming host. Isolation boundaries are kernel boundaries: one actor, one kernel address space, period.

All interaction that crosses those boundaries—whether between actors in different kernels on the same organ, between actors on different organs, or between actors and privileged runtime services—is logically the same kind of operation: serialized messages validated and relayed through capability membranes. Locality changes cost and latency; it does not change the requirement for explicit authority and explicit data copying or capability-checked sharing of immutable object references.

---

## 3.4.2 Organism Hypervisor

The Organism hypervisor is the trusted computing base on every organ for local isolation. Its responsibilities include:

- Allocating hardware resources to actor kernels according to organism-wide placement policy and local organ constraints (power, thermal headroom, real-time requirements).
- Enforcing memory protection domains so that no actor kernel can read or write another’s private memory without an explicit, audited mechanism (there is none for arbitrary read/write; only message-based crossing).
- Scheduling kernel execution and mediating access to devices so that embodied actors (cameras, GPUs, sensors) receive capabilities rather than raw device DMA into foreign memory.
- Exposing attestation interfaces so other organs and remote participants can verify which hypervisor build and policy applied to a message or cryptographic proof at the moment it crossed an organ boundary.

The hypervisor is not the “actor program.” Actor behavior lives in actors inside kernels. The hypervisor is the narrow layer that makes the promise of actor isolation real on commodity hardware.

---

## 3.4.3 Isolated Actor Kernels (One Slim Kernel per Actor)

An **actor kernel** is the minimal trusted runtime paired **one-to-one** with a single actor on an organ. It is intentionally **very slim and small**: not a general OS inside the organism, but the narrowest layer that can still enforce “this actor’s heap is not any other actor’s heap,” attach the mailbox to membranes, and integrate with the hypervisor’s resource accounting.

Each actor kernel hosts **exactly one actor** and exposes only what that pairing requires, for example:

- scheduling that one actor’s turns under hypervisor policy
- mailbox endpoints wired to membranes for ingress and egress
- persistence integration with memory tissues (durably queueing or checkpointing according to that actor’s policy)
- supervision primitives that align with Section 5’s fault domains

Variants of the actor kernel (e.g., a hardened micro-kernel for embodied or real-time actors, a standard minimal build for ordinary actors) trade verified surface area or latency for size and assurance. **They do not multiplex multiple independent actors inside one kernel**—that would recreate a shared address space among peers; multiplexing is the hypervisor’s job across many small kernels, not a single kernel’s job across many actors.

Isolation between actors on the same organ is isolation between sibling actor kernels. A failure, bug, or malicious actor kernel cannot silently corrupt another’s memory by avoiding membranes. Detection and healing propagate through supervision and organism policy rather than through hope that a larger shared runtime stayed sound.

---

## 3.4.4 Capability Membranes

A capability membrane is the enforced interface at which messages cross trust or isolation boundaries. The membrane:

- verifies that the sender holds a capability sufficient to target the recipient’s mailbox or service entry
- may rewrite, filter, or attest message payloads according to policy (for example, stripping fields that the recipient’s contract does not admit)
- logs causal metadata needed for continuity and audit
- applies rate limits, revocation checks, and delegation depth limits

Membranes generalize the idea that “everything is messaging” from Section 2.3: not only is interaction asynchronous and message-based at the actor level, but the implementation is obligated to pass those messages through gates that capabilities govern. There is no “fast path” that bypasses membrane checks for same-organ convenience.

Membranes exist at multiple scales:

- between actor kernels on one organ
- between organs on the network
- between an actor and kernel-provided services that are not themselves ordinary actors (still messaged, still capability-gated)

---

## 3.4.5 Memory Isolation and the Per-Actor Kernel Boundary

Organism does not expose a hierarchical filesystem as a kernel primitive (Section 6), and likewise it does not expose a shared mutable heap between actors as a cooperation primitive.

The **primary isolation unit at runtime is the per-actor kernel**: its private address space and minimal trusted code host **one** actor. There is no organ-local “big kernel heap” where several actors are threads; another actor is always another kernel, another boundary.

- Mutable state visible to an actor stays inside that actor’s kernel unless explicitly serialized into a message or into memory tissues under capability rules.
- References to another actor’s mutable state are not transmitted as pointers across kernels; they are transmitted as messages, capability tokens, or content-addressed immutable object identifiers that memory tissues resolve under capability control.

This aligns durable storage (immutable object graphs, causally versioned) with execution: the same conceptual move—no silent aliasing across trust boundaries—appears in persistence and in the one-actor-one-kernel layout.

---

## 3.4.6 Foreign Code and Minimization

Foreign code is any code whose provenance lies outside the organism’s chosen trust anchor (third-party packages, remotely installed extensions, cross-organism contributed actors). Organism’s approach has two coordinated tracks:

**Minimization.** Reduce the volume and privilege of foreign code by preferring small, capability-bounded actors; declarative composition; and organism-local implementations for sensitive paths. Fewer foreign entry points means fewer membranes to audit and less attack surface at the hypervisor boundary.

**Isolation after compilation.** Regardless of static analysis results, foreign-compiled actors run each in **their own** slim actor kernel under the same membrane rules as first-party actors. Analysis may restrict which capabilities may be granted at spawn time, which organs may host a package, or which membranes it may call—but analysis is policy input, not a substitute for isolation.

Together, minimization and hard isolation let the organism admit the real world of shared libraries and community contributions without treating “imported” as “trusted.”

---

# 3.5 Actors

Actors are isolated computational entities.

Execution of actor code always occurs inside **that actor’s dedicated slim actor kernel** on some organ, subject to the hypervisor and membranes described in Section 3.4. The actor abstraction remains the same for programmers; the implementation constraints ensure that “local” actors cannot secretly become threads sharing a mutable heap or a multiprogrammed kernel with peers without breaking the Organism contract.

## Actor Properties

Every actor possesses:

```text
ActorId
IncarnationId
Mailbox
State
Capabilities
PlacementPolicy
PersistencePolicy
SupervisionLinks
LineageMetadata
```

The mailbox is the actor’s durable ingress surface; membranes sit logically “in front of” mailboxes for messages originating **outside that actor’s kernel**.

---

## Actor Semantics

Actors:

- do not share memory
- cannot directly inspect other actors
- communicate exclusively through messages
- may create actors
- may delegate capabilities
- may supervise actors

---

## Actor Classes

### Ephemeral Actor

Cheap, local, disposable.

No persistence guarantees.

---

### Durable Actor

State persists across:

- restart
- migration
- organ failure

---

### Replicated Actor

Maintains multiple active or standby incarnations.

---

### Embodied Actor

Bound to a specific organ capability.

Examples:

- camera actor
- microphone actor
- GPU actor

Embodied actors depend on the hypervisor’s mediation of hardware capabilities; membranes ensure that capability transfer for device access is explicit and revocable.

---

### Migratory Actor

May relocate dynamically between organs.

Migration preserves:

- ActorId
- mailbox continuity
- capabilities
- supervision relationships

as stated in Section 7.3, and must preserve membrane-relevant authorization: capabilities are revalidated at the destination organ’s entry membranes.

---

# 4. Messaging

---

# 4.1 Message Semantics

Messages are:

- asynchronous
- immutable
- causally tracked
- capability constrained

Messages may be:

- transient
- durable
- replicated

When a message crosses an actor-kernel boundary (hence, **any** message to another actor) or an organ boundary, its payload is serialized and inspected at a membrane. **Within** a single actor kernel—only that actor’s code and its minimal runtime—implementations may optimize local copying when they can prove equivalence to the serialized semantics seen at membranes, but such optimizations must not change authorization outcomes or visibility of data to other actors.

---

# 4.2 Mailboxes

Every actor possesses a mailbox.

Mailboxes:

- are durable endpoints
- may outlive actor incarnations
- support causal ordering metadata
- support selective receipt

Mailboxes are the stable addressing concept; membrane policies determine which senders may reach them over time as capabilities evolve.

---

# 4.3 Delivery Guarantees

The messaging substrate supports configurable delivery semantics:

```text
best_effort
at_least_once
causal
durable
replicated
```

Cross-membrane delivery participates in the same semantic classes; additional attributes (e.g., attested delivery on an organ boundary) may be surfaced without changing the actor-facing classification.

---

# 4.4 Membranes Along the Path

For clarity in implementation and in threat modeling, it helps to name where membranes sit:

- At ingress to an actor kernel from another actor’s kernel or from the network.
- At egress from an actor kernel toward remote organs or toward privileged runtime handlers (hypervisor- or tissue-provided services), when the organism routes a capability-checked request to code that enforces organism-wide invariants—work that must not run inside untrusted actor space alone.

The hypervisor manages actor kernels (scheduling, isolation, device mediation); actors do not “use” it or invoke it as a peer API. Interaction with privileged services is still messaging under capabilities: the slim actor kernel and routing fabric deliver requests; membranes sit where trust level increases, including on one organ when policy says so.

Every crossing is an opportunity to enforce capabilities, preserve causal metadata, and refuse illegal transitions. This does not require the programmer to name membranes explicitly in ordinary code; it requires the runtime to route messages through them consistently.

---

# 5. Supervision

---

# 5.1 Supervision Trees

Actors may supervise other actors.

Supervision relationships define:

- fault domains
- restart policy
- replication policy
- migration policy
- healing strategy

Supervision spans kernels and organs: a supervisor may be remote, but supervision signals travel as messages like any other privileged communication, across membranes with appropriate capability proofs.

---

# 5.2 Healing Semantics

Failure is considered normal system behavior.

The organism continuously attempts to preserve:

- continuity
- capability availability
- actor identity
- causal integrity

Healing actions may include:

- restart
- migration
- replication
- degradation
- substitution
- hibernation

Isolated kernels localize blast radius; supervision and replication recover organism-level function without assuming that a single compromised kernel could silently poison unrelated actors on the same organ.

---

# 6. Memory Tissue

Organism does not expose hierarchical filesystems as a kernel primitive.

Persistent information is represented as immutable object graphs.

---

# 6.1 Objects

Objects are:

- immutable
- content-addressed
- causally versioned
- capability protected

Objects may reference:

- objects
- actors
- streams
- capabilities

Object fetch and publication cross membranes: an actor obtains a capability-gated view of memory tissue contents; it does not map arbitrary remote object stores into its address space as mutable pages.

---

# 6.2 Memory Tiers

Tiers classify where materialized object-graph data lives along axes of latency, durability, locality, and cost. They are not a parallel “filesystem API”: fetch and storage still go through memory tissues, content addresses, capabilities, and membranes. A tier change is a placement and retention decision—what is kept hot, what is durably anchored, what may be discarded—typically orchestrated by the organism together with actor `PersistencePolicy` and resource limits on each organ.

Memory tissues may exist in:

- active consciousness
- working memory
- persistent memory
- distributed long-term memory
- archival memory

Placement is dynamic.

## Active consciousness

Active consciousness is execution-bound **material**: object subgraphs and streams that the placement system keeps **actor-adjacent** on an organ hosting those actors—e.g., resident in fast local memory, prefetched into cache lines or near-execution staging buffers, or otherwise optimized so that permitted actors pay minimal latency to traverse references the policy marks as part of the current **active set**. The metaphor is attention in the narrow sense of **what this stretch of computation is allowed to treat as immediately present**; it is not a theory of mind.

Active-consciousness residence is **revocable by policy**: under memory pressure, power saving, or migration, the same content still exists by content address elsewhere; only the hot local manifestation may be dropped. Actors continue to interact with **handles and capabilities**; the tissue fault may trigger lazy rehydration from a colder tier (often with a membrane checkpoint).

## Working memory

Working memory holds **ephemeral or explicitly non-durable** graph material: scratch derivations, speculative merges, UI render caches, intermediate parse trees, or other values that are cheap to recompute or that have not yet been **committed** into a durable tier. It is usually local to an organ and fast, but its defining property is **impermanence and policy-defined discard**: unless promoted, it may vanish on actor restart, kernel recycling, or eviction without contradicting organism continuity guarantees for durable actors—because those guarantees attach to tiers below that explicitly promise retention.

The contrast with active consciousness is **what is being optimized**: active consciousness optimizes **binding to who is running now**; working memory optimizes **throughput of transformation** with the explicit acceptance that lineage may not yet (or ever) record that material. Promotion from working to persistent (or directly into distributed long-term) is a **deliberate, capability-gated commit**—often the moment an actor sends a `remember`-class operation or a tissue transaction closes.

## Persistent memory

Persistent memory is **durable on the anchoring organ(s)** under organism rules: content-addressed immutable objects and their causal versions **survive** ordinary restart of actor kernels, bounded organ sleep, and the kinds of failure models Section 13 assumes, according to stated guarantees. This is where personal continuity stops being “cache state” and becomes **recoverable fact**—subject to replication policy, backup, and cross-organ reconciliation.

## Distributed long-term memory

Distributed long-term memory is persistent memory **sharded, replicated, or erasure-coded across organs** with routing as ordinary actor messaging and tissue protocols. Latency and consistency follow the delivery and causal classes from Section 4; not every organ keeps a full copy. The same object identity may be materialized in active consciousness on one organ while existing only as encoded fragments on others until read patterns or policy pull it forward.

## Archival memory

Archival memory is the **cold tail**: highest latency, lowest ongoing cost (offline media, remote vaults, compressed packs), often with explicit **retrieve-then-activate** workflows. Objects here are still conceptually part of the tissue graph, but the organ may not map them into addressable form until a membrane-gated **recall** path completes—aligning with cost, legal hold, air gaps, or user consent.

## Tier movement and policy

Promotion and demotion (archival → long-term → persistent → active consciousness; working → persistent; active → working when dropping hot caches without dropping durability) are continuous background concerns for placement (Section 7). They respect:

- **Capabilities**—no tier promotion that would expose object bytes to the wrong actor.
- **Causal ordering**—commits that affect durable lineage are ordered with the organism’s causal metadata.
- **Energy and trust**—moving data to or from archival or remote organs may require attested paths or user acknowledgement.
- **Isolation**—rehydration into an actor kernel never bypasses membranes or substitutes raw DMA for capability-checked object projection.

In short: **active consciousness** is “hot and bound to current execution,” **working memory** is “fast but discardable unless committed,” **persistent** and **distributed long-term** are “durable at increasing geographical and protocol scale,” and **archival** is “economical, latent, explicitly awakened.”

---

# 7. Placement and Scheduling

---

# 7.1 Resource Model

Computation is expressed as resource demand.

Placement is multidimensional: the organism matches what an actor needs and is willing to pay for to what organs can offer under policy, subject to supervision, tissue placement, and collective constraints. Demand is not a single scalar “priority”; it is a vector the scheduler projects onto available organs, often under hard ceilings (battery, thermal, verified isolation budgets) and soft preferences (low latency to a peer, data residency).

Resources include:

- CPU
- GPU
- memory
- bandwidth
- storage
- battery
- thermal headroom
- trust
- locality
- hypervisor-verified isolation overhead

### CPU

Time on general-purpose execution units: throughput for bursty work, eligible CPU time for sustained loads, and scheduling class (best-effort vs latency-sensitive vs deadline-bound). Durable and replicated actors still consume CPU for checkpointing, journal compaction, and replay; embodied actors may need predictable frames for sensor or media pipelines when those pipelines are CPU-driven. Affinity (particular core classes, performance vs efficiency cores) is part of demand. Work that belongs on parallel or fixed-function accelerators is modeled separately under GPU.

### GPU

Parallel and accelerator execution: graphics, machine-learning inference and training, video encode/decode, and other vendor-specific queues. Demand includes device class and API profile (e.g., supported instruction set or intermediate representation), queue latency and throughput, VRAM or accelerator-local memory budget, and whether the workload is batch-tolerant or frame-deadline-bound. GPU work is usually expressed by embodied or organ-bound actors: placement may require a particular organ because the physical device lives there; migration then means re-handoff to another organ’s GPU or remote delegation through capability-gated protocols, not silent peer pointer sharing.

The hypervisor mediates device DMA and submission channels: actors receive capability-checked submission rights, not unrestricted access to another actor’s VRAM or command streams. Time-slicing, partitioning (hardware or software), and power-state transitions are organ policy applied to shared GPUs. GPU demand couples tightly to thermal headroom and battery on mobile organs.

### Memory

Primary-store footprint for the slim actor kernel plus one actor, hot object materialization (memory tiers), stack/heap for the runtime, and kernel bookkeeping. Demand includes working set size, burst allocation, and limits on how much active consciousness the actor may pin on an organ. Memory interacts with eviction: exceeding budget may demote hot material or refuse new pinning until policy reconciles.

### Bandwidth

Data movement over links that matter for cost and latency: inter-organ messaging, tissue fetch and replication, backup or sync to archival tiers, and wide-area relay when organs partition. Demand may specify ceilings, minimum viable throughput for streaming, and partition tolerance (degrade quality vs pause). Local bandwidth inside an organ still matters for DMA-adjacent embodied actors and for kernel-to-hypervisor traffic.

### Storage

Durable and cold capacity: persistent and distributed long-term volume, archival packs, snapshot and checkpoint storage, and write amplification from causal logging. Demand includes IOPS/latency class for commits, retention policy, and erasure or replication overhead. Storage trusts interact with “who may host this replica.”

### Battery

On untethered organs, energy budget and discharge rate constrain which actors may run, at what duty cycle, and whether migration to mains-powered organs is preferable. Placement may throttle non-critical actors or shift them to organs on wall power while preserving capability semantics.

### Thermal headroom

Package temperature and sustained power limits: when headroom is low, the organism may reduce clock, migrate compute away, or lengthen scheduling quanta. Thermal interacts with embodied actors bound to hot silicon (GPUs, NPUs) where relocation is constrained.

### Trust

Minimum organ attestation (hypervisor build, kernel variant, membrane policy hash), jurisdictional or operator identity, data residency rules, and maximum foreign-code privilege on a hosting organ. Trust is not binary: demand may request “verified kernel tier” or “owner-operated organ only” and fail placement otherwise rather than silently downgrading.

### Locality

Preference or hard requirement to run near other actors, specific object subgraphs, human interfaces, or legal regions—latencies and capability paths from this placement to those anchors. Locality includes colocation (same organ), regional colocation (same site or metro), and avoidance (must not run on organ class X). It is the spatial counterpart to trust and bandwidth.

### Hypervisor-verified isolation overhead

Real CPU, memory, and latency costs of enforcing one slim kernel per actor: extra page tables, world switches, cache effects, optional encrypted memory or integrity-checked pages, and membrane work on hot paths. Stronger isolation is a resource with a measurable budget; an actor that insists on maximally hardened kernels consumes more of this overhead class and may displace other work on capacity-constrained organs.

Demand annotations are advisory or contractual according to `PlacementPolicy` and organ policy: some limits are hard (refuse placement), others soft (best effort with telemetry). The organism records decisions for audit, supervision, and future placement.

---

# 7.2 Placement Policies

Actors may express:

- latency sensitivity
- persistence requirements
- trust requirements
- energy preferences
- hardware affinity

The organism determines embodiment.

Placement chooses an organ and spawns or moves the actor into its own slim actor kernel there, selecting a kernel variant (minimal default, real-time-oriented, formally verified, etc.) appropriate to the actor’s class and policy—not a “big” kernel shared with unrelated actors. Some actors may require kernels with stronger verification claims; others may use the smallest standard build on the cheapest organ that satisfies trust.

---

# 7.3 Migration

Actors may migrate between organs when placement policy permits: migration is a form of re-placement, not an exemption from `PlacementPolicy`, organ capabilities, trust ceilings, locality rules, or resource limits from Section 7.1. The organism may refuse or queue a move when no destination satisfies the actor’s declared constraints, when supervision policy requires a specific failover path, or when collective or tissue policy forbids a region or organ class.

Migration preserves:

- ActorId
- mailbox continuity
- capabilities
- supervision relationships

Migration includes a membrane handoff: the destination organ must accept the actor’s capability bundle and reattach mailbox routing without breaking causal ordering guarantees promised by the actor’s persistence policy. Choosing the destination, timing the move, and approving kernel variants on the new organ remain subject to the same placement machinery as initial spawn (Section 7.2).

---

# 8. Identity and Trust

---

# 8.1 Organism Identity

Each organism possesses a cryptographic root identity.

Identity continuity persists independently of organs.

## Otherness

“Otherness” here is not metaphor alone; it is the structural fact that a persistent organism is not the same entity as any organ, and not the same entity as another organism unless roots and policy explicitly unite them.

- Organism versus organ. Hardware is other than the identity it serves. A phone or server may host actor kernels and memory tissues, but replacement, loss, or compromise of that organ does not erase or fuse with the root: the organism remains distinguishable as the same continuing subject only because lineage and keys say so, not because silicon persists. Continuity is of the organism through organs—not in a single organ.
- Organism versus organism. Another root is strictly other: peers, services, and collective structures (Section 3.2) encounter this organism as a non-self they must authenticate, route to, and capabilitize separately. No message, object, or capability is implicitly “shared across everyone”; ambiguity is resolved by proofs anchored to which identity spoke or delegated. Federation and sub-organism nesting refine where the boundary runs, but they do not erase it—every layer still has an inside (subjects governed by a root) and an outside (everything that must cross a membrane or treaty).
- Operational grounding. Cryptographic root identity makes otherness checkable: verification draws a line between self and non-self that membranes, delegation, and attestation enforce. Trust is meaningful because there is a you and a not-you; without that distinction, there would be no grant, no revocation, and no audit of who acted.

In short: identity continuity independent of organs is continuity of a bounded computational subject—other to every organ and other to every foreign organism—whose persistence is what the rest of Chapter 8 turns into trust mechanics.

---

# 8.2 Organ Trust

An organ is other than the organism root (Section 8.1): it is a physical site and software stack that hosts organism work. Organism-wide identity does not automatically make every laptop or cloud slot equally trusted as an execution surface. Organs therefore participate through attested trust relationships: proofs, recorded policy, and ongoing evaluation that this node at this time may carry these responsibilities.

Trust may be:

- revoked
- degraded
- constrained

without terminating the organism.

## What attestation establishes

Attestation binds an organ to evidence remote and local participants can verify before routing messages, placing actors, or replicating tissue. Typical dimensions include:

- Hypervisor identity and build: the trusted computing base that enforces isolation and device mediation on this hardware.
- Actor kernel set or policy class: which verified or standard kernels may run, version pins, and allowed variance for updates.
- Membrane configuration: policy hashes or manifests describing ingress and egress rules, rate limits, and delegation checks at organ boundaries.
- Hardware and firmware context where available: TPM or secure-enclave quotes, boot chain, or IOMMU posture, when policy treats them as part of the trust story.

Evidence has freshness: stale attestation may be treated as degraded trust until renewed. Joining an organ to an organism usually records an initial attestation bundle; re-attestation after update, reboot, or policy push keeps the relationship honest.

## Revocation, degradation, and constraint

- Revoked. The organ is no longer trusted at all for its prior role. The organism stops routing new sensitive work there, may forcibly migrate actors according to supervision and placement policy, and treats in-flight capabilities targeting that organ as suspect until reconciled. Revocation is a clean break: the hardware may still exist on the network, but it is outside the trusted set until explicitly re-enrolled and re-attested if policy allows.
- Degraded. The organ remains partially trusted but at a lower tier: for example it may run only certain actor classes, may not hold archival or high-assurance replicas, or may require extra membranes on every egress. Degradation expresses probation after patch lag, partial attestation failure, or elevated risk signals without a full ban.
- Constrained. Trust is narrowed without necessarily lowering a global tier label: specific capabilities, tissue regions, or collective memberships may be withheld while the organ still performs bounded duties. Constraint implements least privilege per organ—fine-grained where degradation is coarse.

In all three cases the organism continues: other organs absorb load; actors and durable state reconcile under healing (Section 5) and migration (Section 7.3); the root identity and causal lineage are not destroyed because one node fell out of favor.

## Remote reliance

When another organism or a distant organ sends a message or proof, attestation answers which isolation story a participant is relying on if this payload was handled or generated on that organ. Hypervisor builds, kernel sets, and membrane policy versions (and their hashes in signed manifests) let participants refuse paths that no longer match their trust policy, independent of transit encryption.

Placement (Section 7.2) and organ discovery consume the same trust material: an actor’s `PlacementPolicy` may require attestation above a threshold, and the scheduler treats untrusted or stale organs as absent for matching until remediated.

---

# 8.3 Capability Delegation

Capabilities are the concrete form of authority in Organism (Section 2.5): unforgeable tokens that tie a principal to permitted actions on named resources under explicit constraints. Delegation is how one actor, the organism bootstrap, or a privileged handler passes a subset of authority to another without opening ambient permission tables. Membranes and routing consult these proofs on privileged paths (Sections 3.4.4 and 4.4).

Capabilities may be:

- delegated
- restricted
- time-limited
- recursively composed

Delegation chains are cryptographically verifiable.

Membrane enforcement consumes these proofs on every privileged send path; delegation is not merely a logical grant but an auditable token chain.

## Delegated

Delegation transfers a capability from holder to recipient so the recipient may act in the issuer’s stead within the bounds carried by the token. Downstream rights are normally an attenuation of upstream rights: a delegate cannot silently exceed what the parent granted unless policy explicitly allows amplification (a rare, audited case such as bootstrapping from the organism root). The kernel records issuer, recipient, delegation event, and policy annotations so supervision and audit can reconstruct who enabled whom.

## Restricted

Restriction narrows a capability at issuance or delegation: which mailboxes may be targeted, which object types or content hashes may be read or written, which memory-tissue tiers may be touched, which organs qualify as execution sites, or which membrane policies apply. Restriction is how least privilege is expressed in token form. A membrane that receives a privileged send with a proof that does not cover the attempted operation rejects it; the token must match the intent.

## Time-limited

Time limits bind validity to clocks or logical counters under policy: absolute expiry, maximum chain lifetime, not-valid-after lineage depth, or bounds tied to actor incarnation. Expired capabilities fail closed at verification. Short-lived delegation limits damage when an organ or foreign-held principal is compromised; renewal may require fresh proofs under stricter attestation (Section 8.2).

## Recursively composed

Recursive composition allows one capability to embed or reference others so a single presentation can carry bundle authority (for example: read this object and append to that stream and reply on this mailbox), or so delegation forms a tree of attenuations. Recursion depth and allowed composition shapes are policy-bounded so chains stay analyzable. Verifiers check not only signatures but shape: permitted constructors, max depth, and forbidden combinations.

## Chains and verification

A delegation chain is an ordered sequence of issuances and attenuations from a root or bootstrap principal to the presenter. Each hop is authenticated under keys the organism recognizes; revoked intermediates invalidate downstream tokens unless policy defines re-anchoring. Verification asks: Are the links authentic? Is nothing expired? Does each restriction lawfully follow from its parent? Does the operation stay inside the cumulative envelope?

Chains may be compressed for transport (Merkle batching, cached issuer roots) but must remain semantically equivalent to the explicit chain under organism rules.

## Revocation and staleness

Revocation of a capability or of an issuer invalidates dependent proofs (Section 12, `revoke`). Membranes and receivers treat stale or revoked evidence as hard failures for privileged operations. How quickly revocation is visible is a delivery and consistency matter (Section 4): high-assurance paths may require fresh proofs per message; looser paths may admit bounded staleness with explicit risk flags.

## Auditing and membranes

Membrane enforcement consumes delegation proofs on privileged sends—mailbox targeting, tissue recall, device submission, and similar paths—which ties side effects to the chain that authorized them. That is what makes delegation auditable across collective and federated organisms (Section 3.2) rather than a private agreement between peers.

---

# 9. Interfaces

---

# 9.1 Interfaces Are Projections

Interfaces are temporary manifestations of actor behavior.

An actor may expose:

- graphical interfaces
- voice interfaces
- APIs
- notifications
- ambient displays

simultaneously.

---

# 9.2 Projection Semantics

Interfaces:

- are stateless projections
- may migrate independently
- may adapt to organ constraints

User-interface pipelines that touch device hardware route through embodied capabilities and the hypervisor’s device mediation, preserving the same capability story as backend actors.

---

# 10. Networking

Networking is not a distinct subsystem.

All communication is actor messaging.

The organism treats:

- local transport
- wireless transport
- internet transport
- relay transport

as interchangeable routing layers.

Physically, local transport may bypass wide-area links, but logically it still passes through membranes with capability checks; “on-LAN” and "on-organ" are not synonymous with “trusted.”

---

# 11. Trusted Runtime Responsibilities

Organism does not consist of one shared “system kernel” that hosts all actors. Execution is layered (Section 3.4): each organ has an Organism hypervisor; each actor has its own slim actor kernel; membranes and memory tissues enforce authority on crossings. When Section 2.2 says actors are the kernel primitive, that is architectural vocabulary for actors as the irreducible unit of behavior, not the name of a singleton kernel image.

The bullets below are organism-wide guarantees—what holders of actor code are entitled to assume—realized jointly by the trusted computing base on each organ and by distributed services (routing, tissues, discovery, trust state). No single component listed here is “the Organism kernel”; responsibility is split by design.

The organism trusted runtime guarantees:

- actor isolation
- capability enforcement
- message routing
- causal tracking
- persistence coordination
- scheduling
- placement
- healing orchestration
- organ discovery
- trust management

## Where the work lives

- Organism hypervisor (per organ, Section 3.4.2). Spawns and retires slim actor kernels; enforces memory protection between them; schedules their execution on hardware; mediates devices and exposes attestation interfaces; anchors local membrane attachment; accounts resources against organ policy.
- Slim actor kernel (one per actor, Section 3.4.3). Hosts exactly one actor’s runtime: local turns, mailbox plumbing to membranes, hooks for persistence and supervision, integration with hypervisor accounting—without becoming a multiprogrammed “big kernel” for other actors.
- Membranes, routing, and tissues (cross-cutting, often distributed). Inter-kernel and inter-organ messaging, causal metadata, durable effects, organ discovery, and trust reconciliation may be implemented as privileged services, protocols, and supervised components outside any single actor’s address space; they are still part of the trusted story actors depend on.

Actors never receive a Unix-style host API from this runtime. The trusted base does not expose:

- processes
- threads
- files
- sockets
- mounts
- device drivers in the Unix sense

---

# 12. Core Organism API (conceptual)

Example conceptual primitives—the naming is historical; they describe organism operations, not syscalls against a monolithic kernel:

```erlang
spawn_actor(Spec, Policy) -> ActorId.

send(ActorId, Message) -> ok.

receive(Selector) -> Message.

link(ActorA, ActorB) -> LinkRef.

monitor(ActorId) -> MonitorRef.

grant(Capability, ActorId) -> CapabilityRef.

revoke(CapabilityRef) -> ok.

snapshot(ActorId) -> SnapshotRef.

restore(SnapshotRef) -> ActorId.

migrate(ActorId, OrganId) -> ok.

replicate(ActorId, ReplicationPolicy) -> ok.

manifest(ActorId, InterfaceSpec) -> ProjectionRef.

remember(Object) -> ObjectId.

recall(Query) -> ResultSet.
```

Implementation notes implied by v0.3: `spawn_actor` creates or attaches a dedicated slim actor kernel on an organ according to `Policy`; `send` always crosses at least one membrane when the recipient is a different actor (different kernel); `grant` and `revoke` update the capability graphs membranes consult. The API remains conceptual—the exact surface may split into supervisor, tissue, and membrane services—but the semantics stay message- and capability-centric.

---

# 13. Failure Model

Failure is expected and continuous. Organism is designed for a world where nothing is permanently up, perfectly synchronized, or fully trusted—yet the organism’s identity, durable commitments, and authorized behavior must still make sense. The baseline posture is degrade, contain, and heal (Section 5), not “avoid failure by centralizing everything.”

The system assumes:

- organs disconnect
- actors crash
- messages delay
- partitions occur
- clocks diverge
- capabilities expire
- kernels may fail independently on an organ

Correctness derives from:

- supervision
- causality
- replication
- reconciliation
- capability security

not perfect connectivity.

## What the assumptions mean

- Organs disconnect—power loss, sleep, network loss, mobility, or policy-enforced isolation. Routing and placement (Sections 7 and 10) must tolerate organs appearing and vanishing; work continues elsewhere when policy allows, or pauses with explicit causal accounting rather than silent corruption.
- Actors crash—logic faults, resource exhaustion, or malicious behavior contained inside one slim actor kernel. Blast radius is local to that kernel’s tenant; supervision restarts, migrates, or substitutes (Section 5) without assuming shared memory with peers.
- Messages delay—unbounded latency is normal on WANs or contested radios. Delivery semantics (Section 4.3) are chosen per path; correctness does not require synchronous global delivery for every interaction.
- Partitions occur—the network may split the organism into disjoint sets of organs that cannot talk. Different partitions may temporarily make inconsistent progress on replicated state; reconciliation after heal merges or surfaces conflicts according to causal and capability rules, rather than pretending a single global write order existed the whole time.
- Clocks diverge—wall clocks skew and jump. Logical clocks and causal metadata (Section 4.1) anchor ordering where it matters; time-limited capabilities (Section 8.3) may use monotonic or attested time sources where policy demands wall-clock alignment.
- Capabilities expire—by design or compromise. Privileged sends fail closed when proofs are stale or revoked (Sections 8.2–8.3); there is no silent fallback to ambient authority.
- Kernels fail independently on an organ—one slim actor kernel may die while siblings and the hypervisor keep running, or the hypervisor may reset subsets of kernels. Isolation boundaries (Section 3.4) prevent one kernel’s failure from being written off as another’s; recovery reattaches mailboxes and durable state through supervised, capability-checked paths.

## How correctness is achieved

- Supervision structures fault domains: who restarts whom, where migration targets lie, and when to give up and surface error to a user or peer organism. Healing is policy-driven, not a single global panic.
- Causality tracks which effects depend on which messages and durable commits. It supports deterministic replay where modeled, explains divergent histories under partition, and pairs with delivery classes so actors know what “happened before” means across delays.
- Replication spreads actors and tissue shards so organ or kernel loss does not erase policy-critical state. Replication interacts with placement trust: not every replica is on every organ class (Section 8.2).
- Reconciliation runs after partitions heal, after conflicting writes, or when attestation or trust tiers change. It may merge object-graph versions, elect authoritative replicas, or require human or collective arbitration—always bounded by capabilities (who may declare merge rules).
- Capability security ensures that recovery, migration, and rehydration from cold tiers do not become backdoors: participants prove authority afresh; membranes do not relax checks “because it’s local.”

Together these mechanisms mean liveness and safety without perfect connectivity: the organism may slow, narrow function, or fork temporarily, but it does not conflate authorization with reachability.

---

# 14. Non-Goals

Organism does not attempt to:

- emulate POSIX
- expose Unix compatibility as a core primitive
- preserve process/thread abstractions
- centralize around a single machine
- require continuous connectivity
- require cloud dependency
- offer shared mutable address spaces between actors as a supported programming model
- allow cross-actor cooperation through undisciplined in-process memory sharing, even when colocated on one organ

---

# 15. Future Directions

Potential future research areas:

- causal scheduling
- semantic memory tissues
- actor evolution semantics
- adaptive UI projection
- predictive actor placement
- distributed cognition models
- organism federation
- collective organisms
- biological-inspired healing strategies
- temporal debugging and replay
- self-optimizing supervision structures
- verified actor kernels and mechanically checked membrane policies
- hardware-specific hypervisor optimizations that preserve semantic equivalence to the message-membrane model
- automatic minimization of foreign dependency graphs with capability-preserving refactorings

---

# 16. Summary

Organism reconceives the operating system as a persistent distributed actor ecosystem centered on computational continuity rather than machine ownership.

Actors become the fundamental unit of existence.

Devices become organs.

The system becomes a life-like computational organism.

From v0.2 into this v0.3 story: that organism’s body is not a single shared kernel heap. On each organ, an Organism hypervisor hosts **many very small actor kernels—one per actor**; cooperation is only through messages that capability membranes admit. Foreign code is minimized where possible and always confined in its own kernel, so trust in people and communities can grow without pretending that every package runs in a friendly universe.

The abstract principles—capabilities, messaging, persistence, collective identity—unchanged in intent—now meet a concrete execution discipline that can be implemented, audited, and extended on real hardware.