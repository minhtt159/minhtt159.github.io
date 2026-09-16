---
title: "Five dollars a month instead of an open port"
date: 2021-03-16T21:50:00+07:00
draft: true
description: "Every home service you add tempts you to forward one more port on a modem you do not own. Renting a small server as a VPN hub does not remove the internet-facing listener; it replaces several you cannot patch with one you can."
summary: "The honest reason the homelab exists. Port forwarding on the ISP modem accumulates: one listener per service, on a box you do not administer. A 5 USD virtual server does not make the listener disappear - it moves it somewhere you control."
tags: ["homelab", "networking", "vpn", "security"]
---

> *Archive note. Written in March 2021 in Hanoi, for a blog I have since retired, and the earliest post in that blog's homelab sequence - everything else grew out of it. Prices and products are of their time, and the arrangement itself did not last long: I moved to the Netherlands within about a year, and a VPN hub hosted by a Vietnamese carrier stops being the obvious choice once you live on the other side of the planet from it. What follows chronologically is [mining ETH in a homelab](/posts/mining-eth-in-a-homelab/).*

**TL;DR.** You want to reach a machine at home from outside. You can forward a port on the ISP modem, or you can rent a VPS - a virtual private server, a small Linux machine rented by the month - and run a VPN on it that everything dials out to. Both designs put a process on the internet where strangers can reach it. The difference is how many, and whose. Forwarding grows one internet-facing listener per service, on firmware you did not choose and cannot patch. The VPN hub has exactly one listener, on a machine you administer, and that count does not move when the service list grows. The reason this matters is not the first service. It is the fourth.

