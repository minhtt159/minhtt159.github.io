---
title: "Merklision: composing two Merkle bugs into 2047 collisions"
date: 2020-10-24T17:00:00+07:00
draft: false
description: "A SVATTT 2020 qualifier challenge asked for 2020 distinct inputs that all hash to one Merkle root. Neither of the two implementation gaps gets close on its own. Alternating between them does, because each one's output is the other one's input."
summary: "The challenge wanted 2020 distinct strings colliding on one Merkle root. A second-preimage attack gives about ten. An odd-length duplication bug gives about ten more. Alternating the two gives 2047, which I measured rather than derived."
tags: ["ctf", "cryptography", "merkle-tree", "writeup"]
---

> *Archive note. Written in October 2020 for a blog I have since retired, and reproduced here as it stood. The challenge server is long gone and nothing here has been re-verified against a current implementation.*

**TL;DR.** A CTF challenge hands you a random string of `n` characters and asks for 2020 other distinct strings whose Merkle hash matches. A textbook second-preimage attack on a Merkle tree yields about `log2(n)` collisions, roughly ten. A second bug - odd-length input silently duplicating its last element - yields about ten more. Neither is enough, and twenty is not 2020. What closes the gap is that the two rules compose: each one's output is a valid input to the other, so you alternate them up the tree instead of picking one. Choosing `n = 1025`, which makes every row odd down to the final pair, produced 2047 collisions in eleven seconds. I measured that number rather than deriving it, and the distinction matters - see below.

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

Three pieces of notation. `h(x)` is the 6-byte digest of any byte string - a leaf, or a pair of digests glued together. `||` is byte concatenation. `M(...)` is the Merkle root of a whole input list, which is what the challenge compares.

The build, stated exactly, because the detail matters below: hash every element to get a row of digests, then repeatedly replace each adjacent pair `a, b` with `h(a || b)` until one value is left.

The property that breaks it is that every row is just a list of byte strings, and the function that consumes a row does not record which row it was. As far as `M` is concerned, rows are interchangeable. That is the root cause of everything below.

## Rule one: the classic second preimage

A second-preimage attack means: given one input, find a different input with the same hash. For an unvalidated Merkle construction this is not hard.

Take leaves `L1, L2, L3, L4`. The first pass produces digests `h(L1) h(L2) h(L3) h(L4)`. The row above is `h(h(L1)||h(L2))` and `h(h(L3)||h(L4))`, and the root is the hash of those two concatenated.

So what two-element input reproduces that root? Not the internal nodes themselves. The implementation hashes every element before pairing anything, so you have to submit the values that *hash to* the internal nodes. The node is `h(h(L1)||h(L2))`, so the element you send is `h(L1)||h(L2)`: the two child digests, concatenated, twelve bytes.

```
M(L1, L2, L3, L4) = M(h(L1)||h(L2), h(L3)||h(L4))
```

The original writeup glossed this, and the source settles it. The solver builds its rewrite rows as `[hashes[i] + hashes[i+1] for i in range(0, len(hashes), 2)]`, run after the odd-row duplicate has already been appended so the pairing is even, and `+` on `bytes` in Python is concatenation - not addition, and not a further hash. Submitting a twelve-byte element is legal because the wire format base64-decodes to arbitrary bytes and the challenge never inspects them.

Every row gives you one such rewrite, so a tree over `n` leaves gives about `log2(n)` of them. With `n` capped at 2020 that is about ten. Two hundred times short.

## Rule two: the odd-length duplication

The implementation has a second gap. When a row has an odd number of elements, the last element is duplicated so the pairing works out. So padding an odd-length input by repeating its final element is invisible:

```
M(1, 2, 3, 4, 5) = M(1, 2, 3, 4, 5, 5)
```

The original writeup counted this as one collision per odd row, about ten in total, and I repeated that for years. It is wrong, and badly. Nothing stops you appending a second copy, and a third: the padding cascades, and every length from 1026 up to 2048 collides with the original. Rule two on its own is worth 1023 collisions, not ten.

So the framing the original built - two rules worth ten each, twenty against a requirement of 2020, hopeless without some trick - was never the real situation. Rule two alone gets you halfway. What the composition buys is the rest, and a much tidier account of where the total comes from.

<svg class="dg" viewBox="0 0 900 440" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="mk-t mk-d">
<title id="mk-t">Two Merkle collision rules and their composition</title>
<desc id="mk-d">Left: the four generated leaves are hashed to digests, each adjacent pair of digests is concatenated and hashed into an internal node, and the two nodes combine into the root. Right: a two-element submission where each element is the concatenation of two child digests. Because the implementation hashes every element before pairing, hashing those concatenations reproduces the internal nodes exactly, and therefore the same root. Note the elements submitted are the concatenated digests, twelve bytes each, not the six-byte internal nodes. Colour marks role: blue is what the server generated and hashes up from, red is what the attacker submits instead, and green is the root, which is the one value identical on both sides and the whole point of the attack. Below: the odd-length rule repeats a trailing element without changing the root - for instance the five-element list and the same list with its last element doubled - and alternating the two rules across rows is what the post derives the collision count from.</desc>
<defs>
  <marker id="mk-ar-a" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar-a" d="M0,0 L7,3 L0,6 Z"/></marker>
  <marker id="mk-ar-c" markerWidth="9" markerHeight="9" refX="7" refY="3" orient="auto"><path class="ar-c" d="M0,0 L7,3 L0,6 Z"/></marker>
