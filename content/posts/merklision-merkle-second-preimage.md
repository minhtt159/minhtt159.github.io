---
title: "Merklision: composing two Merkle bugs into 2047 collisions"
date: 2020-10-24T17:00:00+07:00
draft: false
description: "A SVATTT 2020 qualifier challenge asked for 2020 distinct inputs that all hash to one Merkle root. Neither of the two implementation gaps gets close on its own. Alternating between them does, because each one's output is the other one's input."
summary: "The challenge wanted 2020 distinct strings colliding on one Merkle root. A second-preimage attack gives about ten. An odd-length duplication bug gives about ten more. Alternating the two gives 2047, which I measured rather than derived."
tags: ["ctf", "cryptography", "merkle-tree", "writeup"]
---

> *Archive note. Written in October 2020 for a blog I have since retired, and reproduced here as it stood. The challenge server is long gone and nothing here has been re-verified against a current implementation.*

**TL;DR.** A CTF challenge hands you a random string of `n` characters and asks for 2020 other distinct strings whose Merkle hash matches. A textbook second-preimage attack on a Merkle tree yields about `log2(n)` collisions, roughly ten. A second bug - odd-length input silently duplicating its last element - yields about ten more. Neither is enough, and twenty is not 2020. What closes the gap is that the two rules compose: each one's output is a valid input to the other, so you alternate them up the tree instead of picking one. Choosing `n = 1025`, which makes every level odd, produced 2047 collisions in eleven seconds. I measured that number rather than deriving it, and the distinction matters - see below.

## The challenge

This was the crypto challenge in the SVATTT 2020 online qualifier - Sinh vien voi An Toan Thong Tin, the ASEAN Student Contest in Information Security - authored by `ndh`. I had played SVATTT from 2015 to 2017 on the crypto side and wrote a chain of elliptic-curve challenges for the 2018 edition, so the first thing I do with one of these is look at who set it. That tells you the ratio. Some authors write challenges that are 50% thinking and 50% code. `ndh` writes challenges that are 80% thinking and 20% code, and this one held to that.

The server does three things:

1. Reads an integer `n` from you and asserts `0 <= n <= 2020`, then builds a tuple of `n` single-letter byte strings drawn from `a-zA-Z`.
2. Shows you that tuple and reads 2020 more. Each one must be pairwise distinct from the others - a repeat trips an assertion and ends the session, it is not silently dropped - and each must have the same Merkle hash.
3. Prints the flag.

Two details from the source matter more than they look.

The cap `n <= 2020` is what makes this a challenge rather than a formality: without it you would name an enormous `n` and let a single rule run away. You pick `n` within the cap, and that choice is the whole exploit.

The submission format is a comma-separated list of base64 blobs, decoded straight back into a tuple of byte strings. Elements are arbitrary bytes. There is no alphabet restriction on what you send back, which turns out to matter a great deal. Nor is there any check that a submission differs from the original tuple - only that the 2020 are pairwise distinct - so the original is itself a legal answer.

## Merkle hashes, briefly

A Merkle tree hashes a block of data by hashing each leaf, then hashing each adjacent pair of hashes, and repeating until one root remains. Here the hash is SHA-256 truncated to its first 6 bytes, a 48-bit digest, which the challenge does to keep the base64 traffic small. It is the structure under Bitcoin and Ethereum block bodies, and its point is that you can verify one leaf against the root without holding the rest of the data.

Three pieces of notation. `h(x)` is the 6-byte digest of one element. `||` is byte concatenation. `M(...)` is the Merkle root of a whole input list, which is what the challenge compares.

The build, stated exactly, because the detail matters below: hash every element to get a row of digests, then repeatedly replace each adjacent pair `a, b` with `h(a || b)` until one value is left.

The property that breaks it is that every row is just a list of byte strings, and the function that consumes a row does not record which row it was. As far as `M` is concerned, rows are interchangeable. That is the root cause of everything below.

## Rule one: the classic second preimage

A second-preimage attack means: given one input, find a different input with the same hash. For an unvalidated Merkle construction this is not hard.

