% PORTO(8)

# NAME

porto - linux container management system

# SYNOPSIS

**portod** \[-h|--help\] \[-v|--version\] \[options...\] \<command\> \[arguments...\]

**portoctl** \[-h|--help\] \[-v|--version\] \[options...\] \<command\> \[arguments...\]

# DESCRIPTION

Porto is a yet another Linux container management system, developed by Yandex.

The main goal is providing single entry point for several Linux subsystems
such as cgroups, namespaces, mounts, networking, etc.
Porto is intended to be a base for large infrastructure projects.

## Key Features
* **Nested containers**       - containers could be put into containers
* **Nested virtualizaion**    - containers could use porto service too
* **Flexible configuration**  - all container parameters are optional
* **Reliable service**        - porto upgrades without restarting containers

Container management software build on top of porto could be transparently
enclosed inside porto container.

Porto provides a protobuf interface via an unix socket /run/portod.socket.

Command line tool **portoctl** and C++, Python and Go APIs are included.

Porto requires Linux kernel 3.18 and optionally some offstream patches.

# CONTAINERS

## Name

Container name could contains only these characters: 'a'..'z', 'A'..'Z', '0'..'9',
'\_', '\-', '@', ':', '.'. Slash '/' separates nested container: "parent/child".

Each container name component should not exceed 128 characters.
Whole name is limited with 200 characters and 220 for superuser.
Also porto limits nesting with 16 levels.

Container could be addressed using short name relative current porto
namespaces: "name", or absolute name "/porto/name" which stays the
same regardless of porto namespace.
See **absolute\_name** and **porto\_namespace** below.

Host is a pseudo-container "/".

"self" points to current container where current task lives and
could be used for relative requests "self/child" or "self/..".

Container "." points to parent container for current porto namespace,
this is common parent for all visible containers.

## States
* **stopped**   - initial state
* **starting**  - start in progress
* **running**   - execution in progress
* **paused**    - frozen, consumes memory but no cpu
* **dead**      - execution complete
* **meta**      - running container without command

## Operations
* **create**    - creates new container in stopped state
* **start**     - stopped -\> starting -\> running | meta
* **stop**      - running | meta | dead -\> stopped
* **restart**   - dead -\> stopped -\> starting -\> running | meta
* **kill**      - running -\> dead
* **death**     - running -\> dead
* **pause**     - running | meta -\> paused
* **resume**    - paused -\> running | meta
* **destroy**   - destroys container in any state
* **list**      - list containers
* **get**       - get container property
* **set**       - set container property
* **wait**      - wait for container death

## Usual Life Cycle:

create -\> (stopped) -\> set -\> start -\> (running) -\> death -\> (dead) -\> get -\> destroy

## Properties

Container configuration and state both represented in key-value interface.  
Some properties are read-only or requires particular container state.

**portoctl** without arguments prints list of possible container properties.

Some properties have internal key-value structure in like \<key>\: \<value\>;...
and provide access to individual values via **property\[key\]**

Values which represent size in bytes could have floating-point and 1024-based
suffixes: B|K|M|G|T|P|E. Porto returns these values in bytes without suffixes.

Values which represents text masks works as **fnmatch(3)** with flag FNM\_PATHNAME:
'\*' and '?' doesn't match '/', with extension: '\*\*\*' - matches everything.

## Context

* **command** - container command string

    Environment variables $VAR are expanded using **wordexp(3)**.

* **core\_command** - command for receiving core dumps

    See [COREDUMPS] below.

* **env** - environment of main container process, syntax: \<variable\>=\<value\>; ...

    Container with **isolate**=false inherits environment variables from parent.

* **user** - uid of container processes, default **owner\_user**

* **group** - gid of container processes, default: **owner\_group**

* **umask** - initial file creation mask, default: 0002, see **umask(2)**

* **ulimit** - resource limits, syntax: \<type\>: \[soft\]|unlimited \<hard\>|unlimited; ... see **getrlimit(2)**

   Default ulimits could be set in portod.conf:
```
container { default_ulimit: "type: soft hard;..." }
```
   Hardcoded default is "core: 0 unlimited; memlock: 8M unlimited; nofile: 8K 1M".

* **virt\_mode** - virtualization mode:
    - *app* - (default) start command as normal process
    - *os* - start command as init process

## State

* **state** - current state, see [States]

