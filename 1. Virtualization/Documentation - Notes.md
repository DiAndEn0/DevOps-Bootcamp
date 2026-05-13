
### Virtualization - 

---
#### Virtualization Basics - 

> **Virtualization** - in itself is a bunch of technologies that allow to divide physical computing resources into series of virtual machines, OS, processes or containers. **Hypervisor** is the software/firmware that creates the VMs. 

![[Pasted image 20260512213055.png]]

> **Desktop Virtualization** is the concept of separating logical desktop from the physical machine (Will get to this later in more detail)

> In **Hardware Virtualization** the host machine is the one being used by the virtualization and the guest machine is the VM. **Hardware Virtualization** is essentially a virtualization of the hardware required to run the different OS, VMs and processes on the host machine, and by allocating resources from the host machine it's possible to create virtualized isolated hardware environments where you're able to install and run OS instead of using different physical hosts. 
> 
> **On Platform Virtualization:**
> 	On the *hardware platform* the **Hypervisor** (host software/control program) creates a simulated computer environment which is called a **VM** for its *guest software* which could range from user applications to complete operating systems. It executes as if running directly on the physical hardware.
> 
> There are a few caveats that come with the technique:
> 	 1. There are performances penalties to run the **hypervisor** and the VMs itself compared to running it native on the physical machine.
> 	 2. Restricted access to physical system resources (such as Network, Disk Usage and etc). Based on policies enacted by the virtualization host.
>  
> So why do we need this **Hardware Virtualization**?
> 	1. *Server consolidation* - Using one physical server instead of many small ones. It's called P2V transformation.
> 	2. Reduces equipment and labor costs, reduces energy consumption.
> 	3. Better control and monitoring over the different VMs since they are on the same physical server.
> 	4. Faster creation of new VMs
> 	5. Easy relocation of VMs from one machine to another.
> 	6. Disaster recovery
> 	
> 	**But**: there is a matter of unpredictability when it comes to resource allocation, need to use different techniques to achieve stability between the performance of different VMs on the same machine. 

![[virtual-machine-diagram.svg]]

> **OS-level Virtualization** - Virtualization which happens at the kernel level, meaning that instead of virtualizing the entire machine, it shares its kernel with the physical machine, and creates *guest environments* directly from it. 
> 
> **Containers** are units of software that hold together the components and functionalities needed for an application to run. They don't require a *guest operating system* and their primary role is to nicely package an application together with its dependencies in a portable unit. They're a lot smaller than VMs and they can be deployed anywhere knowing it will be stable.
> They can also be used to split applications into functions and microservices.


> So, what's the difference and what do we need for what task?


| VMs (Hardware Virtualization)                        | Containers (OS-level Virtualizaion)      |
| ---------------------------------------------------- | ---------------------------------------- |
| Can run different OS                                 | Share a single OS                        |
| Virtualizes hardware                                 | Virtualize the OS or software            |
| Large Size (Gbs)                                     | Small Size (Mbs)                         |
| Takes longer to run                                  | Very lightweight, takes less time to run |
| Uses a lot of system memory                          | Require a lot less memory                |
| More secure, hardware isn't shared between processes | Less secure, memory is shared            |

#### Container Runtimes - 


 **Docker** - is one of the most popular container runtimes currently on the market. It adheres to the **OCI** (Open Container Initiative), and its primary feature is running container images. Its features include: 
- **Client-Server** Architecture - works by forwarding the CLI commands to a background process (**dockerd**) that manages all container lifecycle operations. Sends request through a socket (Docker's REST API).
- **dockerd** - *daemon process* that runs with root privileges by default (rootless limited) and uses container runtime components to process requests from the Docker Binary.
- Can be remote controlled by exposing Docker's API socket on a remote host machine.
- **Dockerfile** - provides instructions to build container image.
- **Docker Compose file**  -  defines running containers and reference a **dockerfile** to build and image to use for particular service.

![[docker_daemon.webp]]


**Podman** - More lightweight, container runtime that is **Docker**-Compatible CLI without the Docker daemon basically. It has its own GUI called Podman Desktop and is a great alternative to Docker. Its features include:
- **Rootless** by default - meaning it cannot be compromised if an attacker gains root inside the container. Works by remapping permissions, making the container think it's running on root while in reality it runs on a different UID.
- **Daemonless** - the container manages its own requests without sending everything to a background process like **dockerd** or by systemd if on linux, which isolates the containers from each other.
- No single point of failure
- Lower overhead - when containers aren't running does not consume resources
- Security - no exposure risk.


**Containerd** - is an open-source, lightweight container runtime focused on running in a **rootless architecture**, it adheres to the **OCI**. Its main functionality is managing container lifecycle, it is powering **Kubernetes**, **Docker** as a backend and **cloud platforms** (AWS Fargate, Google Cloud Run, Azure Container Instances). 
- While lacking many features provided by Docker, it exists as a component in a larger system like **Kubernetes** and **Docker**, not a standalone tool.
- Used in Kubernetes clusters as the de facto standard
- Lighter and faster than docker in startup, memory overhead and image pulling
- Used in IoT devices with limited resources.
- Used in building custom Container Platforms
	- Examples include AWS Firecracker, Rancher/K3s, Nomad
- In order to use it directly there is a tool called **nerdctl** which provides Docker CLI compatibility. 



| Criterion                    | Docker Engine + Desktop                | Podman + Desktop                     | containerd + nerdctl          |
| ---------------------------- | -------------------------------------- | ------------------------------------ | ----------------------------- |
| **Architecture**             | Client-server with root daemon         | Daemonless, fork model               | Minimalist daemon (not root)  |
| **Root requirement**         | Daemon as root (rootless mode limited) | Rootless by default                  | Can run rootless              |
| **Docker CLI compatibility** | 100% (this is Docker)                  | ~95% drop-in replacement             | ~98% via nerdctl              |
| **Compose support**          | Native docker compose                  | podman-compose (~90% coverage)       | nerdctl compose (~95%)        |
| **Build Dockerfiles**        | BuildKit built-in                      | Buildah integrated                   | BuildKit via buildctl/nerdctl |
| **Kubernetes integration**   | Deprecated (v1.24+), via containerd    | podman generate/play kube            | Native (K8s default runtime)  |
| **Startup latency**          | Fast (daemon pre-warmed)               | ~10-15% slower (fork overhead)       | Fastest (minimal runtime)     |
| **Memory overhead**          | ~100MB daemon + containers             | ~0MB daemon + containers             | ~30MB runtime + containers    |
| **Security (rootless)**      | Limited rootless support               | Rootless first-class                 | Supports rootless             |
| **Image registry**           | Docker Hub seamless                    | Compatible (OCI registries)          | Compatible (OCI registries)   |
| **Windows containers**       | ★★★★★ Native support                   | ★★☆☆☆ Limited, improving             | ★★★☆☆ Via containerd shim     |
| **macOS/Windows host**       | Docker Desktop (native-like)           | Podman Machine (VM-based)            | nerdctl + Lima VM             |
| **CI/CD friendliness**       | Good (but shared daemon = risk)        | ★★★★★ Excellent (rootless isolation) | Good (K8s-native pipelines)   |
| **Production runtime**       | Legacy (K8s deprecated)                | Growing (Red Hat OpenShift)          | ★★★★★ K8s default             |
