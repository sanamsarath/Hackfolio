# Cilium Policy Creation Flow

This document describes the architecture and flow of how network policies are created, processed, and applied in Cilium.

## Architecture Overview Diagram
```mermaid
flowchart TB
    %% Color scheme for components - optimized for both light and dark backgrounds
    %% Policy Sources - Medium Blue
    %% Watchers - Medium Green
    %% Policy Importer - Medium Orange
    %% Policy Repository - Medium Purple
    %% Endpoint Manager - Medium Red
    %% Endpoint Policy - Medium Teal
    %% Data Path - Medium Gray

    subgraph "Policy Sources" 
        style Policy Sources fill:#4b89dc,stroke:#2c5499,color:#000
        kube["Kubernetes API"]
        cidrg["CiliumCIDRGroup"]
        file["Directory-based Files"]
        api["Local REST API"]
        
        %% Style all nodes in this subgraph
        style kube fill:#4b89dc,stroke:#2c5499,color:#000
        style cidrg fill:#4b89dc,stroke:#2c5499,color:#000
        style file fill:#4b89dc,stroke:#2c5499,color:#000
        style api fill:#4b89dc,stroke:#2c5499,color:#000
    end

    subgraph "Policy Watchers"
        style Policy Watchers fill:#37bc9b,stroke:#2b9484,color:#000
        k8sWatch["K8s Policy Watcher"]
        cidrWatch["CIDRGroup Watcher"]
        dirWatch["Directory Watcher"]
        
        %% Style all nodes in this subgraph
        style k8sWatch fill:#37bc9b,stroke:#2b9484,color:#000
        style cidrWatch fill:#37bc9b,stroke:#2b9484,color:#000
        style dirWatch fill:#37bc9b,stroke:#2b9484,color:#000
    end
    
    subgraph "Policy Importer"
        style Policy Importer fill:#f6bb42,stroke:#c69426,color:#000
        direction TB
        importer["Policy Importer"]
        prefixHandler["CIDR Prefix Handler"]
        policyQueue["Policy Update Queue"]
        processor["Policy Processor"]
        
        %% Style all nodes in this subgraph
        style importer fill:#f6bb42,stroke:#c69426,color:#000
        style prefixHandler fill:#f6bb42,stroke:#c69426,color:#000
        style policyQueue fill:#f6bb42,stroke:#c69426,color:#000
        style processor fill:#f6bb42,stroke:#c69426,color:#000
    end
    
    subgraph "Policy Repository"
        style Policy Repository fill:#967adc,stroke:#7652c6,color:#000
        direction TB
        repo["Policy Repository"]
        ruleStore["Rule Storage 
        - rules (map)
        - rulesByNamespace
        - rulesByResource"]
        revision["Revision Counter"]
        policyCache["Policy Cache"]
        selCache["Selector Cache"]
        
        %% Style all nodes in this subgraph
        style repo fill:#967adc,stroke:#7652c6,color:#000
        style ruleStore fill:#967adc,stroke:#7652c6,color:#000
        style revision fill:#967adc,stroke:#7652c6,color:#000
        style policyCache fill:#967adc,stroke:#7652c6,color:#000
        style selCache fill:#967adc,stroke:#7652c6,color:#000
    end
    
    subgraph "Endpoint Manager"
        style Endpoint Manager fill:#fc6e51,stroke:#cd5940,color:#000
        epManager["Endpoint Manager"]
        epRegen["Endpoint Regenerator"]
        epCallback["Policy Update Callbacks"]
        
        %% Style all nodes in this subgraph
        style epManager fill:#fc6e51,stroke:#cd5940,color:#000
        style epRegen fill:#fc6e51,stroke:#cd5940,color:#000
        style epCallback fill:#fc6e51,stroke:#cd5940,color:#000
    end
    
    subgraph "Endpoint Policy Application"
        style Endpoint Policy Application fill:#5bc0de,stroke:#3998b6,color:#000
        direction TB
        ep["Endpoint"]
        desiredPolicy["Desired Policy"]
        realizedPolicy["Realized Policy"]
        policyMap["BPF Policy Map"]
        
        %% Style all nodes in this subgraph
        style ep fill:#5bc0de,stroke:#3998b6,color:#000
        style desiredPolicy fill:#5bc0de,stroke:#3998b6,color:#000
        style realizedPolicy fill:#5bc0de,stroke:#3998b6,color:#000
        style policyMap fill:#5bc0de,stroke:#3998b6,color:#000
    end
    
    subgraph "Data Path"
        style Data Path fill:#8e8e93,stroke:#6e6e73,color:#000
        bpf["BPF Programs"]
        
        %% Style all nodes in this subgraph
        style bpf fill:#8e8e93,stroke:#6e6e73,color:#000
    end

    %% Connections between components
    kube --> k8sWatch
    cidrg --> cidrWatch
    file --> dirWatch
    
    k8sWatch --> policyQueue
    cidrWatch --> policyQueue
    dirWatch --> policyQueue
    api --> policyQueue
    
    policyQueue --> processor
    processor --> prefixHandler
    prefixHandler --> repo
    
    processor --> repo
    
    repo --> ruleStore
    repo --> revision
    repo --> policyCache
    repo --> selCache
    
    processor --"UpdatePolicy(idsToRegen, fromRev, toRev)"--> epManager
    
    epManager --> epRegen
    epManager --> epCallback
    
    epRegen --> ep
    
    ep --> desiredPolicy
    desiredPolicy --> realizedPolicy
    realizedPolicy --> policyMap
    
    policyMap --> bpf
    
    %% Special step - selector cache updating identity
    selCache --"Notify changed identities"--> policyCache
    policyCache --"EndpointPolicy updates"--> desiredPolicy
```

