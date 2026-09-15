---
title: "Mining ETH in a homelab, and the day it stopped being about mining"
date: 2021-04-26T17:00:00+07:00
draft: true
description: "Spare GPU slots in machines that were already running, a 4U case because a workstation chassis cooks cards, and a path from GPU passthrough on Proxmox to a Kubernetes Deployment that treats a miner as any other workload."
summary: "The goal was to make idle hardware pay the electricity bill. The route ran through IOMMU passthrough into a Windows VM, then out again into containers, and what survived the mining was the scheduling pattern underneath it."
tags: ["homelab", "proxmox", "kubernetes", "gpu", "virtualization"]
---

> *Archive note. Written in April 2021, with the Kubernetes half added that September, for a blog I have since retired. Ethereum left proof of work in 2022, so the mining is history. The passthrough and scheduling mechanics are the part that outlived it.*

**TL;DR.** During the Covid-era GPU shortage I had spare PCIe slots in machines that were already powered on for other reasons, and a few cards accumulated over the years. Mining ETH on them was a way to make that hardware cover the electricity, water and internet bills. Getting a GPU into a virtual machine meant IOMMU passthrough on Proxmox, because the cards that virtualise themselves properly cost enterprise money. The Windows guest turned out to be the unstable part. Containers removed the passthrough step entirely, and once the miner was a container, a Kubernetes Deployment with a `nodeSelector` scheduled it onto GPU nodes like any other workload. The mining paid for itself. The scheduling pattern is what I actually kept.

## Why bother

Four things lined up in early 2021:

1. Lockdowns gave people time to trade, and crypto and equities both moved, mostly upward.
2. The same lockdowns broke the semiconductor supply chain. GPUs were the scarce part, and GPUs are what mines ETH, so mining got harder and the price went up together.
3. I was already running a homelab, which means several PCs, workstations and servers at home. Almost none of them needed graphics, so I had a lot of empty PCIe slots.
4. I already owned a handful of GPUs, accumulated across years of student machine-building. Running them at full duty felt like getting value out of money already spent.

So: mine to cover the monthly bills, and a bit beyond. The condition attached is worth stating, because it is the whole economic argument. I would not have done this without the background to run it, and I would not have done it if it meant buying the hardware from scratch.

## Not building a rig

A dedicated miner builds a frame holding six to eight GPUs. Everything that is not a GPU is chosen to be as cheap as possible: a board with many PCIe slots, minimal CPU, minimal RAM, a small disk, a power supply that is adequate rather than good. Then a mining-specific operating system such as HiveOS goes on top. The machine mines and does nothing else, which is both the strength and the weakness.

I went the other way. The workstations already existed, so adding GPUs to them raised the utilisation of machines that were running regardless, and left the CPU and RAM free for the rest of the homelab services. That constraint - the machine has a day job - is what forced every decision that follows.

## Getting a GPU into a virtual machine

Mining software runs on Windows or on Linux, so the card needs to reach an operating system. Since I virtualise rather than installing on bare metal, a guest by default has no idea what hardware exists underneath it. Handing it real hardware needs IOMMU, the input-output memory management unit, which is what lets a device be mapped safely into a guest's address space.

In server hardware the clean answer is SR-IOV, single-root IO virtualization, where the card presents itself as several virtual functions. Cards that do that are priced accordingly. Compare an RTX A6000 against an RTX 3090 sometime: the same chip, the same CUDA core count, wildly different price. Enterprise cards were out of reach, so consumer cards it was, and the one I bought for this was an RTX 3080 at 24.5 million VND.

Consumer parts work for passthrough as long as the CPU and motherboard support IOMMU. I tend to buy high-end desktop CPUs, so this was never the blocker.

For the hypervisor there were three serious options: QEMU-KVM, Xen, and VMware ESXi. I was poor and partial to open source, which picks QEMU-KVM. Several distributions expose it, and the friendliest of those is Proxmox.

Then Proxmox got a Windows guest with the GPU passed through. Windows over HiveOS was a personal call - I needed Windows for other things anyway, and one card, an RTX 3060, only reached 48 MH/s after a driver unlock that exists on Windows. Down the rabbit hole: once it had been running a while, the Windows guest turned out to be the least stable component in the stack.

