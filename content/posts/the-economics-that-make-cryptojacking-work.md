---
title: "The arithmetic that makes cryptojacking worth it"
date: 2021-07-25T17:00:00+07:00
draft: true
description: "Renting GPUs to mine has a return-on-investment calculation that usually says no. Removing the cost line from that calculation is what turns a miner into malware, and it explains why GPU credentials are worth stealing."
summary: "A miner that schedules itself onto any GPU node is a workload like any other, and its economics come down to one subtraction. Drive the cost term to zero and the answer is always yes, which is the whole reason cryptojacking exists."
tags: ["security", "threat-model", "kubernetes", "gpu", "cloud"]
---

> *Archive note. Written in July 2021 for a blog I have since retired, when Ethereum still ran on proof of work and EIP-1559 was about to change the yield. The coin is gone and the numbers with it. The economic argument is the part that still holds, and it now applies to whatever GPUs are used for next.*

**TL;DR.** Once a miner is a container scheduled by a replica count, running it on rented cloud GPUs is a straightforward calculation: mined value minus rental cost. That subtraction usually comes out negative, which is why renting to mine is mostly a bad idea and buying the coin is the rational move. The interesting case is what happens to the calculation when the cost term is zero, because someone else is paying for the compute. Then there is no threshold to clear and no break-even to wait for, and that is the entire economic engine behind cryptojacking. If your organisation runs GPUs, that is what makes the credentials in front of them worth stealing.

## Where this picks up

Three months earlier I had been [mining ETH in a homelab](/posts/mining-eth-in-a-homelab/) by passing a GPU into a Windows virtual machine. The conclusion of that exercise was that the passthrough was unnecessary: a container skips it, and once the miner is a container, Kubernetes will place it against an advertised GPU resource like any other workload.

Three months of running it produced some coin, some experience, and one observation I had not expected to make - which is that I had been looking at an attack surface without recognising it.

## Renting, and the problem it creates

Cloud providers are generous with GPU instances, and people do rent them for exactly this. Around that time it was common to see acquaintances renting compute VPS capacity to plot Chia.

The obstacle is not renting one instance; it is managing many. A single VM tops out at around four GPUs. Suppose the price moves and you want ten GPU instances. You can deploy them quickly with a tool like `ansible`, but when one of them breaks you have to work out *which* one and go fix it by hand. Ten machines with individual identities is ten things to babysit.

Kubernetes answers that directly. On a cloud provider it grows the instance count when you need capacity and shrinks it with one action, and a pod that dies is rescheduled rather than investigated. The unit of work becomes fungible. It does start to sound like passive income.

None of this is new. Back in 2017, when ETH last ran up hard, people were already building on this stack to rent GPUs and mine. In 2018 the price crashed and nobody mentioned it again.

## When information technology does return on investment

The idea is a few lines of configuration. The hard question is not how, it is when: when is it worth renting at all?

Buying hardware means accounting for the capital, the cooling, the electricity to run it, and how many months until it pays back. Renting is simpler, and comes down to one subtraction:

```
profit = value mined - rental cost
```

If that is positive you rent. If it is negative you buy the coin on an exchange and skip the engineering, or you leave crypto alone entirely if you have no conviction about it.

<svg class="dg" viewBox="0 0 900 450" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="roi-t roi-d">
<title id="roi-t">Three ways to fund the same miner, and what each one has to clear</title>
<desc id="roi-d">One container image feeds three funding models. Owning the hardware carries capital, power and cooling and must clear a break-even period. Renting cloud GPUs carries an hourly cost that the mined value has to exceed, which it usually does not. Running on compute someone else pays for carries no cost term at all, so the subtraction has nothing to clear and the answer is always yes. That missing cost line is what makes the third column an attack rather than a business.</desc>
<defs>
  <marker id="roi-ar" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar" d="M0,0 L7,3 L0,6 Z"/></marker>
  <marker id="roi-ar-c" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar-c" d="M0,0 L7,3 L0,6 Z"/></marker>
