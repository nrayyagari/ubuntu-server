# Managing Processes, Containers, and Namespaces - Q&A

## Table of Contents
1. [Processes vs Containers](#1-processes-vs-containers)
2. [Docker Containers as Process Trees](#2-docker-containers-as-process-trees)
3. [Apache vs Nginx Process Model](#3-apache-vs-nginx-process-model)
4. [docker ps vs docker top vs top](#4-docker-ps-vs-docker-top-vs-top)
5. [Linux Namespaces](#5-linux-namespaces)
6. [How to Read lsns Output](#6-how-to-read-lsns-output)
7. [Comparing Apache and Nginx Namespaces](#7-comparing-apache-and-nginx-namespaces)
8. [Finding Processes in the Same PID Namespace](#8-finding-processes-in-the-same-pid-namespace)
9. [Why user and time Namespaces May Be Shared](#9-why-user-and-time-namespaces-may-be-shared)
10. [Containers, Root, and Security](#10-containers-root-and-security)
11. [Rootless Podman vs Rootful Docker](#11-rootless-podman-vs-rootful-docker)
12. [Quick Reference Commands](#12-quick-reference-commands)

---

## 1. Processes vs Containers

### Q: From the Linux host perspective, what is a container?

A container is **not a VM** and not a separate kernel.

From the Linux host perspective, a container is:
- a group of normal Linux processes
- placed into namespaces for isolation
- placed into cgroups for resource control

Simple mental model:

- **VM** = separate kernel
- **container** = same kernel, isolated process tree

### Q: Does Linux "see" containers directly?

Not in the same way it sees processes.

Linux directly sees:
- processes
- sockets
- filesystems
- namespaces
- cgroups

The "container" is the combination of those pieces.

---

## 2. Docker Containers as Process Trees

### Q: If one container is running Apache, why do I see multiple `httpd` processes on the host?

Because **one container does not mean one process**.

A container usually contains a **process tree**:
- one parent/main process
- zero or more child processes

For example, Apache in our Docker container showed:
- 1 master/parent `httpd`
- 3 worker `httpd` processes

### Q: So are those real host processes?

Yes.

From the host perspective, they are real Linux processes. They are just:
- in the container's namespaces
- in the container's cgroups

### Q: What is the best one-line summary?

One container is usually **one isolated process tree**, not one single process.

---

## 3. Apache vs Nginx Process Model

### Q: In our containers, did both Apache and Nginx have multiple processes?

Yes.

Apache:
- 1 master process
- 3 worker processes

Nginx:
- 1 master process
- 3 worker processes

### Q: If both have master and worker processes, what is the difference?

The difference is not simply "single-process vs multi-process."

The main difference is **how they handle requests internally**:

- **Apache**
  - commonly uses process or process+thread models
  - in our container it used MPM `event`
  - more module-heavy and historically more process/thread oriented

- **Nginx**
  - uses an event-driven worker model
  - each worker handles many connections asynchronously
  - usually has lower memory overhead for reverse proxy and static content

### Q: What is the simple way to remember this?

- `Apache` = flexible, module-heavy, process/thread oriented
- `Nginx` = event-driven, connection-efficient, great as reverse proxy

---

## 4. docker ps vs docker top vs top

### Q: What does `docker ps` show?

It shows **containers**.

Example:

```bash
docker ps
```

### Q: What does `docker top` show?

It shows the **processes running inside a container**.

Example:

```bash
docker top apache
docker top nginx -eo pid,ppid,user,args
```

This is closer to `ps` than to Linux `top`.

### Q: What is the difference between `top` and `docker top`?

- `top` = interactive live system monitor
- `docker top` = process snapshot for one container

### Q: Why is `docker top` useful?

Because it lets you inspect:
- parent vs child processes
- usernames
- command lines
- process count inside a container

without entering the container.

---

## 5. Linux Namespaces

### Q: What problem do namespaces solve?

Namespaces solve the problem of **visibility and isolation**.

They control what a process can see.

### Q: What are the main namespace types used with containers?

The main Linux namespace types are:

- `pid` = process IDs
- `mnt` = mount points / filesystem view
- `net` = network stack
- `uts` = hostname/domain name
- `ipc` = shared memory, semaphores, message queues
- `user` = UID/GID mappings
- `cgroup` = cgroup view
- `time` = certain clock views

### Q: Is there a `uid` namespace?

No.

The correct term is **`user` namespace**.

It controls user and group ID mappings.

---

## 6. How to Read lsns Output

### Q: What does `lsns` do?

`lsns` means **list namespaces**.

Example:

```bash
sudo lsns --task 22826
```

### Q: How should I interpret this output?

```text
NS TYPE   NPROCS   PID USER COMMAND
4026532184 pid         4 22826 root nginx: master process nginx -g daemon off;
```

Read it like this:

- `NS`
  - the namespace ID
  - same ID means same namespace

- `TYPE`
  - the kind of isolation
  - here it is `pid`

- `NPROCS`
  - number of processes attached to that namespace
  - here `4`

- `PID`
  - representative process shown by `lsns`

- `USER`
  - user of that representative process

- `COMMAND`
  - command line of that representative process

### Q: What is the most important rule when reading `lsns`?

For the same `TYPE`:

- same `NS` number = shared namespace
- different `NS` number = isolated namespace

---

## 7. Comparing Apache and Nginx Namespaces

### Q: How do I compare the namespaces of two containers?

First get their host PIDs:

```bash
APACHE_PID=$(docker inspect -f '{{.State.Pid}}' apache)
NGINX_PID=$(docker inspect -f '{{.State.Pid}}' nginx)
```

Then inspect each one:

```bash
sudo lsns --task "$APACHE_PID"
sudo lsns --task "$NGINX_PID"
```

### Q: In our environment, what did the comparison show?

Apache and Nginx had different namespace IDs for:
- `mnt`
- `uts`
- `ipc`
- `pid`
- `cgroup`
- `net`

They shared:
- `time`
- `user`

### Q: What does that mean?

It means Apache and Nginx were isolated from each other for:
- filesystem view
- hostname
- IPC
- process tree
- cgroup view
- network stack

But they were sharing the default:
- `time` namespace
- `user` namespace

---

## 8. Finding Processes in the Same PID Namespace

### Q: If I know a process is in PID namespace `4026532184`, how do I find all processes in that namespace?

Use `/proc` namespace links:

```bash
for p in /proc/[0-9]*; do
  [ "$(sudo readlink "$p/ns/pid" 2>/dev/null)" = "pid:[4026532184]" ] &&
  ps -p "${p#/proc/}" -o pid,ppid,user,cmd --no-headers
done
```

### Q: What does that show?

It shows all host processes that belong to that same PID namespace.

For our Docker `nginx` container, that corresponded to:
- 1 master process
- 3 worker processes

### Q: Why do processes in the same PID namespace see the same process tree?

Because the PID namespace defines the **process-ID view**.

If processes share the same `pid` namespace:
- they share the same visible process world

If they are in different `pid` namespaces:
- they do not see the same process world

---

## 9. Why user and time Namespaces May Be Shared

### Q: Why did `pid` show `NPROCS 4`, but `user` and `time` showed much larger values?

Because `NPROCS` is counted **per namespace type**.

A process can be:
- in a small private `pid` namespace
- while also being in a large shared `user` namespace
- and a large shared `time` namespace

### Q: Is it common for Docker containers to share the host/default `user` namespace?

Yes, with rootful Docker this is common unless user namespace remapping is enabled.

Common pattern for rootful Docker:
- private `pid`, `mnt`, `net`, `ipc`, `uts`
- shared/default `user`
- shared/default `time`

### Q: Does shared `user` or `time` namespace mean isolation has failed?

No.

It means isolation is **partial and layered**, not absolute.

The container may still have separate:
- PID namespace
- network namespace
- mount namespace
- cgroups
- capabilities/seccomp/AppArmor/SELinux restrictions

### Q: Which shared namespace matters more from a security perspective?

`user` matters more than `time`.

Shared `time` is usually not the main concern.

Shared `user` means container root is closer to host-root semantics than it would be with user namespace remapping or rootless containers.

---

## 10. Containers, Root, and Security

### Q: What does "do not run containers as root" actually mean?

This advice usually mixes two different ideas.

### Q: What is the first meaning?

Run the **application process** inside the container as a non-root user.

That reduces damage if the application is compromised.

### Q: What is the second meaning?

Use:
- user namespaces
- or rootless containers

so that root inside the container is **not real host root**.

### Q: Why are these two ideas related but different?

Because:

- non-root app user
  - reduces power **inside the container**

- user namespaces / rootless runtime
  - reduces how dangerous container root is **relative to the host**

### Q: What is the practical recommendation?

Best baseline:
- run the app as non-root inside the container

Better hardening:
- also use user namespace remapping or rootless containers

Avoid:
- `--privileged` unless absolutely necessary

---

## 11. Rootless Podman vs Rootful Docker

### Q: What changed when we ran `nginx` with rootless Podman?

Rootless Podman created a **private user namespace** for the container environment.

### Q: Did the rootless Podman container still share all namespaces only with app processes?

No.

Different namespace types can have different members.

For our rootless Podman `nginx`:

- `pid` namespace members were mainly the app process tree
- `user` namespace also included helper processes like:
  - `catatonit`
  - `slirp4netns`
  - `rootlessport`
  - `conmon`
- `net` namespace included:
  - `rootlessport`
  - `rootlessport-child`
  - `nginx` master
  - `nginx` workers

### Q: Why are helper processes in the same namespace?

Because runtime helpers may need to participate in the same:
- user-ID mapping context
- network context

to make rootless containers work correctly.

### Q: What is the key difference between rootful Docker and rootless Podman in our tests?

Docker rootful `nginx`:
- shared host/default `user` namespace

Podman rootless `nginx`:
- private `user` namespace owned under the unprivileged user context

### Q: What is the security meaning of that difference?

In rootless Podman:
- root inside the container is mapped through a user namespace
- it is not host root

That is one major reason rootless containers are considered safer.

---

## 12. Quick Reference Commands

### Q: How do I list containers?

```bash
docker ps
podman ps
```

### Q: How do I see processes inside a Docker container?

```bash
docker top nginx -eo pid,ppid,user,args
docker top apache -eo pid,ppid,user,args
```

### Q: How do I get a container's main host PID?

Docker:

```bash
docker inspect -f '{{.State.Pid}}' nginx
```

Podman:

```bash
podman inspect -f '{{.State.Pid}}' podman-nginx
```

### Q: How do I inspect namespaces for that process?

```bash
sudo lsns --task <pid>
sudo lsns --task <pid> -o NS,TYPE,NPROCS,PID,USER,COMMAND
```

### Q: How do I compare two containers?

```bash
APACHE_PID=$(docker inspect -f '{{.State.Pid}}' apache)
NGINX_PID=$(docker inspect -f '{{.State.Pid}}' nginx)

sudo lsns --task "$APACHE_PID"
sudo lsns --task "$NGINX_PID"
```

### Q: How do I find all processes in the same PID namespace as a container?

```bash
PID=$(docker inspect -f '{{.State.Pid}}' nginx)
NS=$(sudo readlink /proc/$PID/ns/pid)

for p in /proc/[0-9]*; do
  [ "$(sudo readlink "$p/ns/pid" 2>/dev/null)" = "$NS" ] &&
  ps -p "${p#/proc/}" -o pid,ppid,user,cmd --no-headers
done
```

### Q: What is the shortest summary of this chapter?

- a container is an isolated process tree, not a VM
- namespaces control what processes can see
- cgroups control what processes can use
- `docker top` shows container processes
- `lsns` shows namespace membership
- rootless containers strengthen isolation by using a private user namespace