With the guest up, the rest is mundane. Install the miner, join a pool. My hashrate was small, so I picked ethermine for its 0.1 ETH payout floor, which suits a small operation - a higher floor means waiting months to see anything.

One quiet benefit of virtualising: when a bare-metal rig hangs, someone has to walk over and power-cycle it. Under Proxmox the guest reboots from a web interface, the same as VMware or VirtualBox.

Heat was the real problem. GPUs crammed into a workstation chassis cook. I bought a 4U case to dedicate to this, and that configuration ran about three months without drama.

## The part that outlived the mining

Some months in, a more experienced friend pointed out the obvious thing I had walked past: run the miner in a Docker container and the passthrough problem disappears. He was right. Containers skip the passthrough setup and avoid the CPU penalty that comes with it. Mining is not CPU-hungry enough for that penalty to matter much, but the simplification does.

Once the miner is a container, it is just another workload, and the whole apparatus for running workloads applies to it. That is where this stopped being about mining.

<svg class="dg" viewBox="0 0 900 470" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="mine-t mine-d">
<title id="mine-t">Two ways to get a miner onto a GPU: passthrough versus container scheduling</title>
<desc id="mine-d">Top row, the April 2021 path: Proxmox maps the card into a Windows guest through IOMMU passthrough, the miner runs inside that guest, and the guest is the fragile link. Bottom row, the September 2021 path: a Dockerfile is built by GitHub Actions into the GitHub container registry, a Kubernetes Deployment pulls it, and a nodeSelector plus the NVIDIA device plugin place the pod on a GPU node. The container path has no passthrough step and scales by changing a replica count.</desc>
<defs>
  <marker id="mine-ar" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar" d="M0,0 L7,3 L0,6 Z"/></marker>
  <marker id="mine-ar-a" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar-a" d="M0,0 L7,3 L0,6 Z"/></marker>
  <marker id="mine-ar-d" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar-d" d="M0,0 L7,3 L0,6 Z"/></marker>
</defs>
<rect class="zone" x="20" y="30" width="860" height="150" rx="6"/>
<text class="t" x="40" y="56">April 2021: pass the card into a guest</text>
<rect class="box" x="40" y="76" width="120" height="52" rx="4"/>
<text class="m" x="100" y="97" text-anchor="middle">Proxmox</text>
<text class="s" x="100" y="115" text-anchor="middle">QEMU-KVM host</text>
<rect class="box-d" x="180" y="76" width="120" height="52" rx="4"/>
<text class="m" x="240" y="97" text-anchor="middle">IOMMU</text>
<text class="s" x="240" y="115" text-anchor="middle">passthrough</text>
<rect class="box-d" x="320" y="76" width="120" height="52" rx="4"/>
<text class="m" x="380" y="97" text-anchor="middle">Windows VM</text>
<text class="s" x="380" y="115" text-anchor="middle">driver unlock</text>
<rect class="box-a" x="460" y="76" width="120" height="52" rx="4"/>
<text class="m" x="520" y="97" text-anchor="middle">miner</text>
<text class="s" x="520" y="115" text-anchor="middle">one card</text>
<rect class="box-e" x="600" y="76" width="120" height="52" rx="4"/>
<text class="m" x="660" y="97" text-anchor="middle">pool</text>
<text class="s" x="660" y="115" text-anchor="middle">0.1 ETH floor</text>
<path class="ln-d" d="M160,102 L174,102" marker-end="url(#mine-ar-d)"/>
<path class="ln-d" d="M300,102 L314,102" marker-end="url(#mine-ar-d)"/>
<path class="ln-d" d="M440,102 L454,102" marker-end="url(#mine-ar-d)"/>
<path class="ln" d="M580,102 L594,102" marker-end="url(#mine-ar)"/>
<text class="s" x="40" y="156">scales by building another machine; the guest is the fragile link</text>
<rect class="zone" x="20" y="210" width="860" height="170" rx="6"/>
<text class="t" x="40" y="236">September 2021: schedule it like any other workload</text>
<rect class="box" x="40" y="256" width="120" height="52" rx="4"/>
<text class="m" x="100" y="277" text-anchor="middle">Dockerfile</text>
<text class="s" x="100" y="295" text-anchor="middle">wraps T-Rex</text>
<rect class="box" x="180" y="256" width="120" height="52" rx="4"/>
<text class="m" x="240" y="277" text-anchor="middle">Actions</text>
<text class="s" x="240" y="295" text-anchor="middle">builds on push</text>
<rect class="box" x="320" y="256" width="120" height="52" rx="4"/>
<text class="m" x="380" y="277" text-anchor="middle">GHCR</text>
<text class="s" x="380" y="295" text-anchor="middle">free 500 MB</text>
<rect class="box-a" x="460" y="256" width="120" height="52" rx="4"/>
<text class="m" x="520" y="277" text-anchor="middle">Deployment</text>
<text class="s" x="520" y="295" text-anchor="middle">replicas: N</text>
<rect class="box-b" x="600" y="256" width="120" height="52" rx="4"/>
<text class="m" x="660" y="277" text-anchor="middle">GPU node</text>
<text class="s" x="660" y="295" text-anchor="middle">device plugin</text>
<rect class="box-e" x="740" y="256" width="120" height="52" rx="4"/>
<text class="m" x="800" y="277" text-anchor="middle">pool</text>
<text class="s" x="800" y="295" text-anchor="middle">same wallet</text>
<path class="ln-a" d="M160,282 L174,282" marker-end="url(#mine-ar-a)"/>
<path class="ln-a" d="M300,282 L314,282" marker-end="url(#mine-ar-a)"/>
<path class="ln-a" d="M440,282 L454,282" marker-end="url(#mine-ar-a)"/>
<path class="ln-a" d="M580,282 L594,282" marker-end="url(#mine-ar-a)"/>
<path class="ln" d="M720,282 L734,282" marker-end="url(#mine-ar)"/>
<text class="s" x="40" y="336">no passthrough step; a failed pod is rescheduled, not walked over to</text>
<text class="s" x="40" y="358">scales by changing one number, on hardware you own or hardware you rent</text>
<rect class="zone" x="20" y="400" width="860" height="52" rx="6"/>
<text class="m" x="40" y="432">nodeSelector keeps the pod off the nodes that have no card to give it</text>
</svg>

