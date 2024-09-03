# tctree

Linux `tc` output but in form of a tree with flexible templating, filtering, and watch mode.

## How it looks

```
net0
====
qdisc 1: htb 3.5 GiB 3636396 pkt
refcnt 2 r2q 10 default 0x160 direct_packets_stat 0 ver 3.17 ...
dropped: 0 overlimits: 0 requeues: 0 backlog: 0 qlen: 0
|
+- class 1:1 htb 3.5 GiB 3636396 pkt
   rate 6Mbit ceil 6Mbit linklayer ethernet burst 1599b/1 mpu...
   dropped: 0 overlimits: 0 requeues: 0 backlog: 0 qlen: 0
   |
   +- class 1:10 htb 3.4 GiB 3564514 pkt
   |  rate 1659Kbit ceil 6Mbit linklayer ethernet burst 1599b...
   |  dropped: 0 overlimits: 0 requeues: 0 backlog: 0 qlen: 0
   |  |
   |  +- qdisc 80e3: fq_codel 3.4 GiB 3564514 pkt
   |     limit 10240p flows 1024 quantum 757 target 5ms inter...
   |     dropped: 0 overlimits: 0 requeues: 0 backlog: 0 qlen: 0
   |
   +- class 1:160 htb 100.5 MiB 71882 pkt
      prio 7 quantum 1000 rate 1Kbit ceil 6Mbit linklayer eth...
      dropped: 0 overlimits: 0 requeues: 0 backlog: 0 qlen: 0
      |
      +- qdisc 80ee: fq_codel 100.5 MiB 71882 pkt
         limit 10240p flows 1024 quantum 757 target 5ms inter...
         dropped: 0 overlimits: 0 requeues: 0 backlog: 0 qlen: 0
```

#### Watch mode:

```
net0
====
Node                            Est. rate        Sent   Dropped   Overlim
qdisc 1: htb                    47.9 kbit     3.5 GiB        14     +9/39
+- class 1:1 htb                47.9 kbit     3.5 GiB         0     +6/21
   +- class 1:10 htb            47.9 kbit     3.4 GiB         0     +3/17
   |  +- qdisc 80e3: fq_codel   47.9 kbit     3.4 GiB         0         0
   +- class 1:160 htb                       100.5 MiB         0         0
      +- qdisc 80ee: fq_codel               100.5 MiB         0         0   
```

## How to install

To run the tool you need only one file `tctree`. It's a script written in POSIX Shell.

```sh
# Download the script using curl:
curl -O https://raw.githubusercontent.com/deflock-linux/tctree/main/tctree

# Or using wget:
wget https://raw.githubusercontent.com/deflock-linux/tctree/main/tctree

# Make it executable.
chmod +x tctree

# That's it! You can already run it.
./tctree
```

Optionally you can move it to any of your `bin`-directories from `PATH` and make it system-wide. E.g.:

```sh
sudo mv tctree /usr/local/bin/tctree

# Now you can run simply:
tctree
```

## Features

```
tctree [options] [...targets]
```

#### Targets

Target is an interface and optionally a handle you want to inspect; in the form: `<interface>[/<handle>]`.  
If handle is not specified then interface's root egress and ingress qdiscs are used.

```sh
tctree net0/1:1 wwan0
```

Targets are optional. If no target is specified tctree discovers all network interfaces (excluding `lo`) and uses them as targets.

### Filtering

It is possible to hide any nodes and their subtrees using special conditions syntax:

```sh
# Hide nodes without traffic
tctree net0 --hide 'sent_bytes = 0'

# All net0 nodes except the one with handle 1:10 (including its subtree)
tctree net0 --hide "handle = '1:10'"

# This may be seen as alias to `--hide 'type = "class" && kind = "fq_codel"'`
tctree net0 --hide-class 'kind = "fq_codel"'

# HTB-tree without fq_codel subtrees
tctree net0 --hide "parent.kind = 'htb' && kind = 'fq_codel'"

# Do not show interface's ingress qdisc (if it exists)
tctree net0 --hide-qdisc 'kind = "ingress"'
```

### Templating

It is possible to modify text of any node.  
These texts are generated from templates using another special syntax:

```sh
# Example of compact output
tctree net0 --template "{type} {handle} {kind} sent: {sent_bytes|traffic}"

# qdisc 1: prio sent: 6.3 GiB
# +- class 1:1 prio sent: 18.5 MiB
# +- class 1:2 prio sent: 6.3 GiB
# +- class 1:3 prio sent: 736.2 KiB
# qdisc ffff: ingress sent: 6.2 GiB

# Conditions in templates
tctree net0/1: --template "{type} {handle} [% if diff:sent_packets > 0 %]Rate: {rate:sent_bytes|traffic_rate}[% endif %]"

# qdisc 1: Rate: 107.4 kbit
# +- class 1:1
# +- class 1:2 Rate: 107.4 kbit
# +- class 1:3 
```

#### More information

```sh
tctree --help-full
```

## How it works

1. Collect distinct interfaces from targets.
2. Fetch `tc` output for these interfaces.
3. Parse output and store the data.
4. Build tree structure using data's parent fields.

For each target:

5. Set target's node as selected.
6. Render selected node's text from template.
7. Get selected node's children.
8. Filter children.
9. Sort non-filtered children.
10. For each sorted non-filtered child set it as selected, then go to (6).

In watch mode ring-buffer snapshots are used to store each update's data.  
Then in steps 6-9 data from several snapshots is used. Number of snapshots is configurable.

## Status

> [!WARNING]
> This project is still in BETA and needs some more testing on various network configurations.

> [!NOTE]
> This project is written in POSIX Shell. 
> It was supposed to be a small script but it's more than 5k lines now.
> At the same time POSIX Shell is not the best choice for such a task: lots of text manipulations, loops, tree structures, etc. 
> Therefore although a lot of optimizations were made performance may suffer when number of nodes grows.

## TODO

* Add tests
* Custom labels for handles: `--label "1:2 classA"`
* Custom transformers support
* Precompile expressions, conditions, and templates

## License

Copyright (c) 2024-present Alex Tsvetkov

This software is licensed under either of:
* Apache License (Version 2.0)
* GPL 2.0 (or any later version)

SPDX-License-Identifier: Apache-2.0 OR GPL-2.0-or-later
