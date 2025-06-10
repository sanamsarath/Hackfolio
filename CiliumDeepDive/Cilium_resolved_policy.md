# Cilium New Policy Creation Flow (with CiliumResolvedPolicy)

This document describes the updated architecture and flow of how network policies are created, processed, and applied in Cilium, focusing on the `CiliumResolvedPolicy` custom resource.

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

    subgraph Policy Source
        style Policy Source fill:#4b89dc,stroke:#2c5499,color:#000
        crp["CiliumResolvedPolicy (K8s CRD)"]
        
        %% Style node
        style crp fill:#4b89dc,stroke:#2c5499,color:#000
    end

    subgraph Policy Watcher
        style Policy Watcher fill:#37bc9b,stroke:#2b9484,color:#000
        crpWatch["CiliumResolvedPolicy Watcher"]
        
        %% Style node
        style crpWatch fill:#37bc9b,stroke:#2b9484,color:#000
    end
    
    subgraph Policy Importer
        style Policy Importer fill:#f6bb42,stroke:#c69426,color:#000
        direction TB
        importer["Policy Importer"]
        policyQueue["Policy Update Queue"]
        processor["Policy Processor"]
        
        %% Style all nodes in this subgraph
        style importer fill:#f6bb42,stroke:#c69426,color:#000
        style policyQueue fill:#f6bb42,stroke:#c69426,color:#000
        style processor fill:#f6bb42,stroke:#c69426,color:#000
    end
    
    subgraph Policy Repository
        style Policy Repository fill:#967adc,stroke:#7652c6,color:#000
        direction TB
        resolvedPolicyCache["ResolvedPolicyCache"]
        
        %% Style node
        style resolvedPolicyCache fill:#967adc,stroke:#7652c6,color:#000
    end
    
    subgraph Endpoint Manager
        style Endpoint Manager fill:#fc6e51,stroke:#cd5940,color:#000
        epManager["Endpoint Manager"]
        epRegen["Endpoint Regenerator"]
        epCallback["Policy Update Callbacks"]
        
        %% Style all nodes in this subgraph
        style epManager fill:#fc6e51,stroke:#cd5940,color:#000
        style epRegen fill:#fc6e51,stroke:#cd5940,color:#000
        style epCallback fill:#fc6e51,stroke:#cd5940,color:#000
    end
    
    subgraph Endpoint Policy Application
        style Endpoint Policy Application fill:#5bc0de,stroke:#3998b6,color:#000
        direction TB
        ep["Endpoint"]
        desiredPolicy["Desired Policy (from ResolvedPolicyCache)"]
        realizedPolicy["Realized Policy"]
        policyMap["BPF Policy Map"]
        
        %% Style all nodes in this subgraph
        style ep fill:#5bc0de,stroke:#3998b6,color:#000
        style desiredPolicy fill:#5bc0de,stroke:#3998b6,color:#000
        style realizedPolicy fill:#5bc0de,stroke:#3998b6,color:#000
        style policyMap fill:#5bc0de,stroke:#3998b6,color:#000
    end
    
    subgraph Data Path
        style Data Path fill:#8e8e93,stroke:#6e6e73,color:#000
        bpf["BPF Programs"]
        
        %% Style node
        style bpf fill:#8e8e93,stroke:#6e6e73,color:#000
    end

    %% Connections between components
    crp --> crpWatch
    
    crpWatch --> policyQueue
    
    policyQueue --> processor
    processor --> resolvedPolicyCache
    
    processor --"UpdatePolicy(idsToRegen)"--> epManager 
    
    epManager --> epRegen
    epManager --> epCallback
    
    epRegen --> ep
    
    ep --"Requests Desired State"--> resolvedPolicyCache
    resolvedPolicyCache --"Provides Desired State"--> desiredPolicy
    desiredPolicy --> realizedPolicy
    realizedPolicy --> policyMap
    
    policyMap --> bpf
    
```

## Flow Summary (New)

\`\`\`
CiliumResolvedPolicy (K8s CRD)
    ↓
CiliumResolvedPolicy Watcher (detects changes in CiliumResolvedPolicy resources)
    ↓
Policy Importer (processes, queues, and batches policy updates)
    ↓
Policy Repository (stores updates in ResolvedPolicyCache)
    ↓
Endpoint Manager (coordinates updates to affected endpoints)
    ↓
Endpoint (requests desired policy state from ResolvedPolicyCache)
    ↓
Endpoint Policy Application (calculates and applies policies to endpoints)
    ↓
BPF Maps and Programs (enforce policies in the datapath)
\`\`\`

## Component Descriptions (Updated)

### Policy Source
- **CiliumResolvedPolicy (K8s CRD)**: The sole source for pre-resolved network policies.

### Policy Watcher
- **CiliumResolvedPolicy Watcher**: Watches Kubernetes API for `CiliumResolvedPolicy` resource changes. Translates these into a common internal representation and forwards them to the Policy Importer.

### Policy Importer
- **Policy Queue**: Buffers incoming policy updates for batched processing.
- **Policy Processor**: Core processing logic that:
  - Updates policies in the `ResolvedPolicyCache`.
  - Tracks affected identities for endpoint regeneration.
  - Sends policy update notifications.

### Policy Repository
Central storage for resolved policies:
- **ResolvedPolicyCache**: Caches the `CiliumResolvedPolicy` objects. Endpoints directly query this cache for their desired policy state.

### Endpoint Manager
- **Endpoint Manager**: Manages all endpoints in the node.
- **Endpoint Regenerator**: Handles endpoint regeneration process when underlying `CiliumResolvedPolicy` changes affect an endpoint.
- **Policy Update Callbacks**: Notifies registered components about policy changes.

### Endpoint Policy Application
- **Endpoint**: Represents a container/pod network endpoint.
- **Desired Policy**: The policy state fetched directly from `ResolvedPolicyCache`.
- **Realized Policy**: The policy that has been successfully translated and applied to the datapath.
- **Policy Map**: BPF map containing policy rules enforced in the datapath.

### Data Path
- **BPF Programs**: Kernel programs that enforce policies at the datapath level.

## New Policy Creation Flow Steps

1.  **Policy Event Detection**:
    *   Changes to `CiliumResolvedPolicy` resources are detected by the dedicated `CiliumResolvedPolicy Watcher`.

2.  **Policy Translation (Minimal)**:
    *   The watcher forwards the `CiliumResolvedPolicy` (or relevant parts) to the policy importer.

3.  **Policy Queuing**:
    *   Updates are sent to the policy importer's queue.
    *   Updates are batched for efficient processing.

4.  **ResolvedPolicyCache Update**:
    *   The Policy Processor pushes the `CiliumResolvedPolicy` updates into the `ResolvedPolicyCache`.
    *   The set of affected identities/endpoints is calculated.

5.  **Endpoint Notification & Regeneration**:
    *   Endpoint Manager is notified with a list of affected endpoints.
    *   For affected endpoints, regeneration is triggered.

6.  **Endpoint Fetches Desired State**:
    *   The Endpoint, during its regeneration or policy update cycle, directly requests its desired policy state from the `ResolvedPolicyCache`.

7.  **Endpoint Policy Application**:
    *   Endpoint translates the desired policy into policy map updates.
    *   Realized policy is set after successful application.

8.  **BPF Map Updates**:
    *   Policy map changes are applied to BPF maps.
    *   Datapath starts enforcing the new policy rules.
