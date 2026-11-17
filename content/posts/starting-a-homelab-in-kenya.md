---
title: "How to Start a Homelab in Kenya"
date: 2026-11-17
draft: false
summary: "A practical guide to starting a homelab in Nairobi for about KES 50,000 with three used mini PCs, covering hardware, power, software and the mistakes to avoid."
tags: ["homelab", "proxmox", "kenya", "self-hosting", "ups"]
categories: ["Infrastructure", "Homelab"]
---

You can start a real homelab in Nairobi for about KES 50,000 with three used mini PCs, or about KES 18,000 for one mini PC.

This guide is based on more than a year of running my own, so it focuses on what I would do differently: buy more RAM, protect the router from KPLC, and set up backups before you need them. The prices are what I paid in 2025 next to what the same listings showed in November 2026, so check current stock before you spend anything.

## Step 1: Buy the hardware

Buy used Lenovo ThinkCentre Tiny PCs, and get 16 GB of RAM in each one if you can. Mine are three M900s with Core i5-6500T chips, and they have run non-stop for over a year without drama.

| Item | What I paid in 2025 | Listed on 10 Nov 2026 | Where |
| --- | --- | --- | --- |
| M900 Tiny, i5 6th gen, 8 GB, 256 GB SSD | KES 15,000 to 18,000 | KES 17,000 ex VAT | [Future Technologies](https://futuretechnologies.co.ke/product/ex-uk-lenovo-thinkcentre-m900-tiny-desktop-micro-tower-pc-intel-core-i5-6500t-8gb-ram-256gb-ssd-usb-3-0-wifi-windows-11) |
| M900 Tiny, i5, 16 GB, 512 GB SSD | not bought, but can work | KES 33,000 | [Jiji, Nairobi Central](https://jiji.co.ke/sw/nairobi-central/16-desktop-computers/lenovo) |
| M720q Tiny, i5-8500T, 256 GB NVMe | about KES 19,000 | KES 23,000 ex VAT | [Future Technologies](https://futuretechnologies.co.ke/?p=8181) |
| Three nodes, total | about KES 48,000 | KES 51,000 ex VAT for three | |

How to buy:

- **Add 16% VAT.** Shops quote "ex VAT", so add it at checkout.
- **Check stock first.** None of the listings showed stock status, so message the seller before you go.
- **Ask about RAM.** The M720q page contradicts itself (8 GB in the specs, 16 GB in the title), so confirm before you pay.
- **Search Jiji for "ThinkCentre Tiny"** for stock that changes daily. The sellers are usually on Moi Avenue, so spend a Saturday afternoon making calls and go see the machines before you pay.
- **Try Zayre Technologies.** [Zayre Technologies](https://www.tiktok.com/%40zayregadget_nrb) is on Instagram and TikTok. I single source from them these days, as they understand the homelab space and offer good support. If you go to them, tell them Duncan sent you, they might give you a discount.

Two choices I would make differently today:

- **RAM first, CPU second.** My three nodes now have 16 to 20 GB each, and memory is still the thing that runs out, never the processor. Start at 16 GB per node and go to 32 GB if the board allows it.
- **Get 512 GB, not 256 GB.** Two of my three nodes have 256 GB drives and one of them is already about 80% full. One node got a 512 GB drive and it has room to breathe.

On Dell vs HP vs Lenovo, this Reddit thread is the one that convinced me about Lenovo: [Lenovo Tiny or Dell Micro for first homelab](https://www.reddit.com/r/homelab/comments/1frq4q2/lenovo_tiny_or_dell_micro_for_first_homelab/). The [ServeTheHome TinyMiniMicro series](https://www.servethehome.com/introducing-project-tinyminimicro-home-lab-revolution/) is the long read.

## Step 2: Set up power and network

KPLC is the real enemy of a Kenyan homelab, so buy the UPS before you buy anything fancy. I got a 1000VA, 600W UPS for KES 11,000 from [Vivahtech](https://vivahtech.co.ke/product/s-tech-1000va-600w-ups/). It gave me about 30 minutes in theory and about 28 minutes in a real power cut, with the machines shut down early to stretch it. Today a Mecer 1000VA, 600W UPS with a USB port is [KES 17,000 ex VAT at Rapidtech](https://rapidtech.co.ke/product/mecer-1000va-1kva-600w-me-1000-vu-line-interactive-ups-with-avr), and a Lightwave 1000VA is [KES 8,500 ex VAT at Lapsol](https://lapsol.ke/product/lightwave-lw-1000va-3-pin-ups-lw-ups1000), though its page doesn't mention a USB port.

- **Put the router on the UPS too.** Mine sits on plain mains, so when the power goes the internet goes with it, instantly, even while the servers are still running. Plug in the router and the fibre box as well.
- **Choose a UPS with a USB data port.** Mine has none, so my lab cannot ask the UPS "are we on battery?". I had to make the nodes watch the router instead. With a USB port, the free NUT software does it properly.
- **Set "AC power recovery" to On in the BIOS.** After a cut, the machine should switch itself back on when power returns. The Lenovos did this by default.
- **Run a cable instead of Wi-Fi.** I pulled a cable from my router to my home office, and the shop crimped it for under KES 2,000. Servers on cable are faster and don't drop.
- **Add a switch once you have more than one node.** To start, one cable from your router is enough. With a cluster you need a switch and a short cable per node. Buy a gigabit one, because a 10/100 switch like the TP-Link LS1005 is slow when you copy VM disks around. The gigabit [LS1005G is KES 2,400 ex VAT at TDK](https://www.tdk.co.ke/product/tp-link-ls1005g-5-port-10-100-1000mbps-desktop-switch).

## Step 3: Install the software in stages

Don't try to build everything on day one. Do one stage, live with it for a few weeks, then move on.

1. **One machine, a few useful things.** Install [Proxmox VE](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso) on one box. You can run VMs for heavier work and LXC containers for small stuff. Then open the [Proxmox community scripts](https://community-scripts.github.io/ProxmoxVE/scripts), which have a one-line install for almost anything you will want to self-host. Add [Tailscale](https://tailscale.com/) so you can reach it from anywhere.
2. **Make the house use it.** Run an ad-blocking DNS server (Pi-hole or AdGuard Home), and put a reverse proxy such as Caddy in front so each service gets a friendly name instead of an IP and port. Add a simple start page like Glance, and run your first Docker apps. This is where the lab starts to feel useful.
3. **Add a second and third node.** Now you can cluster. Try k3s (a small Kubernetes), a self-hosted Git server such as Gitea, and ArgoCD so that the cluster deploys whatever you commit to Git. Add Prometheus, Grafana and Loki so you can see what is happening, and Ansible to patch every machine the same way. Only do this stage if you want to learn Kubernetes. Plenty of good labs stop at stage 2.
4. **Add a separate box for local AI.** My Dell mini PC, which is not in the cluster, runs Ollama and Open WebUI for local models.

## Step 4: Reach your lab from outside

There are two different jobs here.

- **Tailscale is for you.** I run it on one machine, advertise my home network through it, and set up split DNS, so from my phone my lab names work anywhere.
- **Cloudflare is for other people.** Its free Zero Trust tier lets you publish a service on a subdomain without opening a port on your router. This [video](https://www.youtube.com/watch?v=wyKkeb3w5lI) walks through it.

## Mistakes to avoid

1. **Don't cluster for the sake of it.** Three nodes is about voting, not power. A Proxmox cluster needs a majority of its nodes to agree before it will change anything. With three nodes you can lose one and carry on. When one of mine went away for hardware repair, the cluster ran on two votes with nothing to spare, and a second failure would have locked the cluster's config read-only. If you only want one or two machines, run them standalone and skip clustering.
2. **Don't expect a switch to give you live migration.** A switch does the networking part only. My nodes each use their own local disk, so moving a VM means copying its disk across. Smooth live migration needs shared storage, which a first lab doesn't have.
3. **Set up backups on day one.** This is the one I would shout about. Add a backup disk or a second box early, and use the built-in Proxmox backup jobs or Proxmox Backup Server.

## What my lab looks like now

A year and a bit in, mine looks like this. It shows where the road goes, not where you should start.

| Part | What I run |
| --- | --- |
| Hardware | Three Lenovo M900 Tiny PCs in a Proxmox cluster, plus one Dell mini PC outside the cluster for AI |
| Guests | About 27 VMs and containers across the four machines |
| Kubernetes | Three small k3s clusters, deployed from Git with Gitea and ArgoCD |
| Names and DNS | Two DNS layers (ad-blocking for the house, plus my own internal zone) and Caddy giving every service a name |
| Watching it | Prometheus, Grafana and Loki, with alerts to my phone |
| Patching | Ansible runs weekly across every machine |
| Remote access | Tailscale subnet route and split DNS, so my lab names work from my phone |
| Power | UPS on the servers and switch, with scripts that shut guests down cleanly in a long cut |

I can still point at gaps: backups are patchy, and the router should be on the UPS. Every lab has a list like that, and it is part of the fun.

## Further reading

- [r/homelab](https://www.reddit.com/r/homelab/) for hardware, racks and show-offs
- [r/selfhosted](https://www.reddit.com/r/selfhosted/) for things worth running
- [ServeTheHome: Project TinyMiniMicro](https://www.servethehome.com/introducing-project-tinyminimicro-home-lab-revolution/) on Dell vs HP vs Lenovo
- [Proxmox community scripts](https://community-scripts.github.io/ProxmoxVE/scripts) for one-line installs