### Flow Summary

```
Policy Sources (Kubernetes API, CiliumCIDRGroup, Files, REST API)
    ↓
Policy Watchers (detect changes in policies)
    ↓
Policy Importer (processes, queues, and batches policy updates)
    ↓
CIDR Prefix Handler (allocates identities for CIDRs)
    ↓
Policy Repository (stores rules, maintains caches and revision counter)
    ↓
Endpoint Manager (coordinates updates to affected endpoints)
    ↓
Endpoint Policy Application (calculates and applies policies to endpoints)
    ↓
BPF Maps and Programs (enforce policies in the datapath)
```

## Component Descriptions

### Policy Sources
- **Kubernetes API**: Source for CiliumNetworkPolicy, CiliumClusterwideNetworkPolicy, K8sNetworkPolicy resources
- **CiliumCIDRGroup**: Source for IP CIDR grouping policies
- **Directory-based Files**: Source for policies stored as files on disk
- **Local REST API**: Source for policies added via Cilium API/CLI

### Policy Watchers
Policy watchers monitor changes from various policy sources and react to events (create, update, delete):

- **K8s Policy Watcher**: Watches Kubernetes API for policy changes
- **CIDRGroup Watcher**: Watches for CIDR group changes
- **Directory Watcher**: Monitors the filesystem for policy file changes

The watchers translate policy definitions from their respective formats into a common internal representation and forward them to the Policy Importer.

### Policy Importer
- **Policy Queue**: Buffers incoming policy updates for batched processing
- **Prefix Handler**: Manages CIDR prefixes from policies, updating the IPCache
- **Policy Processor**: Core processing logic that:
  - Allocates local identities for CIDR prefixes
  - Updates policies in the repository
  - Tracks affected identities for endpoint regeneration
  - Sends policy update notifications

### Policy Repository
Central storage for all policy rules with several key components:
- **Rule Storage**: Maintains indexed maps of rules:
  - `rules`: Main storage mapping rule keys to rule objects
  - `rulesByNamespace`: Indexes rules by namespace
  - `rulesByResource`: Indexes rules by resource ID
- **Revision Counter**: Tracks policy version for change detection
- **Policy Cache**: Caches computed policies per identity for performance
- **Selector Cache**: Efficiently maps endpoint selectors to matching identities

### Endpoint Manager
- **Endpoint Manager**: Manages all endpoints in the node
- **Endpoint Regenerator**: Handles endpoint regeneration process
- **Policy Update Callbacks**: Notifies registered components about policy changes

### Endpoint Policy Application
- **Endpoint**: Represents a container/pod network endpoint
- **Desired Policy**: The newly calculated policy that should be applied
- **Realized Policy**: The policy that has been successfully applied
- **Policy Map**: BPF map containing policy rules enforced in the datapath

### Data Path
- **BPF Programs**: Kernel programs that enforce policies at the datapath level

## Policy Creation Flow

1. **Policy Event Detection**:
   - Policy changes are detected by dedicated watchers
   - Each policy source (Kubernetes, file, API) has its own watcher

2. **Policy Translation**:
   - Watchers normalize policies into a common format
   - Policies are encapsulated in `PolicyUpdate` objects

3. **Policy Queuing**:
   - Updates are sent to the policy importer's queue
   - Updates are batched for efficient processing

4. **CIDR Identity Allocation**:
   - Prefix handler extracts CIDR prefixes from policies
   - Local identities are allocated for these prefixes
   - IPCache is updated with prefix metadata

5. **Policy Repository Update**:
   - Policies are added/replaced in the repository
   - Rules are indexed by different criteria (namespace, resource)
   - Repository revision number is incremented
   - Set of affected identities is calculated

6. **Selector Cache Updates**:
   - New selectors are added to the selector cache
   - Matching identities are calculated for each selector

7. **Endpoint Policy Propagation**:
   - Endpoint Manager is notified with list of affected identities
   - For affected endpoints, regeneration is triggered
   - Unaffected endpoints only update their policy revision

8. **Endpoint Policy Application**:
   - Endpoint calculates desired policy from repository
   - Policy changes are translated into policy map updates
   - Realized policy is set after successful application

9. **BPF Map Updates**:
   - Policy map changes are applied to BPF maps
   - Datapath starts enforcing the new policy rules