* **exit\_code** - 0: success, 1..255: error code, -64..-1: termination signal, -99: OOM

* **exit\_status** - container exit status, see **wait(2)**

* **oom\_killed** - true, if container has been OOM killed

* **core\_dumped** - true, if main process dumped core

* **respawn\_count** - how many times container has been respawned (using respawn property)

* **root\_pid** - main process pid in client pid namespace (could be unreachable)

* **stdout\[[offset\]\[:length\]]** - stdout text, see **stdout\_path**
    - stdout - get all available text
    - stdout[:1000] - get only last 1000 bytes
    - stdout[2000:] - get bytes starting from 2000 (text might be lost)
    - stdout[2000:1000] - get 1000 bytes starting from 2000

* **stdout\_offset** - offset of stored stdout

* **stderr\[[offset\]\[:length\]]** - stderr text, see **stderr\_path**

    Same as **stdout**.

* **stderr\_offset** - offset of stored stderr

* **time** - container running time in seconds

* **creation\_time** - format: YYYY-MM-DD hh:mm::ss

* **start\_time** - format: YYYY-MM-DD hh:mm::ss

* **absolute\_name** - full container name including porto namespaces

* **absolute\_namespace** - full container namespace including parent namespaces

* **controllers** - enabled cgroup controllers, see [CGROUPS] below

* **cgroups** - paths to cgroups, syntax: \<name\>: \<path\>

* **process\_count** - current process count

* **thread\_count** - current thread count

* **thread\_limit**  - limit for **thread\_count**

    For first level containers default is 10000.

* **parent** - parent container absolute name

* **private** - 4k text to describe container

* **weak** - if true container will be destroyed when client disconnects

    Contianer must be created by special API call, client could clear **weak** flag after that.

* **respawn** - restart container after death

    Delay between respawns is 1 second.

* **max\_respawns** - limit for automatic respawns, default: -1 (unlimited)

* **aging\_time** - time in seconds before auto-destroying dead containers, default: 1 day

## Security

* **isolate** - create new pid/ipc/utc/env namespace
    - *true*          - create new namespaces (default)
    - *false*         - use parent namespaces

* **capabilities** - available capabilities, syntax: CAP;... see **capabilities(7)**

    Porto restricts capabilities depending on other properties:

    Requires **memory\_limit**: IPC\_LOCK.

    Requires **isolate**: SYS\_BOOT, KILL, PTRACE.

    Requires **net** and no **ip\_limit**: NET\_ADMIN.

    Always available: NET\_BIND\_SERVICE, NET\_RAW

    Requires **root** (chroot): SETPCAP, SETFCAP, CHOWN, FOWNER,
    DAC\_OVERRIDE, FSETID, SETGID, SETUID, SYS\_CHROOT, MKNOD, AUDIT\_WRITE.

    Requires no **root**, cannot be ambient, only for suid:
    LINUX\_IMMUTABLE, SYS\_ADMIN, SYS\_NICE, SYS\_RESOURCE.

    Without chroot all these capabilities are available. Capabilities which
    meet requirements could be set as ambient and available within chroot.

    Container inherits capabilities from parent and cannot surpass them.

    Container owned by root by default have all these capabilities and
    ignore any restrictions.

* **capabilities\_ambient** - raise ambient capabilities, syntax: CAP;... see **capabilities(7)**

    All tasks in container will have these capabilities.

    Requires Linux 4.3

