# Virtualization & Containerization Bootcamp

## 📖 Phase 1: Theoretical Foundation
Before diving into the hands-on labs, research and understand the following core concepts:
* **Virtualization Basics:** Understand the difference between Virtual Machines (hardware-level virtualization) and Containers (OS-level virtualization).
* **Container Runtimes:** Explore and compare Docker, Podman, and containerd. Understand what role they play in running containers.
* **Kubernetes & Orchestration:** Learn why orchestration is necessary for managing multiple containers in production.
* **Kubernetes Objects:** Deep dive into K8s primitives:
    * *Pod:* The smallest deployable unit.
    * *ConfigMap:* Managing configuration data.
    * *Deployment:* Managing stateless applications and replica sets.
    * *Service:* Networking and exposing applications.
    * *Ingress:* Managing external access to services (HTTP/HTTPS).
    * *StatefulSet:* Managing stateful applications.
* **Kubernetes Flavors:** Research the differences between K8s (upstream), K3s (lightweight, IoT/Edge), and Rancher (multi-cluster management).

---

## 💻 Phase 2: Practical Lab - VMs and Containers
**Objective:** Set up your base environment and get comfortable with container networking.

1.  **Virtual Machine Setup:**
    * Install VMware on your host machine.
    * Spin up at least two VMs: one Linux (Ubuntu or Fedora) and one Windows.
    * Load a VM image with pre-installed software to understand VM snapshots and cloning.
2.  **Docker Fundamentals:**
    * Install Docker on your Linux VM.
    * Run 3 containers simultaneously. Suggested containers:
        1.  `Prometheus` (Monitoring)
        2.  `Prometheus AppDemo` (Target application)
        3.  A custom app of your choosing.
    * Mount a volume to load files from your desktop host into one of the containers.
    * Connect all three containers to a custom Docker bridge network and ensure they can communicate.
3.  **Basic Integration:**
    * Configure Prometheus to scrape metrics from the AppDemo container using its internal Docker network IP or hostname.

---

## 🚀 Phase 3: Practical Lab - K3s & Orchestration
**Objective:** Deploy and manage applications in a local Kubernetes cluster.

1.  **Cluster Provisioning:**
    * Install **K3s** on your Linux VM to create a lightweight Kubernetes cluster.
2.  **Application Deployment Methods:**
    * *Task:* Install applications using three different package management/deployment strategies. Try deploying Prometheus using each method sequentially to learn the differences:
        1.  `kubectl apply` (Raw YAML manifests).
        2.  `kustomize` (Overlay-based configuration).
        3.  `helm` (Templated package manager).
3.  **Writing Custom Manifests:**
    * Find two pre-existing container images for testing: one acting as an HTTP Server, and another acting as an HTTP Client (sending requests).
    * Write raw Kubernetes manifests for both applications. Your manifests must include:
        * `Deployment` (To run the pods).
        * `Service` (To expose the HTTP server internally).
        * `ConfigMap` (To inject configuration, e.g., telling the client which URL to hit).
    * *Validation:* Apply these with `kubectl apply` and verify the client is successfully communicating with the server via HTTP.
    * *Final Challenge:* Run both Prometheus and AppDemo in the K3s cluster. Use a `ConfigMap` to provide Prometheus with its scraping configuration, and use a `Service` to make AppDemo available to Prometheus.
