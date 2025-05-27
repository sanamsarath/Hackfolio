# GetSelectorPolicy Flow: Complete Documentation

## Overview

This document provides comprehensive documentation of the GetSelectorPolicy flow in Cilium, detailing how L4PolicyMap, L4Policy, and L4Filters are created, merged, and managed. The flow represents the complete journey from Cilium Network Policy (CNP/CCNP) rules to BPF map entries that control traffic in the datapath.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Core Components](#core-components)
3. [GetSelectorPolicy Flow Sequence](#getselectorpolicy-flow-sequence)
4. [Component Creation and Lifecycle](#component-creation-and-lifecycle)
5. [Policy Merging Process](#policy-merging-process)
6. [Selector Cache Operations](#selector-cache-operations)
7. [BPF Map State Generation](#bpf-map-state-generation)
8. [Complete Flow Diagrams](#complete-flow-diagrams)
9. [Class Diagrams](#class-diagrams)
10. [Key Data Structures](#key-data-structures)

## Architecture Overview

The GetSelectorPolicy flow is the core policy resolution mechanism in Cilium that transforms high-level network policies into low-level BPF map entries. The flow involves multiple components working together:

- **Repository**: Central policy storage and resolution engine
- **PolicyCache/Distillery**: Caching layer for resolved policies
- **SelectorCache**: Identity resolution and selector management
- **L4PolicyMap/L4Policy/L4Filter**: L4 policy structures
- **MapState**: BPF map entry generation

## Core Components

### Repository
- Central policy repository storing all CNP/CCNP rules
- Provides policy resolution for specific identities
- Manages policy revisions and rule matching
- **Key Method**: `GetSelectorPolicy()`, `resolvePolicyLocked()`

### PolicyCache (Distillery)
- Caches resolved selector policies per identity
- Manages policy lifecycle (creation, updates, deletion)
- Prevents duplicate policy calculations
- **Key Method**: `updateSelectorPolicy()`, `lookupOrCreate()`

### SelectorCache
- Manages mapping between selectors and numeric identities
- Provides incremental updates when identities change
- Caches selector evaluations for performance
- **Key Method**: `AddIdentitySelector()`, `UpdateIdentities()`

### L4PolicyMap
- Container for both ingress and egress L4 policies
- Top-level L4 policy structure per endpoint
- **Key Fields**: `Ingress`, `Egress` (both of type `L4Policy`)

### L4Policy
- Collection of L4 filters organized by protocol
- **Key Fields**: `HTTP`, `UDP`, `TCP`, `SCTP`, `ICMP`, `ICMPv6`

### L4Filter
- Individual L4 policy rule with port/protocol details
- Contains selector-specific policies and rule origins
- **Key Fields**: `Port`, `Protocol`, `PerSelectorPolicies`, `RuleOrigin`

## GetSelectorPolicy Flow Sequence

```mermaid
sequenceDiagram
    participant E as Endpoint
    participant R as Repository
    participant PC as PolicyCache
    participant SC as SelectorCache
    participant SP as SelectorPolicy
    participant EP as EndpointPolicy
    participant MS as MapState

    E->>R: GetSelectorPolicy(identity, skipRevision, stats, endpointID)
    R->>R: mutex.RLock()
    R->>PC: updateSelectorPolicy(identity, endpointID)
    
    PC->>PC: lookupOrCreate(identity)
    PC->>PC: Check cached policy revision
    
    alt Policy not cached or outdated
        PC->>R: resolvePolicyLocked(identity)
        R->>R: computePolicyEnforcementAndRules(identity)
        R->>R: Create new selectorPolicy
        R->>R: resolveL4IngressPolicy(matchingRules)
        R->>R: resolveL4EgressPolicy(matchingRules)
        R->>SP: Create selectorPolicy with L4PolicyMap
        PC->>PC: setPolicy(newSelectorPolicy, endpointID)
        PC->>SP: oldPolicy.detach() if exists
    end
    
    R->>SP: Return cached/new selectorPolicy
    E->>SP: DistillPolicy(logger, policyOwner, redirects)
    
    SP->>SC: GetVersionHandleFunc()
    SC->>EP: Create EndpointPolicy with version handle
    SP->>SP: insertUser(EndpointPolicy)
    
    EP->>EP: toMapState(logger)
    EP->>MS: Generate BPF map entries
    
    EP->>E: Return EndpointPolicy with MapState
```

## Component Creation and Lifecycle

### L4Filter Creation Process

```mermaid
flowchart TD
    A[CNP/CCNP Rule] --> B[Rule Parsing]
    B --> C[Create L4Filter]
    C --> D[Set Port/Protocol]
    D --> E[Cache Selectors]
    E --> F[Create PerSelectorPolicies map]
    F --> G[Set RuleOrigin tracking]
    G --> H[Add to L4Policy]
    
    E --> E1[SelectorCache.AddIdentitySelector]
    E1 --> E2[Create CachedSelector]
    E2 --> E3[Populate with current identities]
    E3 --> E4[Register for incremental updates]
```

### L4Policy Assembly

```mermaid
flowchart TD
    A[Multiple L4Filters] --> B[Group by Protocol]
    B --> C{Protocol Type?}
    C -->|HTTP| D[Add to L4Policy.HTTP]
    C -->|TCP| E[Add to L4Policy.TCP]
    C -->|UDP| F[Add to L4Policy.UDP]
    C -->|SCTP| G[Add to L4Policy.SCTP]
    C -->|ICMP| H[Add to L4Policy.ICMP]
    C -->|ICMPv6| I[Add to L4Policy.ICMPv6]
    
    D --> J[L4Policy Complete]
    E --> J
    F --> J
    G --> J
    H --> J
    I --> J
```

### L4PolicyMap Structure

```mermaid
flowchart TD
    A[L4PolicyMap] --> B[Ingress L4Policy]
    A --> C[Egress L4Policy]
    
    B --> B1[HTTP map port to L4Filter]
    B --> B2[TCP map port to L4Filter]
    B --> B3[UDP map port to L4Filter]
    B --> B4[SCTP map port to L4Filter]
    B --> B5[ICMP map type to L4Filter]
    B --> B6[ICMPv6 map type to L4Filter]
    
    C --> C1[HTTP map port to L4Filter]
    C --> C2[TCP map port to L4Filter]
    C --> C3[UDP map port to L4Filter]
    C --> C4[SCTP map port to L4Filter]
    C --> C5[ICMP map type to L4Filter]
    C --> C6[ICMPv6 map type to L4Filter]
```

## Policy Merging Process

### Rule Merging Algorithm

```mermaid
flowchart TD
    A[Multiple CNP Rules] --> B[Group by Port/Protocol]
    B --> C[Create base L4Filter]
    C --> D[Merge selectors from all rules]
    D --> E[Combine PerSelectorPolicies]
    E --> F[Merge RuleOrigin tracking]
    F --> G[Resolve conflicts]
    G --> H[Final merged L4Filter]
    
    D --> D1[Rule 1 Selectors]
    D --> D2[Rule 2 Selectors]
    D --> D3[Rule N Selectors]
    D1 --> E
    D2 --> E
    D3 --> E
```

### Selector Policy Merging

```mermaid
flowchart TD
    A[Multiple L4Filters for same port] --> B[Merge PerSelectorPolicies]
    B --> C[Union of CachedSelectors]
    C --> D[Combine per-selector rules]
    D --> E[Resolve L7 policy conflicts]
    E --> F[Create unified L4Filter]
    
    B --> B1[Filter 1: sel1 to policy1, sel2 to policy2]
    B --> B2[Filter 2: sel2 to policy3, sel3 to policy4]
    B1 --> C
    B2 --> C
    C --> C1[Merged: sel1 to policy1, sel2 to merged policy2 and policy3, sel3 to policy4]
```

## Selector Cache Operations

### Identity Resolution Flow

```mermaid
sequenceDiagram
    participant L4F as L4Filter
    participant SC as SelectorCache
    participant CS as CachedSelector
    participant IS as IdentitySelector
    participant ID as IdentityCache

    L4F->>SC: AddIdentitySelector(selector)
    SC->>SC: Check if selector exists in cache
    
    alt Selector not cached
        SC->>IS: Create new identitySelector
        SC->>ID: Scan existing identities
        ID->>IS: Add matching identities to cachedSelections
        SC->>CS: Create CachedSelector wrapper
        SC->>SC: Add to selectors map
    end
    
    SC->>CS: Return CachedSelector
    L4F->>L4F: Store in PerSelectorPolicies[CachedSelector]
    
    Note over SC: Future identity updates trigger notifications
    ID->>SC: UpdateIdentities(added, deleted)
    SC->>IS: Update cachedSelections
    IS->>L4F: IdentitySelectionUpdated()
    L4F->>L4F: Trigger policy recomputation
```

### Incremental Updates

```mermaid
flowchart TD
    A[Identity Added/Removed] --> B[SelectorCache.UpdateIdentities]
    B --> C[Scan all cached selectors]
    C --> D{Selector matches identity?}
    D -->|Yes| E[Update cachedSelections]
    D -->|No| F[Next selector]
    E --> G[Notify all users]
    G --> H[L4Filter.IdentitySelectionUpdated]
    H --> I[Trigger endpoint policy recomputation]
    F --> J{More selectors?}
    J -->|Yes| C
    J -->|No| K[Update complete]
```

## BPF Map State Generation

### toMapState Process

```mermaid
flowchart TD
    A[EndpointPolicy.toMapState] --> B[Iterate L4PolicyMap]
    B --> C[Process Ingress L4Policy]
    B --> D[Process Egress L4Policy]
    
    C --> C1[For each protocol map]
    C1 --> C2[For each L4Filter]
    C2 --> C3[L4Filter.toMapState]
    
    C3 --> C4[Iterate PerSelectorPolicies]
    C4 --> C5[Get CachedSelector.GetSelections]
    C5 --> C6[For each identity]
    C6 --> C7[Create MapState entries]
    C7 --> C8[Handle port ranges/named ports]
    C8 --> C9[Add to policyMapState]
    
    D --> D1[Same process for egress]
    
    C9 --> E[Complete MapState]
    D1 --> E
```

### MapState Entry Creation

```mermaid
flowchart TD
    A[L4Filter with resolved identities] --> B{Port specified?}
    B -->|Specific port| C[Create entry for port]
    B -->|Port range| D[Expand range to individual ports]
    B -->|Named port| E[Resolve to numeric port]
    B -->|Wildcard| F[Create wildcard entry]
    
    C --> G[MapStateEntry with TrafficDirection, Protocol, Port, Identity]
    D --> H[Multiple MapStateEntries for each port]
    E --> I[MapStateEntry with resolved port]
    F --> J[MapStateEntry with port 0]
    
    G --> K[Add to MapState with DenyPreferredInsert or AllowPreferredInsert]
    H --> K
    I --> K
    J --> K
```

## Complete Flow Diagrams

### Full GetSelectorPolicy Flow

```mermaid
graph TD
    A[Endpoint Policy Regeneration] --> B[Repository.GetSelectorPolicy]
    B --> C{Policy in cache?}
    C -->|No| D[PolicyCache.updateSelectorPolicy]
    C -->|Yes, outdated| D
    C -->|Yes, current| E[Return cached policy]
    
    D --> F[Repository.resolvePolicyLocked]
    F --> G[computePolicyEnforcementAndRules]
    G --> H[Create new selectorPolicy]
    H --> I[resolveL4IngressPolicy]
    H --> J[resolveL4EgressPolicy]
    
    I --> K[Create L4PolicyMap.Ingress]
    J --> L[Create L4PolicyMap.Egress]
    K --> M[Populate L4Policy with L4Filters]
    L --> M
    
    M --> N[Cache selectors in SelectorCache]
    N --> O[Store selectorPolicy in cache]
    O --> P[Return selectorPolicy]
    
    E --> Q[selectorPolicy.DistillPolicy]
    P --> Q
    
    Q --> R[Create EndpointPolicy]
    R --> S[Register with selectorPolicy for updates]
    S --> T[EndpointPolicy.toMapState]
    T --> U[Generate BPF MapState entries]
    U --> V[Return EndpointPolicy]
```

### Policy Update Flow

```mermaid
sequenceDiagram
    participant CNP as CNP/CCNP
    participant R as Repository
    participant PC as PolicyCache
    participant SC as SelectorCache
    participant L4F as L4Filter
    participant E as Endpoint

    CNP->>R: Policy Add/Update/Delete
    R->>R: Increment revision
    R->>PC: Invalidate affected cached policies
    
    E->>R: Policy regeneration trigger
    R->>PC: updateSelectorPolicy()
    PC->>R: resolvePolicyLocked()
    R->>R: Create new L4Filters
    R->>SC: AddIdentitySelector() for new selectors
    SC->>L4F: Return CachedSelector
    R->>PC: Cache new selectorPolicy
    
    Note over SC: Identity changes trigger updates
    SC->>L4F: IdentitySelectionUpdated()
    L4F->>L4F: Update cached selections
    L4F->>E: Trigger incremental MapState update
```

## Class Diagrams

### Core Policy Classes

```mermaid
classDiagram
    class Repository {
        -mutex RWMutex
        -rules map[uint64]*rule
        -policyCache *policyCache
        +GetSelectorPolicy(identity, skipRevision, stats, endpointID) SelectorPolicy
        +resolvePolicyLocked(identity) *selectorPolicy
        +computePolicyEnforcementAndRules(identity) matchingRules
        +resolveL4IngressPolicy(rules) L4Policy
        +resolveL4EgressPolicy(rules) L4Policy
    }

    class policyCache {
        -mutex Mutex
        -repo *Repository
        -policies map[NumericIdentity]*cachedSelectorPolicy
        +updateSelectorPolicy(identity, endpointID) *selectorPolicy
        +lookupOrCreate(identity) *cachedSelectorPolicy
        +delete(identity) bool
    }

    class cachedSelectorPolicy {
        -mutex Mutex
        -identity *Identity
        -policy atomic.Pointer[selectorPolicy]
        +getPolicy() *selectorPolicy
        +setPolicy(policy, endpointID)
    }

    class selectorPolicy {
        +Revision uint64
        +SelectorCache *SelectorCache
        +L4Policy *L4PolicyMap
        +IngressPolicyEnabled bool
        +EgressPolicyEnabled bool
        +DistillPolicy(logger, owner, redirects) *EndpointPolicy
        +insertUser(user) 
        +removeUser(user)
        +detach(isDelete, endpointID)
    }

    Repository --> policyCache : contains
    policyCache --> cachedSelectorPolicy : manages
    cachedSelectorPolicy --> selectorPolicy : caches
```

### L4 Policy Structure Classes

```mermaid
classDiagram
    class L4PolicyMap {
        +Ingress L4Policy
        +Egress L4Policy
        +detach(selectorCache, isDelete, endpointID)
        +insertUser(user)
        +removeUser(user)
    }

    class L4Policy {
        +HTTP map[string]*L4Filter
        +TCP map[string]*L4Filter
        +UDP map[string]*L4Filter
        +SCTP map[string]*L4Filter
        +ICMP map[string]*L4Filter
        +ICMPv6 map[string]*L4Filter
        +detach(selectorCache, isDelete, endpointID)
        +insertUser(user)
        +removeUser(user)
        +hasRedirect() bool
    }

    class L4Filter {
        +Port int
        +PortName string
        +Protocol u8proto.U8proto
        +U8Proto u8proto.U8proto
        +PerSelectorPolicies map[CachedSelector]*PerSelectorPolicy
        +RuleOrigin map[CachedSelector]labels.LabelArrayList
        +policy atomic.Pointer[L4PolicyMap]
        +toMapState(logger, owner, direction, features) MapState
        +cacheIdentitySelector(sel, lbls, cache) CachedSelector
        +removeSelectors(cache)
        +detach(cache)
        +IdentitySelectionUpdated(logger, selector, added, deleted)
        +IdentitySelectionCommit(logger, txn)
        +IsPeerSelector() bool
    }

    class PerSelectorPolicy {
        +CanReach *regexp.Regexp
        +HTTPRules *HTTPIngressRuleArray
        +HTTPRedirect *api.HTTPRedirect
        +authentication *api.Authentication
    }

    L4PolicyMap --> L4Policy : contains
    L4Policy --> L4Filter : contains
    L4Filter --> PerSelectorPolicy : maps to
    L4Filter --> CachedSelector : uses
```

### Selector Cache Classes

```mermaid
classDiagram
    class SelectorCache {
        -mutex: RWMutex
        -versioned: versioned Coordinator
        -idCache: scIdentityCache
        -selectors: map of string to identitySelector
        -userNotes: list of userNotification
        +AddIdentitySelector(user, lbls, selector): CachedSelector
        +RemoveSelector(selector, user)
        +UpdateIdentities(added, deleted, wg): bool
        +GetVersionHandleFunc(func)
    }

    class identitySelector {
        -logger: slog Logger
        -source: selectorSource
        -key: string
        -selections: versioned Value of NumericIdentitySlice
        -users: map of CachedSelectionUser to struct
        -cachedSelections: map of NumericIdentity to struct
        -metadataLbls: stringLabels
        +GetSelections(version): NumericIdentitySlice
        +Selects(version, nid): bool
        +addUser(user): bool
        +removeUser(user): bool
        +updateSelections(txn)
        +notifyUsers(sc, added, deleted, wg)
    }

    class CachedSelector {
        <<interface>>
        +GetSelections(version): NumericIdentitySlice
        +GetMetadataLabels(): LabelArray
        +Selects(version, nid): bool
        +IsWildcard(): bool
        +IsNone(): bool
        +String(): string
    }

    class CachedSelectionUser {
        <<interface>>
        +IdentitySelectionUpdated(logger, selector, added, deleted)
        +IdentitySelectionCommit(logger, txn)
        +IsPeerSelector(): bool
    }

    SelectorCache --> identitySelector : manages
    identitySelector ..|> CachedSelector : implements
    L4Filter ..|> CachedSelectionUser : implements
```

### EndpointPolicy and MapState Classes

```mermaid
classDiagram
    class EndpointPolicy {
        -selectorPolicy *selectorPolicy
        -VersionHandle *versioned.VersionHandle
        -policyMapState MapState
        -policyMapChanges MapChanges
        +PolicyOwner PolicyOwner
        +Redirects map[string]uint16
        +toMapState(logger)
        +Ready() error
        +Detach(logger)
        +Len() int
        +Get(key) MapStateEntry
    }

    class MapState {
        -entries map[Key]MapStateEntry
        -logger *slog.Logger
        +DenyPreferredInsert(key, entry, features)
        +AllowPreferredInsert(key, entry, features)
        +Delete(key)
        +ForEach(func)
        +Len() int
    }

    class MapStateEntry {
        +ProxyPort uint16
        +DerivedFromRules labels.LabelArrayList
        +IsDeny bool
        +HasAuth AuthType
        +AuthType AuthType
    }

    class Key {
        +Identity uint32
        +DestPort uint16
        +Nexthdr uint8
        +TrafficDirection uint8
    }

    EndpointPolicy --> MapState : contains
    MapState --> MapStateEntry : contains
    MapState --> Key : indexed by
```

## Key Data Structures

### L4Filter Configuration

```go
type L4Filter struct {
    // Port configuration
    Port       int              // Numeric port (0 = wildcard)
    PortName   string           // Named port reference
    Protocol   u8proto.U8proto  // Protocol (TCP, UDP, etc.)
    
    // Selector-specific policies
    PerSelectorPolicies map[CachedSelector]*PerSelectorPolicy
    
    // Rule origin tracking for debugging
    RuleOrigin map[CachedSelector]labels.LabelArrayList
    
    // Reference to parent policy for incremental updates
    policy atomic.Pointer[L4PolicyMap]
}
```

### Selector Policy Resolution

```go
type selectorPolicy struct {
    Revision             uint64          // Policy revision
    SelectorCache        *SelectorCache  // Cached selectors
    L4Policy            *L4PolicyMap    // Resolved L4 policies
    IngressPolicyEnabled bool           // Ingress enforcement
    EgressPolicyEnabled  bool           // Egress enforcement
}
```

### Identity to Selector Mapping

```go
type identitySelector struct {
    source           selectorSource                    // Label selector logic
    selections       versioned.Value[NumericIdentitySlice] // Current selections
    users            map[CachedSelectionUser]struct{}  // Interested parties
    cachedSelections map[NumericIdentity]struct{}      // Fast lookup cache
}
```

### BPF Map Entries

```go
type MapStateEntry struct {
    ProxyPort         uint16                    // L7 proxy redirect port
    DerivedFromRules  labels.LabelArrayList     // Source policy rules
    IsDeny           bool                      // Deny vs Allow
    AuthType         AuthType                  // Authentication requirement
}
```

## Performance Considerations

### Caching Strategy

1. **Selector Caching**: Selectors are cached globally and shared across all policies
2. **Policy Caching**: Resolved policies are cached per identity to avoid recomputation
3. **Incremental Updates**: Only changed identities trigger selector re-evaluation
4. **Versioned Updates**: All updates are versioned to ensure consistency

### Optimization Techniques

1. **Lazy Evaluation**: Policies are only resolved when needed by endpoints
2. **Shared Selectors**: Identical selectors are deduplicated across policies
3. **Batch Updates**: Identity updates are batched to minimize policy recomputations
4. **Fast Path**: Common operations use optimized data structures and algorithms

## Conclusion

The GetSelectorPolicy flow represents a sophisticated policy resolution system that efficiently transforms high-level network policies into low-level datapath configurations. The multi-layered caching, incremental updates, and shared selector architecture ensure both correctness and performance at scale.

The flow demonstrates key architectural principles:
- **Separation of Concerns**: Clear separation between policy storage, resolution, and caching
- **Incremental Processing**: Minimal recomputation when policies or identities change
- **Consistency**: Versioned updates ensure consistent policy state across components
- **Performance**: Multiple levels of caching and optimization for high-throughput scenarios

This documentation serves as a comprehensive guide for understanding, debugging, and extending Cilium's policy resolution system.
