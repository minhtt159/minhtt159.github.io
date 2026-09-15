---
title: "Merklision: 2020 collisions out of one Merkle tree"
date: 2020-10-24T17:00:00+07:00
draft: true
description: "A SVATTT 2020 qualifier challenge asked for 2020 distinct inputs that all hash to one Merkle root. Two implementation gaps, composed recursively, produce 2047 of them in eleven seconds."
summary: "The challenge wanted 2020 distinct strings colliding on one Merkle root. A second-preimage attack gives about ten. An odd-length duplication bug gives about ten more. Composing the two recursively gives 2047."
tags: ["ctf", "cryptography", "merkle-tree", "writeup"]
---

> *Archive note. Written in October 2020 for a blog I have since retired, and reproduced here as it stood. The challenge server is long gone and nothing here has been re-verified against a current implementation.*

**TL;DR.** A CTF challenge hands you a random string of `n` characters and asks for 2020 other distinct strings whose Merkle hash matches. A textbook second-preimage attack on a Merkle tree yields about `log2(n)` collisions, roughly ten. A second bug - odd-length input silently duplicating its last element - yields about ten more. Neither is enough. Composing the two rules recursively turns the additive count into a multiplicative one, and picking `n = 1025` so that every level of the tree is odd produces 2047 collisions in eleven seconds of brute force.

## The challenge

This was the crypto challenge in the SVATTT 2020 online qualifier, authored by `ndh`. I had played SVATTT from 2015 to 2017 on the crypto side and wrote an ECC chain for the 2018 edition, so the first thing I do with one of these is look at who set it. That tells you the ratio. Some authors write challenges that are 50 percent thinking and 50 percent code. `ndh` writes challenges that are 80 percent thinking and 20 percent code, and this one held to that.

The server does three things:

1. Reads an integer `n` from you, and generates a random string of `n` elements, each a letter from `a-zA-Z`.
2. Shows you that string and asks for 2020 further strings, pairwise distinct and distinct from the original.
3. Accepts if every one of them has the same Merkle hash as the generated string.

You choose `n`. That turns out to be the whole challenge.

## Merkle hashes, briefly

A Merkle tree hashes a block of data by hashing each leaf, then hashing each adjacent pair of hashes, and repeating until one root remains. It is the structure under Bitcoin and Ethereum block bodies, and its point is that you can verify one leaf against the root without holding the rest of the data.

The property that matters here is that the tree is built bottom-up from a list, and the intermediate rows are hashes of exactly the same shape as the leaf row. If the implementation does not record which row an element came from, the rows are interchangeable.

## Rule one: the classic second preimage

A second-preimage attack means: given one input, find a different input with the same hash. For an unvalidated Merkle construction this is not hard.

Take leaves `L1, L2, L3, L4`. The next row up is `H(L1,L2)` and `H(L3,L4)`, and the root is `H` of those two. Now hand the same implementation a two-element input consisting of `H(L1,L2), H(L3,L4)` directly. It hashes them as a pair, gets the same root, and has no way to tell the difference:

```
M(1, 2, 3, 4) = M(H(1,2), H(3,4))
```

Every level of the tree gives you one such rewrite, so a tree over `n` leaves gives `log2(n)` collisions. With `n` bounded below 2020 that is about ten. Two hundred times short.

## Rule two: the odd-length duplication

The implementation has a second gap. When a row has an odd number of elements, the last element is duplicated so the pairing works out. So padding an odd-length input by repeating its final element is invisible:

```
M(1, 2, 3, 4, 5) = M(1, 2, 3, 4, 5, 5)
```

That is worth one collision per odd row, and there are only about ten rows. Another ten. Still short, and adding the two rules gives twenty, not 2020.

<svg class="dg" viewBox="0 0 900 440" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="mk-t mk-d">
<title id="mk-t">Two Merkle collision rules and their composition</title>
<desc id="mk-d">Left: the generated input of four leaves hashes pairwise into two intermediate nodes and then one root. Right: handing the implementation those two intermediate nodes as a fresh two-element input reproduces the same root, because the construction never records which row an element came from. Below: the odd-length rule duplicates a trailing element without changing the hash, and alternating the two rules across levels multiplies the collision count instead of adding to it.</desc>
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
<rect class="box" x="102" y="168" width="86" height="30" rx="4"/>
<text class="m" x="145" y="188" text-anchor="middle">H(L1,L2)</text>
<rect class="box" x="272" y="168" width="86" height="30" rx="4"/>
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
<path class="ln" d="M145,168 L145,146 L230,146 L230,128" marker-end="url(#mk-ar)"/>
<path class="ln" d="M315,168 L315,146 L230,146" />
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
<text class="s" x="470" y="240">submitted as a two-element input</text>
<text class="s" x="470" y="262">same root, different string, counts as a collision</text>
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

One five-element input produced four collisions before the higher levels were even touched, and the last one is longer than anything the first rule alone reaches. The ceiling is the first power of two above `n`: for `n = 5`, `max_length` is 8.

So the choice of `n` is the actual exploit. You want every level of the tree to land on an odd count, so that the padding rule fires at each one. Working downward, `n = (((2*2-1)*2-1)*2-1)...` and staying under 2020 gives `n = 1025`, with `max_length = 2048`.

## Solving it

There is a clever way to enumerate the collisions and a stupid way. I took the stupid way, because the challenge scores a flag and not an algorithm:

1. The first level gets padded regardless, so loop, repeating the last element until the string reaches length 2048.
2. For each of those, compute the Merkle hash and record the collisions found across its levels, applying rule one.
3. Push every candidate into a set, the same structure the challenge uses, which drops the duplicates. There are a lot of duplicates.
4. Send the set back to the server.

```console
$ python ./merklision.py
Maximum length can collide is 2048
There are 2047 collisions
Takes 11 sec to produce
```

2047 against a requirement of 2020, in eleven seconds. Source is in [the merklision solver gist](https://gist.github.com/minhtt159/af8e19e2ac7088be48889ccd5c6e0e0b).

## What it was worth

Five teams solved it inside the four-hour window. For an online CTF with four-person teams and four challenges, that means one person had to land this cleanly while the other three landed theirs, which is a reasonable definition of a hard qualifier problem.

The lesson generalises past the challenge. Neither bug is exotic: one is the documented second-preimage weakness of unvalidated Merkle constructions, the other is a one-line padding convenience. Individually each is worth ten collisions and would probably survive a review as a curiosity. What made them worth 2047 is that one rule's output is the other rule's input, and nothing in the implementation stops you from alternating between them all the way up the tree.
