# Container technologies
February 28

## Reminders

- Lab 3 due March 7
  - Ask for Lab 2 solution if you are unsure about your Lab 1 code
- Send us Project 2 issues via private note on Piazza


## Lecture notes

### Overview
- Key container technologies
- - Kernel namespaces
  - Control groups (cgroups)
  - Overlay copy-on-write (COW) file system
- Software deployment and microservices
- Container orchestration

### Container technologies
- Kernel namespaces: control visibility; determine what a container can see
  - What hostname do containerized processes see? What IP addresses? What process does a given process ID refer to? What user does a given user ID refer to?
- cgroups: control resource usage; limit the resources that a container is allowed to use
- Overlay file system: allows containers to share a container image/dependencies; deduplicates shared resources

#### Kernel namespaces
- Controls visibility 
- E.g. process IDs are only visible with in the same namespace
  - Processes in different namespaces may have the same PID
  - Processes are uniquely identified by both their namespace and process ID
  - Each namespace can have its own root user
```
|-------------------| 
|      default      | hostname: alice
|     namespace     | IP: 123
|                   |
| ┌---┐ ┌---┐ ┌---┐ |
| |100| |200| |200| |---------------|
| └---┘ └---┘ └---┘ | network stack |
|-------------------|---------------|

|-------------------|
|   namespace #1    | hostname: bob
|                   | IP: 456
| ┌---┐ ┌---┐ ┌---┐ |
| |100| |200| |200| |---------------|
| └---┘ └---┘ └---┘ | network stack |
|-------------------|---------------|

|-------------------|
|   namespace #2    | hostname: carol
|                   | IP: 789
| ┌---┐ ┌---┐ ┌---┐ |
| |100| |200| |200| |---------------|
| └---┘ └---┘ └---┘ | network stack |
|-------------------|---------------|
```
- Containers may have their own namespace, or they may share a namespace with other containers (e.g. in Kubernetes pods)
- Kernel namespaces are useful outside of containers to, e.g., run multiple copies of the same process that expects to use the same resources (e.g., same network ports or process ID) 
- Some namespaces are lighter than others
  - A namespace over, e.g., process IDs, are lightweight and straightforward to set up
  - Managing different IP addresses requires setting up a network stack for each namespace - this is very expensive
- Each namespace can have its own root user, but they should not all have the same privileges as the default namespace root user
  - Implemented with *capabilities*
    - ACLs are a way to manage permission/privileges based on *identity* - is this user allowed to perform this action?
      - ACLs are too coarse-grained for many containers
    - Capabilities are like *tokens* that provide permissions - you can perform an action if you hold the right capability token
      - Tradeoff: finer-grained than ACLs, but harder to use and heavier
  - **Non-default root users are given fewer capabilities than the default root**
```
 |----------------|  |----------------| 
 |  container 1   |  |  container 2   |
 |----------------|  |----------------|
 |  namespace 1   |  |  namespace 2   |
 |----------------|--|----------------|
 |              host OS               |  
 |------------------------------------|  
 |                HW                  |
 |------------------------------------|  
```
- Each container has its own namespace (even if the namespaces are configured the same, even if the containers are duplicates)
- There are **6 namespaces**, meaning that there are 6 attributes of a namespace that you can configure
  - Mount namespace 
  - PID namespace 
  - networking namespace 
  - hostname namespace
  - inter-process communication
  - user namespace
- Most are lightweight to set up; networking namespace is heavy!
- Namespace system calls
  - `clone()`
    - Creates a new process and a new namespace
  - `unshare()`
    - Creates a new namespace (without a new process)
  - `setns()`
    - Adds a process to an existing namespace
- Namespaces are handled via the `/proc` file system
  - `/proc` is an FS used by the kernel to share information with user space
    - Exposes configuration knobs to user space
    - Users can read `/proc` to obtain information about the user and write to it to change configurations
    - UNIX philosophy: everything is a file
  - Example: `/proc/pid/ns` lists all namespaces for a given PID
    - E.g. `proc/100/ns` lists namespaces for PID 100 (represented as symbolic links)
  - Namespaces are identified by namespace ID numbers
    - If `/proc/pid/ns` gives you the same namespace ID for two processes, they are in the same namespace
      - If you check it for processes in a container, you will see different IDs from processes running on the host

#### Control groups (cgroups)
- Resource management
- Can control things like:
  - CPU usage %
  - CPU placement (pinning a process to a particular CPU)
    - Useful for running performance experiments - reduces noise from other cores
  - Memory usage
  - Network rules (e.g. limiting throughput that can be used by a container)
- 11 different control groups 
- Resource quotas are enforced by the kernel
- If a process attempts to go beyond its limit (e.g. process tries to allocate 2GB of memory but is only allowed 1GB), the kernel will enforce it/kill the process (e.g OOM killer)
```
 |----------------|  |----------------|
 |  container 1   |  |  container 2   |
 |----------------|  |----------------|
 |     cgroup     |  |     cgroup     |
 |----------------|  |----------------|
 |  namespace 1   |  |  namespace 2   |
 |----------------|--|----------------|
 |              host OS               |
 |------------------------------------|
 |                HW                  |
 |------------------------------------|
```
- Each container has its own cgroup (even if the cgroups are configured the same, even if the containers are duplicates)