Concretely: the VPN was WireGuard, deployed with [Algo](https://github.com/trailofbits/algo) - Ansible scripts from Trail of Bits that stand up a personal WireGuard or IPsec server with hardened defaults - on a virtual machine rented from Viettel IDC - a Vietnamese cloud provider whose small instances sit in much the same bracket as a DigitalOcean droplet, and the hosting arm of the same carrier that supplied the line at home. Both of those choices do more work in the argument than they look, so they get their own sections.

## The network everyone starts with

Home networks are not standardised. Two flats with identical floor plans end up wired differently depending on who lives there. The common shape is an ISP modem and router combo with everything hanging off it: a PC on Ethernet, a laptop and a phone on wifi, maybe a second router or access point bridging a far room.

That is enough for what most people need. Web, streaming, games with low latency. Nothing about it is wrong.

Then one day you want to reach the PC at home while you are not at home, and to do it safely.

## The ready-made answers

The hosted remote-desktop tools do this job and do it well:

- **TeamViewer** is free only while your use counts as personal, and the moment it decides otherwise you are paying. The figure I had in mind at the time was 24.90 USD. I did not write down whether that was per month or per year, and I am not going to reconstruct it now - which means I cannot honestly claim the VPS was cheaper, only that it was not obviously more expensive and did considerably more.
- **AnyDesk** is the same shape with fewer features.
- **Chrome Remote Desktop** is genuinely free, and it is what I would still point a non-technical person at. It also renegotiates to a direct route instead of hauling traffic through Google's servers. Its catch is small: it runs in Chrome, so some key combinations never reach the far machine - press Ctrl-W expecting to close a window over there and you close a tab over here.

So I should be straight about the decision, because the cost framing flatters it. Chrome Remote Desktop was free and solved the stated problem. Paying 5 USD a month to escape a hotkey collision would be absurd. What I actually wanted was a general-purpose machine with a public address, which happens to solve remote access as a side effect and then keeps being useful. The remote desktop was the excuse, not the reason.

Buying domestically mattered more than the price did, though the price was competitive anyway - Viettel IDC's small instances sit where an equivalent DigitalOcean droplet sits. The machine was with the hosting business of the carrier already terminating the line at my flat, so traffic between home and hub stayed inside one national network instead of detouring through Singapore or Frankfurt and back. For a tunnel that every interactive session traverses, that is the difference between a remote desktop feeling local and feeling remote. It is also the part of this design that did not survive moving continents.

Point everything at a VPN running on that VPS and the devices stop being on separate networks. They address each other as though they shared a LAN, which means the built-in remote-desktop protocols - RDP on Windows, VNC elsewhere - work with no product tier involved. Laptop, phone, tablet, all reaching the machines at home.

Worth naming what that does not buy: the VPS is a third party in the path, and unlike Chrome Remote Desktop's direct route, it is in the path for every packet. I traded a company I do not control for a rented box I do, which is a real improvement in authority and a real regression in hop count.

## What is actually running on it

WireGuard, put there by Algo.

Algo is worth naming rather than leaving as "a VPN", because half of what makes this design defensible is which server you expose. It is Ansible, run from your own laptop against a fresh cloud instance, and its opinions are the point. It declines to install OpenVPN or Tor. It drops legacy cipher suites and protocols - no L2TP, no IKEv1, no RSA. It does not put its security on TLS. Users are declared in a config file before deployment and each one gets a generated WireGuard config and a QR code, so adding a phone is scanning a square rather than editing anything on the server. It builds on an Ubuntu LTS with unattended security upgrades switched on.

The WireGuard part matters more than the automation. WireGuard does not reply to packets that fail authentication. A port scan of the VPS finds nothing on that port, because the server says nothing to anyone who cannot already prove they belong. That is a materially different exposure profile from a forwarded port on a modem, where whatever is behind it answers and identifies itself to anybody who knocks. The listener exists, and it is silent.

It is also small. WireGuard's kernel implementation is a few thousand lines against the hundreds of thousands in the IPsec and OpenVPN stacks, and that ratio is the whole reason it is possible to feel relaxed about running one on a public address.

## The port forward, and why I did not

The obvious objection: run the VPN server on the ISP modem, forward one port, save the 5 USD. If the contract has no static IP, dynamic DNS will give you a free name that tracks the address as it moves. People do this and it works.

The catch is that it works only where the ISP lets you configure the modem at all. Plenty of contracts ship locked firmware, and where that is true this alternative does not lose on security, it simply is not available. Where it is available, here is what it costs.

Forwarding a port puts a process in listening state, reachable from the entire internet, on a device you did not choose, do not administer, and cannot patch. Your ISP picked it and ships its firmware on its own schedule. When a vulnerability lands in that firmware, your options are to wait.

Since I already owned a domain, I bought the VPS and pointed a DNS record at it through Cloudflare, so I had a name rather than an address to remember. That is all Cloudflare is doing here - a VPN is not HTTP and does not travel through a caching proxy.

## Then it happens again

Here is the part that changes the arithmetic, and the reason this post exists.

You put up a security camera. Its recorder wants to be reachable so the phone app can see the feed, so you forward a port. Fine - one.

A few days later you want to host something at home, a blog perhaps. A VM, another port. Two.

Then home automation: the garage door, the water heater, the lights. Another VM, another port. Three.

Then the several hundred gigabytes of material on the PC that you want to watch from outside the house. Four.

The list does not terminate. Each addition, alone, is one small hole and a reasonable trade. Together they are a steadily growing inbound attack surface on hardware you do not control, assembled one defensible decision at a time. That is what makes it dangerous: no single step feels like the wrong call.

Two honest caveats before the conclusion.

**The count grows only if you let it.** Three of those four services speak HTTP. One reverse proxy on one VM fronts all of them behind a single port, and now the growth is in proxy configuration rather than in forwarded ports. That is a real answer to the problem and it is what many people do. It also leaves you one internet-facing listener on a VM you patch - which, as the next section admits, is exactly where the VPN hub lands. The two designs are closer relatives than this post originally implied.

**Not everything can join a VPN.** The camera is the first example in my own escalation, and a camera appliance almost certainly cannot run a VPN client, nor can most home-automation hubs. Those devices never join the tunnel. They stay on the LAN and speak to nothing outside it; what joins the tunnel are the clients, and the tunnel gives those clients a route onto the LAN where the appliances already live. The appliance does not need to participate for this to work, which is fortunate, because it cannot.

## What the hub actually buys

The VPN hub does not remove the internet-facing listener. It has one: the VPN server itself, on a public address, which is the whole point of it being reachable.

I spent this entire post arguing against internet-facing listeners, so let me state the real claim rather than the flattering one. The gain is not zero exposure. It is that the count stops tracking the service list, and that the one remaining listener sits on a machine I chose, administer and can patch the same day a vulnerability is published. Four listeners on ISP firmware become one listener on my own box, and the fifth service adds none.

<svg class="dg" viewBox="0 0 900 480" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="hub-t hub-d">
<title id="hub-t">Where the internet-facing listener lives, in two designs</title>
<desc id="hub-d">Both designs have something listening on the internet; the diagram is about how many and on whose hardware. Left: the internet reaches the ISP modem, and each service the owner adds - camera, blog, home automation, media - needs its own forwarded inbound port there, so the number of internet-reachable listeners grows with the service list, on firmware the owner did not choose and cannot patch. Right: the internet reaches one listener, a WireGuard server on a rented virtual machine the owner administers, which does not answer packets that fail authentication and so does not announce itself to a port scan. The ISP modem forwards nothing; the home network and any device away from home both dial outward to that hub and meet on it. Adding a fifth service leaves the listener count at one. Red marks an internet-facing listener in both panels, so the single red arrow on the right is the honest admission that the exposure moved rather than vanished.</desc>
<defs>
  <marker id="hub-ar" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar" d="M0,0 L7,3 L0,6 Z"/></marker>
  <marker id="hub-ar-b" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar-b" d="M0,0 L7,3 L0,6 Z"/></marker>
  <marker id="hub-ar-c" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar-c" d="M0,0 L7,3 L0,6 Z"/></marker>
</defs>
<rect class="zone" x="20" y="30" width="420" height="350" rx="6"/>
<text class="t" x="40" y="56">Four listeners, none of them yours</text>
<text class="s" x="40" y="74">the count grows with the service list</text>
<rect class="box-e" x="175" y="90" width="110" height="34" rx="4"/>
<text class="m" x="230" y="112" text-anchor="middle">internet</text>
<path class="ln-c" d="M230,124 L230,144" marker-end="url(#hub-ar-c)"/>
<rect class="box-c" x="150" y="150" width="160" height="44" rx="4"/>
<text class="m" x="230" y="170" text-anchor="middle">ISP modem</text>
<text class="s" x="230" y="187" text-anchor="middle">4 ports forwarded</text>
<path class="ln-c" d="M230,194 L230,216 L80,216 L80,234" marker-end="url(#hub-ar-c)"/>
<path class="ln-c" d="M230,194 L230,216 L180,216 L180,234" marker-end="url(#hub-ar-c)"/>
<path class="ln-c" d="M230,194 L230,216 L280,216 L280,234" marker-end="url(#hub-ar-c)"/>
<path class="ln-c" d="M230,194 L230,216 L380,216 L380,234" marker-end="url(#hub-ar-c)"/>
<rect class="box" x="34" y="240" width="92" height="42" rx="4"/>
<text class="s" x="80" y="266" text-anchor="middle">camera</text>
<rect class="box" x="134" y="240" width="92" height="42" rx="4"/>
<text class="s" x="180" y="266" text-anchor="middle">blog</text>
<rect class="box" x="234" y="240" width="92" height="42" rx="4"/>
<text class="s" x="280" y="266" text-anchor="middle">home auto</text>
<rect class="box" x="334" y="240" width="92" height="42" rx="4"/>
<text class="s" x="380" y="266" text-anchor="middle">media</text>
<text class="s" x="40" y="318">firmware you did not choose and cannot patch</text>
<text class="s" x="40" y="340">when a bug lands in it, your option is to wait</text>
<text class="s" x="40" y="362">no single one of these feels like the wrong call</text>
<rect class="zone" x="460" y="30" width="420" height="350" rx="6"/>
<text class="t" x="480" y="56">One listener, and it is yours</text>
<text class="s" x="480" y="74">the count does not move when the list grows</text>
<rect class="box-e" x="480" y="90" width="100" height="34" rx="4"/>
<text class="m" x="530" y="112" text-anchor="middle">internet</text>
<path class="ln-c" d="M580,107 L644,107" marker-end="url(#hub-ar-c)"/>
<rect class="box-a" x="650" y="84" width="200" height="46" rx="4"/>
<text class="m" x="750" y="104" text-anchor="middle">VPS: WireGuard</text>
<text class="s" x="750" y="121" text-anchor="middle">1 listener, silent to scans</text>
<rect class="box" x="480" y="168" width="120" height="42" rx="4"/>
<text class="s" x="540" y="188" text-anchor="middle">laptop, phone</text>
<text class="s" x="540" y="204" text-anchor="middle">away from home</text>
<path class="ln-b" d="M540,168 L540,148 L750,148 L750,136" marker-end="url(#hub-ar-b)"/>
<rect class="box" x="670" y="180" width="160" height="42" rx="4"/>
<text class="m" x="750" y="200" text-anchor="middle">ISP modem</text>
<text class="s" x="750" y="216" text-anchor="middle">0 ports forwarded</text>
<path class="ln-b" d="M750,180 L750,136" marker-end="url(#hub-ar-b)"/>
<rect class="box-b" x="670" y="262" width="160" height="50" rx="4"/>
<text class="m" x="750" y="282" text-anchor="middle">home LAN</text>
<text class="s" x="750" y="299" text-anchor="middle">the four, and the next</text>
<path class="ln-b" d="M750,262 L750,228" marker-end="url(#hub-ar-b)"/>
<text class="s" x="480" y="338">green dials outward; nothing at home accepts a connection</text>
<text class="s" x="480" y="360">appliances never join the tunnel - the clients do, and reach them</text>
<rect class="zone" x="20" y="400" width="860" height="60" rx="6"/>
<text class="t" x="40" y="426">The hub does not remove the exposure. It relocates it.</text>
<text class="s" x="40" y="448">Four listeners on firmware you cannot patch become one on a machine you can, and the fifth service adds none.</text>
</svg>

## What it costs, stated plainly

Three bills come with this that the 5 USD does not cover.

**You now run a server.** The modem was condemned for being unpatchable; the replacement is patchable and therefore must be patched. Algo takes the worst of that away by building on an Ubuntu LTS with unattended security upgrades on by default, which is more maintenance than most home routers ever receive. It does not take all of it away: unattended upgrades cover distribution packages, not the day the LTS reaches end of life and the whole instance needs rebuilding. A VPN server abandoned across that boundary is worse than the port forward it replaced.

**It is a single point of failure.** A forwarded port fails one service at a time. A hub fails everything at once, including the route you would use to get back in and fix it. That is a genuinely worse failure mode and the only mitigation is a second way in, which costs either money or the exact exposure you were avoiding.

**Everything transits the rented box.** Including, if you go through with the media plan, video. On the cheapest instance available, which comes with the bandwidth allowance you would expect at that price. Watching something in bed never needed the tunnel - that traffic is local - but watching it from elsewhere does, and that is where this design meets its limit.

## What this actually was

I wrote it as a log of my own decisions, so that when something broke months later I would know where to look, and on the chance it helped somebody circling the same choice.

Reading it back, it is the load-bearing one. Once every machine at home sits on one private network that outside devices can join, the questions stop being about connectivity and start being about capacity: what else could run here, which box should run it, and what happens when one of them dies. That is a homelab. It started as an argument about 5 USD and a port.