Take leaves `L1, L2, L3, L4`. The first pass produces digests `h(L1) h(L2) h(L3) h(L4)`. The row above is `h(h(L1)||h(L2))` and `h(h(L3)||h(L4))`, and the root is the hash of those two concatenated.

So what two-element input reproduces that root? Not the internal nodes themselves. The implementation hashes every element before pairing anything, so you have to submit the values that *hash to* the internal nodes. The node is `h(h(L1)||h(L2))`, so the element you send is `h(L1)||h(L2)`: the two child digests, concatenated, twelve bytes.

```
M(L1, L2, L3, L4) == M(h(L1)||h(L2), h(L3)||h(L4))
```

The original writeup glossed this, and the source settles it. The solver builds its rewrite rows as `[hashes[i] + hashes[i+1] for i in range(0, len(hashes), 2)]`, and `+` on `bytes` in Python is concatenation - not addition, and not a further hash. Submitting a twelve-byte element is legal because the wire format base64-decodes to arbitrary bytes and the challenge never inspects them.

Every level gives you one such rewrite, so a tree over `n` leaves gives about `log2(n)` of them. With `n` capped at 2020 that is about ten. Two hundred times short.

## Rule two: the odd-length duplication

The implementation has a second gap. When a row has an odd number of elements, the last element is duplicated so the pairing works out. So padding an odd-length input by repeating its final element is invisible:

```
M(1, 2, 3, 4, 5) = M(1, 2, 3, 4, 5, 5)
```

That is worth one collision per odd row, and there are only about ten rows. Another ten. Still short, and adding the two rules gives twenty, not 2020.

<svg class="dg" viewBox="0 0 900 440" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="mk-t mk-d">
<title id="mk-t">Two Merkle collision rules and their composition</title>
<desc id="mk-d">Left: the four generated leaves are hashed to digests, each adjacent pair of digests is concatenated and hashed into an internal node, and the two nodes combine into the root. Right: a two-element submission where each element is the concatenation of two child digests. Because the implementation hashes every element before pairing, hashing those concatenations reproduces the internal nodes exactly, and therefore the same root. Note the elements submitted are the concatenated digests, twelve bytes each, not the six-byte internal nodes. Colour marks role: blue is what the server generated, red is what the attacker submits. Below: the odd-length rule repeats a trailing element without changing the root, and alternating the two rules is what produces the collision count the post derives.</desc>
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
<rect class="box-a" x="92" y="168" width="106" height="30" rx="4"/>
<text class="s" x="145" y="187" text-anchor="middle">h(h(L1)||h(L2))</text>
<rect class="box-a" x="262" y="168" width="106" height="30" rx="4"/>
<text class="s" x="315" y="187" text-anchor="middle">h(h(L3)||h(L4))</text>
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
<text class="s" x="470" y="72">submit what hashes to the row above, not the row itself</text>
<rect class="box-b" x="627" y="92" width="76" height="30" rx="4"/>
<text class="m" x="665" y="112" text-anchor="middle">root</text>
<rect class="box-c" x="512" y="168" width="106" height="30" rx="4"/>
<text class="s" x="565" y="187" text-anchor="middle">h(L1)||h(L2)</text>
<rect class="box-c" x="702" y="168" width="106" height="30" rx="4"/>
<text class="s" x="765" y="187" text-anchor="middle">h(L3)||h(L4)</text>
<path class="ln-c" d="M565,168 L565,146 L665,146 L665,128" marker-end="url(#mk-ar-c)"/>
<path class="ln-c" d="M765,168 L765,146 L665,146" />
<text class="s" x="470" y="240">12 bytes each: the child digests, concatenated</text>
<text class="s" x="470" y="262">hashing them reproduces the blue row, so the root matches</text>
<text class="s" x="470" y="284">not the 6-byte nodes themselves - elements get hashed first</text>
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
2. That is six elements, so collapse a level: collides with the three concatenated digest pairs of that row.
3. That row is three long, so pad it: repeat its last element to get four.
4. Expand back down a level and you are holding `1, 2, 3, 4, 5, 5, 5, 5`, a plain list of the original single-letter elements again.

