---
layout: post
title:  "Dokrust Devlog: Week 1"
author: "Shori"
comments: false
tags: Containers
---

*For the GitHub repo of Dokrust, visit [**here**](https://github.com/lishpr/dokrust).*

This is the first week of development of Dokrust. As a fork of [vas-quod](https://github.com/flouthoc/vas-quod), it has a few problems (or features, as I don't know whether the original author intended to make it work that way) that I want to improve upon.

<br />

## Improvements

### Interactive shell

First, vas-quod cannot sustain an interactive bash. When the shell is passed in as the command to execute, it would terminate (and the ssh to my server got closed, too).
```
# ./target/debug/vas-quod -r ../images/rootfs/ -c bash
# root@vasquod:/# logout
Connection to *.*.*.* closed.
```

Inspecting the ```run_container()``` function of vas-quod, it is found that
```rust 
pub fn run_container(rootfs: &str, command: &str, command_args: Vec<&str>){
    // ... preambulatory code ...

    let _child_pid = sched::clone(cb, stack, clone_flags, Some(Signal::SIGCHLD as i32)).expect("Failed to create child process");

}
```
This could allow the parent process that initiated ```clone()``` to exit in prior to the child process, thus prohibited commands like ```bash``` from running.

And the solution turns out to be simple. By adding the following line after where ```clone()``` is being called, the parent process *waits* for the child process's return status. This would sustain the parent process long enough so that an interactive bash could be allowed.
```rust
let _proc_res = nix::sys::wait::waitpid(_child_pid, None);
```

### Some refractoring

I guess the function signatures found in vas-quod, such as the ones shown below, could be improved by OO Design. Therefore I created a "Runtime" class that stores all the related information, so the functions could look cleaner. Also, by following the OO Design, the scalability of the code could be improved.

```rust
fn spawn_child(hostname: &str, cgroup_name: &str, rootfs: &str, command: &str, command_args: &[&str]) -> isize

pub fn run_container(rootfs: &str, command: &str, command_args: Vec<&str>)
```

<br />

## Feature addition

### Customizable hostnames

After the improvements above, the first thing I want to introduce is hostname customization, as it is simple for someone new to rust, and it is meaningful if I want to eventually run multiple instances of Dokrust containers.

This is easy as invoking the ```sethostname()``` syscall.

### CGroup quotas

One important feature that containers offer is the isolation of processes. A part of the isolation is provided by Linux through Control Groups, where you can ascribe the quotas of CPU, memory, disk, network usage, etc for processes in a specfic container.

Docker offers well-wrapped pass-in options such as ```--cpu-quota``` for each of the CGroup, while I just want to leave it to the users to decide (and save me a lot of work).

The setting of CGroup quotas consists of identifying the ```pid``` of the processes in the container, and the quota you wish to set. As Linux leaves the API for CGroups in the form of files. This could be done by parsing and writing the option for quota and target group into the files that Linux provided.

```rust
fs::write(cfs_quota, quota.as_bytes()).unwrap();
fs::write(tasks, format!("{}", unistd::getpid().as_raw())).unwrap();
```

### Directory mounting

Vas-quod provided the mounting of ```/proc``` into the rootfs by invoking the Mount Namespace. This is permanent as we want an isolated process id space. But we also wish to have files being mounted into the rootfs at runtime. 

This could also be done via the ```mount()``` syscall by passing in the ```MS_BIND``` flag. This way, we could have the directories mounted into the rootfs in a transient manner.

<br />

## Next up

In the original vas-quod repo, the author has network bridge support and mounting in the roadmap. As I have mounting somewhat naïvely setup, I feel networking support should be the core of development next week.