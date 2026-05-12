
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


