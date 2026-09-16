---
title: "Five dollars a month instead of an open port"
date: 2021-03-16T21:50:00+07:00
draft: true
description: "Every home service you add tempts you to forward one more port on a modem you do not own. Renting a small VPS as a VPN hub costs less than the remote-desktop licence it replaces and means the count of open ports stays at zero."
summary: "The honest reason the homelab exists. Port forwarding on the ISP modem scales badly and scales dangerously: one hole per service, on a box you do not control. A 5 USD VPS turns every connection into an outbound one."
tags: ["homelab", "networking", "vpn", "security"]
---

> *Archive note. Written in March 2021 for a blog I have since retired, and the earliest post in this sequence - everything else here grew out of it. The prices and the specific products are of their time. The ISP is genericised. What follows next chronologically is [mining ETH in a homelab](/posts/mining-eth-in-a-homelab/).*

**TL;DR.** You want to reach a machine at home from outside. The obvious answers are a hosted remote-desktop product or forwarding a port on the ISP modem. The product costs more than a small VPS and does less; the port forward is free but puts a listening service on a box you do not own, that your ISP may forbid you to change, and that the whole internet can reach. Renting the cheapest VPS available at 5 USD a month and running a VPN on it inverts the direction: every device dials out to the hub, sees every other device as though they shared a LAN, and the number of inbound holes at home stays zero. The reason that matters is not the first service. It is the fourth.

## The network everyone starts with

Home networks are not standardised. Two flats with identical floor plans end up wired differently depending on who lives there. The common shape is an ISP modem and router combo with everything hanging off it: a PC on Ethernet, a laptop and a phone on wifi, maybe a second router or access point bridging a far room.

That is enough for what most people need. Web, streaming, games with low latency. Nothing about it is wrong.

Then one day you want to reach the PC at home while you are not at home, and to do it safely.

## The obvious products, and what they cost

The ready-made answers are the hosted remote-desktop tools, and they work well at the job they do. Each has a catch:

- **TeamViewer** is free only if your use counts as personal, and the moment it decides otherwise you are paying. The licence I was comparing against was 24.90 USD.
- **AnyDesk** is the same shape with fewer features.
- **Chrome Remote Desktop** is genuinely free and it is the one I would point a non-technical person at. It also renegotiates to a direct route rather than hauling traffic through Google's servers. The catch is that it runs in Chrome, so some key combinations never reach the remote machine - press Ctrl-W expecting to close a window over there and you close a tab over here.

I was lazy and short of money, which is a combination that pushes you toward the option that solves more than one problem. The cheapest VPS on offer was 5 USD a month. That is less than the remote-desktop licence, and a VPS is a general-purpose machine rather than a single-purpose product.

Point the devices at a VPN running on that VPS and they stop being machines on separate networks. They see each other as if they were on one LAN, which means the free, built-in remote-desktop tools - RDP, VNC - just work, with no third party in the path and no product tier to worry about. Laptop, phone, tablet, all reaching the machines at home, for 5 USD a month.

## The port forward, and why I did not

The objection is immediate and fair: run the VPN server on the ISP modem itself, forward a port, and save the 5 USD. If the contract does not include a static IP, dynamic DNS services will hand you a free domain that tracks your address as it changes. People do this and it works.

I am not going to call it wrong. I will say what it costs, because the cost is not obvious when there is only one service.

Forwarding a port puts a process in listening state, reachable from the entire internet, on a device you did not choose, do not administer, and cannot patch. Your ISP picked it, ships its firmware, and in many contracts reserves the right to refuse you the configuration anyway. That is a large attack surface for a saving of 5 USD, and the asymmetry is unpleasant: you are betting the security of everything behind that modem against a rounding error on a monthly bill.

Since I already owned a domain, I bought the VPS and pointed Cloudflare at it. That part was preference rather than necessity.

## Then it happens again

Here is the part that changes the arithmetic, and the reason this post exists.

You put up a security camera. It has a box, the box wants a port, you forward one so the phone app can see the feed. Fine - one port.

A few days later you want to host something at home, a blog perhaps. A VM, and a port exposed to the internet. Two.

A few days after that it is home automation: the garage door, the water heater, the lights. Another VM, another port. Three.

Then there is the several hundred gigabytes of material on the PC that you would like to watch in bed, or away from the house entirely. Four.

The list of things you might want to run at home does not terminate. Each one, taken alone, is one small hole and a reasonable trade. Taken together they are a steadily growing inbound attack surface on hardware you do not control, added one defensible decision at a time. That is what makes it dangerous: no single step feels like the wrong call.

