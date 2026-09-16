---
title: "Merklision: composing two Merkle bugs into 2047 collisions"
date: 2020-10-24T17:00:00+07:00
draft: true
description: "A SVATTT 2020 qualifier challenge asked for 2020 distinct inputs that all hash to one Merkle root. Neither of the two implementation gaps gets close on its own. Alternating between them does, because each one's output is the other one's input."
summary: "The challenge wanted 2020 distinct strings colliding on one Merkle root. A second-preimage attack gives about ten. An odd-length duplication bug gives about ten more. Alternating the two gives 2047, which I measured rather than derived."
tags: ["ctf", "cryptography", "merkle-tree", "writeup"]
---

> *Archive note. Written in October 2020 for a blog I have since retired, and reproduced here as it stood. The challenge server is long gone and nothing here has been re-verified against a current implementation.*

**TL;DR.** A CTF challenge hands you a random string of `n` characters and asks for 2020 other distinct strings whose Merkle hash matches. A textbook second-preimage attack on a Merkle tree yields about `log2(n)` collisions, roughly ten. A second bug - odd-length input silently duplicating its last element - yields about ten more. Neither is enough, and twenty is not 2020. What closes the gap is that the two rules compose: each one's output is a valid input to the other, so you alternate them up the tree instead of picking one. Choosing `n = 1025`, which makes every level odd, produced 2047 collisions in eleven seconds. I measured that number rather than deriving it, and the distinction matters - see below.

## The challenge

This was the crypto challenge in the SVATTT 2020 online qualifier - Sinh vien voi An toan thong tin, the Vietnamese inter-university student security competition - authored by `ndh`. I had played SVATTT from 2015 to 2017 on the crypto side and wrote a chain of elliptic-curve challenges for the 2018 edition, so the first thing I do with one of these is look at who set it. That tells you the ratio. Some authors write challenges that are 50 percent thinking and 50 percent code. `ndh` writes challenges that are 80 percent thinking and 20 percent code, and this one held to that.

The server does three things:

1. Reads an integer `n` from you, and generates a random string of `n` elements, each a letter from `a-zA-Z`.
2. Shows you that string and asks for 2020 further strings, pairwise distinct and distinct from the original. It collects them in a set, so exact repeats are silently discarded rather than rejected.
3. Accepts if every one of them has the same Merkle hash as the generated string.

`n` has to stay below 2020. That cap is what makes this a challenge rather than a formality - without it you would pick an enormous `n` and let a single rule run away. You choose `n` within the cap, and that choice turns out to be the whole exploit.

## Merkle hashes, briefly

A Merkle tree hashes a block of data by hashing each leaf, then hashing each adjacent pair of hashes, and repeating until one root remains. It is the structure under Bitcoin and Ethereum block bodies, and its point is that you can verify one leaf against the root without holding the rest of the data.

Two pieces of notation for the rest of this post. `H(a,b)` is the hash of one adjacent pair, one internal node of the tree. `M(...)` is the Merkle hash of a whole input list, the root you get after collapsing every row. The challenge compares `M` values.

The property that matters here is that the tree is built bottom-up from a list, and the intermediate rows are hashes of exactly the same shape as the leaf row. The construction hashes pairs without recording which row they came from, so as far as `M` is concerned, the rows are interchangeable. That is the root cause of everything below.

## Rule one: the classic second preimage

A second-preimage attack means: given one input, find a different input with the same hash. For an unvalidated Merkle construction this is not hard.

Take leaves `L1, L2, L3, L4`. The next row up is `H(L1,L2)` and `H(L3,L4)`, and the root is `H` of those two. Now hand the same implementation a two-element input consisting of `H(L1,L2), H(L3,L4)` directly. It hashes them as a pair, gets the same root, and has no way to tell the difference. Writing the leaves as `1, 2, 3, 4` for brevity:

```
M(1, 2, 3, 4) = M(H(1,2), H(3,4))
```

Every level of the tree gives you one such rewrite, so a tree over `n` leaves gives about `log2(n)` of them. With `n` capped below 2020 that is about ten. Two hundred times short.

## Rule two: the odd-length duplication

The implementation has a second gap. When a row has an odd number of elements, the last element is duplicated so the pairing works out. So padding an odd-length input by repeating its final element is invisible:

```
M(1, 2, 3, 4, 5) = M(1, 2, 3, 4, 5, 5)
```

