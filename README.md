# Project Sentinel: Heterogeneous Hybrid-Cloud AI Platform (K3s/Proxmox)

## Project Summary
This project demonstrates advanced DevOps, Cloud, and Kubernetes orchestration by deploying a private, PII-compliant RAG (Retrieval Augmented Generation) system that intelligently routes workloads based on available hardware and privacy rules.

**Key Technologies:** K3s, Proxmox, NVIDIA GPU, Intel AVX-512, TrueNAS NFS, Ollama, LiteLLM, Open WebUI.

## 1. Core Architectural Requirements Achieved
| Requirement | Hardware/Component | Resume Impact |
| :--- | :--- | :--- |
| **Heterogeneous Scheduling** | K3s Node Selectors (`special.gpu=nvidia`, `special.cpu=avx512`) | **Advanced K8s Orchestration.** Proof of resource-aware scheduling. |
| **GPU Acceleration (AI)** | NVIDIA RTX 3060 Ti | **MLOps Proficiency.** Demonstrated GPU runtime configuration (Containerd/K3s). |
| **Specialized Compute** | Intel AVX-512 Instructions | **High-Performance Compute.** Proof of CPU passthrough for speed (OCR/Vector). |
| **Hybrid Cloud Routing** | LiteLLM Proxy + Open WebUI Functions | **Context-Aware Sovereignty.** PII routes Local (GPU); General routes Cloud (Gemini). |
| **Data Persistence** | TrueNAS NFS Provisioner | **Enterprise Resiliency.** Decoupled state from compute nodes for high availability. |

## 2. Infrastructure Setup (Proxmox/OS)

**Prerequisites:**
1.  **Proxmox IOMMU:** Enabled in GRUB (`intel_iommu=on iommu=pt`).
2.  **VM CPU Type:** Set to **`host`** for all VMs to expose AVX-512 flags.
3.  **GPU Passthrough:** RTX 3060 Ti passed through via PCIe to `node-gpu`.
4.  **K3s Installation:** Completed (Control Plane + 2 Agents).

## 3. Configuration & Deployment

### Initial Configuration (Manual Step)
Run these commands first to label the hardware:
```bash
# Label the Nodes
kubectl label node node-gpu special.gpu=nvidia
kubectl label node node-compute special.cpu=avx512


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