</defs>
<rect class="zone" x="20" y="30" width="400" height="290" rx="6"/>
<text class="t" x="40" y="54">What the server generated</text>
<text class="s" x="40" y="72">blue is the server side, at both rows</text>
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
<text class="s" x="470" y="262">hashing them reproduces the blue digest row above</text>
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
2. That is six elements, so collapse one row: collides with the three concatenated digest pairs of that row.
3. That row is three long, so pad it: repeat its last element to get four.
4. Expand back down one row and you are holding `1, 2, 3, 4, 5, 5, 5, 5`, a plain list of the original single-letter elements again.

The original writeup stopped there and counted four. It missed one: `1, 2, 3, 4, 5, 5, 5` collides too, at length 7. The bottom two rows of a five-element input yield lengths 5 through 8 and 3 through 4, which is six answers, one of them the original - so five collisions, not four. There is a ceiling: padding cannot grow an input past the first power of two at or above `n`, which for `n = 5` is 8.

An earlier draft of this post claimed step 4 mattered because it lands back in the original alphabet, and that the server might be fussy about what a submission contains. Reading the source, it is not: elements arrive base64-decoded as arbitrary bytes and nothing inspects them. Rows of concatenated digests are just as acceptable as rows of letters. Step 4 is a consequence of the composition, not a precaution.

So the choice of `n` is the actual exploit. You want every row on the way up, down to the last pair, to have an odd count, so the padding rule fires at each one. Working the recurrence downward, `n = (((2*2-1)*2-1)*2-1)...` while staying under the cap gives `n = 1025`, whose padding ceiling is 2048.

## Where 2047 comes from

The original writeup asked how many collisions that yields and answered, in as many words, that I did not know. Having the source back, the answer is exact and a little deflating.

Take the rows of the tree for `n = 1025`: 1025, 513, 257, 129, 65, 33, 17, 9, 5, 3, 2.

Rule one lets you submit any one of those rows, expressed as concatenated digest pairs. Rule two then lets you pad whichever row you submitted, repeating its trailing element, up to that row's own ceiling. So each row contributes a contiguous band of lengths, and the bands turn out to tile the whole range without overlapping:

| row | lengths it yields | answers |
|---|---|---|
| the leaves, 1025 | 1025 to 2048 | 1024 |
| 513 | 513 to 1024 | 512 |
| 257 | 257 to 512 | 256 |
| 129 | 129 to 256 | 128 |
| 65 | 65 to 128 | 64 |
| 33 | 33 to 64 | 32 |
| 17 | 17 to 32 | 16 |
| 9 | 9 to 16 | 8 |
| 5 | 5 to 8 | 4 |
| 3 | 3 to 4 | 2 |
| 2 | 2 | 1 |

```
1024 + 512 + 256 + 128 + 64 + 32 + 16 + 8 + 4 + 2 + 1 = 2047
```

Eleven bands, each half the one above it, covering every length from 2 to 2048 exactly once. That is both why the answers come out one per length and where the total comes from. The requirement was 2020, so the margin was 27 lengths - closer than it felt at the time.

The floor is 2 rather than 1 for a reason worth stating plainly: a one-element submission `[x]` has `M([x]) = h(x)`, so producing one means finding a preimage of the 48-bit root. That is not structurally impossible, just a different and much less elegant attack.

One wrinkle worth stating because it affects the number you can actually submit: the solver seeds its result set with the original tuple before searching, so 2047 counts the input itself. There are 2046 genuine collisions. The challenge never checks a submission against the original, only that the 2020 are pairwise distinct, so submitting it is legal and the point is moot - but 2047 and 2046 both being defensible answers is exactly the sort of thing a writeup should not leave implicit.

I re-ran the solver against the original challenge source while preparing this post. It reproduces 2047, every length from 2 to 2048 present exactly once, in about five seconds on current hardware.