That is worth one collision per odd row, and there are only about ten rows. Another ten. Still short, and adding the two rules gives twenty, not 2020.

<svg class="dg" viewBox="0 0 900 440" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="mk-t mk-d">
<title id="mk-t">Two Merkle collision rules and their composition</title>
<desc id="mk-d">Left: the generated input of four leaves hashes pairwise into two intermediate nodes and then one root. Right: the same two intermediate node values, byte for byte identical to the ones on the left, submitted as a fresh two-element input. They reproduce the same root, because the construction never records which row a value came from. Colour marks role, not value: blue is what the server generated, red is what the attacker submits. Below: the odd-length rule duplicates a trailing element without changing the hash, and the chain of four steps shows the two rules being alternated, which is what the prose derives the collision count from.</desc>
<defs>
  <marker id="mk-ar" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar" d="M0,0 L7,3 L0,6 Z"/></marker>
  <marker id="mk-ar-a" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar-a" d="M0,0 L7,3 L0,6 Z"/></marker>
  <marker id="mk-ar-c" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar-c" d="M0,0 L7,3 L0,6 Z"/></marker>
</defs>
<rect class="zone" x="20" y="30" width="400" height="290" rx="6"/>
<text class="t" x="40" y="54">What the server generated</text>
<text class="s" x="40" y="72">four leaves, hashed pairwise upward</text>
<rect class="box-b" x="192" y="92" width="76" height="30" rx="4"/>
<text class="m" x="230" y="112" text-anchor="middle">root</text>
<rect class="box-a" x="102" y="168" width="86" height="30" rx="4"/>
<text class="m" x="145" y="188" text-anchor="middle">H(L1,L2)</text>
<rect class="box-a" x="272" y="168" width="86" height="30" rx="4"/>
<text class="m" x="315" y="188" text-anchor="middle">H(L3,L4)</text>
<rect class="box-a" x="52" y="248" width="56" height="30" rx="4"/>
<text class="m" x="80" y="268" text-anchor="middle">L1</text>
<rect class="box-a" x="132" y="248" width="56" height="30" rx="4"/>
<text class="m" x="160" y="268" text-anchor="middle">L2</text>
<rect class="box-a" x="222" y="248" width="56" height="30" rx="4"/>
<text class="m" x="250" y="268" text-anchor="middle">L3</text>
<rect class="box-a" x="302" y="248" width="56" height="30" rx="4"/>
<text class="m" x="330" y="268" text-anchor="middle">L4</text>
<path class="ln-a" d="M80,248 L80,232 L145,232 L145,204" marker-end="url(#mk-ar-a)"/>
<path class="ln-a" d="M160,248 L160,232 L145,232" />
<path class="ln-a" d="M250,248 L250,232 L315,232 L315,204" marker-end="url(#mk-ar-a)"/>
<path class="ln-a" d="M330,248 L330,232 L315,232" />
<path class="ln-a" d="M145,168 L145,146 L230,146 L230,128" marker-end="url(#mk-ar-a)"/>
<path class="ln-a" d="M315,168 L315,146 L230,146" />
<text class="s" x="40" y="304">input length 4</text>
<rect class="zone" x="450" y="30" width="430" height="290" rx="6"/>
<text class="t" x="470" y="54">Rule one: feed the middle row back in</text>
<text class="s" x="470" y="72">the tree never records which row an element came from</text>
<rect class="box-b" x="627" y="92" width="76" height="30" rx="4"/>
<text class="m" x="665" y="112" text-anchor="middle">root</text>
<rect class="box-c" x="522" y="168" width="86" height="30" rx="4"/>
<text class="m" x="565" y="188" text-anchor="middle">H(L1,L2)</text>
<rect class="box-c" x="722" y="168" width="86" height="30" rx="4"/>
<text class="m" x="765" y="188" text-anchor="middle">H(L3,L4)</text>
<path class="ln-c" d="M565,168 L565,146 L665,146 L665,128" marker-end="url(#mk-ar-c)"/>
<path class="ln-c" d="M765,168 L765,146 L665,146" />
<text class="s" x="470" y="240">same bytes as the blue row on the left, now the whole input</text>
<text class="s" x="470" y="262">same root, different string, counts as a collision</text>
<text class="s" x="470" y="284">red marks what the attacker submits, not a different value</text>
<text class="s" x="470" y="304">input length 2</text>
<rect class="zone" x="20" y="340" width="860" height="80" rx="6"/>
<text class="t" x="40" y="366">Rule two, and why composing them multiplies</text>
<text class="m" x="40" y="390">M(1,2,3,4,5) = M(1,2,3,4,5,5)    odd row duplicates its last element</text>
<text class="m" x="40" y="410">pad, collapse a level, pad again, expand back: 1,2,3,4,5 -&gt; 1,2,3,4,5,5,5,5</text>
</svg>

