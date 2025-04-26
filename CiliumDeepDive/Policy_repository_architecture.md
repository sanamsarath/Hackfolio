# Cilium Policy Repository Architecture

The Policy Repository is a core component of Cilium's networking security system. It stores and manages all network policies that control traffic between workloads in a cluster. This document explains the architecture of the Policy Repository, its key components, storage strategies, and how it's optimized for efficient policy lookups and enforcement.

## Overview

The Policy Repository is designed to efficiently store and query network policies. It maintains multiple indexing strategies to optimize different types of lookups and organizes policies by resource, namespace, and other attributes.
```mermaid
classDiagram
    class Repository {
        +mutex RWMutex
        +rules map[ruleKey]*rule
        +rulesByNamespace map[string]sets.Set[ruleKey]
        +rulesByResource map[ResourceID]map[ruleKey]*rule
        +nextID uint
        +revision atomic.Uint64
        +selectorCache *SelectorCache
        +policyCache *policyCache
        +certManager CertificateManager
        +metricsManager PolicyMetrics
        +l7RulesTranslator EnvoyL7RulesTranslator
        +Search(lbls LabelArray) (Rules, uint64)
        +GetRevision() uint64
        +BumpRevision() uint64
        +ReplaceByResource(rules Rules, resource ResourceID) (set.Set[NumericIdentity], uint64, int)
        +ReplaceByLabels(rules Rules, searchLabelsList []LabelArray) (set.Set[NumericIdentity], uint64, int)
        +GetSelectorPolicy(id *Identity, skipRevision uint64, stats GetPolicyStatistics, endpointID uint64) (SelectorPolicy, uint64, error)
    }
    
    class rule {
        +Rule api.Rule
        +key ruleKey
        +subjectSelector CachedSelector
        +matchesSubject(identity *Identity) bool
        +getSubjects() []NumericIdentity
        +resolveIngressPolicy(policyCtx PolicyContext, state *traceState, result L4PolicyMap, requirements, requirementsDeny []LabelSelectorRequirement) (L4PolicyMap, error)
        +resolveEgressPolicy(policyCtx PolicyContext, state *traceState, result L4PolicyMap, requirements, requirementsDeny []LabelSelectorRequirement) (L4PolicyMap, error)
    }
    
    class ruleKey {
        +resource ResourceID
        +idx uint
    }
    
    class SelectorCache {
        +selectors map[string]*selectorEntry
        +identityToSelectorsMap map[NumericIdentity]map[CachedSelector]struct()
        +Update() void
    }
    
    class policyCache {
        +cache map[NumericIdentity]*cachedSelectorPolicy
        +idMgr identitymanager.IDManager
        +updateSelectorPolicy(identity *Identity, endpointID uint64) (SelectorPolicy, bool, error)
    }

    Repository "1" *-- "*" rule : contains
    rule "1" *-- "1" ruleKey : identified by
    Repository "1" -- "1" SelectorCache : uses
    Repository "1" -- "1" policyCache : uses
```

## Key Components

### Repository

The core structure that manages all policy rules and provides lookups based on different criteria.

### Rule Storage Strategy

The Repository uses multiple indexing strategies for efficient policy lookups:
```mermaid
flowchart TB
    Repository["Policy Repository"]
    Rules["rules map[ruleKey]*rule"]
    ByNamespace["rulesByNamespace map[string]sets.Set[ruleKey]"]
    ByResource["rulesByResource map[ResourceID]map[ruleKey]*rule"]
    
    Repository --> Rules
    Repository --> ByNamespace
    Repository --> ByResource
    
    Rules -- "Main Storage" --> Rule1["rule1"]
    Rules -- "Main Storage" --> Rule2["rule2"]
    Rules -- "Main Storage" --> Rule3["rule3"]
    
    ByNamespace -- "Lookup by namespace" --> Namespace1["namespace1: [key1, key2]"]
    ByNamespace -- "Lookup by namespace" --> Namespace2["namespace2: [key3]"]
    
    ByResource -- "Lookup by resource" --> Resource1["resource1: {key1: rule1}"]
    ByResource -- "Lookup by resource" --> Resource2["resource2: {key2: rule2, key3: rule3}"]
```

### Rule Key Structure

Each rule in the repository is identified by a unique `ruleKey`:
```mermaid
classDiagram
    class ruleKey {
        +resource ResourceID
        +idx uint
    }
    
    note for ruleKey "resource identifies the owning resource\nidx is a unique index within the resource"
```

## Storage Optimization

### Efficient Lookup Strategies

The Policy Repository enables different types of efficient lookups:

1. **Resource-based lookup**: Through `rulesByResource` index
2. **Namespace-based lookup**: Through `rulesByNamespace` index
3. **Label-based lookup**: Through label matching in the `Search` method