</defs>
<rect class="box-a" x="330" y="30" width="240" height="52" rx="4"/>
<text class="t" x="450" y="52" text-anchor="middle">One miner container</text>
<text class="s" x="450" y="70" text-anchor="middle">replicas: N, nodeSelector: gpu</text>
<path class="ln" d="M400,82 L250,82 L250,124" marker-end="url(#roi-ar)"/>
<path class="ln" d="M450,82 L450,124" marker-end="url(#roi-ar)"/>
<path class="ln-c" d="M500,82 L680,82 L680,124" marker-end="url(#roi-ar-c)"/>
<rect class="box-b" x="30" y="130" width="240" height="200" rx="4"/>
<text class="t" x="50" y="156">Own the hardware</text>
<text class="s" x="50" y="182">capital: cards, case, power supply</text>
<text class="s" x="50" y="204">running: electricity, cooling</text>
<text class="s" x="50" y="226">recovery: walk to the machine</text>
<text class="m" x="50" y="262">must clear:</text>
<text class="m" x="50" y="284">break-even, in months</text>
<text class="s" x="50" y="314">answer: sometimes</text>
<rect class="box-d" x="330" y="130" width="240" height="200" rx="4"/>
<text class="t" x="350" y="156">Rent the hardware</text>
<text class="s" x="350" y="182">capital: none</text>
<text class="s" x="350" y="204">running: hourly instance cost</text>
<text class="s" x="350" y="226">recovery: reschedule the pod</text>
<text class="m" x="350" y="262">must clear:</text>
<text class="m" x="350" y="284">mined minus rental</text>
<text class="s" x="350" y="314">answer: usually no</text>
<rect class="box-c" x="630" y="130" width="240" height="200" rx="4"/>
<text class="t" x="650" y="156">Pay for neither</text>
<text class="s" x="650" y="182">capital: none</text>
<text class="s" x="650" y="204">running: billed to the owner</text>
<text class="s" x="650" y="226">recovery: not your problem</text>
<text class="m" x="650" y="262">must clear:</text>
<text class="m" x="650" y="284">nothing at all</text>
<text class="s" x="650" y="314">answer: always yes</text>
<rect class="zone" x="30" y="356" width="840" height="76" rx="6"/>
<text class="t" x="50" y="382">The cost line is the only thing holding the third column back</text>
<text class="s" x="50" y="404">Remove it and there is no threshold, no break-even, and no reason to stop.</text>
<text class="s" x="50" y="424">That is why the credentials in front of a GPU fleet are worth more than they look.</text>
</svg>

## The version without a cost line

Now hold the mined value constant and set the rental cost to zero.

Every threshold in the calculation disappears. There is no break-even period to survive, no price level the coin has to reach, and no reason to ever turn it off. A workload that is marginal at market rates becomes unconditionally profitable the moment somebody else is paying the bill. Nothing about the miner changes - same image, same replica count, same pool. Only the invoice moves.

That is not a hypothetical. It is the standing economic reason that idle compute gets stolen, and it says something specific about what an attacker is shopping for. They do not need to reach your data. They need to reach something that can start a process on a machine with a card in it, and they need it to stay unnoticed while it bills you.

Two things fit that description uncomfortably well in a GPU estate. The first is service credentials for an inference endpoint, which by design sit in front of exactly the hardware a miner wants and are handled by more systems than anyone has fully enumerated. The second is anything that shapes how a node comes up - the initialisation path of a GPU scale set, for instance - because it runs with the node's authority on every instance the group ever creates, and nobody reads it after the first time it works.

## What to do about it

The defensive reading is not "watch for miners". Signature-matching a known miner binary catches the lazy version and nothing else. The useful reading is that GPU capacity is a directly monetisable asset on your balance sheet, and should be protected the way other directly monetisable assets are:

- **Treat inference and GPU-scheduling credentials as money.** Scope them, rotate them, and alert on use from somewhere they have never been used from. They are not read-only API keys; they are a way to spend your compute budget.
- **Version and review node initialisation the way you review production code.** A mutable startup script attached to a scale set is an unreviewed root shell that runs on every machine the group creates.
- **Alert on utilisation, not on binaries.** Sustained high GPU utilisation with no corresponding job, or instance counts that grow without a workload behind them, describe the condition regardless of which miner is running. The billing anomaly and the capacity anomaly are the same event seen from two directions.
- **Ask what a compromised credential is worth in cash.** For most of the industry this question came into focus only once the hardware became scarce and expensive. GPUs got there first.

## Closing

EIP-1559 is coming, and at the rates involved renting GPUs legitimately to mine ETH cannot be made profitable anyway. That kills this particular instance of the arithmetic, not the arithmetic. Some future coin, or some future thing GPUs are scarce for, will rebuild the same subtraction, and the same term will be the one worth removing.

Writing this down probably reduces the number of instances I can quietly rent. That trade seems fine. The interesting work was never the mining.