* **devices** - access to devices, syntax: \<device\> \[r\]\[w\]\[m\]\[-\] \[name\] \[mode\] \[user\] \[group\];...

    By default porto grants access to:  
    /dev/null  
    /dev/zero  
    /dev/full  
    /dev/random  
    /dev/urandom  
    /dev/tty  
    /dev/console  (forwarded to /dev/null)  
    /dev/ptmx  
    /dev/pts/*  

    Access to devices for all containers could be granted in portod.conf:
```
container { extra_devices: "<device> [rwm]... ;..." }
```

* **enable\_porto** - access to porto
    - *false* | *none* - no access
    - *read-isolate*   - read-only access, show only sub-containers
    - *read-only*      - read-only access
    - *isolate*        - read-write access, show only sub-containers
    - *child-only*     - write-access to sub-containers
    - *true* | *full*  - full access (default)

    Containers with restricted view sees truncated container names excluding isolated part.

    All containers with read access could examine **"/"**, **"."**, **"self"** and all own parents.

* **porto\_namespace** - name prefix required to shown containers (deprecated, use **enable\_porto**=isolate)

* **owner\_user** - container owner user, default: creator

* **owner\_group** - container owner group, default: creator

Porto client authentication is based on task pid, uid, gid received via
**socket(7)** SO\_PEERCRED and task freezer cgroup from /proc/pid/cgroup.

Requests from host are executed in behalf or client task uid:gid.

Requests from containers are executed in behalf of container's
**owner\_user**:**owner\_group**.

By defaut all users have read-only access to all visible containers.
Visibility is controlled via **enable\_porto** and **porto\_namespace**.

Write access have root user and users from group "porto".

Write access to container requires any of these conditions:  
* container is a sub-container  
* owner\_user matches to client user  
* owner\_user belongs to group "porto-containers"  
* owner\_user belongs to group "${CLIENT\_USER}-containers"  

## Filesystem

* **cwd** - working directory

    In host root default is temporary directory: /place/porto/*container-name*  
    In choot: '/'

* **root** - container root path, default: /

    Porto creates new mount namespace from parent mount namespace,
    and chroot into this direcotry using **pivot_root(2)**.

    Porto mounts: /dev, /dev/pts, /dev/hugepages, /proc, /run, /sys, /sys/kernel/tracing.

    Also porto recreates in /run directories strictire from underlying filesystem.

    Porto creates in /dev nodes only for devices permitted by property **devices**.

    If container should have access to porto then /run/portod.socket is binded inside.

* **root\_readonly** - remount everything read-only

* **bind** - bind mounts: \<source\> \<target\> \[ro|rw\];...

    This option creates new mount namespace and binds directories and files
    from parent container mount namespace. Bind mounts are invisible from
    host or parent container and cannot be used for volumes.

* **stdout\_path** - stdout file, default: internal rotated storage

    By default *stdout* and *stderr* are redirected into files created in
    default **cwd**.

    Periodically, when size of these files exceeds **stdout\_limit** head bytes
    are removed using **fallocate(2)** FALLOC_FL_COLLAPSE_RANGE. Count of lost
    bytes are show in **stdout\_offset**.

    Path "/dev/fd/*fd*" redirects stream into file descriptor *fd* of
    porto client task who starts container.

* **stdout\_limit** - limits internal stdout/stderr storage, porto keeps tail bytes

    Default is 8Mb, value limited with 1Gb.

* **stderr\_path** - stderr file, default: internal rotated storage

    Same as **stdout\_path**.

* **stdin\_path** - stdin file; default: "/dev/null"

* **place** - places allowed for volumes and layers, syntax: \[default\]\[;mask;...\]

    Default is "/place;\*\*\*" (use /place by default, allow any other),
    for container with chroot default is "/place".

* **place\_limit** - limits sum of volume **space\_limit** owned by subtree

    Format: total|default|\<place\>|tmpfs|lvm \<group\>|rbd: \<bytes\>;...

* **place\_usage** - current sum of volume **space\_limit** owned by subtree

    Format is same as for **place\_limit**.

* **volumes\_owned** - list of volumes charged into **place\_usage**

* **volumes\_linked** - list of volumes linked to this container

* **volumes\_required** - list of volumes reqired to start this container

Setting **bind**, **root**, **stdout\_path**, **stderr\_path** requires
write permissions to the target or owning related volume.

## Memory

* **memory\_usage** - current memory usage: anon + cache

* **anon\_usage** - current anon memory usage

* **cache\_usage** - current cache usage

* **hugetlb\_usage** - current hugetlb memory usage

* **hugetlb\_limit** - hugetlb memory limit

* **max\_rss** - peak anon memory usage (offstream kernel feature)

* **memory\_guarantee** - guarantee for memory\_usage, default: 0

    Memory reclaimer skips container and it's sub-containers if current usage
    is less than guarantee. Overcommit is forbidden, 2Gb are reserved for host system.

* **memory\_limit** limit for memory\_usage, default: 0 (unlimited)

    Allocations over limit reclaims cache or write anon pages into swap.
    On failure syscalls returns ENOMEM\EFAULT, page fault triggers OOM.

* **memory\_guarantee\_total** -  hierarchical memory guarantee

    Upper bound for guaranteed memory for sub-tree.

* **memory\_limit\_total** - hierarchical memory limit

    Upper bound for memory usage for sub-tree.

* **anon\_limit** - limit for anon\_usage, default: 0 (offstream kernel feature)

* **dirty\_limit** limit for dirty memory unwritten to disk, default: 0 (offstream kernel feature)

* **recharge\_on\_pgfault** - if *true* immigrate cache on minor page fault, default: *false* (offstream kernel feature)

* **pressurize\_on\_death** - if *true* set tiny soft memory limit for dead and hollow meta containers, default *false*

* **oom\_is\_fatal** - kill all affected containers on OOM, default: *true*

* **oom\_score\_adj** - OOM score adjustment: -1000..1000, default: 0

    See oom\_score\_adj in **proc(5)**.

* **minor\_faults** - count minor page-faults (file cache hits)

* **major\_faults** - count major page-faults (file cache misses, reads from disk)

## CPU

* **cpu\_usage** - CPU time used in nanoseconds (1 / 1000\_000\_000s)

* **cpu\_usage\_system** - kernel CPU time in nanoseconds

* **cpu\_wait** - total time waiting for execution in nanoseconds (offstream kernel feature)

* **cpu\_weight** - CPU weight, syntax: 0.01..100, default: 1

    Multiplies cpu.shares and increase task priority/nice.

* **cpu\_guarantee** - desired CPU power

    Syntax: 0.0..100.0 (in %) | \<cores\>c (in cores), default: 0

    Increase cpu.shares accourding to required cpu power distribution.
    Offstream kernel patches provides more accurate control.

* **cpu\_guarantee\_total** - effective CPU guarantee

    Porto popagates CPU guarantee from childrents into parent containtes:
    cpu\_guarantee\_total = max(cpu\_guarantee, sum cpu\_guarantee\_total for running childrens)

    For root container this shows total CPU guarantee.

* **cpu\_limit** - CPU usage limit

    Syntax: 0.0..100.0 (in %) | \<cores\>c (in cores), default: 0 (unlimited)

    Porto setup both CFS and RT cgroup limits.
    RT cgroup limit is strictly non-overcommitable in mainline kernel.

    For root container this shows total CPU count.

* **cpu\_limit\_total** - total CPU limit

    cpu\_limit\_total = sum min(cpu\_limit, cpu\_limit\_total) for running or
    meta childrens plus contianer cpu\_limit if it's running.

    For root contianer this shows total CPU commitment.

* **cpu\_period** - CPU limit accounting period

   Syntax: 1ms..1s, default: 100ms \[nanoseconds\]

* **cpu\_policy**  - CPU scheduler policy, see **sched(7)**
    - *normal*   - SCHED\_OTHER (default)
    - *high*     - SCHED\_OTHER (nice = -10, increases cpu.shares by 16 times)
    - *rt*       - SCHED\_RR    (nice = -20, priority = 10)
    - *batch*    - SCHED\_BATCH
    - *idle*     - SCHED\_IDLE  (also decreases cpu.shares by 16 times)
    - *iso*      - SCHED\_ISO   (offstream kernel feature)

* **cpu\_set** - CPU affinity
    - \[N|N-M,\]... - set CPU mask
    - *node* N      - bind to NUMA node
    - *reserve* N   - N exclusive CPUs
    - *threads* N   - only N exclusive CPUs
    - *cores* N     - only N exclusive SMT cores, one thread for each

* **cpu\_set\_affinity** - resulting CPU affinity: \[N,N-M,\]...

## Disk IO

Disk names are single words, like: "sda" or "md0".

"fs" is a statistics and limits at filesystem level (offstream kernel feature).

"hw" is a total statistics for all disks.

Statistics and limits could be requested for filesystem path.
Absolute paths are resolved in host, paths starting with dot in chroot:
**io\_read\[/\]**, **io\_read\[.\]**.

* **io\_read** - bytes read from disk, syntax: \<disk\>: \<bytes\>;...

* **io\_write** - bytes written to disk, syntax: \<disk\>: \<bytes\>;...

* **io\_ops** - disk operations: \<disk\>: \<count\>;...

* **io\_time** - total io time: \<disk\>: \<nanoseconds\>;...

* **io\_limit** - IO bandwidth limit, syntax: fs|\<path\>|\<disk\> \[r|w\]: \<bytes/s\>;...
    - fs \[r|w\]: \<bytes\>     - filesystem level limit (offstream kernel feature)
    - \<path\> \[r|w\]: \<bytes\> - setup blkio limit for disk used by this filesystem
    - \<disk\> \[r|w\]: \<bytes\> - setup blkio limit for disk

* **io\_ops\_limit**  - IOPS limit: fs|\<path\>|\<disk\> \[r|w\]: \<iops\>;...

* **io\_policy** IO scheduler policy, see **ioprio\_set(2)**
    - *none*    - set by **cpu\_policy**, blkio.weight = 500 (default)
    - *rt*      - IOPRIO\_CLASS\_RT(4), blkio.weight = 1000 (highest)
    - *high*    - IOPRIO\_CLASS\_BE(0), blkio.weight = 1000
    - *normal*  - IOPRIO\_CLASS\_BE(4), blkio.weight = 500
    - *batch*   - IOPRIO\_CLASS\_BE(7), blkio.weight = 10
    - *idle*    - IOPRIO\_CLASS\_IDLE, blkio.weight = 10 (lowest)

* **io\_weight** IO weight, syntax: 0.01..100, default: 1

    Additional multiplier for blkio.weight.

## Network

Matching interfaces by name support masks '?' and '\*'.
Interfaces aggregated into groups from /etc/iproute2/group, see **ip-link(8)**.

* **net** - network namespace configuration, Syntax: \<option\> \[args\]...;...
    - *inherited*            - use parent container network namespace (default)
    - *none*                 - empty namespace, no network access (default for virt\_mode=os)
    - *L3* \<name\> \[master\]  - veth pair and ip routing from host
    - *NAT* \[name\]         - same as L3 but assign ip automatically
    - *tap* \<name\>         - tap interface with L3 routing
    - *ipip6* \<name\> \<remote\> \<local\> - ipv4-via-ipv6 tunnel
    - *MTU* \<name\> \<mtu\> - set MTU for device
    - *MAC* \<name\> \<mac\> - set MAC for device
    - *autoconf* \<name\>    - wait for IPv6 SLAAC configuration
    - *ip* \<cmd\> \<args\>... - configure namespace with **ip(8)**
    - *container* \<name\>   - use namespace of this container
    - *netns* \<name\>       - use namespace created by **ip-netns(8)**
    - *steal* \<device\>     - steal device from parent container
    - *macvlan* \<master\> \<name\> \[bridge|private|vepa|passthru\] \[mtu\] \[hw\]
    - *ipvlan* \<master\> \<name\> \[l2|l3\] \[mtu\]
    - *veth* \<name\> \<bridge\> \[mtu\] \[hw\]

* **ip**             - ip addresses, syntax: \<interface\> \<ip\>/\<prefix\>;...

* **ip\_limit**      - ip allowed for sub-containers: none|any|\<ip\>\[/\<mask\>\];...

    If set sub-containers could use only L3 networks and only with these **ip**.

* **default\_gw**    - default gateway, syntax: \<interface\> \<ip\>;...

* **hostname**       - hostname inside container

    Inside container root /etc/hostname must be a regular file,
    porto bind-mounts temporary file over it.

* **resolv\_conf**   - DNS resolver configuration, syntax: \<resolv.conf option\>;...

    Default value for containers with chroot could be set in portod.conf:
```
container { default_resolv_conf: "nameserver <ip>;nameserver <ip>;..." }
```

    Inside container root /etc/resolv.conf must be a regular file,
    porto bind-mounts temporary file over it.

    Setting **resolv\_conf**="" keeps configuraion in chroot as is.

* **sysctl**         - sysctl configuration, syntax: \<sysctl\>: \<value\>;...

    Porto allows to set only virtualized sysctls from hardcoded white-list.

    Default values for network and ipc sysctls are the same as in host and
    could be overriden in portod.conf:
```
container {
    ipc_sysctl {
        key: "sysctl"
        val: "value"
    },
    ...
    net_sysctl {
        key: "sysctl"
        val: "value"
    },
    ...
}
```

* **net\_guarantee** - minimum egress bandwidth: \<interface\>|group \<group\>|default: \<Bps\>;...

* **net\_limit**     - maximum egress bandwidth: \<interface\>|group \<group\>|default: \<Bps\>;...

* **net\_rx\_limit** - maximum ingress bandwidth: \<interface\>|group \<group\>|default: \<Bps\>;...

* **net\_bytes**     - traffic class counters: \<interface\>|group \<group\>: \<bytes\>;...

* **net\_class\_id** - traffic class: \<interface\>: major:minor (hex)

* **net\_drops**     - tc drops: \<interface\>|group \<group\>: \<packets\>;...

* **net\_overlimits** - tc overlimits: \<interface\>|group \<group\>: \<packets\>;...

* **net\_packets**   - tc packets: \<interface\>|group \<group\>: \<packets\>;...

* **net\_rx\_bytes** - device rx bytes: \<interface\>|group \<group\>: \<bytes\>;...

* **net\_rx\_drops** - device rx drops: \<interface\>|group \<group\>: \<packets\>;...

* **net\_rx\_packets** - device rx packets: \<interface\>|group \<group\>: \<packets\>;...

* **net\_tx\_bytes** - device tx bytes: \<interface\>|group \<group\>: \<bytes\>;...

* **net\_tx\_drops** - device tx drops: \<interface\>|group \<group\>: \<packets\>;...

* **net\_tx\_packets** - device tx packets: \<interface\>|group \<group\>: \<packets\>;...

L3 network connects host and container netns with veth pair and configures
routing in both directions. Also it adds neighbour/arp proxy entries to
interfaces with addresses from the same network, this way container becomes
reachable from the outside. This requires sysctl configuration in host:
```
net.ipv4.conf.all.forwarding = 1
net.ipv6.conf.all.forwarding = 1
net.ipv6.conf.all.proxy_ndp = 1
net.ipv6.conf.all.accept_ra = 2
```

NAT requires configuration in iptables and in portod.conf:
```
network {
    nat_first_ipv4: "*ip*"
    nat_first_ipv6: "*ip*"
    nat_count: *count*
}
```

Example:
```
# iptables -t nat -A POSTROUTING -s 192.168.42.0/24 -j MASQUERADE
# ip6tables -t nat -A POSTROUTING -s fec0::42:0/120 -j MASQUERADE
network {
    nat_first_ipv4: "192.168.42.1"
    nat_first_ipv6: "fec0::42:1"
    nat_count: 255
}
```

In new network namespaces porto setup **ip-addrlabel(8)** from portod.conf:
```
network {
        addrlabel {
                prefix: "ip/mask"
                label: number
        },
        ...
```

Porto setup **tc-hfsc(8)** scheduler for all interfaces except listed in
portod.conf as unmanaged:
```
network {
    unmanaged_device: "name"
    unmanaged_group: "group"
}
```

# CGROUPS

Porto enables all cgroup controllers for first level containers or
if any related limits is set. Otherwise container shares cgroups with parent.

Controller "cpuacct" is enabled for all containers if it isn't bound with other controllers.

Controller "freezer" is used for management and enabled for all containers.

Cgroup tree required for systemd is configured automatically for virt\_mode=os
containers if /sbin/init is a symlink to systemd.

Enabled controllers are show in property **controllers** and
could be enabled by: **controllers\[name\]**=true.
For now cgroups cannot be enabled for running container.
Resulting cgroup path show in property **cgroups\[name\]**.

All cgroup knobs are exposed as read-only properties \<name\>.\<knob\>,
for example **memory.status**.

# COREDUMPS

Portod might register itself as a core dump helper and forward cores into container if it has set core\_command.

Variables set in environment and substituted in core\_command:

- CORE\_PID (pid inside container)
- CORE\_TID (crashed thread id)
- CORE\_SIG (signal)
- CORE\_TASK\_NAME (comm for PID)
- CORE\_THREAD\_NAME (comm for TID)
- CORE\_EXE\_NAME (executable file)
- CORE\_CONTAINER
- CORE\_OWNER\_UID
- CORE\_OWNER\_GID

Command executed in non-isolated sub-container and gets core dump as stdin.

For example core\_command='tee core-\$\{CORE\_TASK\_NAME\}-\$\{CORE\_PID\}-\$\{CORE\_SIG\}'
saves core into file in container. Container ulimit\[core\] must be set > 1 or unlimited.

Information in proc about crashed process and thread available too.
Porto makes sure that sysctl kernel.core\_pipe\_limit isn't zero,
otherwise crashed task could exit and dismantle pid namespace too early.


Required setup in portod.conf:
```
core {
   enable: true
   default_pattern: "/coredumps/%e.%p.%s"
   space_limit_mb: 102400
   slot_space_limit_mb: 10240
}
```

Default pattern is used for non-container cores or if core command isn't set.
It might use '%' kernel core template defined in *core(5)*.
If default\_pattern ends with '.gz' or '.xz' core will be compressed.

File owner set according to **owner_user** and **owner_group**.

For uncompressed format porto detects zero pages and turns them into file holes
and flushes written data to disk every 4Mb (set by option core.sync\_size).

Porto also creates hardlink in same directory with name:
```
${CORE_CONTAINER//\//%}%${CORE_EXE_NAME}.${CORE_PID}.S${CORE_SIG}.$(date +%Y%m%dT%H%M%S).core
```

Option space\_limit\_mb limits total size of default pattern directory,
after exceeding new cores are discarded.

Option slot\_space\_limit\_mb limits total size for each first-level container.

Porto never deletes old core dumps.

# VOLUMES

Porto provides "volumes" abstraction to manage disk space.
Each volume is identified by a full path.
You can either manually declare a volume path or delegate this task to porto.

A volume can be linked to one or more containers, links act as reference
counter: unlinked volume will be destroyed automatically. By default volume
is linked to the container that created it: "self", "/" for host.

By default volume unlink calls lazy **umount(2)** with flag MNT\_DETACH,
strict unlink calls normal umount and fails if some files are opened.

## Volume Properties

Like for container volume configuration is a set of key-value pairs.

* **backend** - backend engine, default: autodetect
    - *plain*     - bind mount **storage** to volume path
    - *bind*      - bind mount **storage** to volume path, requires volume path
    - *tmpfs*     - mount new tmpfs instance
    - *quota*     - project quota for volume path
    - *native*    - bind mount **storage** to volume path and setup project quota
    - *overlay*   - mount overlayfs and optional project quota for upper layer
    - *squash*    - overlayfs and quota on top of squashfs image set in **layers**
    - *loop*      - create and mount ext4 image **storage**/loop.img or **storage** if this's file
    - *rbd*       - map and mount ext4 image from caph rbd **storage**="id@pool/image"
    - *lvm*       - ext4 in **lvm(8)** **storage**="\[group\]\[/name\]\[@thin\]\[:origin\]"

    Depending on chosen backend some properties becomes required of not-supported.

