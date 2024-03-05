---
layout: post
title: "Core isolation for dataplane"
date:   2024-03-03 12:00:00 +0100
categories: jekyll update
---

## Subject
Whenever you run your dataplane on the host dedicated to it or share it with some useful processes, you probably want to isolate cpu resources to the dataplane. 

# isolcpus
The most adopted approach to isolate userspace application on Linux is to set `isolcpus` argument. One could notice that this kernel argument was marked as deprecated some time ago. Although, I've tried to use it recently. It works usually fine if the only process is your dataplane.

But I also had some userspace applications running in lxd. And sometimes those processes were scheduled on isolated cores. Sometimes many kthreads were scheduled on the same cores. It went out to be a terrible idea since when a process or kthread is scheduled to an isolated core - it stays there forever.

# cgroups
The 2nd approach that I tried was using cgroups + systemd. I used this manual [0] as a reference. Everything can be done in runtime (without rebooting), so it's a good one. Introducing a new slice for the dataplane (vpp in my case) with isolated cpuset from all the other slices (`system.slice`, `user.slice` and `init.scope`) would help one to exclude userspace processes from running on the dataplane cores.

```
# cat /etc/systemd/system/isolated.slice
[Slice]
AllowedCPUs=0-2
```

This doesn't affect the kernel to run kthreads and irq on those cores. It can be fixed with setting a proper irqaffinity though.

# Hierarchy
The previous solution won't work for LXD/Incus/Docker, since they create their own cgroups. Still, we can limit the containers by some settings:
* set `limits.cpu` per container
* set `limits.cpu` for the whole profile
* use hierarchical cgroups (only works with cgroups2) and pass `lxc.cgroup.relative=1` to the default profile

The latted would force lxd/incus to create its cgroups within the parent (e.g. `system.slice`). And one can enjoy a more beatiful solution for managing isolated and non-isolated cores from one place.

[0] - https://documentation.suse.com/sle-rt/15-SP5/html/SLE-RT-all/cha-shielding-with-systemd.html
[1] - https://github.com/lxc/incus/issues/492#issuecomment-1969975994