The VPN hub does not make any individual service safer. It makes the count stop growing. Every service stays bound to the private network, every client arrives through the tunnel, and the number of ports open on the modem stays at zero no matter how long the list of services gets.

<svg class="dg" viewBox="0 0 900 480" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="hub-t hub-d">
<title id="hub-t">A port per service versus one outbound tunnel</title>
<desc id="hub-d">Left: the internet reaches the ISP modem, and each home service the owner adds - camera, blog, home automation, media - needs its own forwarded inbound port on that modem, so the number of internet-reachable listeners grows with the number of services, on hardware the owner does not administer. Right: a rented VPS runs a VPN hub. The modem opens nothing. Devices away from home and the home network itself both dial outward to the hub and meet there, so adding services leaves the count of inbound ports at zero.</desc>
<defs>
  <marker id="hub-ar" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar" d="M0,0 L7,3 L0,6 Z"/></marker>
  <marker id="hub-ar-b" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar-b" d="M0,0 L7,3 L0,6 Z"/></marker>
  <marker id="hub-ar-c" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar-c" d="M0,0 L7,3 L0,6 Z"/></marker>
</defs>
<rect class="zone" x="20" y="30" width="420" height="350" rx="6"/>
<text class="t" x="40" y="56">A port for each thing</text>
<text class="s" x="40" y="74">the count grows with the service list</text>
<rect class="box-e" x="175" y="90" width="110" height="36" rx="4"/>
<text class="m" x="230" y="113" text-anchor="middle">internet</text>
<path class="ln-c" d="M230,126 L230,144" marker-end="url(#hub-ar-c)"/>
<rect class="box-c" x="150" y="150" width="160" height="44" rx="4"/>
<text class="m" x="230" y="170" text-anchor="middle">ISP modem</text>
<text class="s" x="230" y="187" text-anchor="middle">4 inbound ports open</text>
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
<text class="s" x="40" y="318">every listener is reachable from the whole internet</text>
<text class="s" x="40" y="340">on a box the ISP chose, ships firmware for, and may lock</text>
<text class="s" x="40" y="362">no single one of these feels like the wrong call</text>
<rect class="zone" x="460" y="30" width="420" height="350" rx="6"/>
<text class="t" x="480" y="56">One tunnel out</text>
<text class="s" x="480" y="74">5 USD a month, nothing listening at home</text>
<rect class="box-a" x="600" y="90" width="140" height="44" rx="4"/>
<text class="m" x="670" y="110" text-anchor="middle">VPS</text>
<text class="s" x="670" y="127" text-anchor="middle">VPN hub</text>
<rect class="box" x="480" y="175" width="120" height="42" rx="4"/>
<text class="s" x="540" y="195" text-anchor="middle">laptop, phone</text>
<text class="s" x="540" y="211" text-anchor="middle">anywhere</text>
<rect class="box-b" x="700" y="175" width="160" height="42" rx="4"/>
<text class="m" x="780" y="195" text-anchor="middle">ISP modem</text>
<text class="s" x="780" y="211" text-anchor="middle">0 ports open</text>
<path class="ln-b" d="M540,175 L540,154 L670,154 L670,140" marker-end="url(#hub-ar-b)"/>
<path class="ln-b" d="M780,175 L780,154 L670,154" />
<rect class="box-b" x="690" y="255" width="180" height="48" rx="4"/>
<text class="m" x="780" y="275" text-anchor="middle">home services</text>
<text class="s" x="780" y="292" text-anchor="middle">the same four, and the next</text>
<path class="ln-b" d="M780,255 L780,223" marker-end="url(#hub-ar-b)"/>
<text class="s" x="480" y="330">both ends dial outward and meet on the hub</text>
<text class="s" x="480" y="352">add the fifth service and the port count is still zero</text>
<rect class="zone" x="20" y="400" width="860" height="60" rx="6"/>
<text class="t" x="40" y="426">The VPN does not make any one service safer</text>
<text class="s" x="40" y="448">It stops the number of internet-reachable listeners growing with the number of things you run.</text>
</svg>

## What this actually was

I wrote this as a log of my own decisions, so that when something broke months later I would know where to look, and on the off-chance it helped somebody else circling the same choice.

Reading it back, it is the load-bearing decision. Once every machine at home sits on one private network that outside devices can join, the questions stop being about connectivity and start being about capacity: what else could run here, which box should run it, and what happens when one of them dies. That is a homelab. It started as an argument about 5 USD and a port.