One five-element input produced four collisions from the bottom two levels alone, and the last of them is longer than anything the first rule reaches by itself. There is a ceiling: padding cannot grow an input past the first power of two at or above `n`, which for `n = 5` is 8.

An earlier draft of this post claimed step 4 mattered because it lands back in the original alphabet, and that the server might be fussy about what a submission contains. Reading the source, it is not: elements arrive base64-decoded as arbitrary bytes and nothing inspects them. Rows of concatenated digests are just as acceptable as rows of letters. Step 4 is a consequence of the composition, not a precaution.

So the choice of `n` is the actual exploit. You want every row on the way up to have an odd count, so the padding rule fires at each one. Working the recurrence downward, `n = (((2*2-1)*2-1)*2-1)...` while staying under the cap gives `n = 1025`, whose padding ceiling is 2048.

## Where 2047 comes from

The original writeup asked how many collisions that yields and answered, in as many words, that I did not know. Having the source back, the answer is exact and a little deflating.

Every answer the search produces is a list, and the lists come out at one distinct length each, from 2 elements up to 2048. That is it. One answer per length:

```
2048 - 2 + 1 = 2047
```

The ceiling of 2048 is the padding limit, the floor of 2 is the shortest row a Merkle tree can have above the root, and every length in between is reachable. The requirement was 2020, so the margin was 27 lengths - closer than it felt at the time.

One wrinkle worth stating because it affects the number you can actually submit: the solver seeds its result set with the original tuple before searching, so 2047 counts the input itself. There are 2046 genuine collisions. The challenge never checks a submission against the original, only that the 2020 are pairwise distinct, so submitting it is legal and the point is moot - but 2047 and 2046 both being defensible answers is exactly the sort of thing a writeup should not leave implicit.

I re-ran the solver against the original challenge source while preparing this post. It reproduces 2047, every length from 2 to 2048 present exactly once, in about five seconds on current hardware.

<svg class="dg" viewBox="0 0 900 330" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="lad-t lad-d">
<title id="lad-t">Why the answer count is 2047</title>
<desc id="lad-d">The search emits one valid answer at each possible list length. The shortest is a two-element list, the longest is a 2048-element list, which is the padding ceiling for a 1025-element input, and every length in between is reachable exactly once. The count is therefore 2048 minus 2 plus 1, which is 2047, against a requirement of 2020 - a margin of 27. The original input, 1025 elements long, sits inside that range and is counted by the solver, so 2046 of the 2047 are collisions distinct from it.</desc>
<defs>
  <marker id="lad-ar-a" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar-a" d="M0,0 L7,3 L0,6 Z"/></marker>