* **storage**       - persistent data storage, default: internal non-persistent

    - */path*     - path to directory to be used as storage
    - *name*      - name of internal persistent storage, see [Volume Storage]

    Some backends (*rbd*, *lvm*) expects configuration in special format.

* **ready**         - is construction complete

* **state**         - volume state
    - *initial*
    - *building*
    - *ready*
    - *unlinked*
    - *to-destroy*
    - *destroying*
    - *destroyed*

* **private**       - user-defined property, 4k text

* **owner\_container** - owner container, default: creator

    Used for tracking **place\_usage** and **place\_limit**.

* **owner\_user**   - owner user, default: creator

* **owner\_group**  - owner group, default: creator

* **user**          - directory user, default: creator

* **group**         - directory group, default: creator

* **permissions**   - directory permissions, default: 0775

* **creator**       - container user group

* **read\_only**    - true or false, default: false

* **containers**    - initial container references, syntax: container;...  default: "self"

* **layers**        - layers, syntax: top-layer;...;bottom-layer

    - */path*     - path to layer directory
    - *name*      - name of layer in internal storage, see [Volume Layers]

    Backend *overlay* use layers directly.

    Backend *squash* expects path to a squashfs image as top-layer.

    Other backends copy layers' files into volume during construction.

* **place**         - place for layers and default storage

    This is path to directory where must be sub-directories
    "porto_layers", "porto_storage" and "porto_volumes".

    Default and possible paths are controller by container property **place**.