### Policy Revision Tracking

Each change to the repository increments a revision counter that allows components to detect when policies have changed.
```mermaid
flowchart LR
    Insert["insert(rule)"] --> BumpRevision["BumpRevision()"]
    Delete["del(key)"] --> BumpRevision
    Replace["ReplaceByResource()"] --> BumpRevision
    BumpRevision --> Notify["Notify endpoints of policy change"]
```

## Example Policy Processing Flow

The following diagram shows how policies flow from Kubernetes into the Policy Repository:
```mermaid
flowchart TB
    K8sPolicy["Kubernetes NetworkPolicy CRD"]
    CiliumPolicy["Cilium Network Policy CRD"]
    LocalAPI["Local API Policy"]
    
    K8sPolicy --> |"Converted to API Rules"| Repository["Policy Repository"]
    CiliumPolicy --> |"Converted to API Rules"| Repository
    LocalAPI --> |"Converted to API Rules"| Repository
    
    Repository --> Insert["insert(rule)"]
    Repository --> Delete["del(ruleKey)"]
    Repository --> ReplaceResource["ReplaceByResource(rules, resourceID)"]
    Repository --> ReplaceLabels["ReplaceByLabels(rules, labelArray)"]
    
    Insert --> BumpRevision["Bump Repository Revision"]
    Delete --> BumpRevision
    ReplaceResource --> BumpRevision
    ReplaceLabels --> BumpRevision
    
    BumpRevision --> Recalculate["Recalculate affected endpoints"]
```

## Example: Storage of a Network Policy

Let's examine how a simple Kubernetes NetworkPolicy is stored in the repository:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/16
    ports:
    - protocol: TCP
      port: 5978
```

This policy would be represented in the repository as:

1. A `rule` object with ingress and egress rules.
2. A `ruleKey` with:
   - `resource` identifying the NetworkPolicy resource 
   - `idx` set to a unique index (usually 0 since a single K8s NetworkPolicy maps to one Cilium rule)
3. Entries in all indexing maps:
   - Added to `rules` with the ruleKey as the key
   - Added to `rulesByNamespace["default"]` 
   - Added to `rulesByResource[resourceID]` where resourceID uniquely identifies the K8s NetworkPolicy

## Lookup Operations

### Identity-based Policy Lookup

When determining what policies apply to an endpoint with a given identity:
```mermaid
flowchart TB
    GetSelectorPolicy["GetSelectorPolicy(identity)"]
    CheckCache["Check policyCache for identity"]
    
    subgraph "Cache Miss Path"
    ComputeRules["computePolicyEnforcementAndRules(identity)"]
    MatchClusterWide["Match cluster-wide rules"]
    MatchNamespace["Match namespace rules"]
    ResolvePolicy["resolvePolicyLocked(identity)"]
    end
    
    GetSelectorPolicy --> CheckCache
    CheckCache -- "Not found" --> ComputeRules
    CheckCache -- "Found and not stale" --> ReturnCached["Return cached policy"]
    
    ComputeRules --> MatchClusterWide
    ComputeRules --> MatchNamespace
    ComputeRules --> ResolvePolicy
    ResolvePolicy --> UpdateCache["Update policyCache"]
    UpdateCache --> ReturnPolicy["Return policy"]
```

### Resource-based Policy Management

When replacing policies associated with a resource (e.g., when a NetworkPolicy is updated):
```mermaid
flowchart TB
    ReplaceByResource["ReplaceByResource(rules, resourceID)"]
    RemoveOldRules["Remove all existing rules for resourceID"]
    InsertNewRules["Insert new rules with resourceID"]
    TrackAffectedIDs["Track affected identity IDs"]
    
    ReplaceByResource --> RemoveOldRules
    RemoveOldRules --> InsertNewRules
    InsertNewRules --> TrackAffectedIDs
    TrackAffectedIDs --> BumpRevision["Bump revision & return affected IDs"]
```

## Optimizations

1. **Revision-based Skip**: The `GetSelectorPolicy` method accepts a `skipRevision` parameter to avoid recomputing policies when the repository hasn't changed.

2. **Namespace-based Indexing**: Rules are indexed by namespace for faster lookups when evaluating which policies apply to pods in a specific namespace.

3. **Resource-based Management**: Rules are grouped by the resource that created them, allowing efficient updates when resources change.

4. **Label-based Filtering**: The repository can efficiently find rules that match specific label sets.

## Conclusion

The Cilium Policy Repository architecture is designed for efficiency and scalability, with multiple indexing strategies to optimize different types of lookups. Its structure enables Cilium to efficiently enforce network policies across a large number of endpoints while minimizing the computational overhead of policy evaluation.