<svg class="dg" viewBox="0 0 900 400" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="lad-t lad-d">
<title id="lad-t">Eleven bands of lengths, tiling the range</title>
<desc id="lad-d">Each row of the tree can be submitted in place of the leaves, and then padded up to that row's own ceiling, so each row yields a contiguous band of list lengths. The leaf row of 1025 yields lengths 1025 to 2048, which is 1024 answers; the row of 513 yields 513 to 1024, which is 512; and so on halving down to the final pair, which yields the single length 2. The eleven bands abut without overlapping and cover every length from 2 to 2048 exactly once, so the total is 1024 plus 512 plus 256 and so on down to 1, which is 2047. That is a geometric sum with one term per row, which is why the answer is two to the eleventh minus one: the eleven is the row count, not a count of binary choices.</desc>
<rect class="zone" x="20" y="28" width="860" height="180" rx="6"/>
<text class="t" x="40" y="54">Each row yields a band of lengths, and the bands tile</text>
<text class="s" x="40" y="72">width is proportional to the count; every band is half the one above it</text>
<rect class="box-a" x="440" y="88" width="420" height="22" rx="3"/>
<text class="s" x="650" y="104" text-anchor="middle">leaf row 1025: lengths 1025-2048, 1024 answers</text>
<rect class="box-a" x="230" y="116" width="210" height="22" rx="3"/>
<text class="s" x="335" y="132" text-anchor="middle">row 513: 513-1024, 512</text>
<rect class="box-a" x="125" y="144" width="105" height="22" rx="3"/>
<text class="s" x="177" y="160" text-anchor="middle">257: 256</text>
<rect class="box-a" x="72" y="172" width="53" height="22" rx="3"/>
<text class="s" x="98" y="188" text-anchor="middle">129</text>
<rect class="box-b" x="40" y="172" width="30" height="22" rx="3"/>
<text class="s" x="55" y="188" text-anchor="middle">...</text>
<rect class="zone" x="20" y="226" width="420" height="150" rx="6"/>
<text class="t" x="40" y="252">The sum</text>
<text class="m" x="40" y="280">1024 + 512 + 256 + 128 + 64</text>
<text class="m" x="40" y="300">  + 32 + 16 + 8 + 4 + 2 + 1</text>
<text class="m" x="40" y="326">= 2047 = 2^11 - 1</text>
<text class="s" x="40" y="352">eleven terms, one per row of the tree</text>
<rect class="zone" x="460" y="226" width="420" height="150" rx="6"/>
<text class="t" x="480" y="252">Against the requirement</text>
<text class="s" x="480" y="278">required: 2020 pairwise-distinct submissions</text>
<text class="s" x="480" y="300">produced: 2047, one at every length from 2 to 2048</text>
<text class="s" x="480" y="322">margin: 27</text>
<text class="s" x="480" y="350">2047 counts the original tuple, so 2046 are collisions</text>
<text class="s" x="480" y="368">distinct from it. The challenge accepts the original too.</text>
</svg>

## Solving it

There is a clever way to enumerate the collisions and a stupid way. I took the stupid way, because the challenge scores a flag and not an algorithm:

1. Take the original tuple and append copies of its last element, one more each time, walking every length from 1025 up to 2048.
2. Keep a padded candidate only if its Merkle hash still equals the target.
3. For each survivor, walk the tree and collect the rewrite rows that rule one licenses - the concatenated digest pairs at each row where padding occurred. Add the survivor itself alongside them.
4. Check every one of those against the target again, drop anything already seen, and keep the rest.

Step 2 is a guard rather than a filter, and this is where I had the algorithm wrong for years. I assumed most padded lengths would fail the check and that the loop was cheap generation plus aggressive discarding. Running it says otherwise: all 1024 lengths from 1025 to 2048 pass. They have to, or the total could not reach 2047. The comparison never rejects anything at this step; it only proves the rule.

It is still the stupid way, just not for the reason I thought. The search re-derives and re-hashes candidates it could have enumerated directly, and leans on a set to absorb the duplicates - of which there are many, since the same answer is reachable by more than one route.

```console
$ python ./merklision.py
Maximum length can collide is 2048
There are 2047 collisions
Takes 11 sec to produce
```

2047 against a requirement of 2020, in eleven seconds on 2020 hardware. Both the challenge and the solver are in [the merklision gist](https://gist.github.com/minhtt159/af8e19e2ac7088be48889ccd5c6e0e0b); every claim in this post about how the challenge behaves was checked against that source rather than against my memory of it, and several of them needed correcting.

And 2047 is `2^11 - 1`, which is not a coincidence at all - it is a geometric sum with one term per row, each half the one before. The eleven is the row count of a 1025-element tree. My old intuition, that the power of two came from a binary choice at each level, had the right shape and the wrong mechanism: the doubling is in how many lengths a row can reach, not in a decision taken at it.

Running the same search at smaller sizes shows it holding: `n = 5` gives 7 answers against a ceiling of 8, `n = 9` gives 15 against 16, `n = 17` gives 31 against 32, each covering every length from 2 to its own ceiling exactly once.

This is also what makes `n = 1025` the right pick rather than a lucky one. Every row down to the last pair lands on an odd count, so the padding rule fires at each one and every band is available. Choose 1026 instead and the very first row is even, nothing pads, and the construction collapses at the bottom.

## What it was worth

Five teams solved it inside the four-hour window. The original version of this writeup drew a conclusion from that about four-person teams each landing one challenge perfectly; I have left the observation in and the inference out, because the solve count alone does not support it and I never knew how many teams competed.

The lesson generalises past the challenge. Neither bug is exotic: one is the documented second-preimage weakness of unvalidated Merkle constructions, the other is a one-line padding convenience. Individually each is worth ten collisions and would probably survive a review as a curiosity. What made them worth 2047 is that one rule's output is the other rule's input, and nothing in the implementation stops you from alternating between them all the way up the tree.