## Composing the rules

Treating the two rules as separate buckets gives twenty collisions. They are not separate. Each one changes the shape of the input, and the changed shape is eligible for the other rule.

Start from `1, 2, 3, 4, 5`:

1. Odd length, so pad: collides with `1, 2, 3, 4, 5, 5`.
2. That is six elements, so collapse a level: collides with `H(1,2), H(3,4), H(5,5)`.
3. That row is odd, so pad again: collides with `H(1,2), H(3,4), H(5,5), H(5,5)`.
4. Expand that back down a level and you get `1, 2, 3, 4, 5, 5, 5, 5`, which is a plain string of the original alphabet.

One five-element input produced four collisions from the bottom two levels alone, without touching the levels above, and the last of them is longer than anything the first rule reaches on its own. There is a ceiling: padding cannot grow an input past the first power of two above `n`, which for `n = 5` is 8.

Note what step 4 bought. Steps 2 and 3 pass hash values around as list elements, which works because `M` cannot tell a hash from a leaf. Step 4 expands back down and lands on `1, 2, 3, 4, 5, 5, 5, 5` - a string drawn entirely from the original alphabet. That matters if the server is fussy about what a submission may contain, and it is the step that makes the technique robust rather than clever.

So the choice of `n` is the actual exploit. You want every level of the tree to land on an odd count, so that the padding rule fires at each one on the way up. Working the recurrence downward, `n = (((2*2-1)*2-1)*2-1)...` while staying under the cap gives `n = 1025`, whose tree is 11 levels deep and whose padding ceiling is 2048.

How many collisions does that yield? At the time I wrote this, I did not know. I still do not have a closed form for it. What follows is a count I measured, not one I derived, and the honest version of this writeup says so rather than dressing up the output of a brute-force loop as a result.

## Solving it

There is a clever way to enumerate the collisions and a stupid way. I took the stupid way, because the challenge scores a flag and not an algorithm:

1. The first level gets padded regardless, so loop, repeating the last element until the string reaches length 2048.
2. For each candidate, compute its Merkle hash and keep it only if it matches the target, recording the collisions found across its levels by applying rule one.
3. Push every survivor into a set, the same structure the challenge uses, which drops the duplicates. There are a lot of duplicates.
4. Send the set back to the server.

Step 2 is doing the filtering. Padding to an arbitrary length does not collide in general - a 1027-element input pads to a different node count than the original and produces a different root - so the loop generates candidates freely and the hash comparison throws most of them away. That is what makes this the stupid way: the search is not constructed to only emit valid answers, it is constructed to emit many answers cheaply and check them all.

```console
$ python ./merklision.py
Maximum length can collide is 2048
There are 2047 collisions
Takes 11 sec to produce
```

2047 against a requirement of 2020, in eleven seconds. Source is in [the merklision solver gist](https://gist.github.com/minhtt159/af8e19e2ac7088be48889ccd5c6e0e0b).

2047 is `2^11 - 1`, and 1025 leaves give an 11-level tree. That is a suggestive coincidence and I want to be careful with it: it looks like one binary choice per level with the do-nothing case removed, but I never proved that, and the small case does not obviously match - `n = 5` gives a 3-level tree and I counted 4 collisions from its lower levels, not 7. Treat the formula as a guess and the 2047 as an observation.

## What it was worth

Five teams solved it inside the four-hour window. The original version of this writeup drew a conclusion from that about four-person teams each landing one challenge perfectly; I have left the observation in and the inference out, because the solve count alone does not support it and I never knew how many teams competed.

The lesson generalises past the challenge. Neither bug is exotic: one is the documented second-preimage weakness of unvalidated Merkle constructions, the other is a one-line padding convenience. Individually each is worth ten collisions and would probably survive a review as a curiosity. What made them worth 2047 is that one rule's output is the other rule's input, and nothing in the implementation stops you from alternating between them all the way up the tree.