## Building the container path

The miner itself is someone else's tool. Phoenix Miner and T-Rex are both easy to drive and take a one percent developer fee, switching over to mine for their author for that share of the time. T-Rex has good Linux support, so that is the one I wrapped.

Using T-Rex is a matter of reading its command-line options and filling in the fields. Nothing about it needs modification, which is the point - the less of someone else's tool you touch, the less you maintain.

**Build the image.** Write a Dockerfile on the development machine, so that what runs in production is what you tested. Docker Compose can expose a GPU to a container for local testing. Worth knowing: in production it is not Docker that enables the GPU for the container. That is the device plugin's job, and it is a different mechanism with different failure modes.

**Build it in CI.** Push the repository to GitHub and set up a workflow that publishes the image. GitHub hosts a container registry with a free tier around 500 MB, which is far more than a mining container needs. After that the pipeline builds the package on every push and the cluster pulls it. GitLab, Docker Hub or a self-hosted registry would work identically - GitHub just kept the number of accounts down.

**Deploy it.** With the image in a registry, four things matter on the Kubernetes side:

- Install a device plugin on the GPU nodes. The [NVIDIA k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) advertises the cards as a schedulable resource.
- Set `replicas` to the number of agents you want. On a cloud provider backed by a scale set, that number can grow as far as the budget does.
- Set a `nodeSelector` so the miner only lands on nodes that have a card. Without it the scheduler will happily place the pod somewhere with nothing to mine on.
- Set `hostAliases` for the pool's domains. Skip this and you will end up explaining to whoever runs the DNS server why those lookups exist.

A working example is at [minhtt159/ghcr-tictactoe](https://github.com/minhtt159/ghcr-tictactoe). Fork it, change the wallet address, and it runs.

## What was actually learned

The mining was a means. What the exercise taught, and what stayed useful long after ETH stopped being mineable, is the shape of the answer: a GPU workload is only awkward while it is pinned to one machine. Pass a card into a guest and you have bought yourself a fragile, hand-placed, manually-recovered unit of work. Put the same binary in a container and let a scheduler place it against an advertised resource, and the unit of work becomes fungible. The hardware stops being something you log into and becomes something a replica count consumes.

Which raises a question I did not have a good answer for at the time: if this is fungible, and it schedules onto anything with a card, what stops it scheduling onto someone else's?