* **place\_key**    - key for charging **place\_limit** for **owner\_container**

    Key equal to **place** if backend keeps data in filesystem.

    Key is empty if backend doesn't limit or own data.
    Like plain, bind, quota, or if storage is provided by user.

    Some backend use own keys: "tmpfs", "lvm \<group\>", "rbd".

* **space\_limit**     - disk space limit, default: 0 - unlimited

* **inode\_limit**     - disk inode limit, default: 0 - unlimited

* **space\_guarantee** - disk space guarantee, default: 0

    Guarantees that resource is available time of creation and
    protects from claiming by guarantees in following requests.

* **inode\_guarantee** - disk inode guarantee, default: 0

* **space\_used**      - current disk space usage

* **inode\_used**      - current disk inode used

* **space\_available** - available disk space

* **inode\_available** - available disk inodes

## Volume Layers

Porto provides internal storage for overlayfs layers.
Each layer belongs to particular place and identified by name,
stored in **place**/porto\_layers/**layer**.
Porto remembers owner's user:group and time since last usage.

Layer name shouldn't start with '\_'.

Layer which names starts with '\_weak\_' are removed once last their user is gone.

Porto provide API for importing and exporting layers in form compressed tarballs
in overlay or aufs formats. For details see **portoctl** command layers.

For building layers see **portoctl** command build
and sample scripts in layers/ in porto sources.

## Volume Storage

Storage is a directory used by volume backend for keeping volume data.
Most volume backends by default use non-persistent temporary storage:
**place**/porto\_volumes/id/**backend**.

If storage is specified then volume becomes persistent and could be
reconstructed using same storage.

Porto provides internal volume storage similar to layers storage:
data are stored in **place**/porto\_storage/**storage**.

For details see **portoctl** command storage.

## Backend LVM

Backend *lvm* takes configuration from property **storage** in format: \[group\]\[/name\]\[@thin\]\[:origin\]".

Default volume group could be set in portod.conf
```
volumes { default_lvm_group: "group" }
```

It "name" is set volume becomes persistent: porto keeps and reuse logical volume "group/name".
User have to remove it using **lvremove(8)**.

If "thin" is set them volume is allocated from precreated thin pool "group/thin".

If "origin" is set then volume is created as thin snapshot of "group/origin" and belongs to the same pool.

# EXAMPLES

Run command in foreground:
```
$ portoctl exec hello command='echo "Hello, world!"'
```

Run command in background:
```
$ portoctl run stress command='stress -c 4' cpu\_limit=2.5c
$ portoctl destroy stress
```

Create volume with automatic path and destroy:
```
$ V=$(portoctl vcreate -A space_limit=1G)
$ portoctl vunlink $V
```

Run os level container and enter inside:
```
portoctl run vm layers=template.tgz space_limit=1G virt_mode=os hostname=vm memory_limit=1G cpu_limit=2c net="L3 eth0" ip="eth0 192.168.1.42"
portoctl shell vm
```

Show containers and resource usage:
```
portoctl top
```

See **portoctl(8)** for details.

# FILES

/run/portod.socket

    Porto API unix socket.

/run/portod  
/run/portod.version

    Symlink to currently running portod binary and it's version.

/run/portoloop.pid  
/run/portod.pid

    Pid file for porto master and slave daemon.

/var/log/portod.log

    Porto daemon log file.

/run/porto/kvs  
/run/porto/pkvs

    Container and volumes key-value storage.

/usr/share/doc/yandex-porto/rpc.proto.gz

    Porto API protobuf.

/etc/portod.conf  
/etc/defaults/portod.conf

    Porto daemon configuration in protobuf text format.
    Porto merges it with hardcoded defaults and prints into log when starts.

/place/porto/*container*

    Default current/working directories for containers.

/place/porto\_volumes/*id*

    Default place keeping volumes and their data.

/place/porto\_layers/*layer*

    Default place for keeping overlayfs layers.

/place/porto\_storage/*storage*

    Default place for persistent volume storages.

/place/porto\_containers/*container*.CT

    Container saved by **portoctl** and auto-started by porto service.

# HOMEPAGE

<https://github.com/yandex/porto>

# AUTHORS

The following authors have created the source code of "Porto" published and distributed by YANDEX LLC as the owner:

Roman Gushchin <klamm@yandex-team.ru>  
Stanislav Fomichev <stfomichev@yandex-team.ru>  
Konstantin Khlebnikov <khlebnikov@yandex-team.ru>  
Evgeniy Kilimchuk <ekilimchuk@yandex-team.ru>  
Michael Mayorov <marchael@yandex-team.ru>  
Stanislav Ivanichkin <sivanichkin@yandex-team.ru>  
Vsevolod Minkov <vminkov@yandex-team.ru>  
Vsevolod Velichko <torkve@yandex-team.ru>  
Maxim Samoylov <max7255@yandex-team.ru>  

# COPYRIGHT

Copyright (C) YANDEX LLC, 2015-2017.
Licensed under the GNU LESSER GENERAL PUBLIC LICENSE.
See LICENSE file for more details.
