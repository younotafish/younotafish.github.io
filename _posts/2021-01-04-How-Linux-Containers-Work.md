---
layout: post
title:  "How Linux Containers Work?"
author: "Shori"
comments: false
tags: Containers
---

Our Arm-based Android Gaming-in-the-cloud platforms are developed with Linux container technologies. The use of containers allow multiple instances of user application run in the server simultaneously. To solve the current issue that the servers are not performing at the desired level, investigating the tech specifications of containers would be a good start.
The technologies of choice currently used in the platforms provided by our partners are Linux containers (LXC) and Docker. Intrinsically, they are of the same breed. They both utilizes the set of technologies provided by the Linux kernel, namely Linux Namespaces, Control Groups (CGroups), and Union File Systems (UnionFS).
In this report, we are going to give a broad overview of the tech specification of containers in a whole.  
<br />

## Linux Namespace

The Linux Namespace provides seven types of isolation. Namely ```CLONE_NEWCGROUP```, ```CLONE_NEWIPC```, ```CLONE_NEWNET```, ```CLONE_NEWNS```, ```CLONE_NEWPID```, ```CLONE_NEWUSER```, and ```CLONE_NEWUT```. 

To create a new namespace, we simply use the system call ```clone()```. The ```clone()``` system call creates a new namespace with the desired types of isolation. More to that, we have ```setns()``` and ```unshare()```, which includes/excludes a process to/from the namespace.
For example, in the source code of LXC, we could find the call to the ```clone()``` function:

For example, in the source code of LXC, we could find the call to the ```clone()``` function:

``` c
#define __LXC_STACK_SIZE (8 * 1024 * 1024)
pid_t lxc_clone(int (*fn)(void *), void *arg, int flags, int *pidfd)
{
	pid_t ret;
	void *stack;

	stack = malloc(__LXC_STACK_SIZE);
	if (!stack) {
		SYSERROR("Failed to allocate clone stack");
		return -ENOMEM;
	}

#ifdef __ia64__
	ret = __clone2(fn, stack, __LXC_STACK_SIZE, flags | SIGCHLD, arg, pidfd);
#else
	ret = clone(fn, stack + __LXC_STACK_SIZE, flags | SIGCHLD, arg, pidfd);
#endif
	if (ret < 0)
		SYSERROR("Failed to clone (%#x)", flags);

	return ret;
}
```
<br />

## Control Groups

Control Groups (CGroups) is a way in Linux to allocate (or, restrict) system resources to processes. By using the command ```lssubsys```, we could see the system resources (or, in technical terms, subsystems) that Linux allows us to manipulate.

``` bash
$ lssubsys -m
cpuset /sys/fs/cgroup/cpuset
cpu,cpuacct /sys/fs/cgroup/cpu,cpuacct
blkio /sys/fs/cgroup/blkio
memory /sys/fs/cgroup/memory
devices /sys/fs/cgroup/devices
freezer /sys/fs/cgroup/freezer
net_cls,net_prio /sys/fs/cgroup/net_cls,net_prio
perf_event /sys/fs/cgroup/perf_event
hugetlb /sys/fs/cgroup/hugetlb
pids /sys/fs/cgroup/pids
rdma /sys/fs/cgroup/rdma
```

The manipulation of CGroup subsystems in Linux is done through a file-based approach. Where we could add lines to files in the directories shown above to achieve the manipulation. And, by adding a new directory in any subsystem, we create a new CGroup. The new directory is populated automatically with the same files in its parent directory.

### Docker example

Let's take a closer look through an example with Docker. We start by running a container.
```bash
$ docker run -it -d ubuntu:latest
f17f32a0975aa6db1f28a17b6c88dab6e14a46a9dee5b98cd884dc0bfe5d1c49
```
Then, we check what's going on in the subsystems. There is a new CGroup named "docker", and it is automatically populated with the same stuff in the parent directory. And we could also find that there is another new directory named ```f17f32a0975aa6db1f28a17b6c88dab6e14a46a9dee5b98cd884dc0bfe5d1c49```. This is the CGroup specifically created for our running container instance (see how the hostname matches).

``` bash
$ pwd
/sys/fs/cgroup/cpu/docker
$ ls
cgroup.clone_children  cpuacct.usage         cpuacct.usage_percpu_sys   cpuacct.usage_user  cpu.rt_period_us   cpu.stat        f17f32a0975aa6db1f28a17b6c88dab6e14a46a9dee5b98cd884dc0bfe5d1c49
cgroup.procs           cpuacct.usage_all     cpuacct.usage_percpu_user  cpu.cfs_period_us   cpu.rt_runtime_us  cpu.uclamp.max  notify_on_release
cpuacct.stat           cpuacct.usage_percpu  cpuacct.usage_sys          cpu.cfs_quota_us    cpu.shares         cpu.uclamp.min  tasks
```

And by exiting the container, the CGroup is removed.

More, flags such as ```--cpu-quota=50000``` in docker that allow us to alter the resource allocation in a container. This is effectively the same as directly altering the files. To examine what's going on, again, we fire up a new container.
``` bash
$ docker run -it -d --cpu-quota=50000 ubuntu:latest
e7b02da0f3ea2d7e6459edcdd6e51b896203fb162816c9be1841e1d9932b140d
$ pwd
/sys/fs/cgroup/cpu/docker/e7b02da0f3ea2d7e6459edcdd6e51b896203fb162816c9be1841e1d9932b140d
$ cat cpu.cfs_quota_us 
50000
```
We could see that the quota set via flags in our docker command is channelled into the CGroup subsystem.

<br />

## Union File System
A docker image is essentially just a tarball. We could examine the fact by exporting an image and then checking its insides by unpacking the tarball. And, we could in fact run a container without the UnionFS, we could map any desired root file system to the container we're about to run. 

So, what makes the UnionFS different? When we pull an image from the docker hub, we notice that the image is being pulled in layers. And, that is right, every action we perform on the current file system, in UnionFS, will become another layer upon the current file system. This enables image reuse across different containers.