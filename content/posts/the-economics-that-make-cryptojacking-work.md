---
title: "The economics that make cryptojacking work"
date: 2021-07-25T17:00:00+07:00
draft: false
description: "Renting GPUs to mine has a return-on-investment calculation that usually says no. Removing the cost line from that calculation is what turns a miner into malware, and it explains why GPU credentials are worth stealing. The shape of the argument, without the numbers, which I never wrote down."
summary: "A miner that schedules itself onto any GPU node is a workload like any other, and its economics come down to one subtraction. Drive the cost term to zero and the answer is always yes, which is the whole reason cryptojacking exists. A qualitative argument: the original never recorded the figures."
tags: ["security", "threat-model", "kubernetes", "gpu", "cloud"]
---

> *Archive note. Written in July 2021 for a blog I have since retired, when Ethereum still ran on proof of work, meaning its network was secured by machines racing to solve a puzzle and paid for doing it - which is what made a graphics card into an income source. EIP-1559, the fee-market change then days away, redirected most of each transaction fee to be destroyed rather than paid to whoever mined the block, cutting miner income at a stroke. Ethereum left proof of work entirely the following year. The coin is gone and the numbers with it; the economic argument is the part that still holds, and it applies to whatever GPUs turn out to be scarce for next.*

**TL;DR.** Cryptojacking is mining cryptocurrency on computers somebody else is paying for. This post is about why the arithmetic makes that so attractive, from someone who spent a while doing the paid-for version.

Once a miner is a container scheduled by a replica count, running it on rented cloud GPUs is a straightforward calculation: mined value minus rental cost, minus the fees the miner and the pool take off the top. At mid-2021 rates that subtraction came out negative often enough that renting to mine was mostly not worth doing. The interesting case is what happens when the cost term is zero, because someone else is paying for the compute. Then there is no threshold to clear and no break-even to wait for, and that is the entire economic engine behind cryptojacking. If your organisation runs GPUs, that is what makes the credentials in front of them worth stealing.

A warning about what this post is not: I never wrote down the instance prices, the hashrates, or what any of it earned. The argument here is about the *shape* of the calculation, and you should not take "usually negative" from me as a measured result. It was my impression at the time.

## Where this picks up

Three months earlier I had been [mining ETH in a homelab](/posts/mining-eth-in-a-homelab/) by passing a GPU into a Windows virtual machine, which is a fiddly way to spend a weekend. Since then a more experienced friend had pointed out that the passthrough was unnecessary: a container skips it, and once the miner is a container, Kubernetes will place it against an advertised GPU resource like any other workload. Working through what that implied is what this post is. The build instructions came later still, in September, and are folded into that earlier post - which therefore carries material from after this one despite its earlier date.

Three months of running the original setup produced some coin - I did not record how much - some experience, and one observation I had not expected to make, which is that I had been looking at an attack surface without recognising it.

## Renting, and the problem it creates

Cloud providers are generous with GPU instances, and renting one to mine on is entirely possible. Renting cloud capacity to farm a coin was certainly in the air: acquaintances were renting plain compute instances - a VPS, a virtual private server, with no GPU attached - to plot Chia, which is a storage-and-CPU workload rather than a GPU one. That is the same instinct applied to different hardware, not evidence about GPUs, and I should not have offered it as though it were.

The obstacle is not renting one instance; it is managing many. The original post asserted that a single VM tops out at around four GPUs, which was already wrong when written - the large instance types of the day carried eight accelerators, and some carried sixteen. The argument survives the correction, because the ceiling is a ceiling wherever it sits: past it you are managing a fleet rather than a machine. Suppose the price moves and you want ten GPU instances. You can deploy them quickly with a tool like `ansible`, but when one of them breaks you have to work out *which* one and go fix it by hand. Ten machines with individual identities is ten things to babysit.

Kubernetes answers that directly. On a cloud provider it grows the instance count when you need capacity and shrinks it with one action, and a pod that dies is rescheduled rather than investigated. The unit of work becomes fungible. It does start to sound like passive income.

None of this is new. Back in 2017, when ETH last ran up hard, people were already building on this stack to rent GPUs and mine. In 2018 the price crashed and the technique stopped being discussed as a business. That is worth stating precisely, because the rest of this post argues the economics are permanent: what the crash ended was the version where you pay for the compute. The version where you do not has no price floor to fall through, and it never went away.

## When IT does the ROI maths

The idea is a few lines of configuration. The hard question is not how, it is when: when is it worth renting at all?

Buying hardware means accounting for the capital, the cooling, the electricity to run it, and how many months until it pays back. Renting is simpler, and comes down to one subtraction:

```
profit = value mined - rental cost - miner fee - pool fee
```

The two fee terms are not rounding errors: T-Rex and its peers take about one percent by mining for their own author part of the time, and the pool takes its own cut. A calculation that omits them flatters the rental case.

If that comes out positive you rent. If it comes out negative, renting is not the way to get exposure to the coin - whether you should buy it instead is a different question with different risk, and nothing in this subtraction answers it. All this arithmetic tells you is whether to run the machines.

