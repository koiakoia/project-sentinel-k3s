# Project Sentinel Network Architecture

### Hybrid Cloud Routing Diagram

```mermaid
graph TD
    %% --- Styles ---
    classDef user fill:#fff,stroke:#333,stroke-width:2px;
    classDef private fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,stroke-dasharray: 5 5;
    classDef public fill:#fbe9e7,stroke:#d84315,stroke-width:2px;
    classDef logic fill:#fff9c4,stroke:#fbc02d,stroke-width:2px;
    classDef hardware fill:#424242,stroke:#000,stroke-width:2px,color:#fff;

    %% --- The User Layer ---
    subgraph Interface [Frontend Layer]
        User([User / Client]):::user -->|HTTPS / Secure| WebUI[Open WebUI]:::user
    end

    %% --- The Logic Layer ---
    subgraph Routing [Decision Layer]
        WebUI -->|API Request| Proxy[LiteLLM Proxy]:::logic
        Proxy -->|Analyze Payload| Router{Contains PII?}:::logic
    end

    %% --- The Private Cloud (Your Basement) ---
    subgraph Private_Cloud [Private Infrastructure - Proxmox/K3s]
        style Private_Cloud fill:#e3f2fd,stroke:#1565c0
        
        Router -->|YES: Sensitive Data| LocalService[Local Inference Service]:::private
        
        subgraph Hardware [Hardware Allocation]
            style Hardware fill:#eeeeee,stroke:#9e9e9e
            
            NodeGPU[Node-GPU <br/> NVIDIA RTX 3060 Ti <br/> (Passthrough)]:::hardware
            NodeCPU[Node-Compute <br/> Intel AVX-512 <br/> (Vector Search)]:::hardware
        end

        LocalService -->|Run Model| NodeGPU
        LocalService -->|Embeddings| NodeCPU
        
        %% Storage
        NodeGPU -.->|Persistent Storage| NAS[(TrueNAS Core <br/> NFS)]:::hardware
    end

    %% --- The Public Cloud ---
    subgraph Public_Cloud [Public Cloud]
        style Public_Cloud fill:#fbe9e7,stroke:#d84315
        
        Router -->|NO: General Queries| CloudAPI[Google Gemini API]:::public
    end

    %% --- Returns ---
    NodeGPU -->|Secure Response| Proxy
    CloudAPI -->|Public Response| Proxy
    Proxy -->|Final Answer| User