#### Overlay FS
- Handle container access to storage
- Every container is based on an *image* made of some base + commands run on top of it
  - E.g.: base Ubuntu 18 with Python and PHP installed
  - Each command creates a new layer 
```
 |-------------------|
 | Ubuntu + Python + | <-- Diff of PHP installation onto Ubuntu + PHP
 |        PHP        |
 |-------------------|
 |  Ubuntu + Python  | <-- Diff of Python installation onto Ubuntu
 |-------------------|
 |     Ubuntu 18     | <-- Base
 |-------------------|
```
- The entire image is created by adding a diff of the changes from one layer to the next one
- If we then want to create a similar container, but without PHP, we can have one copy of the base and the Python layer shared between both containers
- Whenever a container makes changes, the changes are added in a new layer
- Containers maintain pointers to the layers they want to use
  - There only needs to be one copy of each layer, regardless of how many containers are using it. Saves space and network bandwidth
  - Provenance: semantically understanding what is in storage; associating each layer with the command. 
```
      |----------------------------------------------------------|
      |                                          |----------|    |
      |                      |--------------|--> | private2 |    |
 |----------------|  |----------------|     |    |----------|    |
 |  container 1   |  |  container 2   |     |    | private1 | <--|
 |----------------|  |----------------|     |    |----------|    |
 |     cgroup     |  |     cgroup     |     |    |   PHP    | <--|
 |----------------|  |----------------|     |    |----------|    |
 |  namespace 1   |  |  namespace 2   |     |--> |  Python  | <--|
 |----------------|--|----------------|     |    |----------|    |
 |              host OS               |     |--> |  Ubuntu  | <--|
 |------------------------------------|          |----------|
 |                HW                  |
 |------------------------------------|
```
- There is a single overlay file system on the system, shared by all of the containers
  - Docker or similar management services set up the overlay FS for you
- What if a container wants to perform some writes?
  - Containers usually have some private storage space that they can write to
  - Containers are *deployment units* containing executables: they are not meant to store data permanently. 
    - Containers have a private layer to write to in the overlay FS, but the private layer is deleted when the container dies
  - Data that must be stored durably must be written to an external database, key value store, file system, etc.

### Deployment
- In the past, software was built as large, monolithic units providing a variety of services
- Larger codebase ==> harder to maintain
  - Different teams may be working on different parts of the same unit of software
  - Teams must be careful to not break things done by other teams
- Amazon Web Services model
  - Each team builds a stand-alone service with an API accessible by other teams
  - Teams can function independently
```
    Team 1                  Team 2
 |----------|---|    |---|----------|
 |   foo    |API|    |API|   bar    |
 |----------|---|    |---|----------|
```
- Team 1 handles running service `foo`, team 2 handles running service `bar`
- E.g., `foo` can access `bar` via `bar`'s API
- Each service `foo` and `bar` will be run in a container!
  - When, e.g., `bar` needs to be updated, the old container will be killed and a new one with the updated service will be brought up with the same configuration
  - These are **microservices**
  - Containers are convenient for microservices - lightweight, spin up and down quickly, can be moved around as necessary, development and deployment environment is the same

### Container orchestration
- Problem: we have lots of microservices that need to run together. How do we manage all of thse containers?
- Answer: **container orchestration**
- Some orchestration services: Docker Swarm, Kubernetes
  - Kubernetes is more advanced and scalable
- Example: suppose you have a database, web server, and caching layer all running as separate microservices on a cluster in the cloud. How do we decide where to run these? How do we manage them?
```
        |------------|
        |   Swarm    | <-- Determines placement of microservice on physical hosts
        |   Manager  |
        |------------|

  |------------|                    |------------||------------|
  |  Database  |                    | Web server ||   Cache    |
  |------------|                    |------------||------------|
  |--------------------------|      |--------------------------|
  | Container management sys |      | Container management sys |
  |--------------------------|      |--------------------------|
  |          Host 1          |      |          Host 2          |
  |--------------------------|      |--------------------------|
```
- Manager has several jobs:
  - Keeps track of utilization on each host and decides where to put containers based on that information
    - Container management system on each host provides this info to the manager
  - Keeps microservices alive
    - Each container periodically pings the manager; if a container misses a ping, the manager assumes the container is dead and starts a new instance of that container
- Kubernetes pods
```           
  |--------------------------------------|
  |                 Pod                  |
  |  |--------|  |--------|  |--------|  |
  |  |   c0   |  |   c1   |  |   c2   |  | 
  |  |--------|  |--------|  |--------|  |
  |--------------------------------------|
  |--------------------------------------|
  |                Host                  |
  |--------------------------------------|
```
- A pod is a set of containers
- All containers in a pod are deployed on the same host and share the same namespace
  - Microservices that frequently communicate are more efficient if they are co-located and can use IPC to exchange messages
- Containers within a pod can still be managed separately by different teams