<svg class="dg" viewBox="0 0 900 450" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="roi-t roi-d">
<title id="roi-t">Three ways to fund the same miner, and what each one has to clear</title>
<desc id="roi-d">The same container image feeds three funding models, so the three columns differ only in who pays for the node underneath. Owning the hardware carries capital, power and cooling, and must clear a break-even period measured in months; this post does not work out whether it does. A dead card reschedules its pod like any other, but you are the one buying the replacement. Renting cloud GPUs carries an hourly cost that the mined value has to exceed, which the author believed it usually did not, though he recorded no figures to check that against. Running on compute someone else pays for carries no cost term at all, so the subtraction has nothing to clear and the answer is always yes. That missing cost line, not any difference in the software, is what makes the third column an attack rather than a business.</desc>
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
<rect class="box" x="30" y="130" width="240" height="200" rx="4"/>
<text class="t" x="50" y="156">Own the hardware</text>
<text class="s" x="50" y="182">capital: cards, case, power supply</text>
<text class="s" x="50" y="204">running: electricity, cooling</text>
<text class="s" x="50" y="226">recovery: reschedule, then buy</text>
<text class="m" x="50" y="262">must clear:</text>
<text class="m" x="50" y="284">break-even, in months</text>
<text class="s" x="50" y="314">answer: not worked out here</text>
<rect class="box-d" x="330" y="130" width="240" height="200" rx="4"/>
<text class="t" x="350" y="156">Rent the hardware</text>
<text class="s" x="350" y="182">capital: none</text>
<text class="s" x="350" y="204">running: hourly instance cost</text>
<text class="s" x="350" y="226">recovery: reschedule the pod</text>
<text class="m" x="350" y="262">must clear:</text>
<text class="m" x="350" y="284">mined minus rental</text>
<text class="s" x="350" y="314">answer: my impression, no</text>
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

Two things fit that description uncomfortably well in any organisation that runs GPUs.

The first is machine-learning serving. Mining is not the only reason to buy a GPU fleet; the far more common one is running models, and a business that serves model predictions puts an inference endpoint - an internal service that takes a request and answers it using a model on a GPU - in front of that hardware.

A credential for that endpoint is not by itself enough, and it is worth being exact about why, because the sloppy version of this claim is everywhere. Being able to send inference requests is not being able to run a miner; sitting next to a GPU is not executing on it. What matters is the surrounding platform rather than the endpoint: the control-plane credential that can deploy a new model server, the model-loading path that will fetch and unpickle an artifact you supply, the notebook service that exists precisely to run arbitrary code on the expensive hardware. Those meet the bar. The inference key on its own does not, and an attacker who has one is looking for the next hop.

The second is anything that shapes how a node comes up. A cloud scale set - the managed group that creates identical machines on demand, which every major provider offers under its own name - takes an initialisation script that runs on every instance it ever creates, with that instance's authority. If that script can be edited, editing it once is enough, and nobody reads it again after the first time it works.

## What to do about it

The defensive reading is not "watch for miners". Signature-matching a known miner binary catches the lazy version and nothing else. The useful reading is that GPU capacity is a directly monetisable asset on your balance sheet, and should be protected the way other directly monetisable assets are:

- **Treat GPU-scheduling credentials as money, and inference credentials as the hop before them.** Scope them, rotate them, and alert on use from somewhere they have never been used from. They are not read-only API keys; they are a way to spend your compute budget.
- **Version and review node initialisation the way you review production code.** A mutable startup script attached to a scale set is an unreviewed root shell that runs on every machine the group creates.
- **Alert on utilisation, not on binaries or names.** DNS-based detection is barely better than signatures: a pod spec can pin a pool's name to an address in the pod's own hosts file, so the lookup never reaches your resolver, and anyone who can submit a pod spec can do it. What does not dodge is the resource itself - sustained GPU utilisation with no job behind it, or instance counts growing without a workload to explain them. The billing anomaly and the capacity anomaly are the same event seen from two directions.
- **Assume the miner will hide in the noise.** The obvious counter to utilisation alerting is to throttle, sitting at a level that looks like ordinary load, and an attacker paying nothing has no reason to be greedy. That is an argument for baselining what each workload normally draws rather than alerting on a single global threshold.
- **Ask what a compromised credential is worth in cash.** Most credential risk gets modelled as data loss, because that is what a breach usually costs. A credential in front of a GPU fleet has a second price: what an attacker can bill you for by simply using it. GPUs made that second price obvious earlier than the rest of the estate did, because the hardware was scarce and the resale value of its output was quoted publicly, by the hour.

## Closing

EIP-1559 is coming, and my impression at the time - an impression, per the caveat at the top, not a figure I ever wrote down - was that renting GPUs at market rates to mine ETH was not clearing its own cost. That kills this particular instance of the arithmetic, not the arithmetic. Some future coin, or some future thing GPUs are scarce for, will rebuild the same subtraction, and the same term will be the one worth removing.

The original version of this post signed off by joking that publishing it would cut the number of instances I could quietly rent. Two things about that line. The renting was on my own accounts, with my own card, the same way the hardware at home was mine - "quietly" meant without telling anyone, not without paying. And the joke does not survive the argument above it anyway: if renting is barely profitable, publishing costs me nothing, and the only version the disclosure could affect is the version I was arguing against. The interesting work was never the mining.