</defs>
<text class="t" x="40" y="42">One answer per list length</text>
<text class="s" x="40" y="62">every length from 2 to 2048 is reachable, and each is reached exactly once</text>
<rect class="box-b" x="40" y="86" width="120" height="44" rx="4"/>
<text class="m" x="100" y="106" text-anchor="middle">length 2</text>
<text class="s" x="100" y="123" text-anchor="middle">shortest row</text>
<rect class="box" x="188" y="86" width="104" height="44" rx="4"/>
<text class="m" x="240" y="106" text-anchor="middle">length 3</text>
<rect class="box" x="320" y="86" width="104" height="44" rx="4"/>
<text class="m" x="372" y="106" text-anchor="middle">length 4</text>
<rect class="zone" x="452" y="86" width="104" height="44" rx="6"/>
<text class="m" x="504" y="112" text-anchor="middle">...</text>
<rect class="box-a" x="584" y="86" width="132" height="44" rx="4"/>
<text class="m" x="650" y="106" text-anchor="middle">length 1025</text>
<text class="s" x="650" y="123" text-anchor="middle">the original</text>
<rect class="box-b" x="744" y="86" width="132" height="44" rx="4"/>
<text class="m" x="810" y="106" text-anchor="middle">length 2048</text>
<text class="s" x="810" y="123" text-anchor="middle">padding ceiling</text>
<path class="ln-a" d="M160,108 L182,108" marker-end="url(#lad-ar-a)"/>
<path class="ln-a" d="M292,108 L314,108" marker-end="url(#lad-ar-a)"/>
<path class="ln-a" d="M424,108 L446,108" marker-end="url(#lad-ar-a)"/>
<path class="ln-a" d="M556,108 L578,108" marker-end="url(#lad-ar-a)"/>
<path class="ln-a" d="M716,108 L738,108" marker-end="url(#lad-ar-a)"/>
<rect class="zone" x="40" y="164" width="400" height="120" rx="6"/>
<text class="t" x="60" y="190">The count</text>
<text class="m" x="60" y="218">2048 - 2 + 1 = 2047</text>
<text class="s" x="60" y="242">required: 2020</text>
<text class="s" x="60" y="264">margin: 27 lengths</text>
<rect class="zone" x="470" y="164" width="406" height="120" rx="6"/>
<text class="t" x="490" y="190">2047 or 2046?</text>
<text class="s" x="490" y="214">2047 counts the original input, which the solver</text>
<text class="s" x="490" y="234">seeds into its own result set before searching.</text>
<text class="s" x="490" y="258">2046 are collisions distinct from it. Either clears</text>
<text class="s" x="490" y="278">2020, and the challenge accepts the original too.</text>
</svg>

## Solving it

There is a clever way to enumerate the collisions and a stupid way. I took the stupid way, because the challenge scores a flag and not an algorithm:

1. Take the original tuple and append copies of its last element, one more each time, walking every length from 1025 up to 2048.
2. Keep a padded candidate only if its Merkle hash still equals the target. Most lengths do not survive this.
3. For each survivor, walk the tree and collect the rewrite rows that rule one licenses - the concatenated digest pairs at each row where padding occurred. Add the survivor itself alongside them.
4. Check every one of those against the target again, drop anything already seen, and keep the rest.

Step 2 is the filter, and it is why this is the stupid way. Padding to an arbitrary length does not collide in general; the loop emits candidates cheaply and lets the hash comparison discard most of them, rather than being constructed to only produce valid ones. The recorded rewrites in step 3 come from the rows the implementation had to pad, which is exactly where the two rules meet.

```console
$ python ./merklision.py
Maximum length can collide is 2048
There are 2047 collisions
Takes 11 sec to produce
```

2047 against a requirement of 2020, in eleven seconds on 2020 hardware. Both the challenge and the solver are in [the merklision gist](https://gist.github.com/minhtt159/af8e19e2ac7088be48889ccd5c6e0e0b); every claim in this post about how the challenge behaves was checked against that source rather than against my memory of it, and several of them needed correcting.

It is tempting to read 2047 as `2^11 - 1`, one binary choice per level of a 1025-element tree with the empty case removed. That reading gets the right number for the wrong reason, which is worse than getting it wrong.

The count is the size of a range. The answers fill every length from 2 up to the ceiling, so there are `ceiling - 1` of them, and the ceiling is by construction the first power of two at or above `n`. A count of `2^k - 1` therefore falls out every time, without any per-level choice being involved. The power of two is the ceiling, not a tally of decisions.

Running the same search at smaller sizes shows the shape holding: `n = 5` gives 7 answers against a ceiling of 8, `n = 9` gives 15 against 16, `n = 17` gives 31 against 32. Each one covers every length from 2 to its own ceiling exactly once.

## What it was worth

Five teams solved it inside the four-hour window. The original version of this writeup drew a conclusion from that about four-person teams each landing one challenge perfectly; I have left the observation in and the inference out, because the solve count alone does not support it and I never knew how many teams competed.

The lesson generalises past the challenge. Neither bug is exotic: one is the documented second-preimage weakness of unvalidated Merkle constructions, the other is a one-line padding convenience. Individually each is worth ten collisions and would probably survive a review as a curiosity. What made them worth 2047 is that one rule's output is the other rule's input, and nothing in the implementation stops you from alternating between them all the way up the tree.
